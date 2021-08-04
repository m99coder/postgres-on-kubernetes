# postgres-on-kubernetes
PostgreSQL on Kubernetes

## Prerequisites

```bash
$ # install minikube to create a single-node cluster
$ brew install minikube

$ # start cluster using VMs
$ minikube start --vm=true

$ # create a custom namespace and context
$ kubectl create namespace postgres
$ kubectl config set-context postgres --namespace postgres --cluster minikube --user minikube
$ kubectl config use-context postgres
```

## Persistent Volumes

In order to persist the data stored in PostgreSQL it’s necessary to create [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) that have a pod-independent lifecycle. Within a Stateful Set a so called _Persistent Volume Claim_ with a specific [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes/) can be configured.

There are two ways to create Persistent Volumes. Either you manually create a volume per replica of PostgreSQL or you configure [dynamic provisioning](https://minikube.sigs.k8s.io/docs/handbook/persistent_volumes/#dynamic-provisioning-and-csi). For simplicity we choose the manual approach first.

```bash
$ # create 3 persistent volumes
$ kubectl apply -f pv-0.yaml
$ kubectl apply -f pv-1.yaml
$ kubectl apply -f pv-2.yaml

$ # list persistent volumes
$ kubectl get pv
NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-postgresql-0   1Gi        RWO            Retain           Available           default                 9s
pv-postgresql-1   1Gi        RWO            Retain           Available           default                 7s
pv-postgresql-2   1Gi        RWO            Retain           Available           default                 3s
```

## Headless Service

A [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) is specified with `clusterIP: None` and omits to use a L4 load balancer. By also defining a selector, the endpoints controller creates `Endpoint` records and also modifies the DNS configuration so that `A` records are returned that point directly to the pods.

```bash
$ # create headless service
$ kubectl apply -f svc.yaml

$ # describe headless service
$ kubectl describe svc postgresql-svc
Name:              postgresql-svc
Namespace:         postgres
Labels:            sfs=postgresql-sfs
Annotations:       <none>
Selector:          sfs=postgresql-sfs
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                None
IPs:               None
Port:              postgresql-port  5432/TCP
TargetPort:        5432/TCP
Endpoints:         <none>
Session Affinity:  None
Events:            <none>
```

## Secrets

PostgreSQL uses environment variables for configuration. The most important one for the official [PostgreSQL Docker image](https://hub.docker.com/_/postgres) is `POSTGRES_PASSWORD`. We utilize [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) to inject the respective value into the container later on.

```bash
$ # create secret from literal
$ kubectl create secret generic postgresql-secrets \
    --from-literal=POSTGRES_PASSWORD=tes6Aev8

$ # describe secret
$ kubectl describe secrets postgresql-secrets
Name:         postgresql-secrets
Namespace:    postgres
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
POSTGRES_PASSWORD:  8 bytes
```

## Stateful Sets

A [Stateful Set](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) is similar to a _Replica Set_ in a sense that it also handles pods for the configured number of replicas. In contrast to a _Replica Set_, it maintains a sticky identity for each of them. This means they are created in a fixed, sequential order and deleted counterwise. Their network identity is stable as well, what enables us to reference them by the automatically assigned DNS host inside of the cluster.

```bash
$ # create stateful set with 3 replicas
$ kubectl apply -f sfs.yaml

$ # list stateful sets
$ kubectl get statefulsets
NAME             READY   AGE
postgresql-sfs   3/3     16s

$ # list pods
$ kubectl get pods
NAME               READY   STATUS    RESTARTS   AGE
postgresql-sfs-0   1/1     Running   0          86s
postgresql-sfs-1   1/1     Running   0          83s
postgresql-sfs-2   1/1     Running   0          80s

$ # inspect logs of a random pod
$ kubectl logs postgresql-sfs-0

PostgreSQL Database directory appears to contain a database; Skipping initialization

2021-08-04 08:19:50.832 UTC [1] LOG:  starting PostgreSQL 13.3 (Debian 13.3-1.pgdg100+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit
2021-08-04 08:19:50.832 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2021-08-04 08:19:50.832 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2021-08-04 08:19:50.835 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2021-08-04 08:19:50.838 UTC [26] LOG:  database system was shut down at 2021-08-03 14:33:17 UTC
2021-08-04 08:19:50.843 UTC [1] LOG:  database system is ready to accept connections

$ # describe the service to see that 3 endpoints were created automatically
$ kubectl describe svc postgresql-svc
Name:              postgresql-svc
Namespace:         postgres
Labels:            sfs=postgresql-sfs
Annotations:       <none>
Selector:          sfs=postgresql-sfs
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                None
IPs:               None
Port:              postgresql-port  5432/TCP
TargetPort:        5432/TCP
Endpoints:         172.17.0.3:5432,172.17.0.4:5432,172.17.0.5:5432
Session Affinity:  None
Events:            <none>
```

## Connect to

You are able to directly connect to PostgreSQL by starting `bash` within a particular pod.

```bash
$ kubectl exec -it postgresql-sfs-0 -- bash
root@postgresql-sfs-0:/# PGPASSWORD=tes6Aev8 psql -U postgres
psql (13.3 (Debian 13.3-1.pgdg100+1))
Type "help" for help.

postgres=# exit
root@postgresql-sfs-0:/# exit
exit
```

An alternative approach is running a temporary PostgreSQL container and using the included `psql` to connect to one of the database instances. Where the hostname is the automatically created DNS host of the service we deployed earlier. The format of that hostname is: `<service-name>.<namespace>.svc.cluster.local` and will be resolved to a random pod running a database server.

```bash
$ kubectl run -it --rm pg-psql --image=postgres:13.3 --restart=Never \
    --env="PGPASSWORD=tes6Aev8" -- \
    psql -h postgresql-svc.postgres.svc.cluster.local -U postgres
If you don't see a command prompt, try pressing enter.
postgres=# \q
pod "pg-psql" deleted
```

To check that the DNS hostname works we deploy a busybox instance.

```bash
$ kubectl run -it --rm busybox --image=busybox --restart=Never -- sh
If you don't see a command prompt, try pressing enter.
/ # ping postgresql-svc.postgres.svc.cluster.local
PING postgresql-svc.postgres.svc.cluster.local (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.828 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.17.0.3: seq=2 ttl=64 time=0.080 ms
^C
--- postgresql-svc.postgres.svc.cluster.local ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.080/0.329/0.828 ms

/ # nslookup postgresql-svc.postgres.svc.cluster.local
Server:		10.96.0.10
Address:	10.96.0.10:53

Name:	postgresql-svc.postgres.svc.cluster.local
Address: 172.17.0.5
Name:	postgresql-svc.postgres.svc.cluster.local
Address: 172.17.0.4
Name:	postgresql-svc.postgres.svc.cluster.local
Address: 172.17.0.3

*** Can't find postgresql-svc.postgres.svc.cluster.local: No answer

/ # exit
pod "busybox" deleted
```

## Connection pooling

> [Postgres connection pool on Kubernetes in 1 minute](https://jmrobles.medium.com/postgres-connection-pool-on-kubernetes-in-1-minute-80b8020315ef)

First we create a _Config Map_ for the following values:

- `DB_HOST`: DNS hostname of our PostgreSQL service
- `DB_USER`: PostgreSQL database user

`DB_PASSWORD` will be set using the previously created secret. `POOL_MODE` is set to `transaction` and `SERVER_RESET_QUERY` to `DISCARD ALL` by default in the respective deployment manifest.

```bash
$ kubectl create configmap pgbouncer-configs \
    --from-literal=DB_HOST=postgresql-svc.postgres.svc.cluster.local \
    --from-literal=DB_USER=postgres
```

Now we can apply our deployment for [PgBouncer](https://www.pgbouncer.org/) that is based on this [Docker image](https://hub.docker.com/r/edoburu/pgbouncer/) for PgBouncer 1.15.0.

```bash
$ kubectl apply -f pgbouncer.yaml
deployment.apps/pgbouncer created

$ # now we create a service for the deployment
$ kubectl expose deployment pgbouncer --name=pgbouncer-svc
service/pgbouncer exposed
```

Let’s check the server list.

```bash
$ kubectl run -it --rm pg-psql --image=postgres:13.3 --restart=Never \
    --env="PGPASSWORD=tes6Aev8" -- \
    psql -h pgbouncer-svc.postgres.svc.cluster.local -U postgres -d pgbouncer
If you don't see a command prompt, try pressing enter.
pgbouncer=# \x
Expanded display is on.

pgbouncer=# SHOW SERVERS;
-[ RECORD 1 ]+------------------------
type         | S
user         | postgres
database     | postgres
state        | used
addr         | 172.17.0.5
port         | 5432
local_addr   | 172.17.0.6
local_port   | 59960
connect_time | 2021-08-04 11:25:19 UTC
request_time | 2021-08-04 11:25:59 UTC
wait         | 0
wait_us      | 0
close_needed | 0
ptr          | 0x7fa02cb54100
link         |
remote_pid   | 183
tls          |

pgbouncer=# \q
pod "pg-psql" deleted
```

As we can so see PgBouncer only detects one server so far. The reason for that is, each server is listening on the same host and port. We need to fix that.
