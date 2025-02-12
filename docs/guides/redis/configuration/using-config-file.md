---
title: Run Redis with Custom Configuration
menu:
  docs_{{ .version }}:
    identifier: rd-using-config-file-configuration
    name: Quickstart
    parent: rd-configuration
    weight: 10
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/README.md).

# Using Custom Configuration File

KubeDB supports providing custom configuration for Redis. This tutorial will show you how to use KubeDB to run Redis with custom configuration.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/README.md).

- To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial.

  ```bash
  $ kubectl create ns demo
  namespace/demo created

  $ kubectl get ns demo
  NAME    STATUS  AGE
  demo    Active  5s
  ```

> Note: YAML files used in this tutorial are stored in [docs/examples/redis](https://github.com/kubedb/docs/tree/{{< param "info.version" >}}/docs/examples/redis) folder in GitHub repository [kubedb/docs](https://github.com/kubedb/docs).

## Overview

Redis allows configuration via a config file. When redis docker image starts, it executes `redis-server` command. If we provide a `.conf` file directory as an argument of this command, Redis server will use configuration specified in the file. To know more about configuring Redis see [here](https://redis.io/topics/config).

At first, you have to create a config file named `redis.conf` with your desired configuration. Then you have to put this file into a [volume](https://kubernetes.io/docs/concepts/storage/volumes/). You have to specify this volume in `spec.configSecret` section while creating Redis crd. KubeDB will mount this volume into `/usr/local/etc/redis` directory of the pod and the `redis.conf` file path will be sent as an argument of `redis-server` command.

In this tutorial, we will configure `databases` and `maxclients` via a custom config file. We will use Secret as volume source.

## Custom Configuration

At first, let's create `redis.conf` file setting `databases` and `maxclients` parameters. Default value of `databases` is 16 and `maxclients` is 10000.

```bash
$ cat <<EOF >redis.conf
databases 10
maxclients 500
EOF

$ cat redis.conf
databases 10
maxclients 500
```

> Note that config file name must be `redis.conf`

Now, create a Secret with this configuration file.

```bash
$ kubectl create secret generic -n demo rd-configuration --from-file=./redis.conf
secret/rd-configuration created
```

Verify the Secret has the configuration file.

```yaml
$ kubectl get secret -n demo rd-configuration -o yaml

apiVersion: v1
stringData:
  redis.conf: |
    databases 10
    maxclients 500
kind: Secret
metadata:
  name: rd-configuration
  namespace: demo
```

Now, create Redis crd specifying `spec.configSecret` field.

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/examples/redis/configuration/redis-custom.yaml
redis.kubedb.com "custom-redis" created
```

Below is the YAML for the Redis crd we just created.

```yaml
apiVersion: kubedb.com/v1alpha2
kind: Redis
metadata:
  name: custom-redis
  namespace: demo
spec:
  version: 6.0.6
  configSecret:
    name: rd-configuration
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

Now, wait a few minutes. KubeDB operator will create necessary statefulset, services etc. If everything goes well, we will see that a pod with the name `custom-redis-0` has been created.

Check that the statefulset's pod is running

```bash
$ kubectl get pod -n demo custom-redis-0
NAME             READY     STATUS    RESTARTS   AGE
custom-redis-0   1/1       Running   0          25s
```

Check the pod's log to see if the database is ready

```bash
$ kubectl logs -f -n demo custom-redis-0
1:C 01 Oct 08:07:45.274 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 01 Oct 08:07:45.274 # Redis version=4.0.6, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 01 Oct 08:07:45.274 # Configuration loaded
1:M 01 Oct 08:07:45.275 * Running mode=standalone, port=6379.
1:M 01 Oct 08:07:45.275 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 01 Oct 08:07:45.275 # Server initialized
1:M 01 Oct 08:07:45.275 * Ready to accept connections
```

Once we see `Ready to accept connections` in the log, the database is ready.

Now, we will check if the database has started with the custom configuration we have provided. We will `exec` into the pod and use [CONFIG GET](https://redis.io/commands/config-get) command to check the configuration.

```bash
 $ kubectl exec -it -n demo custom-redis-0 sh
/data # redis-cli
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> config get databases
1) "databases"
2) "10"
127.0.0.1:6379> config get maxclients
1) "maxclients"
2) "500"
127.0.0.1:6379> exit
/data #
```

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl patch -n demo rd/custom-redis -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo rd/custom-redis

kubectl patch -n demo drmn/custom-redis -p '{"spec":{"wipeOut":true}}' --type="merge"
kubectl delete -n demo drmn/custom-redis

kubectl delete -n demo secret rd-configuration

kubectl delete ns demo
```

## Next Steps

- Learn how to use KubeDB to run a Redis server [here](/docs/guides/redis/README.md).
- Want to hack on KubeDB? Check our [contribution guidelines](/docs/CONTRIBUTING.md).
