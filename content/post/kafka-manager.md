---
title: "Deploy the Kafka Manager on K8s (and connect with a Strimzi deployed Kafka cluster)"
date: 2019-09-13T09:15:01+02:00
draft: true
---

This document is a short guide on how to deploy the Yahoo Kafka Manager and connect it with a Kafka cluster deployed using the Strimzi operator.

--- 
## Prerequisites

The Kafka Manager requires the `helm` tool for installation and a running Kafka cluster for operations. In the previous post I documented how to deploy a Kafka cluster using the Strimzi operator. 

The Kafka Manager requires a connection to the Zookeeper ensemble that comes with a Kafka installation. But since the Zookeeper pods are protected an additional `stunnel` pod is required which tunnels the traffic from the outside to the Zookeeper server using the advertised `my-cluster-zookeeper-client` K8s service. To quote [Jakub Schulz](https://github.com/scholzj) from the Strimzi team:

> This is intentional. We do not want third party applications use the Zookeeper because it could have negative impact on Kafka cluster availability and because Zookeeper is quite hard to secure. If you really need a workaround, you can use this deployment which can proxy Zookeeper (it expects your Kafka cluster to be named `my-cluster` - if you use different name you should change it in the fields where `my-cluster` is used). Afterwards you should be just able to connect to `zoo-entrance:2181`.
> <cite>Jakub Scholz[^1]</cite>

[^1]: The quote is extracted from a comment on [GitHub](https://github.com/strimzi/strimzi-kafka-operator/issues/1337#issuecomment-464018845)

### Deploy the Zookeeper Entrance 

Luckily there already exists a K8s resource configuration to easily deploy the `stunnel` pod:

{{< gist scholzj 6cfcf9f63f73b54eaebf60738cfdbfae >}}

This configuration is already correctly parametrized to work with a Strimzi operator deployed Kafka cluster. Please run the following command to deploy the `stunnel` pod:

```bash
kubectl apply -f https://gist.github.com/scholzj/6cfcf9f63f73b54eaebf60738cfdbfae/raw/068d55ac65e27779f3a5279db96bae03cea70acb/zoo-entrance.yaml
```

This will deploy a `stunnel` pod and connect it with the `my-cluster-zookeeper-client` service. For external services to connect with Zookeeper the K8s service address is `zoo-entrance:2181`. This service will tunnel all requests to the Strimzi provided Zookeeper nodes. 

## Deploy the Kafka Manager

Now the with a running Zookeeper entrance the Kafka Manager can be deployed. The already exists a `helm` chart that can be used, the only configuration to change is the `zkHosts` parameter. The following command will deploy the Kafka Manager:

```bash
helm install stable/kafka-manager --name=km-testing --set zkHosts=zoo-entrance:2181
```
## Access the Kafka Manager

To access the Kafka Manager use `kubectl` to forward a port from the Kafka Manager pod:

```bash
export POD_NAME=$(kubectl get pods --namespace default -l "app=kafka-manager,release=km-testing" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:9000
```

Now the Kafka Manager can be accessed via http://localhost:8080. Optionally the Kafka Manager can also be deployed with an ingress, please see the [Helm Chart](https://github.com/helm/charts/tree/master/stable/kafka-manager) for further configuration options.


