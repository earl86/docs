---
title: Horizontal Scaling MySQL group replication
menu:
  docs_{{ .version }}:
    identifier: guides-mysql-scaling-horizontal-group
    name: Group Replication
    parent: guides-mysql-scaling-horizontal
    weight: 20
menu_name: docs_{{ .version }}
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/README.md).

{{< notice type="warning" message="This is an Enterprise-only feature. Please install [KubeDB Enterprise Edition](/docs/setup/install/enterprise.md) to try this feature." >}}

# Horizontal Scale MySQL Group Replication

This guide will show you how to use `KubeDB` enterprise operator to increase/decrease the number of members of a `MySQL` Group Replication.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the `kubectl` command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/).

- Install `KubeDB` community and enterprise operator in your cluster following the steps [here](/docs/setup/README.md).

- You should be familiar with the following `KubeDB` concepts:
  - [MySQL](/docs/guides/mysql/concepts/database/index.md)
  - [MySQLOpsRequest](/docs/guides/mysql/concepts/opsrequest/index.md)
  - [Horizontal Scaling Overview](/docs/guides/mysql/scaling/horizontal-scaling/overview/index.md)

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```bash
$ kubectl create ns demo
namespace/demo created
```

> **Note:** YAML files used in this tutorial are stored in [docs/guides/mysql/scaling/horizontal-scaling/group-replication/yamls](https://github.com/kubedb/docs/tree/{{< param "info.version" >}}/docs/guides/mysql/scaling/horizontal-scaling/group-replication/yamls) directory of [kubedb/doc](https://github.com/kubedb/docs) repository.

### Apply Horizontal Scaling on MySQL Group Replication

Here, we are going to deploy a  `MySQL` group replication using a supported version by `KubeDB` operator. Then we are going to apply horizontal scaling on it.

#### Prepare Group Replication

At first, we are going to deploy a group replication server with 3 members. Then, we are going to add two additional members through horizontal scaling. Finally, we will remove 1 member from the cluster again via horizontal scaling.

**Find supported MySQL Version:**

When you have installed `KubeDB`, it has created `MySQLVersion` CR for all supported `MySQL` versions.  Let's check the supported MySQL versions,

```bash
$ kubectl get mysqlversion
NAME            VERSION   DISTRIBUTION   DB_IMAGE                    DEPRECATED   AGE
5.7.35-v1       5.7.35    Official       mysql:5.7.35                             56m
5.7.36          5.7.36    Official       mysql:5.7.36                             56m
8.0.17          8.0.17    Official       mysql:8.0.17                             56m
8.0.27          8.0.27    Official       mysql:8.0.27                             56m
8.0.27-innodb   8.0.27    MySQL          mysql/mysql-server:8.0.27                56m
8.0.29          8.0.29    Official       mysql:8.0.29                             56m
8.0.3-v4        8.0.3     Official       mysql:8.0.3                              56m
```

The version above that does not show `DEPRECATED` `true` is supported by `KubeDB` for `MySQL`. You can use any non-deprecated version. Here, we are going to create a MySQL Group Replication using `MySQL`  `8.0.27`.

**Deploy MySQL Group Replication:**

In this section, we are going to deploy a MySQL group replication with 3 members. Then, in the next section we will scale-up the cluster using horizontal scaling. Below is the YAML of the `MySQL` cr that we are going to create,

```yaml
apiVersion: kubedb.com/v1alpha2
kind: MySQL
metadata:
  name: my-group
  namespace: demo
spec:
  version: "8.0.29"
  replicas: 3
  topology:
    mode: GroupReplication
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

Let's create the `MySQL` cr we have shown above,

```bash
$ kubectl create -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/mysql/scaling/horizontal-scaling/group-replication/yamls/group-replication.yaml
mysql.kubedb.com/my-group created
```

**Wait for the cluster to be ready:**

`KubeDB` operator watches for `MySQL` objects using Kubernetes API. When a `MySQL` object is created, `KubeDB` operator will create a new StatefulSet, Services, and Secrets, etc. A secret called `my-group-auth` (format: <em>{mysql-object-name}-auth</em>) will be created storing the password for mysql superuser.
Now, watch `MySQL` is going to  `Running` state and also watch `StatefulSet` and its pod is created and going to `Running` state,

```bash
$ watch -n 3 kubectl get my -n demo my-group

NAME       VERSION   STATUS    AGE
my-group   8.0.29    Ready     2m

$ watch -n 3 kubectl get sts -n demo my-group
NAME       READY   AGE
my-group   3/3     2m

$ watch -n 3 kubectl get pod -n demo -l app.kubernetes.io/name=mysqls.kubedb.com,app.kubernetes.io/instance=my-group
NAME         READY   STATUS    RESTARTS   AGE
my-group-0   2/2     Running   0          3m22s
my-group-1   2/2     Running   0          3m18s
my-group-2   2/2     Running   0          3m14s
```

Let's verify that the StatefulSet's pods have joined into a group replication cluster,

```bash
$ kubectl get secrets -n demo my-group-auth -o jsonpath='{.data.\username}' | base64 -d
root

$ kubectl get secrets -n demo my-group-auth -o jsonpath='{.data.\password}' | base64 -d
gcv5(gaHfVjHg)RI

$ kubectl exec -it -n demo my-group-0 -c mysql -- mysql -u root --password=sWfUMoqRpOJyomgb --host=my-group-0.my-group-pods.demo -e "select * from performance_schema.replication_group_members"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------------------+--------------------------------------+-----------------------------------+-------------+--------------+-------------+----------------+----------------------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST                       | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION | MEMBER_COMMUNICATION_STACK |
+---------------------------+--------------------------------------+-----------------------------------+-------------+--------------+-------------+----------------+----------------------------+
| group_replication_applier | 84eeaf81-e311-11ec-8713-ca8a36a676b8 | my-group-2.my-group-pods.demo.svc |        3306 | ONLINE       | SECONDARY   | 8.0.29         | XCom                       |
| group_replication_applier | 8620667d-e311-11ec-b0bd-2e61c086151c | my-group-0.my-group-pods.demo.svc |        3306 | ONLINE       | PRIMARY     | 8.0.29         | XCom                       |
| group_replication_applier | 8e790eba-e311-11ec-b867-2a6a31edcdec | my-group-1.my-group-pods.demo.svc |        3306 | ONLINE       | SECONDARY   | 8.0.29         | XCom                       |
+---------------------------+--------------------------------------+-----------------------------------+-------------+--------------+-------------+----------------+----------------------------+

```

So, we can see that our group replication cluster has 3 members. Now, we are ready to apply the horizontal scale to this group replication.

#### Scale Up

Here, we are going to add 2 members in our group replication using horizontal scaling.

**Create MySQLOpsRequest:**

To scale up your cluster, you have to create a `MySQLOpsRequest` cr with your desired number of members after scaling. Below is the YAML of the `MySQLOpsRequest` cr that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MySQLOpsRequest
metadata:
  name: my-scale-up
  namespace: demo
spec:
  type: HorizontalScaling  
  databaseRef:
    name: my-group
  horizontalScaling:
    member: 5
```

Here,

- `spec.databaseRef.name` specifies that we are performing operation on `my-group` `MySQL` database.
- `spec.type` specifies that we are performing `HorizontalScaling` on our database.
- `spec.horizontalScaling.member` specifies the expected number of members after the scaling.

Let's create the `MySQLOpsRequest` cr we have shown above,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/mysql/scaling/horizontal-scaling/group-replication/yamls/scale_up.yaml
mysqlopsrequest.ops.kubedb.com/my-scale-up created
```

**Verify Scale-Up Succeeded:**

If everything goes well, `KubeDB` enterprise operator will scale up the StatefulSet's `Pod`. After the scaling process is completed successfully, the `KubeDB` enterprise operator updates the replicas of the `MySQL` object.

First, we will wait for `MySQLOpsRequest` to be successful.  Run the following command to watch `MySQlOpsRequest` cr,

```bash
$ watch kubectl get mysqlopsrequest -n demo
NAME            TYPE                STATUS       AGE
my-scale-up     HorizontalScaling   Successful   2m55s
```

You can see from the above output that the `MySQLOpsRequest` has succeeded. If we describe the `MySQLOpsRequest`, we shall see that the `MySQL` group replication is scaled up.

```bash
$ kubectl describe myops -n demo my-scale-up
Name:         my-scale-up
Namespace:    demo
Labels:       <none>
Annotations:  <none>
API Version:  ops.kubedb.com/v1alpha1
Kind:         MySQLOpsRequest
Metadata:
  Creation Timestamp:  2022-06-03T07:56:47Z
  Generation:          1
  Managed Fields:
    API Version:  ops.kubedb.com/v1alpha1
    Fields Type:  FieldsV1
    Manager:         kubedb-ops-manager
    Operation:       Update
    Time:            2022-06-03T07:56:47Z
  Resource Version:  456415
  UID:               a9c900cc-8b67-4ce3-b097-0516b6cdb25b
Spec:
  Database Ref:
    Name:  my-group
  Horizontal Scaling:
    Member:  5
  Type:      HorizontalScaling
Status:
  Conditions:
    Last Transition Time:  2022-06-03T07:56:47Z
    Message:               Controller has started to Progress the MySQLOpsRequest: demo/my-scale-up
    Observed Generation:   1
    Reason:                OpsRequestProgressingStarted
    Status:                True
    Type:                  Progressing
    Last Transition Time:  2022-06-03T07:56:47Z
    Message:               Horizontal scaling started in MySQL: demo/my-group for MySQLOpsRequest: my-scale-up
    Observed Generation:   1
    Reason:                HorizontalScalingStarted
    Status:                True
    Type:                  Scaling
    Last Transition Time:  2022-06-03T07:58:02Z
    Message:               Horizontal scaling Up performed successfully in MySQL: demo/my-group for MySQLOpsRequest: my-scale-up
    Observed Generation:   1
    Reason:                SuccessfullyPerformedHorizontalScaling
    Status:                True
    Type:                  ScalingUp
    Last Transition Time:  2022-06-03T07:58:02Z
    Message:               Controller has successfully scaled the MySQL demo/my-scale-up
    Observed Generation:   1
    Reason:                OpsRequestProcessedSuccessfully
    Status:                True
    Type:                  Successful
  Observed Generation:     3
  Phase:                   Successful
Events:
  Type    Reason      Age    From                        Message
  ----    ------      ----   ----                        -------
  Normal  Starting    3m52s  KubeDB Enterprise Operator  Start processing for MySQLOpsRequest: demo/my-scale-up
  Normal  Starting    3m52s  KubeDB Enterprise Operator  Pausing MySQL databse: demo/my-group
  Normal  Successful  3m52s  KubeDB Enterprise Operator  Successfully paused MySQL database: demo/my-group for MySQLOpsRequest: my-scale-up
  Normal  Starting    3m52s  KubeDB Enterprise Operator  Horizontal scaling started in MySQL: demo/my-group for MySQLOpsRequest: my-scale-up
  Normal  Successful  2m37s  KubeDB Enterprise Operator  Horizontal scaling Up performed successfully in MySQL: demo/my-group for MySQLOpsRequest: my-scale-up
  Normal  Starting    2m37s  KubeDB Enterprise Operator  Resuming MySQL database: demo/my-group
  Normal  Successful  2m37s  KubeDB Enterprise Operator  Successfully resumed MySQL database: demo/my-group
  Normal  Successful  2m37s  KubeDB Enterprise Operator  Controller has Successfully scaled the MySQL database: demo/my-group

```

Now, we are going to verify whether the number of members has increased to meet up the desired state, Let's check,

```bash
$ kubectl get secrets -n demo my-group-auth -o jsonpath='{.data.\username}' | base64 -d
root

$ kubectl get secrets -n demo my-group-auth -o jsonpath='{.data.\password}' | base64 -d
gcv5(gaHfVjHg)RI

$ kubectl exec -it -n demo my-group-0 -c mysql -- mysql -u root --password='gcv5(gaHfVjHg)RI' --host=my-group-0.my-group-pods.demo -e "select * from performance_schema.replication_group_members"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------------------+--------------------------------------+-----------------------------------+-------------+--------------+-------------+----------------+----------------------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST                       | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION | MEMBER_COMMUNICATION_STACK |
+---------------------------+--------------------------------------+-----------------------------------+-------------+--------------+-------------+----------------+----------------------------+
| group_replication_applier | 84eeaf81-e311-11ec-8713-ca8a36a676b8 | my-group-2.my-group-pods.demo.svc |        3306 | ONLINE       | SECONDARY   | 8.0.29         | XCom                       |
| group_replication_applier | 8620667d-e311-11ec-b0bd-2e61c086151c | my-group-0.my-group-pods.demo.svc |        3306 | ONLINE       | PRIMARY     | 8.0.29         | XCom                       |
| group_replication_applier | 8e790eba-e311-11ec-b867-2a6a31edcdec | my-group-1.my-group-pods.demo.svc |        3306 | ONLINE       | SECONDARY   | 8.0.29         | XCom                       |
| group_replication_applier | be1d1775-e312-11ec-89bc-b2bf75b2ae52 | my-group-3.my-group-pods.demo.svc |        3306 | ONLINE       | SECONDARY   | 8.0.29         | XCom                       |
| group_replication_applier | d00b4702-e312-11ec-bb43-0e8307eb9313 | my-group-4.my-group-pods.demo.svc |        3306 | ONLINE       | SECONDARY   | 8.0.29         | XCom                       |
+---------------------------+--------------------------------------+-----------------------------------+-------------+--------------+-------------+----------------+----------------------------+

```

You can see above that our `MySQL` group replication now has a total of 5 members. It verifies that we have successfully scaled up.

#### Scale Down

Here, we are going to remove 1 member from our group replication using horizontal scaling.

**Create MysQLOpsRequest:**

To scale down your cluster, you have to create a `MySQLOpsRequest` cr with your desired number of members after scaling. Below is the YAML of the `MySQLOpsRequest` cr that we are going to create,

```yaml
apiVersion: ops.kubedb.com/v1alpha1
kind: MySQLOpsRequest
metadata:
  name: my-scale-down
  namespace: demo
spec:
  type: HorizontalScaling  
  databaseRef:
    name: my-group
  horizontalScaling:
    member: 4
```

Let's create the `MySQLOpsRequest` cr we have shown above,

```bash
$ kubectl apply -f https://github.com/kubedb/docs/raw/{{< param "info.version" >}}/docs/guides/mysql/scaling/horizontal-scaling/group-replication/yamls/scale_down.yaml
mysqlopsrequest.ops.kubedb.com/my-scale-down created
```

**Verify Scale-down Succeeded:**

If everything goes well, `KubeDB` enterprise operator will scale down the StatefulSet's `Pod`. After the scaling process is completed successfully, the `KubeDB` enterprise operator updates the replicas of the `MySQL` object.

Now, we will wait for `MySQLOpsRequest` to be successful.  Run the following command to watch `MySQlOpsRequest` cr,

```bash
$ watch kubectl get mysqlopsrequest -n demo my-scale-down

NAME            TYPE                STATUS       AGE
my-scale-down   HorizontalScaling   Successful   2m27s
```

You can see from the above output that the `MySQLOpsRequest` has succeeded. If we describe the `MySQLOpsRequest`, we shall see that the `MySQL` group replication is scaled down.

```bash
$kubectl describe myops -n demo my-scale-down
Name:         my-scale-down
Namespace:    demo
Labels:       <none>
Annotations:  <none>
API Version:  ops.kubedb.com/v1alpha1
Kind:         MySQLOpsRequest
Metadata:
  Creation Timestamp:  2022-06-03T08:03:42Z
  Generation:          1
  Managed Fields:
    Manager:         kubedb-ops-manager
    Operation:       Update
    Time:            2022-06-03T08:03:42Z
  Resource Version:  457238
  UID:               4a2dc4d7-1117-4042-87f3-13313cc4c031
Spec:
  Database Ref:
    Name:  my-group
  Horizontal Scaling:
    Member:  4
  Type:      HorizontalScaling
Status:
  Conditions:
    Last Transition Time:  2022-06-03T08:03:42Z
    Message:               Controller has started to Progress the MySQLOpsRequest: demo/my-scale-down
    Observed Generation:   1
    Reason:                OpsRequestProgressingStarted
    Status:                True
    Type:                  Progressing
    Last Transition Time:  2022-06-03T08:03:42Z
    Message:               Horizontal scaling started in MySQL: demo/my-group for MySQLOpsRequest: my-scale-down
    Observed Generation:   1
    Reason:                HorizontalScalingStarted
    Status:                True
    Type:                  Scaling
    Last Transition Time:  2022-06-03T08:04:42Z
    Message:               Horizontal scaling down performed successfully in MySQL: demo/my-group for MySQLOpsRequest: my-scale-down
    Observed Generation:   1
    Reason:                SuccessfullyPerformedHorizontalScaling
    Status:                True
    Type:                  ScalingDown
    Last Transition Time:  2022-06-03T08:04:42Z
    Message:               Controller has successfully scaled the MySQL demo/my-scale-down
    Observed Generation:   1
    Reason:                OpsRequestProcessedSuccessfully
    Status:                True
    Type:                  Successful
  Observed Generation:     4
  Phase:                   Successful
Events:
  Type    Reason      Age   From                        Message
  ----    ------      ----  ----                        -------
  Normal  Starting    3m    KubeDB Enterprise Operator  Start processing for MySQLOpsRequest: demo/my-scale-down
  Normal  Starting    3m    KubeDB Enterprise Operator  Pausing MySQL databse: demo/my-group
  Normal  Successful  3m    KubeDB Enterprise Operator  Successfully paused MySQL database: demo/my-group for MySQLOpsRequest: my-scale-down
  Normal  Starting    3m    KubeDB Enterprise Operator  Horizontal scaling started in MySQL: demo/my-group for MySQLOpsRequest: my-scale-down
  Normal  Successful  2m    KubeDB Enterprise Operator  Horizontal scaling down performed successfully in MySQL: demo/my-group for MySQLOpsRequest: my-scale-down
  Normal  Starting    2m    KubeDB Enterprise Operator  Resuming MySQL database: demo/my-group
  Normal  Successful  2m    KubeDB Enterprise Operator  Successfully resumed MySQL database: demo/my-group
  Normal  Successful  2m    KubeDB Enterprise Operator  Controller has Successfully scaled the MySQL database: demo/my-gro
```

Now, we are going to verify whether the number of members has decreased to meet up the desired state, Let's check,

```bash
$ kubectl get secrets -n demo my-group-auth -o jsonpath='{.data.\username}' | base64 -d
root

$ kubectl get secrets -n demo my-group-auth -o jsonpath='{.data.\password}' | base64 -d
gcv5(gaHfVjHg)RI

$  kubectl exec -it -n demo my-group-0 -c mysql -- mysql -u root --password='gcv5(gaHfVjHg)RI' --host=my-group-0.my-group-pods.demo -e "select * from performance_schema.replication_group_members"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------------------+--------------------------------------+-----------------------------------+-------------+--------------+-------------+----------------+----------------------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST                       | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION | MEMBER_COMMUNICATION_STACK |
+---------------------------+--------------------------------------+-----------------------------------+-------------+--------------+-------------+----------------+----------------------------+
| group_replication_applier | 84eeaf81-e311-11ec-8713-ca8a36a676b8 | my-group-2.my-group-pods.demo.svc |        3306 | ONLINE       | SECONDARY   | 8.0.29         | XCom                       |
| group_replication_applier | 8620667d-e311-11ec-b0bd-2e61c086151c | my-group-0.my-group-pods.demo.svc |        3306 | ONLINE       | PRIMARY     | 8.0.29         | XCom                       |
| group_replication_applier | 8e790eba-e311-11ec-b867-2a6a31edcdec | my-group-1.my-group-pods.demo.svc |        3306 | ONLINE       | SECONDARY   | 8.0.29         | XCom                       |
| group_replication_applier | be1d1775-e312-11ec-89bc-b2bf75b2ae52 | my-group-3.my-group-pods.demo.svc |        3306 | ONLINE       | SECONDARY   | 8.0.29         | XCom                       |
+---------------------------+--------------------------------------+-----------------------------------+-------------+--------------+-------------+----------------+----------------------------+
```

You can see above that our `MySQL` group replication now has a total of 4 members. It verifies that we have successfully scaled down.

## Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```bash
kubectl delete my -n demo my-group
kubectl delete myops -n demo my-scale-up
kubectl delete myops -n demo my-scale-down
```