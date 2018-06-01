# Kubernetes The Hard Way

## Introduction

The goal of this repo is to document and store scripts I use to setup a Kubernetes cluster by following tutorial [Kubernetes The Hardway](https://github.com/kelseyhightower/kubernetes-the-hard-way). The goal of the whole exercise is to be able to setup a Kubernetes cluster from scratch without using additional tools such as Kops, Kubeadm, or Tectonic. The whole exercise will include:
1. Setting up a Kubernetes cluster on Google Cloud Platform using scripts
2. Setting up a Kubernetes cluster on AWS, with its underlying infrastructure provisioned with Terraform and cloud-init script in Terraform
3. Setting up a Kubenrnets cluster on VMs in non-cloud-provider local datacenter

# Setting Up Kubernetes Cluster in GCP

## 1. Prerequisites

- Get `gcloud` SDK installed
- Set compute region `gcloud set compute/region asia-east1`
- Set compute zone `gcloud set compute/zone asia-east-1a`

## 2. Installing Client Tools

There are three main client tools that we need to install:
- `cfssl`
- `cfssljson`
- `kubectl`

Now let's install them.

1. Install `cfssl`  

     In my experience, installing `cfssl` using binary somehow did not work so I used `brew install cfssl` instead. The version installed in my setup is:

     ```
     Version: 1.3.0
     Revision: dev
     Runtime: go1.10
     ```

2. Install `cfssljson`  

     As for `cfssljson`, in my experience, I could not install it using binary from the tutorial as well so I used `go get -u github.com/cloudflare/cfssl/cmd/cfssljson` instead. The version installed in my setup is:

     ```
     Version: 1.3.0
     Revision: dev
     Runtime: go1.10
     ```

3. Install `kubectl`  

     For `kubectl`, I initially have version 1.10.1 installed with Homebrew. Since the latest tutorial uses 1.10.2, I reinstalled using this commands:

     ```
     curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/darwin/amd64/kubectl
     chmod +x kubectl
     sudo mv kubectl /usr/local/bin/
     ```

## 3. Provisioning Compute Resource

### 3.1. Networking

Kubernetes assumes a flat network in which containers can communicate with each other.

#### 3.1.1. Virtual Private Cloud

1. Create the VPC.

     ```
     gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom
     ```

2. Create a subnet named `kubernetes`.

     ```
     gcloud compute networks subnets create kubernetes \
     --network kubernetes-the-hard-way \
     --range 10.240.0.0/24
     ```

#### 3.1.2. Firewall Rules

1. Create a firewall rule that allows internal communication across all protocols.

     ```
     gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
     --allow tcp,udp,icmp \
     --network kubernetes-the-hard-way \
     --source-ranges 10.240.0.0/24,10.200.0.0/16
     ```

2. Create a firewall rule that allows SSH, ICMP, and HTTPS from external.

     ```
     gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
     --allow tcp:22,tcp:6443,icmp \
     --network kubernetes-the-hard-way \
     --source-ranges 0.0.0.0/0
     ```

3. To verify our setup is correct, run `gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"`. The result should look like this:

     ```
     NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY
     kubernetes-the-hard-way-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp
     kubernetes-the-hard-way-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp
     ```


#### 3.1.3. Kubernetes Public IP Address

1. Allocate a static public IP address for our Kubernetes API Servers   
     ```
     gcloud compute addresses create kubernetes-the-hard-way \
     --region $(gcloud config get-value compute/region)
     ```

2. To verify, run:

     ```
     gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
     ```
     
     The result should look like this:

     ```
     NAME                     REGION      ADDRESS      STATUS
     kubernetes-the-hard-way  asia-east1  XX.XXX.X.XX  RESERVED
     ```

### 3.2. Compute Instances

We are going to provision three compute instances for Kubernetes controllers and two compute instances for Kubernetes workers.

1. Provision Kubernetes Controllers

     ```
     for i in 0 1 2; do
       gcloud compute instances create controller-${i} \
       --async \
       --boot-disk-size 200GB \
       --can-ip-forward \
       --image-family ubuntu-1604-lts \
       --image-project ubuntu-os-cloud \
       --machine-type n1-standard-1 \
       --private-network-ip 10.240.0.1${i} \
       --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
       --subnet kubernetes \
       --tags kubernetes-the-hard-way,controller
     done
     ```

2. Provision Kubernetes Workers  
 
     ```
     for i in 0 1 2; do
       gcloud compute instances create worker-${i} \
       --async \
       --boot-disk-size 200GB \
       --can-ip-forward \
       --image-family ubuntu-1604-lts \
       --image-project ubuntu-os-cloud \
       --machine-type n1-standard-1 \
       --metadata pod-cidr=10.200.${i}.0/24 \
       --private-network-ip 10.240.0.2${i} \
       --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
       --subnet kubernetes \
       --tags kubernetes-the-hard-way,worker
     done
     ```

3. To verify, run `gcloud compute instances list`. The result should look like this:

     ```
     NAME          ZONE          MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
     controller-0  asia-east1-a  n1-standard-1               10.240.0.10  35.234.32.234   RUNNING
     controller-1  asia-east1-a  n1-standard-1               10.240.0.11  35.234.20.221   RUNNING
     controller-2  asia-east1-a  n1-standard-1               10.240.0.12  35.234.55.208   RUNNING
     worker-0      asia-east1-a  n1-standard-1               10.240.0.20  35.234.50.172   RUNNING
     worker-1      asia-east1-a  n1-standard-1               10.240.0.21  35.194.229.3    RUNNING
     worker-2      asia-east1-a  n1-standard-1               10.240.0.22  35.194.152.171  RUNNING
     ```
