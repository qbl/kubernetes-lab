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

## 4. Provisioning a Certificate Authority and Generating TLS Certificates

### 4.1. Certificate Authority (CA)

1. Generate CA configuration file, certificate, and private key.

     Generate ca-config.json file:

     ```
     cat > ca-config.json <<EOF
     {
       "signing": {
         "default": {
           "expiry": "8760h"
         },
         "profiles": {
           "kubernetes": {
             "usages": ["signing", "key encipherment", "server auth", "client auth"],
             "expiry": "8760h"
           }
         }
       }
     }
     EOF
     ```

     Generate ca-csr.json file:

     ```
     cat > ca-csr.json <<EOF
     {
       "CN": "Kubernetes",
       "key": {
         "algo": "rsa",
         "size": 2048
       },
       "names": [
         {
           "C": "ID",
           "L": "Jakarta",
           "O": "Kubernetes",
           "OU": "CA",
           "ST": "DKI Jakarta"
         }
       ]
     }
     EOF
     ```

     Generate the actual key:

     ```
     cfssl gencert -initca ca-csr.json | cfssljson -bare ca
     ```

### 4.1. Client and Server Certificates

1. Generate the admin client certificate and private key.

     Generate admin-csr.json file:

     ```
     cat > admin-csr.json <<EOF
     {
       "CN": "admin",
       "key": {
         "algo": "rsa",
         "size": 2048
       },
       "names": [
         {
           "C": "ID",
           "L": "Jakarta",
           "O": "system:masters",
           "OU": "Kubernetes The Hard Way",
           "ST": "DKI Jakarta"
         }
       ]
     }
     EOF
     ```
     
     Generate the actual key:

     ```
     cfssl gencert \
       -ca=ca.pem \
       -ca-key=ca-key.pem \
       -config=ca-config.json \
       -profile=kubernetes \
       admin-csr.json | cfssljson -bare admin
     ```

### 4.2. Kubelet Client Certificates

1. Generate a certificate and private key for each Kubernetes worker node.

     ```
     for instance in worker-0 worker-1 worker-2; do
     cat > ${instance}-csr.json <<EOF
     {
       "CN": "system:node:${instance}",
       "key": {
         "algo": "rsa",
         "size": 2048
       },
       "names": [
         {
           "C": "ID",
           "L": "Jakarta",
           "O": "system:nodes",
           "OU": "Kubernetes The Hard Way",
           "ST": "DKI Jakarta"
         }
       ]
     }
     EOF
     
     EXTERNAL_IP=$(gcloud compute instances describe ${instance} \
       --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')
     
     INTERNAL_IP=$(gcloud compute instances describe ${instance} \
       --format 'value(networkInterfaces[0].networkIP)')
     
     cfssl gencert \
       -ca=ca.pem \
       -ca-key=ca-key.pem \
       -config=ca-config.json \
       -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
       -profile=kubernetes \
       ${instance}-csr.json | cfssljson -bare ${instance}
     done
     ```

     Results:
     
     ```
     worker-0-key.pem
     worker-0.pem
     worker-1-key.pem
     worker-1.pem
     worker-2-key.pem
     worker-2.pem
     ```

### 4.3. The Controller Manager Client Certificate

1. Generate the `kube-controller-manager` client certificate and private key.

     Generate kube-controller-manager-csr.json file:

     ```
     cat > kube-controller-manager-csr.json <<EOF
     {
       "CN": "system:kube-controller-manager",
       "key": {
         "algo": "rsa",
         "size": 2048
       },
       "names": [
         {
           "C": "ID",
           "L": "Jakarta",
           "O": "system:kube-controller-manager",
           "OU": "Kubernetes The Hard Way",
           "ST": "DKI Jakarta"
         }
       ]
     }
     EOF
     ```

     Generate the actual key:

     ```
     cfssl gencert \
       -ca=ca.pem \
       -ca-key=ca-key.pem \
       -config=ca-config.json \
       -profile=kubernetes \
       kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
     ```

### 4.4. The Kube Proxy Client Certificate

