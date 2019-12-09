---
title: "Deploy Seldon Operator and a simple model on K8s"
date: 2019-12-09T14:11:21+01:00
draft: true
---

This document is a short guide on how to deploy the Seldon operator and serve ml models. Please note that this document only highlights the very first steps to deploy the Seldon operator itself and a very simple model as a `SeldonDeployment`. 

## Prerequisites

To deploy the Seldon operator a Kubernetes cluster is required, an installation of Helm v3 is also required as the Seldon operator comes packaged as a Helm chart. First check that Helm is installed:

```bash
helm version
```

which should return a version string like this:

```
version.BuildInfo{Version:"v3.0.0", GitCommit:"e29ce2a54e96cd02ccfce88bee4f58bb6e2a28b6", GitTreeState:"clean", GoVersion:"go1.13.4"}
```

If Helm is installed create a new namespace:

```bash
kubectl create namespace seldon-system
```

## Install the Seldon operator Helm chart

```bash
helm install seldon-core seldon-core-operator --repo https://storage.googleapis.com/seldon-charts --set usageMetrics.enabled=true --namespace seldon-system
```

If the installation is successful the output looks like:

```
NAME: seldon-core
LAST DEPLOYED: Mon Dec  9 14:16:44 2019
NAMESPACE: seldon-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

## Install a Seldon Deployment

Once the operator is installed a `SeldonDeployment` is deployed that allows to query the model:

```yaml
apiVersion: machinelearning.seldon.io/v1alpha2
kind: SeldonDeployment
metadata:
  name: seldon-model
spec:
  name: test-deployment
  predictors:
  - componentSpecs:
    - spec:
        containers:
        - name: classifier
          image: seldonio/mock_classifier:1.0
    graph:
      children: []
      endpoint:
        type: REST
      name: classifier
      type: MODEL
    name: example
    replicas: 1
```

To deploy the sample model run the following command:

```
kubectl apply -f seldon-model.yaml
```

This will deploy a seldon model on Kubernetes.


## Update a Seldon Deployment

To update a running a Seldon Deployment, update the `SeldonDeployment` with your custom image and run again:

```bash
kubectl apply -f seldon-model.yaml
```

This will update the `SeldonDeployment` with a custom image containing the model code.
