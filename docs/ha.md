# MSR4 Installation with High Availability

## Introduction

This document provides a comprehensive guide for installing `MSR4` in a lab environment with High Availability (HA). Future updates may include additional ingress options and advanced configuration settings.

### Prerequisites

Before proceeding with the installation, the following prerequisites must be met:

#### A Kubernets Cluster

An existing `Kubernetes` cluster with at least two Linux worker nodes is required. For this guide, TerraTrain was used to create the cluster.

#### Helm Installation

Helm must be installed and properly configured. If `TerraTrain` is being used, it should be noted that the version of Helm included in the container is outdated. It is recommended to upgrade Helm by running the following script:

```
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

After upgrading, Helm commands should reference the full path to the updated Helm binary.

#### Client Bundle Setup

The client bundle must be configured to execute Helm commands successfully. Instructions for setting up the client bundle can be found in the official documentation.

Note: The initial steps in this guide align closely with the non-HA installation documentation for MSR4, with additional instructions for enabling High Availability.

## Step 1: Add the MSR Helm Repository

```
helm repo add harbor https://registry.mirantis.com/charts/harbor/helm
```

## Step 2: Get the values.yaml file

```
helm show values harbor/harbor > values.yaml
```

## Step Three: Create the K8s Secret with the Keys

Create a folder called "certs"

```
mkdir certs
```

Create a text file called "harbor.conf" in the certs directory and fill it out with the following:

```
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
C = US
ST = State
L = City
O = Organization
OU = Organizational Unit
CN = msr

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
IP.1 = <ip-address-of-workernode>  # Replace with your actual IP address
```

Generate the crt and the key using the harbor.conf file you just created

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -config harbor.conf
```

Create the k8s secret (run from outside of the certs folder):

```
kubectl create secret tls <name of your secret> \
--cert=certs/tls.crt \
--key=certs/tls.key
```

## Step 3: Set up NFS and storageclass

Set Data Persistence method to true in the values.yaml file:

```
persistence:
enabled: true
```

Set up the NFS folder on the NFS Server(note this assumes that NFS server is already installed) :

```
sudo mkdir -p <directory you want to use>
sudo chown nobody:nogroup <directory you want to use>
sudo chmod 777 <directory you want to use>
```

Add directory to the file /etc/exports on the NFS Server  with the following line:

```
</directory/you/want/touse> <node-ip>(rw,sync,no_subtree_check,no_root_squash)
```

Exit the NFS Server and install NFS tools on your manager nodes:
On Ubuntu:

```
sudo apt-get update
sudo apt-get install nfs-common
```

On Redhat, Centos,  or Rocky:

```
sudo yum install nfs-utils
```

Run the following helm commands against your cluster:

```
helm repo add nfs-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
```

Make sure you add your nfs server ip in the appropriate spot in this command:

```
helm install nfs-client-provisioner nfs-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=<nfs-server-ip> \
--set nfs.path=</directory/you/want/touse> \
--set storageClass.name=nfs-storage \
--set storageClass.defaultClass=true
```

You should see a pod start up and if you run:

```
kubectl get storageclass
```

You should see the storageclass you created. Once the NFS pod is healthy you can move onto the next step.

## Step 4: Set up HA Redis

Add the bitnami repo

