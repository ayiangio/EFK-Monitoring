# EFK On Kubernetes

A simple environment for monitoring logging application using EFK stack.

#### Includes :
 * [ECK](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)
 * [Fluent-bit](https://fluentbit.io/)

This configuration for collect and filtering log application, in this config Fluent-bit is a collector log from each node and filtering and parsing log to json after that passing to elastic for save as a index

## Prerequisite
- Cluster kubernetes using more than 1 node
- Helm chart 

## Steps

- Instalation CRD ECK in Cluster 
```bash
kubectl create -f https://download.elastic.co/downloads/eck/1.7.1/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/1.7.1/operator.yaml
```

- Deploy ECK in cluster
```bash
kubectl apply -f eck/elastic.yaml
```
You will Deploy 4 Pods with each pod having a different size of pvc. 
- Master
- Data-Hot
- Data-Warm
- Data-Cold

Example for Data hot with the spec for pods using limit resources, CPU 1 Core and Memory 2Gb, and size PVC 10Gb.
```yaml
    - config: 
          node.store.allow_mmap: false
          node.master: false
          node.data: true
          node.attr.data: hot
      name: data-hot
      count: 1
      podTemplate:
        metadata: {}
        spec:
          containers:
            - env:
                - name: ES_JAVA_OPTS
                  value: '-Xms1g -Xmx1g'
              name: elasticsearch
              resources:
                limits:
                  cpu: 1000m
                  memory: 2000Mi
                requests:
                  cpu: 200m
                  memory: 200Mi
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 10Gi
            storageClassName: standard
```

## Password ECK

Don't forget to equate the value like namespace and elastic cluster name.

```bash
kubectl get secret elastic-es-elastic-user -o=jsonpath='{.data.elastic}' -n logging | base64 --decode
```

## Deploy Fluent-bit Using Helm 
Before deploy the fluent-bit, rename the index-name and change the password for authentication for ECK

```yaml
  outputs: |
    [OUTPUT]
        Name es
        Match kube.*
        Host elastic-es-http.logging
        HTTP_User elastic
        HTTP_Passwd Password 
        index example
        Buffer_Size     False
        Retry_Limit     False
        TLS             On
        TLS.verify      Off
        Trace_Error On
```

Fluentbit collecting all log service file from the cluster, you can specify the log file, change the path in this configuration, the name of the log file begins with the name of the pods
```yaml
  inputs: |
    [INPUT]
        Name tail
        Path /var/log/containers/*
        Parser service-parser
        Tag kube.*
        Mem_Buf_Limit 10MB
        Skip_Long_Lines On
        Read_from_Head true
```

```bash
helm install fluent-bit fluent/fluent-bit -n logging
```

## Deploy Kibana With Ingress
Don't forget to equate the value with the elastic cluster name.
```yaml
  elasticsearchRef:
    name: elastic
```

```
kubectl apply -f eck/kibana.yaml -n logging
```