1. Generate the `kube-proxy` client and private key.

     Generate kube-proxy-csr.json file:

     ```
     cat > kube-proxy-csr.json <<EOF
     {
       "CN": "system:kube-proxy",
       "key": {
         "algo": "rsa",
         "size": 2048
       },
       "names": [
         {
           "C": "ID",
           "L": "Jakarta",
           "O": "system:node-proxier",
           "OU": "Kubernetes The Hard Way",
           "ST": "DKI Jakarta"
         }
       ]
     }
     EOF
     ```

     Generate the actual key:

     ```
     cfssl gencert \
       -ca=ca.pem \
       -ca-key=ca-key.pem \
       -config=ca-config.json \
       -profile=kubernetes \
       kube-proxy-csr.json | cfssljson -bare kube-proxy
     ```

### 4.5. The Scheduler Client Certificate

1. Generate the `kube-scheduler` client certificate and private key:

     Generate kube-scheduler-csr.json file:

     ```
     cat > kube-scheduler-csr.json <<EOF
     {
       "CN": "system:kube-scheduler",
       "key": {
         "algo": "rsa",
         "size": 2048
       },
       "names": [
         {
           "C": "ID",
           "L": "Jakarta",
           "O": "system:kube-scheduler",
           "OU": "Kubernetes The Hard Way",
           "ST": "DKI Jakarta"
         }
       ]
     }
     EOF
     ```

     Generate the actual key:

     ```
     cfssl gencert \
       -ca=ca.pem \
       -ca-key=ca-key.pem \
       -config=ca-config.json \
       -profile=kubernetes \
       kube-scheduler-csr.json | cfssljson -bare kube-scheduler
     ```

### 4.6. The Kubernetes API Server Certificate

1. Generate the Kubernetes API Server certificate and private key.

     Export Kubernetes public address:

     ```
     KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
       --region $(gcloud config get-value compute/region) \
       --format 'value(address)')
     ```

     Generate kubernetes-csr.json file:

     ```
     cat > kubernetes-csr.json <<EOF
     {
       "CN": "kubernetes",
       "key": {
         "algo": "rsa",
         "size": 2048
       },
       "names": [
         {
           "C": "ID",
           "L": "Jakarta",
           "O": "Kubernetes",
           "OU": "Kubernetes The Hard Way",
           "ST": "DKI Jakarta"
         }
       ]
     }
     EOF
     ```

     Generate the actual key:

     ```
     cfssl gencert \
       -ca=ca.pem \
       -ca-key=ca-key.pem \
       -config=ca-config.json \
       -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,$     {KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
       -profile=kubernetes \
       kubernetes-csr.json | cfssljson -bare kubernetes
     ```

### 4.7. The Service Account Key Pair

1. Generate the Service Account Key Pair certificate and private key.

     Generate service-account-csr.json file:

     ```
     cat > service-account-csr.json <<EOF
     {
       "CN": "service-accounts",
       "key": {
         "algo": "rsa",
         "size": 2048
       },
       "names": [
         {
           "C": "ID",
           "L": "Jakarta",
           "O": "Kubernetes",
           "OU": "Kubernetes The Hard Way",
           "ST": "DKI Jakarta"
         }
       ]
     }
     EOF
     ```

     Generate the actual key:

     ```
     cfssl gencert \
       -ca=ca.pem \
       -ca-key=ca-key.pem \
       -config=ca-config.json \
       -profile=kubernetes \
       service-account-csr.json | cfssljson -bare service-account
     ```

### 4.8. Distribute the Client and Server Certificates

1. Copy the appropriate certificates and private keys to each worker instance.

     ```
     for instance in worker-0 worker-1 worker-2; do
       gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
     done
     ```

2. Copy the appropriate certificates and private keys to each controller instance.

     ```
     for instance in controller-0 controller-1 controller-2; do
       gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem      kubernetes.pem \
         service-account-key.pem service-account.pem ${instance}:~/
     done
     ```

## 5. Generating Kubernetes Configuration Files for Authentications

### 5.1. Client Authentication Configs

1. Export Kubernetes public IP address.

     ```
     KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe      kubernetes-the-hard-way \
       --region $(gcloud config get-value compute/region) \
       --format 'value(address)')
     ```

