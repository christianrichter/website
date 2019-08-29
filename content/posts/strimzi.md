---
title: "Deploy Kafka & Kafka Connect w/ Strimzi operator on a Kubernates Cluster"
date: 2019-08-29T17:47:07+02:00
draft: true
---


# This document is a summary of the commands and steps to install the Strimzi Kafka operator on a newly provisioned k8s cluster using RKE

## Prerequisites

 * Provison cluster and download k8s config into ~/.kube/config
 * Install `helm` tool from https://helm.sh/docs/using_helm/#installing-helm and put the binary into PATH



## Set up Helm/Tiller

```bash
kubectl create serviceaccount tiller -n kube-system
kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount kube-system:tiller
helm init --service-account tiller
```


## Deploy Strimzi cluster operator

```bash
helm repo add strimzi http://strimzi.io/charts/
helm install strimzi/strimzi-kafka-operator
helm ls
```

## Deploy Kafka cluster

 * Download Strimzi from https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.13.0/strimzi-0.13.0.tar.gz and extract

```bash
wget https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.13.0/strimzi-0.13.0.tar.gz
tar xzvf strimzi-0.13.0.tar.gz
cd strimzi-0.13.0
kubectl apply -f examples/kafka/kafka-ephemeral.yaml
```

 * Check that everything is working by starting a producer and consumer each in a separate shell


Open a new shell and run:

```bash
kubectl run kafka-producer -ti --image=strimzi/kafka:0.13.0-kafka-2.3.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
```

Open a new shell and run:

```bash
kubectl run kafka-consumer -ti --image=strimzi/kafka:0.13.0-kafka-2.3.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```

Now type something in the producer shell and watch the messages appear in the cosumer shell. If that works, Kafka (and Zookeeper) have been deployed correctly.


## Deploy Kafka connect

 * From within the strimzi distribution directory run:

```bash
kubectl apply -f examples/kafka-connect/kafka-connect.yaml
```

## Start a connector

Run the following command to register a running connector

```bash
 kubectl exec -i my-cluster-kafka-0 -- curl -X POST -H "Content-Type: application/json" --data '{"name": "local-file-sink", "config": {"connector.class":"FileStreamSinkConnector", "value.converter":"org.apache.kafka.connect.storage.StringConverter","key.converter":"org.apache.kafka.connect.storage.StringConverter", "tasks.max":"1", "file":"/tmp/test.sink.txt", "topics":"my-topic" }}' http://my-connect-cluster-connect-api:8083/connectors
```

Now check if the file is in ephemeral storage (replace my-connect-cluster identifier)

```bash
kubectl exec -i my-connect-cluster-connect-74c8b9d4fc-b2thg -- cat /tmp/test.sink.txt
```


## Delete a connector

```bash
kubectl exec -i my-cluster-kafka-0 -- curl -X DELETE http://my-connect-cluster-connect-api:8083/connectors/local-file-sink
```


# Deploy a custom image with kafka-connect-s3 plugin and move data from kafka to s3


## Build custom image with kafka-connect-s3

Download the project and switch to released version 5.3.0

```bash
git clone https://github.com/confluentinc/kafka-connect-storage-cloud
git checkout v5.3.0
```

I had to explicitly set the repository url and applied the following change:

```diff
diff --git a/pom.xml b/pom.xml
index 374fa20..bee455c 100644
--- a/pom.xml
+++ b/pom.xml
@@ -68,7 +68,7 @@
         <repository>
             <id>confluent</id>
             <name>Confluent</name>
-            <url>${confluent.maven.repo}</url>
+            <url>http://packages.confluent.io/maven/</url>
         </repository>
     </repositories>
```

Now build the project:

```bash
mvn -U clean install -DskipTests
```


In a new directory create  a Dockerfile with the following content:

```bash
mkdir connect-s3-image

cat << EOF
FROM strimzi/kafka:0.13.0-kafka-2.3.0
USER root:root
COPY ./my-plugins/ /opt/kafka/plugins/
USER 1001
EOF
> connect-s3-image/Dockerfile

mkdir connect-s3-image/my-plugins
```

that should now like:

```bash
drwxrwxr-x  3 richter richter 4096 Aug 23 15:07 ./
drwxrwxr-x 10 richter richter 4096 Aug 23 22:21 ../
-rw-rw-r--  1 richter richter  102 Aug 22 17:20 Dockerfile
drwxrwxr-x  3 richter richter 4096 Aug 23 15:12 my-plugins/
```

Now copy the zip file from kafka-connect-s3 in `my-plugins/`:
```bash
cp kafka-connect-storage-cloud/kafka-connect-s3/target/components/packages/confluentinc-kafka-connect-s3-5.3.0.zip connect-s3-image/my-plugins/
```

Build, tag and upload the docker image:

```bash
cd connect-s3-image
docker build .
docker tag <image_id> <image/name>:<tag>
docker push <image/name>:<tag>
```

Following these steps should result in a docker image uploaded to a registry.


## Deploy kafka-connect using the custom image


Go back into your strimzi distribution directory and modify the file `vi examples/kafka-connect/kafka-connect.yaml` and add the line `image: pappnase/connect:latest` so that finally looks like this:


```
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
metadata:
  name: my-connect-cluster
spec:
  image: pappnase/connect:latest
  version: 2.3.0
  replicas: 1
  bootstrapServers: my-cluster-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
        certificate: ca.crt
```

Now apply the changes with
```bash
kubectl apply -f examples/kafka-connect/kafka-connect.yaml
```

Now verify if the s3 connect plugin is available

```bash
kubectl exec -i my-cluster-kafka-0 -- curl -s -X GET http://my-connect-cluster-connect-api:8083/connector-plugins
```

which should result in
```json
[
  {
    "class": "io.confluent.connect.s3.S3SinkConnector",
    "type": "sink",
    "version": "5.3.0"
  },
  {
    "class": "io.confluent.connect.storage.tools.SchemaSourceConnector",
    "type": "source",
    "version": "2.3.0"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "2.3.0"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "2.3.0"
  }
]
```

Since the first entry is the `S3SinkConnector` everything looks good.

Now a task using the s3 connector can be started, please check the POST data and use a s3 bucket with write access.



```bash
kubectl exec -i my-cluster-kafka-0 -- curl -X POST -H "Content-Type: application/json" --data '{"name":"s3-sink","config":{"connector.class":"S3SinkConnector","s3.region":"eu-central-1","format.class":"io.confluent.connect.s3.format.bytearray.ByteArrayFormat","flush.size":"3","tasks.max":"1","topics":"my-topic","name":"s3-sink","value.converter":"org.apache.kafka.connect.converters.ByteArrayConverter","storage.class":"io.confluent.connect.s3.storage.S3Storage","key.converter":"org.apache.kafka.connect.converters.ByteArrayConverter","s3.bucket.name":"li.wireframe.connect.test"}}',"type":"sink"}' http://my-connect-cluster-connect-api:8083/connectors
```
