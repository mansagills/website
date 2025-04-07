---
reviewers:
- eparis
- pmorie
title: Configuring Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

Learn how to manage Redis configurations in a Kubernetes environment using ConfigMaps - a powerful technique for separating configuration from your contrainer images. 

# Overview

This tutorial walks through using ConfigMaps to store Redis settings outside your container. This approach lets you change Redis settings without having to create new container images. You'll learn how to:

* Create a ConfigMap with Redis configuration values
* Create a Redis Pod that mounts and uses the created ConfigMap
* Verify that the configuration was correctly applie

New to Kubernetes? Start with the [Kubernetes basics tutorial](https://kubernetes.io/docs/tutorials/kubernetes-basics/) before diving in. 


## Before your start

You'll need:

* A Kubernetes cluster (with at least two non-control plane nodes recommended)
* The kubectl command-line tool configured to communicate with your cluster
* kubectl version 1.14 or higher

If you don't already have a cluster, you can create one using  [minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download) or try out one of these Kubernetes playgrounds:
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)
* [Play with Kubernetes](https://labs.play-with-k8s.com/)

> Note: This tutorial builds upon the concepts in [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/).

## Configuring Redis with ConfigMap data

Follow the steps below to configure a Redis cache using data stored in a ConfigMap.

### Step 1: Create and apply a ConfigMap and Redis Pod

First create a ConfigMap with an empty configuration block:

```shell
cat <<EOF >./example-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: ""
EOF
```

Next, apply the ConfigMap created above, along with a Redis pod manifest:

```shell
kubectl apply -f example-redis-config.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```
### Step 2: Examine the Redis Pod Configuration

Let's examine the contents of the Redis pod manifest and note the following:

* A volume named `config` is created by `spec.volumes[1]`
* The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the 
  `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
* The `config` volume is then mounted at `/redis-master` by `spec.containers[0].volumeTMounts[1]`.

This has the net effect of exposing the data in `data.redis-config` from the `example-redis-config`
ConfigMap above as `/redis-master/redis.conf` inside the Pod.

Here's the Redis Pod configuration for reference:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis:5.0.4
    command:
      - redis-server
      - "/redis-master/redis.conf"
    env:
    - name: MASTER
      value: "true"
    ports:
    - containerPort: 6379
    resources:
      limits:
        cpu: "0.1"
    volumeMounts:
    - mountPath: /redis-master-data
      name: data
    - mountPath: /redis-master
      name: config
  volumes:
  - name: data
    emptyDir: {}
  - name: config
    configMap:
      name: example-redis-config
      items:
      - key: redis-config
        path: redis.conf
```

### Step 3: Verify the resources were created

Let's check that both the Pod and ConfigMap were created successfully:

```shell
kubectl get pod/redis configmap/example-redis-config 
```

You should see a similar output:

```
NAME        READY   STATUS    RESTARTS   AGE
pod/redis   1/1     Running   0          8s

NAME                             DATA   AGE
configmap/example-redis-config   1      14s
```

Now, recall that we left the `redis-config` key in the `example-redis-config` ConfigMap blank. Let's look at the ConfigMap

```shell
kubectl describe configmap/example-redis-config
```
You should see that the `redis-config` key is empty:

```
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
```

## Step 4: Verify the initial Redis configuration


Use `kubectl exec` to enter the pod and run the `redis-cli` tool to check the current configuration:

```bash
kubectl exec -it redis -- redis-cli
127.0.0.1:6379> CONFIG GET maxmemory
1) "maxmemory"
2) "0"
127.0.0.1:6379> CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "noeviction"
127.0.0.1:6379> exit
```

## Step 5: Updating the ConfigMap and testing configuration persistence

Now let's add configuration values to the `example-redis-config` ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: |
    maxmemory 2mb
    maxmemory-policy allkeys-lru    
```

Apply the updated ConfigMap:

```shell
kubectl apply -f example-redis-config.yaml
```

Confirm that the ConfigMap was updated:

```shell
kubectl describe configmap/example-redis-config
```

You should see the configuration values we just added:

```
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
----
maxmemory 2mb
maxmemory-policy allkeys-lru
```

Confirm the configuration was applied

Now let's check the Redis Pod again using `redis-cli` via `kubectl exec` to confirm the configuration was applied:

```bash
kubectl exec -it redis -- redis-cli
127.0.0.1:6379> CONFIG GET maxmemory
1) "maxmemory"
2) "0"
127.0.0.1:6379> CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "noeviction"
127.0.0.1:6379>
```

## Step 6: Restart the Redis Pod to apply the configuration

As we can see, the configuration values have not changed because the Pod needs to be restarted to grab updated values from associated ConfigMaps. Let's delete and recreate the Pod:

```bash
kubectl delete pod redis
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

### Step 7: Verify the configuration

Use `kubectl exec` to enter the pod and run the `redis-cli` tool to check the current configuration:

```bash
kubectl exec -it redis -- redis-cli
127.0.0.1:6379> CONFIG GET maxmemory
1) "maxmemory"
2) "2097152"
127.0.0.1:6379> CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "allkeys-lru"
127.0.0.1:6379> exit
```

The output confirms that Redis is now using the configuration values we defined in the ConfigMap:
- `maxmemory` is set to 2MB (2097152 bytes)
- `maxmemory-policy` is set to `allkeys-lru`

## Cleanup

When you're finished with the tutorial, you can clean up the resources:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## Next steps


* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* See an [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/) example.