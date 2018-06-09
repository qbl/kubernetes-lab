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

## 6. Generating The Data Encryption Config and Key

1. Generate an encryption key:

     ```
     ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
     ```

2. Create the `encryption-config.yaml` file:

     ```
     cat > encryption-config.yaml <<EOF
     kind: EncryptionConfig
     apiVersion: v1
     resources:
       - resources:
           - secrets
         providers:
           - aescbc:
               keys:
                 - name: key1
                   secret: ${ENCRYPTION_KEY}
           - identity: {}
     EOF
     ```

3. Copy the `encryption-config.yaml` file to each controller instance:

     ```
     for instance in controller-0 controller-1 controller-2; do
       gcloud compute scp encryption-config.yaml ${instance}:~/
     done
     ```

## 7. Bootstrapping The Etcd Cluster

1. `ssh` into each controller:

     ```
     gcloud compute ssh controller-0
     gcloud compute ssh controller-1
     gcloud compute ssh controller-2
     ```

     The next steps should be performed in each controller.

2. Download `etcd` binary:

     ```
     wget -q --show-progress --https-only --timestamping \
       "https://github.com/coreos/etcd/releases/download/v3.3.5/etcd-v3.3.5-linux-amd64.tar.gz"
     ```

3. Extract and install `etcd` server and `etcdctl` command line utility:

     ```
     tar -xvf etcd-v3.3.5-linux-amd64.tar.gz
     sudo mv etcd-v3.3.5-linux-amd64/etcd* /usr/local/bin/
     ```

4. Configure `etcd` server.

     Create `etcd` directory and copy certificate files to that directory:

     ```
     sudo mkdir -p /etc/etcd /var/lib/etcd
     sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/     
     ```

     Export required environment variables:

     ```
     INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
       http://metadata.google. internal/computeMetadata/v1/instance/network-interfaces/0/ip)
     ETCD_NAME=$(hostname -s)
     ```

     Create `etcd.service` systemd unit file:

     ```
     cat <<EOF | sudo tee /etc/systemd/system/etcd.service
     [Unit]
     Description=etcd
     Documentation=https://github.com/coreos
     
     [Service]
     ExecStart=/usr/local/bin/etcd \\
       --name ${ETCD_NAME} \\
       --cert-file=/etc/etcd/kubernetes.pem \\
       --key-file=/etc/etcd/kubernetes-key.pem \\
       --peer-cert-file=/etc/etcd/kubernetes.pem \\
       --peer-key-file=/etc/etcd/kubernetes-key.pem \\
       --trusted-ca-file=/etc/etcd/ca.pem \\
       --peer-trusted-ca-file=/etc/etcd/ca.pem \\
       --peer-client-cert-auth \\
       --client-cert-auth \\
       --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
       --listen-peer-urls https://${INTERNAL_IP}:2380 \\
       --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
       --advertise-client-urls https://${INTERNAL_IP}:2379 \\
       --initial-cluster-token etcd-cluster-0 \\
       --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
       --initial-cluster-state new \\
       --data-dir=/var/lib/etcd
     Restart=on-failure
     RestartSec=5
     
     [Install]
     WantedBy=multi-user.target
     EOF
     ```

5. Start `etcd` server.

     ```
     sudo systemctl daemon-reload
     sudo systemctl enable etcd
     sudo systemctl start etcd
     ```

6. To verify that our etcd cluster is up and running:

     ```
     sudo ETCDCTL_API=3 etcdctl member list \
       --endpoints=https://127.0.0.1:2379 \
       --cacert=/etc/etcd/ca.pem \
       --cert=/etc/etcd/kubernetes.pem \
       --key=/etc/etcd/kubernetes-key.pem
     ```

## 8. Bootstrapping The Kubernetes Control Plane

### 8.1. Preparation

1. `ssh` into each controller:

     ```
     gcloud compute ssh controller-0
     gcloud compute ssh controller-1
     gcloud compute ssh controller-2
     ```

     The next steps should be performed in each controller.

2. Create Kubernetes configuration directory:

     ```
     sudo mkdir -p /etc/kubernetes/config
     ```

### 8.2. Download and Install Kubernetes Controller Binaries

1. Download Kubernetes binaries:

     ```
     wget -q --show-progress --https-only --timestamping \
       "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kube-apiserver" \
       "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kube-controller-manager" \
       "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kube-scheduler" \
       "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kubectl"
     ```

2. Install Kubernetes binaries:

     ```
     chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
     sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
     ```

### 8.3. Configure The Kubernetes API Server

