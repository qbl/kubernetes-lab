# Kubernetes The Hard Way

## 0. Intro

The goal of this repo is to document and store scripts I use to setup a Kubernetes cluster by following tutorial [Kubernetes The Hardway](https://github.com/kelseyhightower/kubernetes-the-hard-way). The goal of the whole exercise is to be able to setup a Kubernetes cluster from scratch without using additional tools such as Kops, Kubeadm, or Tectonic. The whole exercise will include:
1. Setting up a Kubernetes cluster on Google Cloud Platform using scripts
2. Setting up a Kubernetes cluster on AWS, with its underlying infrastructure provisioned with Terraform and its configuration scripts performed with Chef
3. Setting up a Kubenrnets cluster on VMs in non-cloud-provider local datacenter

## 1. Setting Up Kubernetes Cluster in GCP

### 1.1. Prerequisites

1. Get `gcloud` SDK installed
2. Set compute region `gcloud set compute/region asia-east1`
3. Set compute zone `gcloud set compute/zone asia-east-1a`

### 1.2. Installing Client Tools

There are three main client tools that we need to install:
- `cfssl`
- `cfssljson`
- `kubectl`

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


