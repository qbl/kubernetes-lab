# Setting Up Kubernetes Cluster in AWS

## 0. Requirements

1. Install [kops](https://github.com/kubernetes/kops)
   
   For MacOS:  
   `brew update && brew install kops`

2. Install and Setup AWS CLI Tools

   For MacOS:
   `sudo easy_install pip` (if you don't have python installed already)
   `pip install awscli` (or use `sudo -H pip install awscli --upgrade --ignore-installed six` if you have issues installing it the normal way)

   Setup AWS CLI:  
   Ask Vijay or Taufan for AWS Access Key ID and AWS Secret Access Key, use these keys to fill in fields prompted by this command:  
   `aws configure`  
   When prompted for zone, use: ap-southeast-1

3. Ask for access to `gopay-internal` VPN

## 1. Steps

1. Create S3 Bucket

   `aws s3api create-bucket --bucket alpha-kube-cluster --region us-east-1`  

   `aws s3api create-bucket --bucket beta-golabs-io-kops-store --region ap-southeast-1 --create-bucket-configuration LocationConstraint=ap-southeast-1`  

2. Enable S3 Bucket Versioning

   `aws s3api put-bucket-versioning --bucket alpha-kube-cluster --versioning-configuration Status=Enabled`  
   
   `aws s3api put-bucket-versioning --bucket beta-golabs-io-kops-store --versioning-configuration Status=Enabled`  

3. Setup a Subdomain

   `D=$(uuidgen) && aws route53 create-hosted-zone --name alpha.golabs.io --caller-reference $ID | jq .DelegationSet.NameServers`  

   `D=$(uuidgen) && aws route53 create-hosted-zone --name beta-golabs-io-kops-store --caller-reference $ID | jq .DelegationSet.NameServers`  

4. Create a file named "subdomain.json"

   Fill the "Value" fields with values generated by the previous command.

```
{
  "Comment": "Create a subdomain NS record in the parent domain",
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "subdomain.example.com",
        "Type": "NS",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "ns-1.awsdns-1.co.uk"
          },
          {
            "Value": "ns-2.awsdns-2.org"
          },
          {
            "Value": "ns-3.awsdns-3.com"
          },
          {
            "Value": "ns-4.awsdns-4.net"
          }
        ]
      }
    }
  ]
}
```

5. Apply the Subdomain NS records to the Parent hosted zone

   `aws route53 list-hosted-zones | jq '.HostedZones[] | select(.Name=="golabs.io.") | .Id'`  
   Pass the id generated from command above to <parent-zone-id> in the command below:
   `aws route53 change-resource-record-sets --hosted-zone-id <parent-zone-id> --change-batch file://subdomain.json`

6. Prepare local environment

   `export NAME=alpha.golabs.io`  
   `export KOPS_STATE_STORE=s3://alpha-kube-cluster`  
   `export VPC_ID=vpc-23f0a846`  

   `export NAME=beta.golabs.io`  
   `export KOPS_STATE_STORE=s3://beta-golabs-io-kops-store`  
   `export VPC_ID=vpc-23f0a846`  

7. Execute the command to actually create the kubernetes cluster

   `kops create cluster $NAME --vpc=${VPC_ID} --zones=ap-southeast-1a,ap-southeast-1b,ap-southeast-1c --master-zones=ap-southeast-1a,ap-southeast-1b,ap-southeast-1c --topology public --node-count=4 --node-size=m4.4xlarge --master-size=m4.2xlarge`  
   note: somehow the cluster created with this command has different nodes size, so we need to update it manually after we verify it

8. To verify cluster
     
   `kops update cluster $NAME --yes`  
   `kops rolling-update cluster --yes`  

   Kops will be setting up our k8s cluster and setting our `kubectl` context to that cluster. To verify if it's success, try:  
   `kubectl get nodes`  
   `kops validate cluster`  

9. Update cluster
     
   `kops get ig`  
   `kops edit ig nodes`, `kops edit ig master-ap-southeast-1a`, `kops edit ig master-ap-southeast-1b`, `kops edit ig master-ap-southeast-1c` (if needed)
   `kops update cluster`  
   `kops update cluster --yes`  
   `kops rolling-update cluster`  
   `kops rolling-update cluster --yes`

## 2. References:

1. [Kops](https://github.com/kubernetes/kops/blob/master/docs/aws.md)
2. [Manage Kubernetes Clusters on AWS Using Kops](https://aws.amazon.com/blogs/compute/kubernetes-clusters-aws-kops/)
3. [Kops Topology](https://github.com/kubernetes/kops/blob/master/docs/topology.md)
4. [Kops Commands](https://github.com/kubernetes/kops/blob/master/docs/commands.md#other-interesting-modes)
5. [On High Availability](https://github.com/kubernetes/kops/blob/master/docs/high_availability.md)