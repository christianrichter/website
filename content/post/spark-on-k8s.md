---
title: "Spark on K8s"
date: 2019-10-03T22:25:10+02:00
draft: false
---

This document is a short guide on how to run a PySpark job on a Kubernetes cluster accessing data in a S3 bucket.

## Prerequisites

To run the example PySpark job a Kubernetes cluster is required. A service account for Spark needs to be set up. Run the following command to create one:


```
kubectl create serviceaccount spark
```

A cluster role binding is created between the account and a role on the cluster. To grant privileges to the spark service account bind the account the role `edit` by running:

```
kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=default:spark --namespace=default
```

This allows the Spark application to start new pods running driver and executors.

## Fixing Kubernetes & S3 access

Spark comes with a Kubernetes resoucrce manager to manage its own resources. Unfortunately the default client libs in Spark v2.4.4. are outdated and in addition the S3 aws sdk needs to be added to the distribution. For me the following patch worked after obtaining Spark from GitHub:

```bash
git clone https://github.com/apache/spark
cd spark
git checkout v2.4.4
```

Now apply the following patch which will:

 * Update the K8s libs from 4.1.2 to 4.4.2
 * Add the `aws-java-sdk` to the final package

Apply the patch:


```bash
cat <<'EOF' | git apply
diff --git a/hadoop-cloud/pom.xml b/hadoop-cloud/pom.xml
index e82c41c..5cfbeb9 100644
--- a/hadoop-cloud/pom.xml
+++ b/hadoop-cloud/pom.xml
@@ -69,6 +69,12 @@
       intra-jackson-module version problems.
       -->
     <dependency>
+      <groupId>com.amazonaws</groupId>
+      <artifactId>aws-java-sdk</artifactId>
+      <version>1.7.4</version>
+      <scope>compile</scope>
+    </dependency>
+    <dependency>
       <groupId>org.apache.hadoop</groupId>
       <artifactId>hadoop-aws</artifactId>
       <version>${hadoop.version}</version>
diff --git a/resource-managers/kubernetes/core/pom.xml b/resource-managers/kubernetes/core/pom.xml
index fbd5925..f514a0c 100644
--- a/resource-managers/kubernetes/core/pom.xml
+++ b/resource-managers/kubernetes/core/pom.xml
@@ -29,7 +29,7 @@
   <name>Spark Project Kubernetes</name>
   <properties>
     <sbt.project.name>kubernetes</sbt.project.name>
-    <kubernetes.client.version>4.1.2</kubernetes.client.version>
+    <kubernetes.client.version>4.4.2</kubernetes.client.version>
   </properties>

   <dependencies>
EOF
```



Now a new Spark distribution needs to be built:

```bash
./dev/make-distribution.sh --name custom-spark --pip --r --tgz -Psparkr -Phadoop-2.7 -Phive -Phive-thriftserver -Pyarn -Pkubernetes  -Phadoop-cloud
```

This will take a few minutes to run. Once the distribution is built the Docker images can be built and pushed to the registry:

```bash
./bin/docker-image-tool.sh  -r <registry> -t <tag> build
./bin/docker-image-tool.sh  -r <registry> -t <tag> push
```
In my case it looks like this:

```bash
./bin/docker-image-tool.sh  -r docker.io/pappnase -t v2.4.4-s3-k8-fix build
./bin/docker-image-tool.sh  -r docker.io/pappnase -t v2.4.4-s3-k8-fix push
```

The images will be named `docker.io/pappnase/spark:v2.4.4-s3-k8-fix` and `spark-py` and `spark-r`.

Now the images can be used to run a PySpark job. First upload a py spark job to a bucket and set the Kubernetes cluster (e.g. `kubectl cluster-info`). Please make sure the pods / nodes have proper permissions to read from the input and write to the output bucket. If everything is in place run:


```bash
S3_BUCKET=<BUCKET_NAME>
cat <<'EOF' > s3_test.py
from pyspark.sql import SparkSession

if __name__ == '__main__':

    session = SparkSession \
        .builder \
        .getOrCreate()

    print(session
              .sparkContext
              .textFile("s3a://${S3_BUCKET}/s3_test.py")
              .count())

    session.stop()
EOF
aws s3 cp s3_test.py s3://${S3_BUCKET}/
```
This will upload a Python script to a S3 bucket. The next command will execute that script as a Spark job running on Kubernetes:


```bash
export K8S_CLUSTER=<K8S-CLUSTER>
./bin/spark-submit \
--master k8s://${K8S_CLUSTER} \
--deploy-mode cluster \
--conf spark.executor.instances=3 \
--conf spark.driver.extraJavaOptions="-Dcom.amazonaws.services.s3.enableV4 -Dcom.amazonaws.services.s3.enforceV4" \
--conf spark.executor.extraJavaOptions="-Dcom.amazonaws.services.s3.enableV4 -Dcom.amazonaws.services.s3.enforceV4" \
--conf spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem  \
--conf spark.hadoop.fs.s3a.endpoint=s3.eu-central-1.amazonaws.com  \
--conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
--conf spark.kubernetes.container.image=pappnase/spark-py:v2.4.4-s3-k8-fix  \
--conf spark.kubernetes.executor.request.cores=100m \
--conf spark.kubernetes.pyspark.pythonVersion=3 \
s3a://${S3_BUCKET}/s3_test.py
```
This simply counts the number of lines within a file.

## Python dependencies

In case the Python standard libraries are not sufficient, I had success loading additional libraries on the fly following this pattern:

```python
try:
    import boto3
except ImportError:
    os.system('python3 -m pip install boto3')

import boto3
```
