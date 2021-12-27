#  Création et utilisation de spark dans un cluster AKS

## Création du Cluster AKS

```
RESOURCES_UID=$(uuidgen | cut -c1-8)
RESOURCES_UID=d24f7f56
AKS_NAME="aks-$RESOURCES_UID"
AKS_RG="rg-$AKS_NAME"
SUBSCRIPTION="Jef_Live_Abo"
REGION="francecentral"
```

### Création d'un groupe de resources
```
az group create --location $REGION --name $AKS_RG --subscription "$SUBSCRIPTION"
```

### Création d'un service principal

Le "service principal" est la création d'un utilisateur dédié à une application azure.

```
az ad sp create-for-rbac --name sp-$RESOURCES_UID --role contributor
```

On obtient des identifiants et un **mot de passe** 
> c'est la seule fois qu'on aura accès à ce mot de passe.

```
{
  "appId": "a210e9ee-cf16-48da-8cfa-251997f115e6",
  "displayName": "sp-d24f7f56",
  "name": "a210e9ee-cf16-48da-8cfa-251997f115e6",
  "password": "Tu8XjlC.Jrq7dQJyLYRsA9V6dN-bvkeOZG",
  "tenant": "d97f60c9-c736-4554-923c-63e3d2441ef0"
}
```

### Création du cluster AKS 

* Kubenet pas azureCLI
* utilise une clé ssh existante.
* Associe le service principal
> C'est indispensable d'utiliser le service principal pour pouvoir l'associer à d'autres services 
> comme le registre de container docker ou des espaces disques

```
az aks create \
	--location $REGION \
	--subscription "$SUBSCRIPTION" \
	--resource-group $AKS_RG \
	--name $AKS_NAME \
	--ssh-key-value $HOME/.ssh/id_rsa.pub \
 	--service-principal "a210e9ee-cf16-48da-8cfa-251997f115e6" \
  	--client-secret "Tu8XjlC.Jrq7dQJyLYRsA9V6dN-bvkeOZG" \
	--network-plugin kubenet \
	--load-balancer-sku basic \
	--outbound-type loadBalancer \
	--node-vm-size Standard_B4ms \
    --node-count 1
```

On récupère les crédentials pour pouvoir utiliser l'utilitaire Kubenet.

```
FILE="./$AKS_NAME.kubeconfig"
az aks get-credentials \
  -n $AKS_NAME \
  -g $AKS_RG \
  --subscription "$SUBSCRIPTION" \
  --admin \
  --file $FILE
export KUBECONFIG=$FILE
```
#### Vérification du cluster AKS

```
>kubectl get nodes
NAME                                STATUS   ROLES   AGE    VERSION
aks-nodepool1-93297334-vmss000000   Ready    agent   107m   v1.20.9
```

Installation de l'image docker de `busybox`. `busybox` est un exécutable qui embarque toute les commandes unix.
Très léger, prévu pour l'embarqué.

Pour cela, on crée un fichier `busybox.yaml` qui va crée un pod embarquant busybox.

```
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
```

On applique ensuite le fichier `yaml` et cela va déployer dans le cluster, notre pod `busybox`.

```
> kubectl apply -f busybox.yaml
pod/busybox created
```

On vérifie qu'il tourne bien
```
>kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          24s
```

On peut se logguer dessus.
```
> kubectl exec -it busybox -- sh
/ # uname -a
Linux busybox 5.4.0-1063-azure #66~18.04.1-Ubuntu SMP Thu Oct 21 09:59:28 UTC 2021 x86_64 GNU/Linux
/ # exit
```


## Génération du docker Spark

### Création de l'image Docker

On va créer des images docker permettant de créer des pods spark pour exécuter notre code dans un cluster spark.
Pour cela, on le fait sur une machine ou `docker` est installé.

On récupère le code de Spark
```
wget https://dlcdn.apache.org/spark/spark-3.2.0/spark-3.2.0-bin-hadoop3.2.tgz
tar zxvf spark-3.2.0-bin-hadoop3.2.tgz
cd spark-3.2.0-bin-hadoop3.2
bin/docker-image-tool.sh -r myrepo -t v3.2.0 -p kubernetes/dockerfiles/spark/bindings/python/Dockerfile build
```

On peut être amené à modifier le fichier `kubernetes/dockerfiles/spark/bindings/python/Dockerfile` qui sera
responsable de la création de l'image `docker` pour executer pyspark.
On doit maintenant avoir

```
> docker images | grep spark
myrepo/spark-py                   v3.2.0        51a36bc19623   54 minutes ago   958MB
myrepo/spark                      v3.2.0        f11fc5eb6d09   54 minutes ago   602MB
```

