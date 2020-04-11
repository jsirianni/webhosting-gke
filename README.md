# webhosting-gke

- [webhosting-gke](#webhosting-gke)
  * [About](#about)
  * [Goals](#goals)
  * [Architecture](#architecture)
  * [Usage](#usage)
    + [Deploy Infrastructure](#deploy-infrastructure)
    + [Deploy Kubernetes Services](#deploy-kubernetes-services)
      - [Ingress](#ingress)
      - [LetsEncrypt HTTPS Certificates](#letsencrypt-https-certificates)
      - [NFS Server](#nfs-server)
      - [NFS Provisioner](#nfs-provisioner)
    + [Create MySQL User, Password, Database](#create-mysql-user--password--database)
    + [WordPress](#wordpress)
    + [Verify](#verify)
    + [Test](#test)
  * [Notes:](#notes-)
    + [Production Concerns](#production-concerns)

## About

Reference implementation for hosting scalable WordPress on GKE with minimal dependencies.
This is just an example, carries no warranty, and should not be used for production.
Please do your own due diligence and research. See the Notes section.

This project purposely ignores the existence of several useful tools in order to
show what is really going on behind the scenes:
- [Helm](https://helm.sh/)
- [Kustomize](https://kubernetes.io/blog/2018/05/29/introducing-kustomize-template-free-configuration-customization-for-kubernetes/)
- [Terraform](https://www.terraform.io/)

## Goals

The goal of this project is to provide a reference architecture for deploying
scalable and reliable WordPress on GKE. This is accomplished by leveraging the
following technologies:
- Network TCP Loadbalancer
- Google Kubernetes Engine
- Google Compute Persistent Disk
- Google CloudSQL
- LetsEncrypt for free HTTPS certificates

To make this example as cheep as possible, the smallest instance types possible are used
- db-f1-micro
- e2-small
- preemptive compute instances
- zonal GKE master

## Architecture

![Architecture](assets/arch_v1.png?raw=true)

## Usage

### Deploy Infrastructure

1. Create Project
2. Enable Billing or [free trial](https://cloud.google.com/free)
3. Configure [gcloud cli](https://cloud.google.com/sdk/gcloud)
```
gcloud auth login
gcloud config set project <project id>
gcloud components update
gcloud components install beta
```
4. Enable required APIs
```
gcloud services enable \
    container.googleapis.com \
    compute.googleapis.com \
    sql-component.googleapis.com \
    sqladmin.googleapis.com \
    servicenetworking.googleapis.com
```

5. Deploy [CloudSQL using the GCP web interface](https://cloud.google.com/sql/docs/mysql/quickstart)
  - Mysql 5.7
  - random root password
  - us-central1-c
  - instance type: db-f1-micro

6. Deploy GKE:
  - Kubernetes version 1.15.x
  - single zone for [free tier](https://cloud.google.com/kubernetes-engine/pricing)
  - [preemptive instance group](https://cloud.google.com/compute/docs/instances/preemptible)
  - e2-small [instance types](https://cloud.google.com/compute/docs/machine-types#e2_machine_types)
  - node autoscaling from 1 to 3 instances
  - node labels: purpose=wordpress
  - disable HTTP loadbalancing (we will use TCP loadbalancer with Nginx Ingress to keep costs down)
  - disable monitoring and logging (keeps resource usage and costs down. Production systems should have this enabled)
  - default network and subnet is used
  - auto upgrade is disabled
```
PROJECT=<project id>
```
```
gcloud beta container \
    --project "${PROJECT}" \
    clusters create "wordpress" \
        --zone "us-central1-c" \
        --no-enable-basic-auth \
        --cluster-version "1.15.11-gke.5" \
        --machine-type "e2-small" \
        --image-type "COS" \
        --disk-type "pd-standard" \
        --disk-size "30" \
        --node-labels purpose=wordpress \
        --metadata disable-legacy-endpoints=true \
        --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
        --max-pods-per-node "110" \
        --preemptible \
        --num-nodes "1" \
        --no-enable-stackdriver-kubernetes \
        --enable-ip-alias \
        --network "projects/${PROJECT}/global/networks/default" \
        --subnetwork "projects/${PROJECT}/regions/us-central1/subnetworks/default" \
        --default-max-pods-per-node "110" \
        --enable-autoscaling \
        --min-nodes "1" \
        --max-nodes "3" \
        --no-enable-master-authorized-networks \
        --addons HorizontalPodAutoscaling \
        --no-enable-autoupgrade \
        --enable-autorepair \
        --max-surge-upgrade 1 \
        --max-unavailable-upgrade 0
```

7. Add the NFS node pool. A dedicated node pool (single node) is used to host the
NFS server for the cluster. This pool does not use preemptive instances, therefore
it is suitable for hosting a service that requires1 100% uptime
```
gcloud beta container \
    --project "${PROJECT}" \
    node-pools create "nfs" \
        --cluster "wordpress" \
        --zone "us-central1-c" \
        --node-version "1.15.11-gke.5" \
        --machine-type "e2-small" \
        --image-type "COS" \
        --disk-type "pd-standard" \
        --disk-size "30" \
        --node-labels purpose=nfs \
        --metadata disable-legacy-endpoints=true \
        --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
        --num-nodes "1" \
        --no-enable-autoupgrade \
        --no-enable-autorepair
```

8. Create the NFS server's disk. A dedicated Persistent Disk is used for the NFS server.
This is created outside of Kubernetes in order to guarantee that the disk is not
accidentally deleted when deleting the NFS server from the cluster
```
gcloud beta compute disks create wordpress \
    --project=${PROJECT} \
    --type=pd-standard \
    --size=50GB \
    --zone=us-central1-c \
    --physical-block-size=4096
```

### Deploy Kubernetes Services

**Connect to the cluster**
```
PROJECT=<project id>
gcloud container clusters get-credentials wordpress \
    --zone us-central1-c \
    --project $PROJECT
```

#### Ingress

[nginx ingress](https://kubernetes.github.io/ingress-nginx/) is used for proxying
incoming HTTP and HTTPS requests. This allows for the use of a TCP load balancer
instead of the GCP HTTP load balancer. HTTP load balancers have a charge for each
hosted domain, therefore, the pricing does not scale well. Nginx allows the cluster
to host any number of domains without an increase in load balancer costs.

**Deploy**
```
kubectl apply -f config/ingress/mandatory.yaml
kubectl apply -f config/ingress/cloud-generic.yaml
```

Once deployed, a loadbalancer will be created for you, with healthchecks.

![Load Balancer](assets/loadbalancer.png?raw=true)

**Explanation:**
- The Nginx service creates a Google LoadBalancer, which forwards traffic to the node pool(s)
- The LoadBalancer has a public ip address
- The Nginx deployment has 1 pod, but will scale up to 3 pods when CPU usage is high

#### LetsEncrypt HTTPS Certificates

[LetsEncrypt](https://letsencrypt.org/) is a certificate authority that is free to use.
Cert Manager is used to issue certificates dynamically. We will use the "staging" LetsEncrypt
issuer because the example site does not have a DNS record. Production sites should use
the staging issuer to start, and then the production issuer once DNS is configured with
your DNS provider.

**Set Email**
- Add your email to the configuration
```
vim config/cert-manager/staging_issuer.yaml
```

**Deploy Cert Manager**
```
kubectl apply -f config/cert-manager/cert-manager.yaml
kubectl apply -f config/cert-manager/staging_issuer.yaml
```

Cert Manager handles requesting certificates. Cert Manager detects your WordPress
deployment in realtime, allowing certificates to be requested during deployment.

#### NFS Server

The NFS server will only run on nodes with label `purpose=nfs`. These nodes are
not preemptive, which improves the uptime of this critical service.

**Deploy**
```
kubectl apply -f config/nfs/nfs.yaml
```

In the GCE console, check that the disk is mounted to the NFS worker node.

#### NFS Provisioner

An [NFS provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)
is a service that handles creating and archiving NFS volumes for any deployment
using the `managed-nfs-storage` storage class
```
metadata:
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
```

**Deploy**
```
kubectl apply -f config/nfs/provisioner.yaml
```

### Create MySQL User, Password, Database

Each WordPress deployment will need its own user, password, and database. CloudSQL
does not have the ability to create user's with specific permissions, therefore we
must use the Mysql client

connect (this requires a valid mysql client installed on your workstation)
```
gcloud sql connect <instance id>
```

create database, user, password, privileges
```
CREATE DATABASE example;
GRANT ALL PRIVILEGES ON example.* TO 'example' IDENTIFIED BY 'password';
```

store the password and mysql address as a secret
```
kubectl create secret generic mysql-address --from-literal=address=<ip address>
kubectl create secret generic mysql-example --from-literal=password=<password>
```

Please use a secure password, not `password`

### WordPress

The WordPress deployment consists of two parts. A volume and a deployment. The volume should be created
once and never destroyed. If it is deleted, you can find its contents archived on the NFS server
due to the option `archiveOnDelete: "true"` in the provisioner configuration.

Create the example site's volume
```
kubectl create -f config/wordpress/volume/volume.yaml
```

Deploy the site
```
kubectl apply -f config/wordpress/wordpress.yaml
```

### Verify

At this point, all required services should be in a running state
- nfs-server
- nfs-client-provisioner
- example (wordpress)

```
kubectl get all && kubectl get all -n ingress-nginx
```
```
NAME                                          READY   STATUS    RESTARTS   AGE
pod/cm-acme-http-solver-dmnc5                 1/1     Running   0          19m
pod/cm-acme-http-solver-z9ctv                 1/1     Running   0          19m
pod/example-6b995854b8-5fgkq                  1/1     Running   0          12m
pod/example-6b995854b8-ptfxb                  1/1     Running   0          12m
pod/nfs-client-provisioner-7ddf48748c-z4m85   1/1     Running   0          49m
pod/nfs-server-bf5dbb45d-58ln4                1/1     Running   0          75m

NAME                                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
service/cm-acme-http-solver-hz6zv   NodePort    10.24.6.231   <none>        8089:30783/TCP               19m
service/cm-acme-http-solver-zwl7h   NodePort    10.24.15.6    <none>        8089:30736/TCP               19m
service/example                     ClusterIP   10.24.3.229   <none>        80/TCP                       19m
service/kubernetes                  ClusterIP   10.24.0.1     <none>        443/TCP                      104m
service/nfs-server                  ClusterIP   10.24.3.146   <none>        2049/TCP,20048/TCP,111/TCP   75m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/example                  2/2     2            2           19m
deployment.apps/nfs-client-provisioner   1/1     1            1           49m
deployment.apps/nfs-server               1/1     1            1           75m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/example-6b995854b8                  2         2         2       12m
replicaset.apps/nfs-client-provisioner-7ddf48748c   1         1         1       49m
replicaset.apps/nfs-server-bf5dbb45d                1         1         1       75m

NAME                                          REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/example   Deployment/example   1%/70%    2         5         2          19m

NAME                                            READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-75cfcf7f77-gn865   1/1     Running   0          100m

NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                      AGE
service/ingress-nginx   LoadBalancer   10.24.7.255   35.192.18.106   80:32569/TCP,443:32363/TCP   100m

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress-controller   1/1     1            1           100m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-ingress-controller-75cfcf7f77   1         1         1       100m

NAME                                                           REFERENCE                             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/nginx-ingress-controller   Deployment/nginx-ingress-controller   8%/70%    1         3         1          100m
```

### Test

Edit your [hosts file](https://www.siteground.com/kb/how_to_use_the_hosts_file/)
to point `example.com` to your load balancer ip address
```
35.192.18.106 example.com
```

Browse to the site `https://example.com`, accept the certificate error as this certificate
was generated with the "staging / test" issuer provided by LetsEncrypt

![WordPress](assets/wordpress.png?raw=true)


## Notes:

This example does not take advantage of several factors that you may find important

### Production Concerns
- Nginx Ingress starts with a single pod, a production environment should probably start with 2 or 3 pods
- Nginx CPU / MEM limit and request is set very low
- GKE master is in a single zone, production should use multi zone
- CloudSQL and Compute Engine instance types are the smallest possible, and not suitable for production environments
- CloudSQL and NFS server use "hdd" storage type, a production instance should use an SSD
- CloudSQL uses a public address, [you should take the extra steps required to use a Private Address](https://cloud.google.com/sql/docs/mysql/private-ip)
- Infrastructure is deployed using gcloud cli, a production environment should consider using something like [Terraform](https://www.terraform.io/)
- Everything uses the `default` [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/). This may not be suitable for multi tenant or secure environments
- NFS is a pod in he cluster. Production environment should consider several alternatives:
  - Upgrading or replacing the NFS worker node will cause brief downtime for all wordpress sites
  - Use a dedicated NFS server on GCE
  - Use [FileStore](https://cloud.google.com/filestore), fairly expensive
- Backup, replication, and snapshots should be leveraged with the NFS server's persistent disk
