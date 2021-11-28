---
title: Upgrading Postgres group replication major version
menu:
  docs_{{ .version }}:
    identifier: guides-postgres-upgrading-major-group
    name: Group Replication
    parent: guides-postgres-upgrading-major
    weight: 20
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/README.md).

{{< notice type="warning" message="This is an Enterprise-only feature. Please install [KubeDB Enterprise Edition](/docs/setup/install/enterprise.md) to try this feature." >}}

# Upgrade major version of Postgres Group Replication

This guide will show you how to use `KubeDB` enterprise operator to upgrade the major version of `Postgres` Group Replication.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `KubeDB` community and enterprise operator in your cluster following the steps [here](/docs/setup/README.md).

- You should be familiar with the following `KubeDB` concepts:
  - [Postgres](/docs/guides/postgres/concepts/postgres.md)
  - [PostgresOpsRequest](/docs/guides/postgres/concepts/opsrequest.md)
  - [Upgrading Overview](/docs/guides/postgres/upgrading/overview/index.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/postgres/upgrading/majorversion/group-replication/yamls](/docs/guides/postgres/upgrading/majorversion/group-replication/yamls) directory of [kubedb/docs](https://github.com/kubedb/docs) repository.

### Apply Version Upgrading on Group Replication

Here, we are going to deploy a `Postgres` group replication using a supported version by `KubeDB` operator. Then we are going to apply upgrading on it.

#### Prepare Group Replication

At first, we are going to deploy a group replication using supported that `Postgres` version whether it is possible to upgrade from this version to another. In the next two sections, we are going to find out the supported version and version upgrade constraints.

**Find supported Postgres Version:**

When you have installed `KubeDB`, it has created `PostgresVersion` CR for all supported `Postgres` versions. Let’s check the supported `Postgres` versions,

```bash
$ kubectl get postgresversion 
NAME        VERSION   DB_IMAGE                  DEPRECATED   AGE
5.7.25-v2   5.7.25    kubedb/postgres:5.7.25-v2                 3h55m
5.7.36   5.7.29    kubedb/postgres:5.7.36                 3h55m
5.7.36   5.7.31    kubedb/postgres:5.7.36                 3h55m
5.7.36   5.7.33    kubedb/postgres:5.7.36                 3h55m
8.0.14-v2   8.0.14    kubedb/postgres:8.0.14-v2                 3h55m
8.0.20-v1   8.0.20    kubedb/postgres:8.0.20-v1                 3h55m
8.0.27   8.0.21    kubedb/postgres:8.0.27                 3h55m
8.0.27   8.0.23    kubedb/postgres:8.0.27                 3h55m
8.0.3-v2    8.0.3     kubedb/postgres:8.0.3-v2                  3h55m
```

The version above that does not show `DEPRECATED` true is supported by `KubeDB` for `Postgres`. You can use any non-deprecated version. Now, we are going to select a non-deprecated version from `PostgresVersion` for `Postgres` group replication that will be possible to upgrade from this version to another version. In the next section, we are going to verify version upgrade constraints.

**Check Upgrade Constraints:**

Database version upgrade constraints is a constraint that shows whether it is possible or not possible to upgrade from one version to another. Let's check the version upgrade constraints of `Postgres` `5.7.36`,

```bash
$ kubectl get postgresversion 5.7.36 -o yaml | kubectl neat
apiVersion: catalog.kubedb.com/v1alpha1
kind: PostgresVersion
metadata:
  annotations:
    meta.helm.sh/release-name: kubedb-catalog
    meta.helm.sh/release-namespace: kube-system
  labels:
    app.kubernetes.io/instance: kubedb-catalog
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubedb-catalog
    app.kubernetes.io/version: v0.16.2
    helm.sh/chart: kubedb-catalog-v0.16.2
  name: 5.7.36
spec:
  db:
    image: kubedb/postgres:5.7.36
  distribution: Oracle
  exporter:
    image: kubedb/postgresd-exporter:v0.11.0
  initContainer:
    image: kubedb/toybox:0.8.4
  podSecurityPolicies:
    databasePolicyName: postgres-db
  replicationModeDetector:
    image: kubedb/replication-mode-detector:v0.3.2
  stash:
    addon:
      backupTask:
        name: postgres-backup-5.7.25-v7
      restoreTask:
        name: postgres-restore-5.7.25-v7
  upgradeConstraints:
    denylist:
      groupReplication:
      - < 5.7.31
      standalone:
      - < 5.7.31
  version: 5.7.31
```

The above `spec.upgradeConstraints.denylist` of `5.7.36` is showing that upgrading below version of `5.7.36` is not possible for both group replication and standalone. That means, it is possible to upgrade any version above `5.7.36`. Here, we are going to create a `Postgres` Group Replication using Postgres  `5.7.36`. Then we are going to upgrade this version to `8.0.27`.

**Deploy Postgres Group Replication:**

In this section, we are going to deploy a Postgres group replication with 3 members. Then, in the next section we will upgrade the version of the  members using upgrading. Below is the YAML of the `Postgres` cr that we are going to create,

```yaml
apiVersion: kubedb.com/v1alpha2
kind: Postgres
metadata:
  name: pg-group
  namespace: demo
spec:
  version: "5.7.36"
  replicas: 3
  topology:
    mode: GroupReplication
    group:
      name: "dc002fc3-c412-4d18-b1d4-66c1fbfbbc9b"
  storageType: Durable
  storage:
    storageClassName: "standard"
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  terminationPolicy: WipeOut
```

Let's create the `Postgres` cr we have shown above,

```bash
$ kubectl create -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/postgres/upgrading/majorversion/group-replication/yamls/group_replication.yaml
postgres.kubedb.com/pg-group created
```

**Wait for the cluster to be ready:**

`KubeDB` operator watches for `Postgres` objects using Kubernetes API. When a `Postgres` object is created, `KubeDB` operator will create a new StatefulSet, Services, and Secrets, etc. A secret called `pg-group-auth` (format: <em>{postgres-object-name}-auth</em>) will be created storing the password for postgres superuser.
Now, watch `Postgres` is going to  `Running` state and also watch `StatefulSet` and its pod is created and going to `Running` state,

```bash
$ watch -n 3 kubectl get my -n demo pg-group
Every 3.0s: kubectl get my -n demo pg-group                      suaas-appscode: Thu Jun 18 14:30:24 2020

NAME       VERSION      STATUS    AGE
pg-group   5.7.36    Running   5m52s

$ watch -n 3 kubectl get sts -n demo pg-group
Every 3.0s: kubectl get sts -n demo pg-group                     suaas-appscode: Thu Jun 18 14:31:44 2020

NAME       READY   AGE
pg-group   3/3     7m12s

$ watch -n 3 kubectl get pod -n demo -l app.kubernetes.io/name=postgress.kubedb.com,app.kubernetes.io/instance=pg-group
Every 3.0s: kubectl get pod -n demo -l app.kubernetes.io/name=postgress.kubedb.com...  suaas-appscode: Thu Jun 18 14:35:35 2020

NAME         READY   STATUS    RESTARTS   AGE
pg-group-0   2/2     Running   0          11m
pg-group-1   2/2     Running   0          9m53s
pg-group-2   2/2     Running   0          6m48s
```

Let's verify the `Postgres`, the `StatefulSet` and its `Pod` image version,

```bash
$ kubectl get my -n demo pg-group -o=jsonpath='{.spec.version}{"\n"}'
5.7.36

$ kubectl get sts -n demo -l app.kubernetes.io/name=postgress.kubedb.com,app.kubernetes.io/instance=pg-group -o json | jq '.items[].spec.template.spec.containers[1].image'
"kubedb/postgres:5.7.36"

$ kubectl get pod -n demo -l app.kubernetes.io/name=postgress.kubedb.com,app.kubernetes.io/instance=pg-group -o json | jq '.items[].spec.containers[1].image'
"kubedb/postgres:5.7.36"
"kubedb/postgres:5.7.36"
"kubedb/postgres:5.7.36"
```

Let's also verify that the StatefulSet’s pods have joined into a group replication,

```bash
$ kubectl get secrets -n demo pg-group-auth -o jsonpath='{.data.\username}' | base64 -d
root

$ kubectl get secrets -n demo pg-group-auth -o jsonpath='{.data.\password}' | base64 -d
sWfUMoqRpOJyomgb

$ kubectl exec -it -n demo pg-group-0 -c postgres -- postgres -u root --password=OUta0G6QnyBuKmi0 --host=pg-group-0.pg-group-gvr.demo -e "select * from performance_schema.replication_group_members"
postgres: [Warning] Using a password on the command line interface can be insecure.
+---------------------------+--------------------------------------+------------------------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST                  | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+------------------------------+-------------+--------------+
| group_replication_applier | 0f29bc2e-dc76-11ea-b66f-02f24f164817 | pg-group-1.pg-group-gvr.demo |        3306 | ONLINE       |
| group_replication_applier | 3741c056-dc76-11ea-ac3e-120989b9c9d5 | pg-group-2.pg-group-gvr.demo |        3306 | ONLINE       |
| group_replication_applier | e2ba6b64-dc75-11ea-93e3-f60ec5bc86b8 | pg-group-0.pg-group-gvr.demo |        3306 | ONLINE       |
+---------------------------+--------------------------------------+------------------------------+-------------+--------------+
```

We are ready to apply upgrading on this `Postgres` group replication.

#### Upgrade

Here, we are going to upgrade the `Postgres` group replication from `5.7.36` to `8.0.27`.

**Create PostgresOpsRequest:**

To upgrade your database cluster, you have to create a `PostgresOpsRequest` cr with your desired version that supported by `KubeDB`. Below is the YAML of the `PostgresOpsRequest` cr that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: PostgresOpsRequest
metadata:
  name: pg-upgrade-major-group
  namespace: demo
spec:
  type: Upgrade
  databaseRef:
    name: pg-group
  upgrade:
    targetVersion: "8.0.27"
```

Here,

- `spec.databaseRef.name` specifies that we are performing operation on `pg-group` Postgres database.
- `spec.type` specifies that we are going to perform `Upgrade` on our database.
- `spec.upgrade.targetVersion` specifies expected version `8.0.27` after upgrading.

Let's create the `PostgresOpsRequest` cr we have shown above,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/postgres/upgrading/majorversion/group-replication/yamls/upgrade_major_version_group.yaml
postgresopsrequest.ops.kubedb.com/pg-upgrade-major-group created
```

> Note: During the upgradation of the major version of Postgres group replication, a new StatefulSet is created by the KubeDB enterprise operator and the old one is deleted. The name of the newly created StatefulSet is formed as follows: `<postgres-name>-<suffix>`.
Here, `<suffix>` is a positive integer number and starts with 1. It's determined as follows:
For one-time major version upgrading of group replication, the suffix will be 1.
For the 2nd time major version upgrading of group replication, the suffix will be 2.
It will be continued...

**Verify Postgres version upgraded successfully:**

If everything goes well, `KubeDB` enterprise operator will create a new `StatefulSet` named `pg-group-1` with the desire updated version and delete the old one.

At first, we will wait for `PostgresOpsRequest` to be successful.  Run the following command to watch `MySQlOpsRequest` cr,

```bash
$ watch -n 3 kubectl get myops -n demo pg-upgrade-major-group
Every 3.0s: kubectl get myops -n demo pg-upgrade-major-group     suaas-appscode: Sat Jul 25 21:41:39 2020

NAME                     TYPE      STATUS       AGE
pg-upgrade-major-group   Upgrade   Successful   5m26s
```

You can see from the above output that the `PostgresOpsRequest` has succeeded. If we describe the `PostgresOpsRequest`, we shall see that the `Postgres` group replication is updated with new images and the `StatefulSet` is created with a new image.

```bash
$ kubectl describe myops -n demo pg-upgrade-major-group
Name:         pg-upgrade-major-group
Namespace:    demo
Labels:       <none>
Annotations:  <none>
API Version:  ops.kubedb.com/v1alpha1
Kind:         PostgresOpsRequest
Metadata:
  Creation Timestamp:  2021-03-10T08:50:47Z
  Generation:          3
    Manager:         kubedb-enterprise
    Operation:       Update
    Time:            2021-03-10T08:54:07Z
  Resource Version:  1058663
  Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/postgresopsrequests/pg-upgrade-major-group
  UID:               a1bc17f3-5537-4d15-a60a-5b4b73744637
Spec:
  Database Ref:
    Name:                pg-group
  Stateful Set Ordinal:  1
  Type:                  Upgrade
  Upgrade:
    Target Version:  8.0.27
Status:
  Conditions:
    Last Transition Time:  2021-03-10T08:50:47Z
    Message:               Controller has started to Progress the PostgresOpsRequest: demo/pg-upgrade-major-group
    Observed Generation:   2
    Reason:                OpsRequestProgressingStarted
    Status:                True
    Type:                  Progressing
    Last Transition Time:  2021-03-10T08:50:47Z
    Message:               Postgres version UpgradeFunc stated for PostgresOpsRequest: demo/pg-upgrade-major-group
    Observed Generation:   2
    Reason:                DatabaseVersionUpgradingStarted
    Status:                True
    Type:                  Upgrading
    Last Transition Time:  2021-03-10T08:54:07Z
    Message:               Image successfully updated in Postgres: demo/pg-group for PostgresOpsRequest: pg-upgrade-major-group 
    Observed Generation:   2
    Reason:                SuccessfullyUpgradedDatabaseVersion
    Status:                True
    Type:                  UpgradeVersion
    Last Transition Time:  2021-03-10T08:54:07Z
    Message:               Controller has successfully upgraded the Postgres demo/pg-upgrade-major-group
    Observed Generation:   3
    Reason:                OpsRequestProcessedSuccessfully
    Status:                True
    Type:                  Successful
  Observed Generation:     3
  Phase:                   Successful
Events:
  Type    Reason      Age    From                        Message
  ----    ------      ----   ----                        -------
  Normal  Starting    3m23s  KubeDB Enterprise Operator  Start processing for PostgresOpsRequest: demo/pg-upgrade-major-group
  Normal  Starting    3m23s  KubeDB Enterprise Operator  Pausing Postgres databse: demo/pg-group
  Normal  Successful  3m23s  KubeDB Enterprise Operator  Successfully paused Postgres database: demo/pg-group for PostgresOpsRequest: pg-upgrade-major-group
  Normal  Starting    3m23s  KubeDB Enterprise Operator  Upgrading Postgres images: demo/pg-group for PostgresOpsRequest: pg-upgrade-major-group
  Normal  Successful  2m18s  KubeDB Enterprise Operator  Joning new node with upgraded image: demo/pg-group-1-0
  Normal  Successful  88s    KubeDB Enterprise Operator  Joning new node with upgraded image: demo/pg-group-1-1
  Normal  Successful  28s    KubeDB Enterprise Operator  Joning new node with upgraded image: demo/pg-group-1-2
  Normal  Successful  3s     KubeDB Enterprise Operator  Joning new node with upgraded image: demo/pg-group-1-2
  Normal  Successful  3s     KubeDB Enterprise Operator  Image successfully updated in Postgres: demo/pg-group for PostgresOpsRequest: pg-upgrade-major-group
  Normal  Starting    3s     KubeDB Enterprise Operator  Resuming Postgres database: demo/pg-group
  Normal  Successful  3s     KubeDB Enterprise Operator  Successfully resumed Postgres database: demo/pg-group
  Normal  Successful  3s     KubeDB Enterprise Operator  Controller has Successfully upgraded the version of Postgres : demo/pg-group
```

Now, we are going to verify whether the `Postgres` and `StatefulSet` and it's `Pod` have updated with new image. Let's check,

```bash
$ kubectl get my -n demo pg-group -o=jsonpath='{.spec.version}{"\n"}'
8.0.27

$ kubectl get sts -n demo -l app.kubernetes.io/name=postgress.kubedb.com,app.kubernetes.io/instance=pg-group -o json | jq '.items[].spec.template.spec.containers[1].image'
"kubedb/postgres:8.0.27"

$ kubectl get pod -n demo -l app.kubernetes.io/name=postgress.kubedb.com,app.kubernetes.io/instance=pg-group -o json | jq '.items[].spec.containers[1].image'
"kubedb/postgres:8.0.27"
"kubedb/postgres:8.0.27"
"kubedb/postgres:8.0.27"
```

Let's also check the StatefulSet pods have joined the `Postgres` group replication,

```bash
$ kubectl get secrets -n demo pg-group-auth -o jsonpath='{.data.\username}' | base64 -d
root

$ kubectl get secrets -n demo pg-group-auth -o jsonpath='{.data.\password}' | base64 -d
sWfUMoqRpOJyomgb

$ kubectl exec -it -n demo pg-group-1-0 -c postgres -- postgres -u root --password=OUta0G6QnyBuKmi0 --host=pg-group-1-0.pg-group-gvr.demo -e "select * from performance_schema.replication_group_members"
postgres: [Warning] Using a password on the command line interface can be insecure.
+---------------------------+--------------------------------------+--------------------------------+-------------+--------------+-------------+----------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST                    | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION |
+---------------------------+--------------------------------------+--------------------------------+-------------+--------------+-------------+----------------+
| group_replication_applier | 16a6270a-dc80-11ea-b911-1e3c8e142114 | pg-group-1-2.pg-group-gvr.demo |        3306 | ONLINE       | SECONDARY   | 8.0.21         |
| group_replication_applier | 6e0709ba-dc7f-11ea-9fb1-aaa4bd57b6ff | pg-group-1-0.pg-group-gvr.demo |        3306 | ONLINE       | PRIMARY     | 8.0.21         |
| group_replication_applier | ca40ac85-dc7f-11ea-9ff0-3621cd123c11 | pg-group-1-1.pg-group-gvr.demo |        3306 | ONLINE       | SECONDARY   | 8.0.21         |
+---------------------------+--------------------------------------+--------------------------------+-------------+--------------+-------------+----------------+
```

You can see above that our `Postgres` group replication now has updated members. It verifies that we have successfully upgraded our cluster.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete my -n demo pg-group
kubectl delete myops -n demo pg-upgrade-major-group
```