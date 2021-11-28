---
title: Upgrading Postgres standalone minor version
menu:
  docs_{{ .version }}:
    identifier: guides-postgres-upgrading-minor-standalone
    name: Standalone
    parent: guides-postgres-upgrading-minor
    weight: 10
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/README.md).

{{< notice type="warning" message="This is an Enterprise-only feature. Please install [KubeDB Enterprise Edition](/docs/setup/install/enterprise.md) to try this feature." >}}

# Upgrade minor version of Postgres Standalone

This guide will show you how to use `KubeDB` enterprise operator to upgrade the minor version of `Postgres` standalone.

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

> **Note:** YAML files used in this tutorial are stored in [docs/guides/postgres/upgrading/minorversion/standalone/yamls](/docs/guides/postgres/upgrading/minorversion/standalone/yamls) directory of [kubedb/docs](https://github.com/kube/docs) repository.

### Apply Version Upgrading on Standalone

Here, we are going to deploy a `Postgres` standalone using a supported version by `KubeDB` operator. Then we are going to apply upgrading on it.

#### Prepare Standalone

At first, we are going to deploy a standalone using supported `Postgres` version whether it is possible to upgrade from this version to another. In the next two sections, we are going to find out the supported version and version upgrade constraints.

**Find supported PostgresVersion:**

When you have installed `KubeDB`, it has created `PostgresVersion` CR for all supported `Postgres` versions. Let's check support versions,

```bash
$ kubectl get postgresversion
NAME        VERSION   DB_IMAGE                 DEPRECATED   AGE
5.7.25-v2   5.7.25    kubedb/postgres:5.7.25-v2                8s
5.7.36   5.7.29    kubedb/postgres:5.7.36                8s
5.7.36   5.7.31    kubedb/postgres:5.7.36                8s
5.7.36   5.7.33    kubedb/postgres:5.7.36                8s
8.0.14-v2   8.0.14    kubedb/postgres:8.0.14-v2                8s
8.0.20-v1   8.0.20    kubedb/postgres:8.0.20-v1                8s
8.0.27   8.0.21    kubedb/postgres:8.0.27                8s
8.0.27   8.0.23    kubedb/postgres:8.0.27                8s
8.0.3-v2    8.0.3     kubedb/postgres:8.0.3-v2                 8s
```

The version above that does not show `DEPRECATED` `true` is supported by `KubeDB` for `Postgres`. You can use any non-deprecated version. Now, we are going to select a non-deprecated version from `PostgresVersion` for `Postgres` standalone that will be possible to upgrade from this version to another version. In the next section, we are going to verify version upgrade constraints.

**Check Upgrade Constraints:**

Database version upgrade constraints is a constraint that shows whether it is possible or not possible to upgrade from one version to another. Let's check the version upgrade constraints of `Postgres` `5.7.36`,

```bash
$ kubectl get postgresversion 5.7.36 -o yaml
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
        name: postgres-backup-5.7.25-v6
      restoreTask:
        name: postgres-restore-5.7.25-v6
  upgradeConstraints:
    denylist:
      groupReplication:
      - < 5.7.29
      standalone:
      - < 5.7.29
  version: 5.7.29
```

The above `spec.upgradeConstraints.denylist` is showing that upgrading below version of `5.7.36` is not possible for both standalone and group replication. That means, it is possible to upgrade any version above `5.7.36`. Here, we are going to create a `Postgres` standalone using Postgres  `5.7.36`. Then we are going to upgrade this version to `5.7.36`.

**Deploy Postgres standalone:**

In this section, we are going to deploy a Postgres standalone. Then, in the next section, we will upgrade the version of the database using upgrading. Below is the YAML of the `Postgres` cr that we are going to create,

```yaml
apiVersion: kubedb.com/v1alpha2
kind: Postgres
metadata:
  name: pg-standalone
  namespace: demo
spec:
  version: "5.7.36"
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
$ kubectl create -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/postgres/upgrading/minorversion/standalone/yamls/standalone.yaml
postgres.kubedb.com/pg-standalone created
```

**Wait for the database to be ready:**

`KubeDB` operator watches for `Postgres` objects using Kubernetes API. When a `Postgres` object is created, `KubeDB` operator will create a new StatefulSet, Services, and Secrets, etc. A secret called `pg-standalone-auth` (format: <em>{postgres-object-name}-auth</em>) will be created storing the password for postgres superuser.
Now, watch `Postgres` is going to  `Running` state and also watch `StatefulSet` and its pod is created and going to `Running` state,

```bash
$ watch -n 3 kubectl get my -n demo pg-standalone
Every 3.0s: kubectl get my -n demo pg-standalone                 suaas-appscode: Thu Jun 18 01:28:43 2020

NAME            VERSION      STATUS    AGE
pg-standalone   5.7.36    Running   3m

$ watch -n 3 kubectl get sts -n demo pg-standalone
Every 3.0s: kubectl get sts -n demo pg-standalone                suaas-appscode: Thu Jun 18 12:57:42 2020

NAME            READY   AGE
pg-standalone   1/1     3m42s

$ watch -n 3 kubectl get pod -n demo pg-standalone-0
Every 3.0s: kubectl get pod -n demo pg-standalone-0              suaas-appscode: Thu Jun 18 12:56:23 2020

NAME              READY   STATUS    RESTARTS   AGE
pg-standalone-0   1/1     Running   0          5m23s
```

Let's verify the `Postgres`, the `StatefulSet` and its `Pod` image version,

```bash
$ kubectl get my -n demo pg-standalone -o=jsonpath='{.spec.version}{"\n"}'
5.7.36

$ kubectl get sts -n demo pg-standalone -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubedb/my:5.7.36

$ kubectl get pod -n demo pg-standalone-0 -o=jsonpath='{.spec.containers[0].image}{"\n"}'
kubedb/my:5.7.36
```

We are ready to apply upgrading on this `Postgres` standalone.

#### Upgrade

Here, we are going to upgrade `Postgres` standalone from `5.7.36` to `5.7.36`.

**Create PostgresOpsRequest:**

To upgrade the standalone, you have to create a `PostgresOpsRequest` cr with your desired version that supported by `KubeDB`. Below is the YAML of the `PostgresOpsRequest` cr that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: PostgresOpsRequest
metadata:
  name: pg-upgrade-minor-standalone
  namespace: demo
spec:
  databaseRef:
    name: pg-standalone
  type: Upgrade
  upgrade:
    targetVersion: "5.7.36"
```

Here,

- `spec.databaseRef.name` specifies that we are performing operation on `pg-group` Postgres database.
- `spec.type` specifies that we are going to perform `Upgrade` on our database.
- `spec.upgrade.targetVersion` specifies expected version `5.7.36` after upgrading.

Let's create the `PostgresOpsRequest` cr we have shown above,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/postgres/upgrading/minorversion/standalone/yamls/upgrade_minor_version_standalone.yaml
postgresopsrequest.ops.kubedb.com/pg-upgrade-minor-standalone created
```

**Verify Postgres version upgraded successfully:**

If everything goes well, `KubeDB` enterprise operator will update the image of `Postgres`, `StatefulSet`, and its `Pod`.

At first, we will wait for `PostgresOpsRequest` to be successful.  Run the following command to watch `MySQlOpsRequest` cr,

```bash
$ watch -n 3 kubectl get myops -n demo pg-upgrade-minor-standalone
Every 3.0s: kubectl get myops -n demo pg-up...  suaas-appscode: Wed Aug 12 16:12:03 2020

NAME                          TYPE      STATUS       AGE
pg-upgrade-minor-standalone   Upgrade   Successful   3m57s
```

We can see from the above output that the `PostgresOpsRequest` has succeeded. If we describe the `PostgresOpsRequest`, we shall see that the `Postgres`, `StatefulSet`, and its `Pod` have updated with a new image.

```bash
$ kubectl describe myops -n demo pg-upgrade-minor-standalone
Name:         pg-upgrade-minor-standalone
Namespace:    demo
Labels:       <none>
Annotations:  <none>
API Version:  ops.kubedb.com/v1alpha1
Kind:         PostgresOpsRequest
Metadata:
  Creation Timestamp:  2021-03-10T06:23:10Z
  Generation:          2
  Resource Version:  1027880
  Self Link:         /apis/ops.kubedb.com/v1alpha1/namespaces/demo/postgresopsrequests/pg-upgrade-minor-standalone
  UID:               c7c3148b-e768-4395-885d-12d6403c225e
Spec:
  Database Ref:
    Name:                pg-standalone
  Stateful Set Ordinal:  0
  Type:                  Upgrade
  Upgrade:
    Target Version:  5.7.36
Status:
  Conditions:
    Last Transition Time:  2021-03-10T06:23:10Z
    Message:               Controller has started to Progress the PostgresOpsRequest: demo/pg-upgrade-minor-standalone
    Observed Generation:   2
    Reason:                OpsRequestProgressingStarted
    Status:                True
    Type:                  Progressing
    Last Transition Time:  2021-03-10T06:23:10Z
    Message:               Postgres version UpgradeFunc stated for PostgresOpsRequest: demo/pg-upgrade-minor-standalone
    Observed Generation:   2
    Reason:                DatabaseVersionUpgradingStarted
    Status:                True
    Type:                  Upgrading
    Last Transition Time:  2021-03-10T06:23:50Z
    Message:               Image successfully updated in Postgres: demo/pg-standalone for PostgresOpsRequest: pg-upgrade-minor-standalone 
    Observed Generation:   2
    Reason:                SuccessfullyUpgradedDatabaseVersion
    Status:                True
    Type:                  UpgradeVersion
    Last Transition Time:  2021-03-10T06:23:50Z
    Message:               Controller has successfully upgraded the Postgres demo/pg-upgrade-minor-standalone
    Observed Generation:   2
    Reason:                OpsRequestProcessedSuccessfully
    Status:                True
    Type:                  Successful
  Observed Generation:     3
  Phase:                   Successful
Events:
  Type    Reason      Age   From                        Message
  ----    ------      ----  ----                        -------
  Normal  Starting    117s  KubeDB Enterprise Operator  Start processing for PostgresOpsRequest: demo/pg-upgrade-minor-standalone
  Normal  Starting    117s  KubeDB Enterprise Operator  Pausing Postgres databse: demo/pg-standalone
  Normal  Successful  117s  KubeDB Enterprise Operator  Successfully paused Postgres database: demo/pg-standalone for PostgresOpsRequest: pg-upgrade-minor-standalone
  Normal  Starting    117s  KubeDB Enterprise Operator  Upgrading Postgres images: demo/pg-standalone for PostgresOpsRequest: pg-upgrade-minor-standalone
  Normal  Starting    112s  KubeDB Enterprise Operator  Restarting Pod (master): demo/pg-standalone-0
  Normal  Successful  77s   KubeDB Enterprise Operator  Image successfully updated in Postgres: demo/pg-standalone for PostgresOpsRequest: pg-upgrade-minor-standalone
  Normal  Starting    77s   KubeDB Enterprise Operator  Resuming Postgres database: demo/pg-standalone
  Normal  Successful  77s   KubeDB Enterprise Operator  Successfully resumed Postgres database: demo/pg-standalone
  Normal  Successful  77s   KubeDB Enterprise Operator  Controller has Successfully upgraded the version of Postgres : demo/pg-standalone
```

Now, we are going to verify whether the `Postgres`, `StatefulSet` and it's `Pod` have updated with new image. Let's check,

```bash
$ kubectl get my -n demo pg-standalone -o=jsonpath='{.spec.version}{"\n"}'
5.7.36

$ kubectl get sts -n demo pg-standalone -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubedb/my:5.7.36

$ kubectl get pod -n demo pg-standalone-0 -o=jsonpath='{.spec.containers[0].image}{"\n"}'
kubedb/my:5.7.36
```

You can see above that our `Postgres`standalone has been updated with the new version. It verifies that we have successfully upgraded our standalone.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete my -n demo pg-standalone
kubectl delete myops -n demo pg-upgrade-minor-standalone
```