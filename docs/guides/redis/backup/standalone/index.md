---
title: Redis | Stash
description: Backup and restore standalone Redis database using Stash
menu:
  docs_{{ .version }}:
    identifier: guides-rd-backup-standalone
    name: Standalone Redis
    parent: guides-rd-backup
    weight: 20
menu_name: docs_{{ .version }}
section_menu_id: guides
---


# Backup and Restore standalone Redis database using Stash

Stash 0.9.0+ supports backup and restoration of Redis databases. This guide will show you how you can backup and restore your Redis database with Stash.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using Minikube.
- Install KubeDB in your cluster following the steps [here](/docs/setup/README.md).
- Install Stash Enterprise in your cluster following the steps [here](https://stash.run/docs/latest/setup/install/enterprise/).
- If you are not familiar with how Stash backup and restore Redis databases, please check the following guide [here](/docs/guides/redis/backup/overview/index.md):

You have to be familiar with following custom resources:

- [AppBinding](/docs/guides/redis/concepts/appbinding.md)
- [Function](https://stash.run/docs/latest/concepts/crds/function/)
- [Task](https://stash.run/docs/latest/concepts/crds/task/)
- [BackupConfiguration](https://stash.run/docs/latest/concepts/crds/backupconfiguration/)
- [RestoreSession](https://stash.run/docs/latest/concepts/crds/restoresession/)

To keep things isolated, we are going to use a separate namespace called `demo` throughout this tutorial. Create the `demo` namespace if you haven't created it already.

```bash
$ kubectl create ns demo
namespace/demo created
```

## Backup Redis

This section will demonstrate how to backup a Redis database. Here, we are going to deploy a Redis database using KubeDB. Then, we are going to get backup of this database into a GCS bucket. Finally, we are going to restore the backed-up data into another Redis database.

### Deploy Sample Redis Database

Let's deploy a sample Redis database and insert some data into it.

**Create Redis CRD:**

Below is the YAML of a sample Redis crd that we are going to create for this tutorial:

```yaml
apiVersion: kubedb.com/v1alpha2
kind: Redis
metadata:
  name: demo-rd
  namespace: demo
spec:
  version: 6.2.5
  storageType: Durable
  storage:
    resources:
      requests:
        storage: 1Gi
    storageClassName: "standard"
    accessModes:
      - ReadWriteOnce
  terminationPolicy: WipeOut
```

Create the above `Redis` crd,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/redis/backup/standalone/examples/redis.yaml
redis.kubedb.com/demo-rd created
```

KubeDB will deploy a Redis database according to the above specification. It will also create the necessary secrets and services to access the database.

Let's check if the database is ready to use,

```bash
❯ kubectl get redis -n demo demo-rd
NAME      VERSION   STATUS   AGE
demo-rd   6.2.5     Ready    11m

```

The database is `Ready`. Verify that KubeDB has created a Secret and a Service for this database using the following commands,

```bash
❯ kubectl get secret -n demo -l=app.kubernetes.io/instance=demo-rd
NAME                   TYPE                       DATA   AGE
demo-rd-auth           kubernetes.io/basic-auth   2      2m42s


❯ kubectl get service -n demo -l=app.kubernetes.io/instance=demo-rd
NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
demo-rd                ClusterIP   10.96.242.0   <none>        5432/TCP   3m9s
demo-rd-pods           ClusterIP   None          <none>        5432/TCP   3m9s
```

Here, we have to use the service `demo-rd` and secret `demo-rd-auth` to connect with the database. KubeDB creates an [AppBinding](/docs/guides/redis/concepts/appbinding.md) crd that holds the necessary information to connect with the database.

**Verify AppBinding:**

Verify that the `AppBinding` has been created successfully using the following command,

```bash
❯ kubectl get appbindings -n demo
NAME              TYPE                  VERSION   AGE
demo-rd           kubedb.com/redis      6.2.5      3m54s
```

Let's check the YAML of the above `AppBinding`,

```bash
❯ kubectl get appbindings -n demo demo-rd -o yaml
```

```yaml
apiVersion: appcatalog.appscode.com/v1alpha1
kind: AppBinding
metadata:
  labels:
    app.kubernetes.io/component: database
    app.kubernetes.io/instance: demo-rd
    app.kubernetes.io/managed-by: kubedb.com
    app.kubernetes.io/name: redises.kubedb.com
  name: demo-rd
  namespace: demo
  ownerReferences:
    - apiVersion: kubedb.com/v1alpha2
      blockOwnerDeletion: true
      controller: true
      kind: Redis
      name: demo-rd
      uid: 34a5bc95-4a47-4805-8de3-48010153250a
  resourceVersion: "11653"
  uid: 82e115e1-af32-4500-ba1d-47f12d3c5cb9
spec:
  clientConfig:
    service:
      name: demo-rd
      port: 6379
      scheme: redis
  parameters:
    apiVersion: config.kubedb.com/v1alpha1
    kind: RedisConfiguration
    stash:
      addon:
        backupTask:
          name: redis-backup-6.2.5
        restoreTask:
          name: redis-restore-6.2.5
  secret:
    name: demo-rd-auth
  type: kubedb.com/redis
  version: 6.2.5
```

Stash uses the `AppBinding` crd to connect with the target database. It requires the following two fields to set in AppBinding's `Spec` section.

- `spec.clientConfig.service.name` specifies the name of the service that connects to the database.
- `spec.secret` specifies the name of the secret that holds necessary credentials to access the database.
- `spec.parameters.stash` specifies the Stash Addons that will be used to backup and restore this database.
- `spec.type` specifies the types of the app that this AppBinding is pointing to. KubeDB generated AppBinding follows the following format: `<app group>/<app resource type>`.

**Insert Sample Data:**

Now, we are going to exec into the database pod and create some sample data. At first, find out the database pod using the following command,

```bash
❯ kubectl get pods -n demo --selector="app.kubernetes.io/instance=demo-rd"
NAME                READY   STATUS    RESTARTS   AGE
demo-rd-0           1/1     Running   0          18m
```

Now, let's exec into the pod and write some data,

```bash
❯ kubectl exec -it -n demo demo-rd-0 -- sh


# redis-cli
127.0.0.1:6379> set hi appscode
OK
127.0.0.1:6379> set hello world
OK

127.0.0.1:6379> exit
# exit
```

Now, we are ready to backup this sample database.

### Prepare Backend

We are going to store our backed-up data into a GCS bucket. At first, we need to create a secret with GCS credentials then we need to create a `Repository` crd. If you want to use a different backend, please read the respective backend configuration doc from [here](https://stash.run/docs/latest/guides/latest/backends/overview/).

**Create Storage Secret:**

Let's create a secret called `gcs-secret` with access credentials to our desired GCS bucket,

```bash
$ echo -n 'changeit' > RESTIC_PASSWORD
$ echo -n '<your-project-id>' > GOOGLE_PROJECT_ID
$ cat downloaded-sa-json.key > GOOGLE_SERVICE_ACCOUNT_JSON_KEY
$ kubectl create secret generic -n demo gcs-secret \
    --from-file=./RESTIC_PASSWORD \
    --from-file=./GOOGLE_PROJECT_ID \
    --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
secret/gcs-secret created
```

**Create Repository:**

Now, crete a `Repository` using this secret. Below is the YAML of Repository crd we are going to create,

```yaml
apiVersion: stash.appscode.com/v1alpha1
kind: Repository
metadata:
  name: gcs-repo
  namespace: demo
spec:
  backend:
    gcs:
      bucket: stash-testing
      prefix: demo/redis/demo-rd
    storageSecretName: gcs-secret
```

Let's create the `Repository` we have shown above,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/redis/backup/standalone/examples/repository.yaml
repository.stash.appscode.com/gcs-repo created
```

Now, we are ready to backup our database to our desired backend.

### Backup

We have to create a `BackupConfiguration` targeting the respective AppBinding object of our desired database. Stash will create a CronJob to periodically backup the database.

**Create BackupConfiguration:**

Below is the YAML for `BackupConfiguration` crd to backup the `demo-rd` database we have deployed earlier.,

```yaml
apiVersion: stash.appscode.com/v1beta1
kind: BackupConfiguration
metadata:
  name: sample-redis-backup
  namespace: demo
spec:
  schedule: "*/5 * * * *"
  repository:
    name: gcs-repo
  target:
    ref:
      apiVersion: appcatalog.appscode.com/v1alpha1
      kind: AppBinding
      name: demo-rd
  retentionPolicy:
    name: keep-last-5
    keepLast: 5
    prune: true
```

Here,

- `spec.schedule` specifies that we want to backup the database at 5 minutes interval.
- `spec.repository.name` specifies the name of the `Repository` crd the holds the backend information where the backed up data will be stored.
- `spec.target.ref` refers to the `AppBinding` crd that was created for `demo-rd` database.
- `spec.retentionPolicy`  specifies the policy to follow for cleaning old snapshots.

Let's create the `BackupConfiguration` object we have shown above,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/redis/backup/standalone/examples/backupconfiguration.yaml
backupconfiguration.stash.appscode.com/sample-redis-backup created
```

**Verify CronJob:**

If everything goes well, Stash will create a CronJob with the schedule specified in `spec.schedule` field of `BackupConfiguration` crd.

Verify that the CronJob has been created using the following command,

```bash
❯ kubectl get cronjob -n demo
NAME                                  SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
stash-backup-sample-redis-backup      */5 * * * *   False     0        <none>          30s
```

**Wait for BackupSession:**

The `sample-redis-backup` CronJob will trigger a backup on each scheduled slot by creating a `BackupSession` crd.

Wait for a schedule to appear. Run the following command to watch `BackupSession` crd,

```bash
❯ kubectl get backupsession -n demo -w
NAME                                INVOKER-TYPE          INVOKER-NAME             PHASE       AGE
sample-redis-backup-1613390711      BackupConfiguration   sample-redis-backup   Running     15s
sample-redis-backup-1613390711      BackupConfiguration   sample-redis-backup   Succeeded   78s
```

We can see above that the backup session has succeeded. Now, we are going to verify that the backed up data has been stored in the backend.

**Verify Backup:**

Once a backup is complete, Stash will update the respective `Repository` object to reflect the backup completion. Check that the repository `gcs-repo` has been updated by the following command,

```bash
❯ kubectl get repository -n demo gcs-repo
NAME       INTEGRITY   SIZE        SNAPSHOT-COUNT   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repo   true        1.770 KiB   1                2m                       4m16s
```

Now, if we navigate to the GCS bucket, we are going to see backed up data has been stored in `demo/redis/demo-rd` directory as specified by `spec.backend.gcs.prefix` field of Repository crd.

<figure align="center">
 <img alt="Backup data in GCS Bucket" src="/docs/guides/redis/backup/standalone/images/sample-redis-backup.png">
  <figcaption align="center">Fig: Backup data in GCS Bucket</figcaption>
</figure>

> Note: Stash keeps all the backed-up data encrypted. So, data in the backend will not make any sense until they are decrypted.

## Restore Redis

Now, we are going to restore the database from the backup we have taken in the previous section. We are going to deploy a new database and initialize it from the backup.

**Stop Taking Backup of the Old Database:**

At first, let's stop taking any further backup of the old database so that no backup is taken during the restore process. We are going to pause the `BackupConfiguration` crd that we had created to get backup from the `demo-rd` database. Then, Stash will stop taking any further backup for this database.

Let's pause the `sample-redis-backup` BackupConfiguration,

```bash
❯ kubectl patch backupconfiguration -n demo sample-redis-backup --type="merge" --patch='{"spec": {"paused": true}}'
backupconfiguration.stash.appscode.com/sample-redis-backup patched
```

Now, wait for a moment. Stash will pause the BackupConfiguration. Verify that the BackupConfiguration  has been paused,

```bash
❯ kubectl get backupconfiguration -n demo sample-redis-backup
NAME                    TASK                      SCHEDULE      PAUSED   AGE
sample-redis-backup     redis-backup-6.2.5        */5 * * * *   true     5m55s
```

Notice the `PAUSED` column. Value `true` for this field means that the BackupConfiguration has been paused.

**Deploy Restored Database:**

Now, we are going to deploy the restored database similarly as we have deployed the original `sample-psotgres` database.

Below is the YAML for `Redis` crd we are going deploy to initialize from backup,

```yaml
apiVersion: kubedb.com/v1alpha2
kind: Redis
metadata:
  name: restore-rd
  namespace: demo
spec:
  version: 6.2.5
  storageType: Durable
  storage:
    resources:
      requests:
        storage: 1Gi
    storageClassName: "standard"
    accessModes:
      - ReadWriteOnce
  init:
    waitForInitialRestore: true
  terminationPolicy: WipeOut

```

Notice the `init` section. Here, we have specified `waitForInitialRestore: true` which tells KubeDB to wait for the first restore to complete before marking this database as ready to use.

Let's create the above database,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/redis/backup/standalone/examples/restore-rd.yaml
redis.kubedb.com/restore-rd created
```

This time, the database will get stuck in the `Provisioning` state because we haven't restored the data yet.

```bash
❯ kubectl get redis -n demo restore-rd
NAME                VERSION   STATUS         AGE
restore-rd          6.2.5     Provisioning   6m7s
```

You can check the log from the database pod to be sure whether the database is ready to accept connections or not.

```bash
❯ kubectl logs -n demo restore-rd-0
....
1:C 15 Dec 2021 07:23:35.716 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 15 Dec 2021 07:23:35.716 # Redis version=6.2.5, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 15 Dec 2021 07:23:35.716 # Configuration loaded
1:M 15 Dec 2021 07:23:35.716 * monotonic clock: POSIX clock_gettime
1:M 15 Dec 2021 07:23:35.717 * Running mode=standalone, port=6379.
1:M 15 Dec 2021 07:23:35.718 # Server initialized
1:M 15 Dec 2021 07:23:35.718 * Ready to accept connections

```

As you can see from the above log that the database is ready to accept connections. Now, we can start restoring this database.

**Create RestoreSession:**

Now, we need to create a `RestoreSession` object pointing to the AppBinding for this restored database.

Check AppBinding has been created for the `restore-rd` database using the following command,

```bash
❯ kubectl get appbindings -n demo restore-rd
NAME          TYPE               VERSION   AGE
restore-rd    kubedb.com/redis   6.2.5     13m
```

> If you are not using KubeDB to deploy the database, then create the AppBinding manually.

Below is the YAML for the `RestoreSession` crd that we are going to create to restore backed up data into `restore-rd` database.

```yaml
apiVersion: stash.appscode.com/v1beta1
kind: RestoreSession
metadata:
  name: demo-rd-restore
  namespace: demo
spec:
  repository:
    name: gcs-repo
  target:
    ref:
      apiVersion: appcatalog.appscode.com/v1alpha1
      kind: AppBinding
      name: restore-rd
  rules:
  - snapshots: [latest]
```

Here,

- `spec.repository.name` specifies the `Repository` crd that holds the backend information where our backed up data has been stored.
- `spec.target.ref` refers to the AppBinding crd for the `restore-rd` database where the backed up data will be restored.
- `spec.rules` specifies that we are restoring from the latest backup snapshot of the original database.

Let's create the `RestoreSession` crd we have shown above,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/redis/backup/standalone/examples/restoresession.yaml
restoresession.stash.appscode.com/demo-rd-restore created
```

Once, you have created the `RestoreSession` object, Stash will create a job to restore the database. We can watch the `RestoreSession` phase to check whether the restore process has succeeded or not.

Run the following command to watch `RestoreSession` phase,

```bash
❯ kubectl get restoresession -n demo -w
NAME                      REPOSITORY   PHASE     AGE
demo-rd-restore   gcs-repo     Running   4s
demo-rd-restore   gcs-repo     Running   15s
demo-rd-restore   gcs-repo     Succeeded   15s
demo-rd-restore   gcs-repo     Succeeded   15s
```

So, we can see from the output of the above command that the restore process succeeded.

**Verify Restored Data:**

In this section, we are going to verify that the desired data has been restored successfully. We are going to connect to the database and check whether the table we had created in the original database has been restored or not.

At first, check if the database has gone into `Ready` state using the following command,

```bash
❯ kubectl get redis -n demo restore-rd
NAME                VERSION   STATUS   AGE
restore-rd          6.2.5     Ready    11m
```

Now, exec into the database pod and verify restored data.

```bash
❯ kubectl exec -it -n demo restore-rd-0 -- /bin/sh
/ # redis-cli
127.0.0.1:6379> get hi
"appscode"
127.0.0.1:6379> get hello
"world"
127.0.0.1:6379> exit
/ # exit
```

So, from the above output, we can see the `demo` database we had created in the original database `demo-rd` has been restored in the `restore-rd` database.

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete -n demo backupconfiguration sample-redis-backup
kubectl delete -n demo restoresession demo-rd-restore
kubectl delete -n demo redis demo-rd restore-rd
kubectl delete -n demo repository gcs-repo
```
