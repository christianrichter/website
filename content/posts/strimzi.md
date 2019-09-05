---
title: "Deploy Kafka & Kafka Connect w/ Strimzi operator on a Rancher Kubernates Cluster"
date: 2019-08-29T17:47:07+02:00
draft: true
---


This document outlines the necessary steps to install a 3 node Kafka and Zookeeper cluster using the Strimzi operator on a newly provisioned Rancher Kubernetes cluster.

---

## Prerequisites

The first step is to provision a K8s cluster using the Rancher Quickstart [tutorial](https://rancher.com/docs/rancher/v2.x/en/quick-start-guide/deployment/amazon-aws-qs/).

Once the cluster is provisioned log into the Rancher console, navigate to the cluster view and download the `.kubeconfig` configuration and put it into `~./kube/config`. Check access by running:

```bash
kubectl cluster-info
```

Next step is to install the `helm` tool. Either [download](https://github.com/helm/helm/releases/tag/v2.14.3) the binary or install as [snap](https://helm.sh/docs/using_helm/#installing-helm) package.

After installing the `helm` it needs to be set up by creating a service account and initializing the `tiller` service.


```bash
kubectl create serviceaccount tiller -n kube-system
kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount kube-system:tiller
helm init --service-account tiller
```

If successful a `helm version` should return a message similar to:

```bash
Client: &version.Version{SemVer:"v2.14.3", GitCommit:"0e7f3b6637f7af8fcfddb3d2941fcc7cbebb0085", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.3", GitCommit:"0e7f3b6637f7af8fcfddb3d2941fcc7cbebb0085", GitTreeState:"clean"}
```

As the last step a S3 bucket with read/write access is required that can be accessed via an instance role which is assigned to the Kubernetes worker nodes. If that is in place then everything is prepared to install the [Strimzi](https://strimzi.io/) Kafka K8s operator.

---
## Deploy Strimzi cluster operator

The Strimzi Kafka operator simplifies the deployment of Kafka and components by introducing custom resources for Kubernetes([CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions)). Once installed there will be a resource `Kafka`.

The process to install a Kafka cluster will be to first install the operator using `helm`, then using a provided Kubernetes resource configuration making use of the CRD by configuring and applying a `Kafka` resource.

First install the Strimzi operator using `helm`:

```bash
helm repo add strimzi http://strimzi.io/charts/
helm install strimzi/strimzi-kafka-operator
```

Verify by running `helm ls`. You should see a deployed chart called `strimzi-kafka-operator-0.13.0`. Now the `KafkaCluster` can be deployed.

---
## Deploy Kafka cluster

First a corresponding Strimzi distribution needs to be obtained. Please [download](https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.13.0/strimzi-0.13.0.tar.gz) Strimzi and extract it:

```bash
wget https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.13.0/strimzi-0.13.0.tar.gz
tar xzvf strimzi-0.13.0.tar.gz
```

Now change into the Strimzi directory and apply the `Kafka` resource configuration:

```
cd strimzi-0.13.0
kubectl apply -f examples/kafka/kafka-ephemeral.yaml
```

After 1-2 minutes check if the pods are up and running. It should look similar to this:

```bash
NAME                                         READY   STATUS      RESTARTS   AGE
my-cluster-entity-operator-5cd484569d-vpqx8  3/3     Running     0          113s
my-cluster-kafka-0                           2/2     Running     0          2m16s
my-cluster-kafka-1                           2/2     Running     0          2m16s
my-cluster-kafka-2                           2/2     Running     0          2m16s
my-cluster-zookeeper-0                       2/2     Running     0          2m58s
my-cluster-zookeeper-1                       2/2     Running     0          2m58s
my-cluster-zookeeper-2                       2/2     Running     0          2m58s
strimzi-cluster-operator-7ff64d4b7-qt4tk     1/1     Running     0          14m
```

Congratulation! The Kafka cluster is up and running.


## Verify the Kafka cluster installation

Now the Kafka console producer and consumer can be used to send some messages around.

Open a new shell and run the Kafka console producer:

```bash
kubectl run kafka-producer -ti --image=strimzi/kafka:0.13.0-kafka-2.3.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
```

The warnings from the producer can be ignored, they appear because the topic doesn't exists (but will be created). Now open a second shell and run the Kafka console consumer:

```bash
kubectl run kafka-consumer -ti --image=strimzi/kafka:0.13.0-kafka-2.3.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```

Now type something in the producer shell and watch the messages appear in the cosumer shell. If that works, Kafka has been deployed correctly.

---
## Deploy Kafka connect

The next step is to configure and deploy Kafka Connect to have it cp data from a topic into a S3 bucket. The default installation of Kafka Connect only comes with a file sink, which means a custom docker image packaged with the required sink needs to be built. The required sink is the `S3SinkConnector`. But first the FileStreamSinkConnector will be used to verify an unmodified Kafka Connect will work. To deploy Kafka Connect run from within the Strimzi distribution directory:

```bash
kubectl apply -f examples/kafka-connect/kafka-connect.yaml
```

This should deploy a Kafka Connect pod. Please run `kubectl get pods` and verify a pod called similar to `my-connect-cluster-connect-684cccf6d4-mmv78` is in status `Running`. If successful proceed with starting a connector.


## Start a Kafka Connect connector

Run the following command to register a connector. This connector will pull data from the topic `my-topic` and store the events in a file `/tmp/test.sink.txt` inside the Kafka Connect container:

```bash
kubectl exec -i my-cluster-kafka-0 -- curl -s -X POST -H "Content-Type: application/json" --data '{"name": "local-file-sink", "config": {"connector.class":"FileStreamSinkConnector", "value.converter":"org.apache.kafka.connect.storage.StringConverter","key.converter":"org.apache.kafka.connect.storage.StringConverter", "tasks.max":"1", "file":"/tmp/test.sink.txt", "topics":"my-topic" }}' http://my-connect-cluster-connect-api:8083/connectors
```

This should return something like:
```
{
  "name": "local-file-sink",
  "config": {
    "connector.class": "FileStreamSinkConnector",
    "value.converter": "org.apache.kafka.connect.storage.StringConverter",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "tasks.max": "1",
    "file": "/tmp/test.sink.txt",
    "topics": "my-topic",
    "name": "local-file-sink"
  },
  "tasks": [
    {
      "connector": "local-file-sink",
      "task": 0
    }
  ],
  "type": "sink"
}
```


Verify if the connector is runnig:

```
kubectl exec -i my-cluster-kafka-0 -- curl -s -X GET -H "Content-Type: application/json" http://my-connect-cluster-connect-api:8083/connectors
```

which should return:

```
[
  "local-file-sink"
]
```


Now check if the file is created in the ephemeral storage. Please make sure to replace the name of the pod with the one return by `kubectl get pods`.

```bash
kubectl exec -i my-connect-cluster-connect-684cccf6d4-mmv78 -- cat /tmp/test.sink.txt
```

This should return the messages typed into the Kafka console producer as described in the previous section. Finally the Kafka Connect connector can be deleted:


## Delete a Kafka Connect connector


Run the following command to delete the `FileStreamSinkConnector`:

```bash
kubectl exec -i my-cluster-kafka-0 -- curl -s -X DELETE http://my-connect-cluster-connect-api:8083/connectors/local-file-sink
```

Verify the connector has shut down:

```
kubectl exec -i my-cluster-kafka-0 -- curl -s -X GET -H "Content-Type: application/json" http://my-connect-cluster-connect-api:8083/connectors
```

which should return an empty arry `[]`. If the above has worked for you, congrats, you now have a simple pipeline that moves messages from Kafka to local file system. But since the local file system will be gone when the container is terminated, a better approach could be to store the data in S3 instead of the local file system.

Please see the next section on how to build the `kafka-connect-s3` plugin, build a custom Kafka Connect image with it and finally run it.

---
## Build custom Kafka Connect image with kafka-connect-s3 plugin

Clone the project from GitHub and switch to the released version 5.3.0:

```bash
git clone https://github.com/confluentinc/kafka-connect-storage-cloud
cd kafka-connect-storage-cloud
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

This will build the Kafka Connect plugin which will be put inside the container.


In a new directory create a Dockerfile with the following content:

```bash
mkdir connect-s3-image

cat << EOF > connect-s3-image/Dockerfile
FROM strimzi/kafka:0.13.0-kafka-2.3.0
USER root:root
COPY ./my-plugins/ /opt/kafka/plugins/
USER 1001
EOF

mkdir connect-s3-image/my-plugins
```

that should now like:

```bash
drwxrwxr-x  3 richter richter 4096 Aug 23 15:07 ./
drwxrwxr-x 10 richter richter 4096 Aug 23 22:21 ../
-rw-rw-r--  1 richter richter  102 Aug 22 17:20 Dockerfile
drwxrwxr-x  3 richter richter 4096 Aug 23 15:12 my-plugins/
```

Now unzip the built `kafka-connect-s3` plugin into the `connect-s3-image/my-plugins/` folder:
```bash
unzip kafka-connect-storage-cloud/kafka-connect-s3/target/components/packages/confluentinc-kafka-connect-s3-5.3.0.zip -d connect-s3-image/my-plugins/
```

Build, tag and upload the docker image, replace the repository, image name and tag:

```bash
cd connect-s3-image
docker build .
docker tag <image_id> <repo/name>:<tag>
docker push <repo/name>:<tag>
```

Following these steps should result in a docker image uploaded to a registry.

## Deploy kafka-connect using the custom image

Now with that custom image, Kafka Connect can be deployed again. Go back into your Strimzi distribution directory and modify the file `vi examples/kafka-connect/kafka-connect.yaml` and add the line `image: <repo>/<name>:<tag>` so that it finally looks like this:


```
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
metadata:
  name: my-connect-cluster
spec:
  image: <repo>/<name>:<tag>
  version: 2.3.0
  replicas: 1
  bootstrapServers: my-cluster-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
        certificate: ca.crt
```

Now apply the changes with:
```bash
kubectl apply -f examples/kafka-connect/kafka-connect.yaml
```

It'll take a few moments to swap out the image underneath, once that has been finished it can be verified that the s3 connect plugin is available:

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

Since the first entry is the `S3SinkConnector` everything looks good. Now a task using the s3 connector can be started, please check the POST data and replace the S3 bucket with the one setup at the beginning of this document:

```bash
kubectl exec -i my-cluster-kafka-0 -- curl -X POST -H "Content-Type: application/json" --data '{"name":"s3-sink","config":{"connector.class":"S3SinkConnector","s3.region":"eu-central-1","format.class":"io.confluent.connect.s3.format.bytearray.ByteArrayFormat","flush.size":"3","tasks.max":"1","topics":"my-topic","name":"s3-sink","value.converter":"org.apache.kafka.connect.converters.ByteArrayConverter","storage.class":"io.confluent.connect.s3.storage.S3Storage","key.converter":"org.apache.kafka.connect.converters.ByteArrayConverter","s3.bucket.name":"<NAME_S3_BUCKET>"}},"type":"sink"}' http://my-connect-cluster-connect-api:8083/connectors
```

To verify that everything works run:

```bash
kubectl exec -i my-cluster-kafka-0 -- curl -s -X GET -H "Content-Type: application/json" http://my-connect-cluster-connect-api:8083/connectors
```

which should result in:

```
[
  "s3-sink"
]
```


In addition also check the S3 bucket for any data. You can always use the Kafka console producer to send more data to the Kafka topic, Kafka Connect should fetch it and move to S3.
