# Kubernetes by Example

## 0. Intro

This is my note when I learn Kubernetes from [Kubernetes by Example](http://kubernetesbyexample.com/). This note is mainly created so that I can learn the materials as if I'm about to teach it to someone else, just as suggested in [The Feynman Technique](https://collegeinfogeek.com/feynman-technique/). If you find this learning note useful, most credits goes to the people at [Kubernetes by Example](http://kubernetesbyexample.com/).

### Requirements

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

1. To show rc:

   `kubectl get rc`  

2. To scale up:
   
   `kubectl scale --replicas=[number] rc/[rc_name]`  
   ex: `kubectl scale --replicas=3 rc/rcex`  

3. To delete an rc and all its pods:

   `kubectl delete rc [name]` 
   ex: `kubectl delete rc rcex`  

## 4. Deployments

### Concept

A deployment is a supervisor for pods and replica sets. A deployment gives fine-grained control over how and when a new pod version is rolled out or rolled back from one state to another.

### Steps

1. Create a file named "deployment-0.9.yaml" with content as follows:

```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: sise-deploy
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: sise
    spec:
      containers:
      - name: sise
        image: mhausenblas/simpleservice:0.5.0
        ports:
        - containerPort: 9876
        env:
        - name: SIMPLE_SERVICE_VERSION
          value: "0.9"
```

2. Execute command to create the deployment:
   
   `kubectl create -f [file_source]`  
   ex: `kubectl create -f deployment-0.9.yaml`

3. Check if the deployment is successful:
   
```
kubectl get deploy
kubectl get rs
kubectl get pods
```

Commands above should return list of deployment, replica sets, and pods that we set using "deployment-0.9.yaml" file. Afterward we can check the services that run by getting inside our deployment using `minikube ssh` and then check it by using `curl 172.17.0.8:9876/info` (this IP address and URL endpoint is just an example, check the IP address of your pods using `kubectl describe pods | grep IP:`) to see if it serves the right version of our service (in this case 0.9).

4. Change our deployment version to 1.0

```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: sise-deploy
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: sise
    spec:
      containers:
      - name: sise
        image: mhausenblas/simpleservice:0.5.0
        ports:
        - containerPort: 9876
        env:
        - name: SIMPLE_SERVICE_VERSION
          value: "1.0"
```

Save the file above as "deployment-1.0.yaml" and execute the following command:

`kubectl apply -f deployment-1.0.yaml`.

Afterward we can check the services that run by getting inside our deployment using `minikube ssh` and then check it by using `curl 172.17.0.9:9876/info` (this IP address and URL endpoint is just an example, check the IP address of your pods using `kubectl describe pods | grep IP:`) to see if it serves the right version of our service (in this case version 1.0).

### Commonly Used Commands

1. To view list of deployments

   `kubectl get deploy`

2. To apply new deployment

   `kubectl apply -f [file_source]`  
   ex: `kubectl apply -f deployment-1.0.yaml`

3. To check the progress of deployment rollout

   `kubectl rollout status deploy/[name]`  
   ex: `kubectl rollout status deploy/sise-deploy`

4. To view the history of all deployments

   `kubectl rollout history deploy/[name]`  
   ex: `kubectl rollout history deploy/sise-deploy`

5. To undo deployment

   `kubectl rollout undo deploy/[name] --to-revision=[number]`  
   ex: `kubectl rollout undo deploy/sise-deploy --to-revision=1`

6. To delete a deployment

   `kubectl delete deploy [name]`  
   ex: `kubectl delete deploy sise-deploy`

