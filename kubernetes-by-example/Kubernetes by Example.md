# Kubernetes by Example

## Requirements

- Install minikube
- [Install xhyve](https://github.com/mist64/xhyve)
- Optional [Install minishift](https://docs.openshift.org/latest/minishift/getting-started/installing.html)
- Install docker-machine-driver-xhyve ()
- [Setup docker-machine-driver-xhyve](https://docs.openshift.org/latest/minishift/getting-started/setting-up-driver-plugin.html#xhyve-driver-install)
- [Optional] Setup minishift oc-env (minishift oc-env, eval $(minishift oc-env))
- [Optional] login as admin (oc login -u system:admin)

## 1. Pods

### Concept

A pod is a collection of containers sharing a network and mount namespace. A pod is the basic unit of deployment in Kubernetes. All containers in a pod are scheduled on the same node.

### Commonly Used Commands

1. To create and run a pod:

   `kubectl run [name] —image=[image_source]:[version] —port=[port]`  
   ex: `kubectl run sise --image=mhausenblas/simpleservice:0.5.0 --port=9876`  

2. To view all pods:
   
   `kubectl get pods`  

3. To view detailed information about pods:
   
   `kubectl describe pods`  

4. To view detailed information about specific pod:
   
   `kubectl describe pod [name]`   
   ex: `kubectl describe pod sise-2343179185-9pl08`  

5. To create a pod with configuration file

   `kubectl create -f [source]`  
   ex: `kubectl create -f https://raw.githubusercontent.com/mhausenblas/kbe/master/specs/pods/pod.yaml`  

### Issues:

* [Solved] Can’t ssh to pod (or node?) from cmd, must do it from terminal in Minishift web app. Solved by using minikube and use “minikube ssh” command.
* [Solved] Can’t run “kubectl create -f [source]”. At first it displayed “Error from server (Forbidden): unknown”. After I managed to login as admin with “oc login -u system:admin”, it displayed “Error from server (NotFound): the server could not find the requested resource”. Solved by using minikube which has the latest kubernetes server version (minishift has 1.6 version while minikube has 1.9 version).

## 2. Labels

### Concept

Labels are mechanism to organize Kubernetes objects. A Label is a key-value pair with certain restrictions (length and value) but without any pre-defined meaning.

### Steps

- Define label in a configuration file. See “kubernetes-by-example/labels/pod.yaml” file to see how to define labels in a configuration file.

### Commonly Used Commands

1. To create label from command line:

   `kubectl label pods [pod_name] [label]`  
   ex: `kubectl label pods labelex owner=iqbal`  

2. To show pods with their labels:

   `kubectl get pods --show-label`  

3. To use a label for filtering:

   `kubectl get pods --selector [label]`  
   ex: `kubectl get pods --selector owner=iqbal`  
   `kubectl get pods -l [label]`  
   ex: `kubectl get pods -l owner=iqbal`  
   `kubectl get pods -l ‘[complex conditions]’`  
   ex: `kubectl get pods -l ‘env in (development, production)’`  

## 3. Replication Controllers

### Concept

A replication controller (RC) is a supervisor for long-running pods. An RC will launch a specified number of pods called “replicas” and makes sure that they keep running.

### Steps

- Define a replica in configuration file. See “kubernetes-by-example/replication-controllers/rc.yaml” to see how to define replicas in a configuration file.

### Commonly Used Commands

- To show rc:
    - kubectl get rc
- To scale up:
    - kubectl scale --replicas=[number] rc/[rc_name]
    - ex: kubectl scale --replicas=3 rc/rcex
- To delete an rc and all its pods:
    - kubectl delete rc [name]
    - ex: kubectl delete rc rcex

## 4. Deployments

