FROM openjdk:11-jre-slim

#set proxy
ENV http_proxy=http://irproxy:8082/
ENV https_proxy=http://irproxy:8082/

RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y \
        vim \
        tmux \
        wget \
        curl \
        gnupg \
        watch 

RUN apt update && apt install -yq apt-transport-https apt-utils dnsutils

RUN curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
RUN echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | tee -a /etc/apt/sources.list.d/kubernetes.list
RUN apt update && apt install -y kubectl 

WORKDIR /

RUN wget -q https://dlcdn.apache.org/spark/spark-3.2.0/spark-3.2.0-bin-hadoop3.2.tgz && \
    tar zxvf spark-3.2.0-bin-hadoop3.2.tgz && \
    ln -s spark-3.2.0-bin-hadoop3.2 spark && \
    rm spark-3.2.0-bin-hadoop3.2.tgz && \
    rm -rf /var/cache/apt/* 



#unset proxy
ENV http_proxy=
ENV https_proxy=

