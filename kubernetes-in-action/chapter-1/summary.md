# Introducing Kubernetes

**From Monolithic Applications to Microservices**

Before discussing Kubernetes, we need to discuss about the increasing popularity of microservice architecture and container technology. Understanding both phenomenon will give us a proper context on why we need a system like Kubernetes.

In the decade that I became a student in Informatics Engineering and then started my professional career as a software engineer (2000-2010), most software applications I learned to build or I developed for my enterprise clients were developed with monolithic architecture. Scaling were rarely an issue as most of my projects involved only a handful or at most thousands of users.

There are two ways to scale up a software application: vertically (scale up) and horizontally (scale out). To scale up, we just need to add more resources (CPU, RAM, etc) to the machine running our app. As you may guest, there is a low limit on how high we can scale up because there is only limited amount of resources we can add to a single machine. To scale out, we use additional machines to run different copies of the same app. 

For monolithic applications, scaling out approach may require major change in most monolithic apps. For some parts of a monolithic application, it is even hard or next to impossible to scale out to multiple machines. That is the main reason why companies that operate on huge scale such as Google, Amazon, or Netflix turn into microservices. While the term "microservices" has been around way before them, these companies have put microservice architecture in the spotlight.

There are a lot of benefits of microservice architecture. For the purpose of scalability, we will emphasize on two benefits:
- components of microservices run as independent processes while all components of monolithic apps run as a single process
- components of microservices allow us to develop, deploy, update, and scale each of them individually while components of monolithic apps have to be developed, deployed, updated, and scaled as a one entity