```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Create a file called redis-values.yaml and populate with the following:

```
replica:
replicaCount: 3
persistence:
enabled: true
size: 8Gi
resources:
requests:
memory: 256Mi
cpu: 100m
limits:
memory: 512Mi
cpu: 200m
```

Install redis via helm:

```
helm install redis bitnami/redis -f redis-values.yaml
```

Get and save the redis password (save it as environmental variable, that way you can recall it if you lose it):

```
export REDIS_PASSWORD=$(kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}" | base64 -d)
echo $REDIS_PASSWORD
```

Get the service name and ip

```
kubectl get service
redis-master                                   ClusterIP   10.96.84.98     <none>        6379/TCP
```

Modify the values.yaml file to set redis to external:

```
redis:
# if external Redis is used, set "type" to "external"
# and fill the connection information in "external" section
type: external
```

Modify the values.yaml file and add the details under redis external:

```
external:
# support redis, redis+sentinel
# addr for redis: <host_redis>:<port_redis>
# addr for redis+sentinel: <host_sentinel1>:<port_sentinel1>,<host_sentinel2>:<port_sentinel2>,<host_sentinel3>:<port_sentinel3>
addr: "<servicename:port>"
# The name of the set of Redis instances to monitor, it must be set to support redis+sentinel
sentinelMasterSet: ""
# The "coreDatabaseIndex" must be "0" as the library Harbor
# used doesn't support configuring it
# harborDatabaseIndex defaults to "0", but it can be configured to "6", this config is optional
# cacheLayerDatabaseIndex defaults to "0", but it can be configured to "7", this config is optional
coreDatabaseIndex: "0"
jobserviceDatabaseIndex: "1"
registryDatabaseIndex: "2"
trivyAdapterIndex: "5"
# harborDatabaseIndex: "6"
# cacheLayerDatabaseIndex: "7"
# username field can be an empty string, and it will be authenticated against the default user
username: ""
password: "<password>"
```

## Step 5: Set up Postgresql

Create and populate postgresql-values.yaml file

```
primary:
persistence:
enabled: true
size: 10Gi
resources:
limits:
cpu: 500m
memory: 512Mi
requests:
cpu: 100m
memory: 256Mi
replica:
replicaCount: 3
replication:
enabled: true
synchronousCommit: "on"
numSynchronousReplicas: 1
```

Install Postgresql via helm

```
helm install postgresql bitnami/postgresql-ha -f postgresql-values.yaml
```

Get Postgresql password

```
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql-postgresql-ha-postgresql -o jsonpath="{.data.password}" | base64 -d)
echo $POSTGRES_PASSWORD
```

Login to postgresql

```
kubectl run postgresql-postgresql-ha-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql-repmgr:16.4.0-debian-12-r12 --env="PGPASSWORD=$POSTGRES_PASSWORD"  \
--command -- psql -h postgresql-postgresql-ha-pgpool -p 5432 -U postgres -d postgres
```

Create Databases

```
CREATE DATABASE harbor_core;
CREATE DATABASE clair;
CREATE DATABASE notary_signer;
CREATE DATABASE notary_server;
```

Find "pool" service and port

```
kubectl get service
postgresql-postgresql-ha-pgpool                ClusterIP   10.96.40.233    <none>        5432/TCP
```

Set postgresql to be external in MSR4 values.yaml

```
database:
# if external database is used, set "type" to "external"
# and fill the connection information in "external" section
type: external
```

Fill out the portion under database external

```
external:
host: "<poolservicename>"
port: "<port>"
username: "postgres"
password: "<password>"
coreDatabase: "harbor_core"
```

## Step 6: Set up the Method of ingress by modifying the values.yaml file

Set the expose type:

```
expose:
# Set how to expose the service. Set the type as "ingress", "clusterIP", "nodePort" or "loadBalancer"
# and fill the information in the corresponding section
type: nodePort
```

Set the certsource to TLS and the Secret name to what you want

```
certSource: secret
secret:
# The name of secret which contains keys named:
# "tls.crt" - the certificate
# "tls.key" - the private key
secretName: "<name of your secret>"
```

set the nodePort ports to allow nodePort ingress

```
nodePort:
# The name of NodePort service
name: harbor
ports:
http:
# The service port Harbor listens on when serving HTTP
port: 80
# The node port Harbor listens on when serving HTTP
nodePort: 32769
https:
# The service port Harbor listens on when serving HTTPS
port: 443
# The node port Harbor listens on when serving HTTPS
nodePort: 32770
```

Use a worker node ip address (the same one that you used in generating the cert):

```
externalURL: <a worker node external IP:httpsnodePort>
```

## Step 7: Modify the Yaml for replication

Set the replica number to at least 2 under portal, registry, core, trivy and jobservice in the values.yaml file

## Step 8: Install via Helm

```
helm install my-release harbor/harbor -f <pathto/values.yaml>
```

## Step 9: Allow docker environment to trust Self Signed Cert

Create directory

```
/etc/docker/certs.d/<ipaddress:nodeport>
```

Move and rename tls.crt

```
mv tls.crt /etc/docker/certs.d/<ipaddress:nodeport>/ca.crt
```

The UI should be accessible at https://<worker-node-external-ip>:32770, provided the same NodePort numbers were used as specified in this guide. You should also be able to log in using Docker.
Default Username: admin
Default Password: Harbor12345
