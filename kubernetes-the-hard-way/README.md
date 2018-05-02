# Kubernetes The Hard Way

## Introduction

The goal of this repo is to document and store scripts I use to setup a Kubernetes cluster by following tutorial [Kubernetes The Hardway](https://github.com/kelseyhightower/kubernetes-the-hard-way). The goal of the whole exercise is to be able to setup a Kubernetes cluster from scratch without using additional tools such as Kops, Kubeadm, or Tectonic. The whole exercise will include:
1. Setting up a Kubernetes cluster on Google Cloud Platform using scripts
2. Setting up a Kubernetes cluster on AWS, with its underlying infrastructure provisioned with Terraform and its configuration scripts performed with Chef
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
   For `kubectl`, I used `brew install kubectl` command to install it. The version installed in my setup is (the result of `kubectl version --client`):

```
Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.1", GitCommit:"d4ab47518836c750f9949b9e0d387f20fb92260b", GitTreeState:"clean", BuildDate:"2018-04-13T22:27:55Z", GoVersion:"go1.9.5", Compiler:"gc", Platform:"darwin/amd64"}
```

## 3. Provisioning Compute Resource

### 3.1 Networking

Kubernetes assumes a flat network in which containers can communicate with each other.

#### 3.1.1. Virtual Private Cloud

1. Create the VPC  
   `gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom`

2. Create a subnet named `kubernetes`  

```
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24
```

#### 3.1.2. Firewall Rules

1. Create a firewall rule that allows internal communication across all protocols

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```

2. Create a firewall rule that allows SSH, ICMP, and HTTPS from external

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