On verifie qu'on est capable d'exécuter un job spark dans le container `docker`

```
> docker run -it --rm myrepo/spark-py:v3.2.0 bash
185@c9ace8e1555f:/opt/spark/work-dir$ cd ..
185@c9ace8e1555f:/opt/spark/work-dir$ bin/spark-submit /opt/spark/examples/src/main/python/pi.py 
21/12/17 11:32:19 INFO SparkContext: Running Spark version 3.2.0
...
Pi is roughly 3.132000
...
185@c9ace8e1555f:/opt/spark/work-dir$ 
```

Le résultat de pi est dans l'ensemble des logs.

#### Création d'une image docker utilisant miniconda

Pour faire cela, on va éditer le fichier `kubernetes/dockerfiles/spark/bindings/python/Dockerfile`.
On remplace le `Dockerfile` par le fichier [Dockerfile](./Dockerfile) contenu dans le répertoire racine de ce
github.

Ce Dockerfile s'appuie sur un Dockerfile crée par [Continuum Analytics](https://hub.docker.com/r/conda/miniconda3/dockerfile)

> Notons au passage qu'on crée dans l'image une variable `PYTHONPATH` qui pointe sur une zone disque qui correspondra à notre 
espace `Azure file` où installer les modules partagés par tous les codes python depuis tous les pods.

### Création d'un repository docker azure.

