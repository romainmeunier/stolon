# Stolon inside kubernetes

In this example you'll see how stolon can provide an high available postgreSQL cluster inside kubernetes.


## Docker image
Prebuilt images are available on the dockerhub, the images' tags are their release version. Additional images are available:

* `latest`: latest released image (actually v0.4.0).
* `master`: automatically built after every commit to the master branch.


In the [image](examples/kubernetes/image/docker) directory you'll find the Dockerfile to build the image used in this example. Once the image is built you should push it to the docker registry used by your kubernetes infrastructure.

`sorintlab/stolon:master` is the one used by the kubernetes definitions in this example.
For a more stable testing you can use `sorintlab/stolon:latest`

## Cluster setup and tests

This example has some predefined values that you'd like to change:
* The cluster name is `kube-cluster`
* It points to a single node etcd cluster on `10.245.1.1:2379` without tls. You can change the ST${COMPONENT}_STORE_ENDPOINTS environment variables in the definitions to point to the right etcd cluster.


### Initialize the cluster

The first step is to initialize a cluster with a cluster specification. For now we'll just initialize a cluster without providing a cluster specification but using a default one that will just start with an empty database cluster.

```
./bin/stolonctl --cluster-name=kube-stolon --store-backend=etcd init
```

If you want to automate this step you can just pass an initial cluster specification to the sentinels with the `--initial-cluster-spec` option.

### Create the sentinel(s)

```
kubectl create -f stolon-sentinel.yaml
```

This will create a replication controller with one pod executing the stolon sentinel. You can also increase the number of replicas for stolon sentinels in the rc definition or do it later.

### Create the keeper's password secret

This creates a password secret that can be used by the keeper to set up the initial database user. This example uses the value 'password1' but you will want to replace the value with a Base64-encoded password of your choice.

```
kubectl create -f secret.yaml
```

### Create the first stolon keeper
Note: In this example the stolon keeper is a replication controller that, for every pod replica, uses a volume for stolon and postgreSQL data of `emptyDir` type. So it'll go away when the related pod is destroyed. This is just for easy testing. In production you should use a persistent volume. Actually (kubernetes 1.0), for working with persistent volumes you should define a different replication controller with `replicas=1` for every keeper instance.


```
kubectl create -f stolon-keeper.yaml
```

This will create a replication controller that will create one pod executing the stolon keeper.
The first keeper will initialize an empty postgreSQL instance and the sentinel will elect it as the master.

Once the leader sentinel has elected the first master and created the initial cluster view you can add additional stolon keepers. Will do this later.

### Setup superuser password

Now, you should add a password for the `stolon` user (since it's the os user used by the image for initializing the database). In future this step should be automated.

#### Get the pod id
```
kubectl get pods

NAME                       READY     STATUS    RESTARTS   AGE
stolon-keeper-rc-qpqp9     1/1       Running   0          1m
```

### Create the proxies

```
kubectl create -f stolon-proxy.yaml
```
Also the proxies can be created from the start with multiple replicas.

### Create the proxy service

The proxy service is used as an entry point with a fixed ip and dns name for accessing the proxies.

```
kubectl create -f stolon-proxy-service.yaml
```

### Connect to the db

#### Get the proxy service ip

```
kubectl get svc
NAME                   LABELS                                    SELECTOR                                       IP(S)           PORT(S)
stolon-proxy-service   <none>                                    stolon-cluster=kube-stolon,stolon-proxy=true   10.247.50.217   5432/TCP
```

#### Connect to the proxy service

The password for the stolon user will be the value specified in your `secret.yaml` above (or `password1` if you did not change it). 

```
psql --host 10.247.50.217 --port 5432 postgres -U stolon -W
Password for user stolon:
psql (9.4.5, server 9.4.4)
Type "help" for help.

postgres=#
```

### Create a test table and insert a row

```
postgres=# create table test (id int primary key not null, value text not null);
CREATE TABLE
postgres=# insert into test values (1, 'value1');
INSERT 0 1
postgres=# select * from test;
 id | value
----+--------
  1 | value1
(1 row)
```

### Add another keeper

```
kubectl scale --replicas=2 rc stolon-keeper-rc
```

you'll have a situation like this:

```
kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
stolon-keeper-rc-2fvw1     1/1       Running   0          1m
stolon-keeper-rc-qpqp9     1/1       Running   0          5m
stolon-proxy-rc-up3x0      1/1       Running   0          5m
stolon-sentinel-rc-9cvxm   1/1       Running   0          5m
```

you should wait some seconds (take a look at the pod's logs) to let the postgresql in the new keeper pod to sync with the master

### Simulate master death
There are different ways to tests this. The simplest one is to delete the firstly created keeper pod (that is the master postgresql)

```
kubectl delete pods stolon-keeper-rc-qpqp9
```

You can take a look at the leader sentinel log and will see that after some seconds it'll declare the master keeper as not healthy and elect the other one as the new master:
```
no keeper info available db=cb96f42d keeper=d8de7dc1
no keeper info available db=cb96f42d keeper=d8de7dc1
master db is failed db=cb96f42d keeper=d8de7dc1
trying to find a standby to replace failed master
electing db as the new master db=087ce88a keeper=1749142f
```

Now, inside the previous `psql` session you can redo the last select. The first time `psql` will report that the connection was closed and then it successfully reconnected:

```
postgres=# select * from test;
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.
postgres=# select * from test;
 id | value
----+--------
  1 | value1
(1 row)
```


### Scale your cluster

you can also add additional stolon keepers and also increase/decrease the number of stolon sentinels and proxies:

```
kubectl scale --replicas=2 rc stolon-sentinel-rc
```

```
kubectl scale --replicas=2 rc stolon-proxy-rc
```