1. Move certificate files:

     ```
     sudo mkdir -p /var/lib/kubernetes/
     sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
       service-account-key.pem service-account.pem \
       encryption-config.yaml /var/lib/kubernetes/
     ```

2. Export internal IP address:

     ```
     INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
       http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
     ```

3. Create `kube-apiserver.service` systemd unit file:

     ```
     cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
     [Unit]
     Description=Kubernetes API Server
     Documentation=https://github.com/kubernetes/kubernetes
     
     [Service]
     ExecStart=/usr/local/bin/kube-apiserver \\
       --advertise-address=${INTERNAL_IP} \\
       --allow-privileged=true \\
       --apiserver-count=3 \\
       --audit-log-maxage=30 \\
       --audit-log-maxbackup=3 \\
       --audit-log-maxsize=100 \\
       --audit-log-path=/var/log/audit.log \\
       --authorization-mode=Node,RBAC \\
       --bind-address=0.0.0.0 \\
       --client-ca-file=/var/lib/kubernetes/ca.pem \\
       --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
       --enable-swagger-ui=true \\
       --etcd-cafile=/var/lib/kubernetes/ca.pem \\
       --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
       --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
       --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
       --event-ttl=1h \\
       --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
       --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
       --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
       --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
       --kubelet-https=true \\
       --runtime-config=api/all \\
       --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
       --service-cluster-ip-range=10.32.0.0/24 \\
       --service-node-port-range=30000-32767 \\
       --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
       --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
       --v=2
     Restart=on-failure
     RestartSec=5
     
     [Install]
     WantedBy=multi-user.target
     EOF
     ```

### 8.4. Configure The Kubernetes Controller Manager

1. Move `kube-controller-manager` kubeconfig into place:

     ```
     sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
     ```

2. Create `kube-controller-manager.service` systemd unit file:

     ```
     cat <<EOF | sudo tee      /etc/systemd/system/kube-controller-manager.service
     [Unit]
     Description=Kubernetes Controller Manager
     Documentation=https://github.com/kubernetes/kubernetes
     
     [Service]
     ExecStart=/usr/local/bin/kube-controller-manager \\
       --address=0.0.0.0 \\
       --cluster-cidr=10.200.0.0/16 \\
       --cluster-name=kubernetes \\
       --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
       --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
       --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
       --leader-elect=true \\
       --root-ca-file=/var/lib/kubernetes/ca.pem \\
       --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
       --service-cluster-ip-range=10.32.0.0/24 \\
       --use-service-account-credentials=true \\
       --v=2
     Restart=on-failure
     RestartSec=5
     
     [Install]
     WantedBy=multi-user.target
     EOF
     ```

### 8.5. Configure The Kubernetes Scheduler

1. Move `kube-scheduler` kubeconfig file into place:

     ```
     sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
     ```

2. Create `kube-scheduler.yaml` configuration file:

     ```
     cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
     apiVersion: componentconfig/v1alpha1
     kind: KubeSchedulerConfiguration
     clientConnection:
       kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
     leaderElection:
       leaderElect: true
     EOF
     ```

3. Create `kube-scheduler.service` systemd unit file:

     ```
     cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
     [Unit]
     Description=Kubernetes Scheduler
     Documentation=https://github.com/kubernetes/kubernetes
     
     [Service]
     ExecStart=/usr/local/bin/kube-scheduler \\
       --config=/etc/kubernetes/config/kube-scheduler.yaml \\
       --v=2
     Restart=on-failure
     RestartSec=5
     
     [Install]
     WantedBy=multi-user.target
     EOF
     ```

### 8.6. Start The Controller Services

Run:

```
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

### 8.7. Enable HTTP Health Checks

1. Install nginx:

     ```
     sudo apt-get install -y nginx
     ```

2. Create nginx configuration file:

     ```
      cat > kubernetes.default.svc.cluster.local <<EOF
      server {
        listen      80;
        server_name kubernetes.default.svc.cluster.local;
      
        location /healthz {
           proxy_pass                    https://127.0.0.1:6443/healthz;
           proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
        }
      }
      EOF
     ```

3. Set nginx to use our newly created configuration file:

     ```
     sudo mv kubernetes.default.svc.cluster.local \
       /etc/nginx/sites-available/kubernetes.default.svc.cluster.local

     sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
     ```

4. Restart and enable nginx:

     ```
     sudo systemctl restart nginx
     sudo systemctl enable nginx
     ```

### 8.8. Verification

To verify, run:

```
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```

The result should look like this:

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

To check nginx HTTP health check proxy, run:

```
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

The result should look like this:

```
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Mon, 14 May 2018 13:45:39 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive

ok
```