2. Generate `kubelet` Kubernetes configuration files.

     In the same directory where we store our certificate files, run:

     ```
     for instance in worker-0 worker-1 worker-2; do
       kubectl config set-cluster kubernetes-the-hard-way \
         --certificate-authority=ca.pem \
         --embed-certs=true \
         --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
         --kubeconfig=${instance}.kubeconfig
     
       kubectl config set-credentials system:node:${instance} \
         --client-certificate=${instance}.pem \
         --client-key=${instance}-key.pem \
         --embed-certs=true \
         --kubeconfig=${instance}.kubeconfig
     
       kubectl config set-context default \
         --cluster=kubernetes-the-hard-way \
         --user=system:node:${instance} \
         --kubeconfig=${instance}.kubeconfig
     
       kubectl config use-context default --kubeconfig=${instance}     .kubeconfig
     done
     ```

3. Generate `kube-proxy` Kubernetes configuration files.

     In the same directory where we store our certificate files, run:

     ```
     kubectl config set-cluster kubernetes-the-hard-way \
       --certificate-authority=ca.pem \
       --embed-certs=true \
       --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
       --kubeconfig=kube-proxy.kubeconfig
     
     kubectl config set-credentials system:kube-proxy \
       --client-certificate=kube-proxy.pem \
       --client-key=kube-proxy-key.pem \
       --embed-certs=true \
       --kubeconfig=kube-proxy.kubeconfig
     
     kubectl config set-context default \
       --cluster=kubernetes-the-hard-way \
       --user=system:kube-proxy \
       --kubeconfig=kube-proxy.kubeconfig
     
     kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
     ```

4. Generate `kube-controller-manager` Kubernetes configuration files.

     In the same directory where we store our credentials files, run:

     ```
     kubectl config set-cluster kubernetes-the-hard-way \
       --certificate-authority=ca.pem \
       --embed-certs=true \
       --server=https://127.0.0.1:6443 \
       --kubeconfig=kube-controller-manager.kubeconfig
   
     kubectl config set-credentials system:kube-controller-manager \
       --client-certificate=kube-controller-manager.pem \
       --client-key=kube-controller-manager-key.pem \
       --embed-certs=true \
       --kubeconfig=kube-controller-manager.kubeconfig
   
     kubectl config set-context default \
       --cluster=kubernetes-the-hard-way \
       --user=system:kube-controller-manager \
       --kubeconfig=kube-controller-manager.kubeconfig
   
     kubectl config use-context default    --kubeconfig=kube-controller-manager.kubeconfig
     ```

5. Generate `kube-scheduler` Kubernetes configuration files.

     In the same directory where we store our credentials files, run:

     ```
     kubectl config set-cluster kubernetes-the-hard-way \
       --certificate-authority=ca.pem \
       --embed-certs=true \
       --server=https://127.0.0.1:6443 \
       --kubeconfig=kube-scheduler.kubeconfig
     
     kubectl config set-credentials system:kube-scheduler \
       --client-certificate=kube-scheduler.pem \
       --client-key=kube-scheduler-key.pem \
       --embed-certs=true \
       --kubeconfig=kube-scheduler.kubeconfig
     
     kubectl config set-context default \
       --cluster=kubernetes-the-hard-way \
       --user=system:kube-scheduler \
       --kubeconfig=kube-scheduler.kubeconfig
     
     kubectl config use-context default      --kubeconfig=kube-scheduler.kubeconfig
     ```

6. Generate `admin` Kubernetes configuration files.

     In the same directory where we store our credentials files, run:

     ```
     kubectl config set-cluster kubernetes-the-hard-way \
       --certificate-authority=ca.pem \
       --embed-certs=true \
       --server=https://127.0.0.1:6443 \
       --kubeconfig=admin.kubeconfig
     
     kubectl config set-credentials admin \
       --client-certificate=admin.pem \
       --client-key=admin-key.pem \
       --embed-certs=true \
       --kubeconfig=admin.kubeconfig
     
     kubectl config set-context default \
       --cluster=kubernetes-the-hard-way \
       --user=admin \
       --kubeconfig=admin.kubeconfig
     
     kubectl config use-context default --kubeconfig=admin.kubeconfig
     ```

### 5.2. Distribute The Kubernetes Configuration Files

1. Copy the appropriate `kubelet` and `kube-proxy` kubeconfig files to each worker instance:

     ```
     for instance in worker-0 worker-1 worker-2; do
       gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
     done
     ```

2. Copy the appropriate `kube-controller-manager` and `kube-scheduler` kubeconfig files to each controller instance:

     ```
     for instance in controller-0 controller-1 controller-2; do
       gcloud compute scp admin.kubeconfig      kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
     done
     ```