[Doc Microsoft ACR](https://docs.microsoft.com/fr-fr/azure/container-registry/container-registry-get-started-azure-cli)

L'idée est de rendre disponible les images crées pour le cluster `kubernetes` puissent les déployer de
manière automatique par un pull. on va donc pousser les images dockers de Spark dans un répository
d'images docker hébergé par azure.

```
ACR_NAME=acr$RESOURCES_UID
az acr create --resource-group $AKS_RG --name    --sku Basic
```

On peut ensuite se logguer à ce repository

```
az acr login --name $ACR_NAME
```

On peut ensuite pousser dans ce répository les images de spark créees.
Pour cela sur la machine ou nous avons crée nos images `docker`, on va générer une image tagguée 
`docker tag` utilisant le repository fraichement crée et on fait ensuite un `docker push`

```
docker tag myrepo/spark:v3.2.0 acrd24f7f56.azurecr.io/spark:v3.2.0
docker push acrd24f7f56.azurecr.io/spark:v3.2.0

docker tag myrepo/spark-py:v3.2.0 acrd24f7f56.azurecr.io/spark-py:v3.2.0
docker push acrd24f7f56.azurecr.io/spark-py:v3.2.0
```

Nos images sont maintenant dans le container azure et disponible pour être `pullées`

```
> az acr repository list --name acrd24f7f56
[
  "spark",
  "spark-py"
]
```

Il faut toutefois laisser la possibilité à Kubernetes d'accéder aux images contenues dans le 
repository docker azure. Pour cela il va falloir créer un `secrets` kubernetes qui peut accéder
au répository (On fait un pont entre les deux.)
Pour cela on va passer par le service principal défini pour créer le cluster AKS.

[Doc Microsoft Container Registry Auth Kubernetes](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-kubernetes)

On récupère et on mémorise l'id du service principal d'AKS

```
SERVICE_PRINCIPAL_ID=$(az ad sp list --display-name sp-d24f7f56 --query "[].appId" --output tsv)
```

```
kubectl create secret docker-registry secret-d24f7f56 \
    --docker-server=acrd24f7f56.azurecr.io \
    --docker-username=$SERVICE_PRINCIPAL_ID \
    --docker-password="Tu8XjlC.Jrq7dQJyLYRsA9V6dN-bvkeOZG"
```

la clé `secret-d24f7f56` sera utilisé par kubernetes. Le password est celui qui est crée au moment de la
création du service principal et qu'on a mémorisé
On peut voir que cette clé est bien enregistrée dans les secrets kubernetes

```
> kubectl get secrets -n default
NAME                  TYPE                                  DATA   AGE
default-token-d4w88   kubernetes.io/service-account-token   3      5h5m
secret-d24f7f56       kubernetes.io/dockerconfigjson        1      9m49s
```

## Running in `cluster`mode

```
# Create spark-driver service account
kubectl create serviceaccount spark

# Create a cluster and namespace "role-binding" to grant the account administrative privileges
kubectl create rolebinding spark-rb --clusterrole=admin --serviceaccount=default:spark
```

```
kubectl proxy
Starting to serve on 127.0.0.1:8001
```

```
./bin/spark-submit \
  --master k8s://http://127.0.0.1:8001 \
  --deploy-mode cluster \
  --name spark-pi \
  --class org.apache.spark.examples.SparkPi \
  --conf spark.executor.instances=3 \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
  --conf spark.kubernetes.container.image=acrd24f7f56.azurecr.io/spark:v3.2.0 \
  --conf spark.kubernetes.container.image.pullSecrets=secret-d24f7f56 \
  local:///opt/spark/examples/jars/spark-examples_2.12-3.2.0.jar
```

```
> kubectl get pod
NAME                               READY   STATUS      RESTARTS   AGE
spark-pi-338feb7ddd34f8ab-driver   0/1     Completed   0          12m
> kubectl logs spark-pi-338feb7ddd34f8ab-driver
...
21/12/21 13:36:54 INFO DAGScheduler: Job 0 finished: reduce at SparkPi.scala:38, took 0.953225 s
Pi is roughly 3.140575702878514
...
```

```
./bin/spark-submit \
  --master k8s://http://127.0.0.1:8001 \
  --deploy-mode cluster \
  --name spark-pi \
  --conf spark.executor.instances=3 \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
  --conf spark.kubernetes.container.image=acrd24f7f56.azurecr.io/spark-py:v3.2.0 \
  --conf spark.kubernetes.container.image.pullSecrets=secret-d24f7f56 \
  local:///opt/spark/examples/src/main/python/pi.py 100
```

## Running in `client` mode


### Création du namespace et des rôles

```
#creates logical domain within the cluster
kubectl create namespace isolation

# within this namespace, creates a new service account
kubectl create serviceaccount jumppod -n isolation

# Giving full control within namespace bounds
kubectl create rolebinding jumppod-rb --clusterrole=admin --serviceaccount=isolation:jumppod -n isolation
```

### Récupère un shell dans cet espace de nom

```
export DRIVER_NAME=jump-1
export DRIVER_PORT=20020
kubectl run $DRIVER_NAME -ti --rm=true -n isolation --image=acrd24f7f56.azurecr.io/spark-py:v3.2.0 --serviceaccount='jumppod' bash
```

A l'intérieur de ce shell, configuré pour avoir python et spark. Il faut lancer un service pour 
que l'on puisse résoudre ce container.

A l'extérieur du container mais quand il est lancé (dans une autre fenêtre ayant accès à kubectl),
```

kubectl expose pod $DRIVER_NAME --port=$DRIVER_PORT --type=ClusterIP --cluster-ip=None
```

Ensuite à l'intérieur du container `jump-1`, on peut lancer `pyspark`

```
 export DRIVER_NAME=jump-1
 export DRIVER_PORT=20020
 export NAMESPACE=isolation
```

```
/spark/bin/pyspark \
 --master k8s://https://kubernetes.default.svc.cluster.local:443  \
 --deploy-mode client  \
 --conf spark.executor.instances=3 \
 --conf spark.kubernetes.driver.pod.name=$DRIVER_NAME  \
 --conf spark.kubernetes.authenticate.driver.serviceAccountName=jumppod \
 --conf spark.kubernetes.namespace=$NAMESPACE  \
 --conf spark.kubernetes.container.image=acrd24f7f56.azurecr.io/spark-py:v3.2.0 \
 --conf spark.driver.host=$DRIVER_NAME.$NAMESPACE.svc.cluster.local  \
 --conf spark.driver.port=$DRIVER_PORT \
 --conf spark.kubernetes.container.image.pullSecrets=secret-d24f7f56
```

```{.python}
# Create a distributed data set to test the session.
t = sc.parallelize(range(10))

# Calculate the approximate sum of values in the dataset
r = t.sumApprox(3)
print('Approximate sum: %s' % r)
```

## Définition d'une zone disque partagée dans Azure

### Création du volume

Si on se réfère au cours [Udemy](https://www.udemy.com/course/azure-kubernetes-service-aks/), il existe 
plusieurs façon d'avoir une zone disque accessible depuis un cluster AKS.
On choisit dans cet éventail, la façon la plus simple un `Volume Dynamique` managé par AKS qui repose sur 
un `Azure File`. 

Pour cela, on crée le fichier `azure_disk_pvc.yaml` suivant

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azurefile
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile
  resources:
    requests:
      storage: 1Gi
```

Lorsqu'on l'applique :
```
kubectl apply -f azure_disk_pvc.yaml
```

Cela nous crée automatiquement:
 - un `storage account` (`f4765e654b4f9413f8ce62a`) azure dans le groupe de resource `mc_$AKS_NAME` qui a été crée par azure à la création du cluster AKS.
 - un `Persistent Volume` pour Kubernetes
 - un `Persistent Volume Claim` pour Kubernetes

Nous obtenons donc tout ce qu'il faut pour travailler de manière automatique.
Pour créer un pod qui pointe sur cet espace disque, un fichier `yaml` étendu pour notre `busybox` qui
va monter ce disque sur `/mnt/azure`

```
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
    volumeMounts:
      - mountPath: "/mnt/azure"
        name: volume
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: azurefile
  restartPolicy: Always
```

### Transfert de données sur le volume

Une fois un pod actif qui a monté cette zone (comme notre `busybox` par exemple), on peut utilser 
`kubectl`

``` 
kubectl cp ./random_walk.py busybox:/mnt/azure/
```

#### Montage de la zone azure sous linux.

Le portail `azure` nous donne un moyen de monter un `storage account` sous linux

```
sudo mkdir /mnt/kubernetes-dynamic-pvc-ba3bc1c6-c56d-4d7c-9480-550296f347b8
if [ ! -d "/etc/smbcredentials" ]; then
sudo mkdir /etc/smbcredentials
fi
if [ ! -f "/etc/smbcredentials/f4765e654b4f9413f8ce62a.cred" ]; then
    sudo bash -c 'echo "username=f4765e654b4f9413f8ce62a" >> /etc/smbcredentials/f4765e654b4f9413f8ce62a.cred'
    sudo bash -c 'echo "password=B+j69GW1DheUT/Klj8+CtYqL57bFy3r0vYqQelti2tR4b4wUlXJNchwh7A4O4EYPo1+yTFVhOoLPeoWaHwzegA==" >> /etc/smbcredentials/f4765e654b4f9413f8ce62a.cred'
fi
sudo chmod 600 /etc/smbcredentials/f4765e654b4f9413f8ce62a.cred

sudo bash -c 'echo "//f4765e654b4f9413f8ce62a.file.core.windows.net/kubernetes-dynamic-pvc-ba3bc1c6-c56d-4d7c-9480-550296f347b8 /mnt/kubernetes-dynamic-pvc-ba3bc1c6-c56d-4d7c-9480-550296f347b8 cifs nofail,vers=3.0,credentials=/etc/smbcredentials/f4765e654b4f9413f8ce62a.cred,dir_mode=0777,file_mode=0777,serverino" >> /etc/fstab'
sudo mount -t cifs //f4765e654b4f9413f8ce62a.file.core.windows.net/kubernetes-dynamic-pvc-ba3bc1c6-c56d-4d7c-9480-550296f347b8 /mnt/kubernetes-dynamic-pvc-ba3bc1c6-c56d-4d7c-9480-550296f347b8 -o vers=3.0,credentials=/etc/smbcredentials/f4765e654b4f9413f8ce62a.cred,dir_mode=0777,file_mode=0777,serverino
```

Ce script :
* crée un point de montage  pour notre disque dans `/mnt`
* Obtient des `credentials` pour un montage smb
* rajoute une entrée pour ce disque dans `/etc/fstab`
* monte la zone avec pour type de volume `cifs` 

Une fois ce montage fait on peut faire ce qu'on veut depuis la machine linux qui a fait le montage.


### Exécution d'un job spark utilisant ce volume

```
~/Documents/spark-3.2.0-bin-hadoop3.2/bin/spark-submit  \
   --master k8s://http://127.0.0.1:8001 \
   --deploy-mode cluster  \
   --name random-walk \
   --conf spark.executor.instances=3  \
   --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark  \
   --conf spark.kubernetes.container.image=acrd24f7f56.azurecr.io/spark-py:v3.2.0 \
   --conf spark.kubernetes.container.image.pullSecrets=secret-d24f7f56 \
   --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.azurefile.mount.path=/mnt/azure \
   --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.azurefile.mount.readOnly=false \
   --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.azurefile.options.claimName=azurefile \
   --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.azurefile.mount.path=/mnt/azure \
   --conf spark.kubernetes.executor.volumes.persistentVolumeClaim.azurefile.options.claimName=azurefile \
   local:///mnt/azure/random_walk.py
```


# Références
* ['Jump' Pod on Kubernetes](http://blog.brainlounge.de/memoryleaks/how-to-deploy-a-jump-pod-on-kubernetes/) Excellent article qui explique la façon sous Kubernetes de lancer un pod dans un espace de nom avec les bons droits.
* [Spark on Kubernetes: First Steps](https://www.oak-tree.tech/blog/spark-kubernetes-primer)
