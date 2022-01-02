# k8-examples
A sample of files on how to create different entities in k8 (pods, replicas, deployments...)

## Intended For 
People who have basic knowledge of k8, and just need a quick reminder of how to create yaml files for different k8 objects and some of the options allowed for each type

## Pod
The smallest building unit for k8, the yaml file for a basic pod would have the **spec** keyword used for defining the containers running on this pod and each container health check policy. You can check out the pod yaml file [here](./k8-pod.yaml)

### Pod Scheduling 
- After running the command `kubectl create -f ./k8-pod.yaml` a request is sent to the master node in the k8 cluster (which runs the **API server** and **Scheduler**) to the API server for creating the new pod 

- Since the scheduler is watching for events from tha API server, it detects a new pod creation events so it's responsible for **assigning that pod to a node in the cluster** 
  
- After the scheduler assigns the pod to a node the **kubelet** running on that node is also listening for events on the **API server**, it detects that a pod was assigned to it so it commands the docker daemon on it's node to create the specified containers...

### Container Communication 
Each pod in a k8 cluster is assigned an **IP address**, that been said all the containers in **the same pod** can communicated with each other using **localhost** as for pod to pod communication we will ge to that in the [Services](#services) section.

## Replica Set
By default a **pod** doesn't have any fault tolerant behavior (if it's created without any **controllers**), so a replica set is one of the **controllers** that is responsible for making sure the right amount of pods are running in our cluster 

### Assigning pods
The replica set identifies the pods it's responsible for by using the **matchLabels** so if the replica set has `matchLabels` x:1 y:2 **any pod** with at least those 2 labels and same value will be considered as part of the replica set pod pool

### Replica set creation
- After running the command `kubectl create -f ./k8-replica-set.yaml` a request is sent to the **API server** and the **replica set controller** is watching for events on the API server and detects a replica set creation event occurred 

- The replica set controller creates the same number of pods as specified in it's yaml file
  
- The scheduler detects new unassigned pod(s) so it assigns those pods to the available nodes and so on ...

## Services

### Motivation
As we established pods can be spun up and destroyed as the replica set responsible for that pod sees fit. So pod to pod communication seems impossible (since each pod has a dynamic IP address and we can't hardcode it's value for communication purposes). So services were created to **expose pods** for providing a stable interface for communication  

### Types

#### ClusterIP (default)
Used to expose our pod **only inside our cluster** so other pods in our cluster are allowed to communicate with the associated pod. That pod **can not** be accessed outside our cluster.

#### NodePort
Used to expose a pod on a certain node to **the outside world** (other entities outside our k8 cluster). By default `NodePort` also creates a clusterIP port so nodes in the same cluster can communicate with the associated internally using the cluster network.

#### LoadBalancer 
usually used with the cloud provider (AWS, GCP, Azure..) own's load balancer, we will dive deeper into this later

### Service Creation
Usually any service created is defined in the **same file** the pod is defined in as they are tightly coupled (the service is created to expose a certain pod)

- After running the command `kubectl create -f k8-replica-set-and-service.yaml` The normal flow for creating a replica set is followed

- In the service section we can see that we defined a service of type `NodeType`, the service needs a selector to define which pods it needs to traffic incoming requests to (more on that in the next steps) with a bunch of options, all that is sent to the API server as request for service creation.

- Another controller named **Endpoint controller** is listening to events from the API server, and detects a new service creation request is fired.

- The controller creates new **endpoint object(s)** (the number of endpoint objects is equal to the number of pods the service is responsible for) each endpoint object corresponds to a pod IP and port (the port specified in the node service) all this info is used for re-routing incoming request to any of these pods.

- **Kube-proxy** then detects new endpoint object(s) are created and a new service, so it creates **ip table rules** for the service to re-route incoming requests to the endpoints and rules for each endpoint to re-route to it's corresponding pod.

- **Kube-dns** also detects there is a new service created so it adds an entry fo the new service.

### Incoming requests flow

#### Inside the cluster
 Given that the service was created for a specific port(let's say 2000) any incoming traffic **inside the cluster** to port 2000, will be intercepted by **kube-proxy** and forwarded to the service which forwards it to the appropriate endpoint.

#### Outside the cluster 
Given that the service was created for a specific port (let's also say 2000) any incoming request **outside the cluster** will be intercepted by the **kube-dns** and forwarded to the right service which forwards it to the right endpoint.

#### Choosing the right pod
Like we said a service can represent **many pods** (if we deploy the same pod using a replica set with value 3 for example) so the service forwards the request to **a random endpoint out of the 3** so it resembles something like round robin where each pod receives almost the same amount of requests 

## Deployments
### Motivation
Since we can now know how to deploy **pods** and if they crash **Replica sets** are responsible for restarting and making sure the right count of pods are up, But what happens if we update the image (containers running inside our pods) for instance we might've added a new feature to our codebase or we are just upgrading the version used of our database. That means we have to take down **all our pods** (since they have the old image), delete the old replica-set and re-apply the replica set yaml file. But this is bound to have some down time which in some cases we can't afford. Here is where **Deployments** come into play they **roll back** the old pods one by one and in each step it rolls back a pod **a new pod** replaces it (so by the end the same number of pods is up) so there is zero down-time.

### Deployment Creation
- After Running the command `kubectl create -f ./k8-deployment.yaml` a request is sent to the **API server** to create a new deployment object which is considered as an event, which is listened to by the deployment controller which on receiving that event sends a command to the **API server** to create a replica set with the given spec in the yaml file, and the replica set creation flow starts..., this might not seem of a great value but deployments really shine when it comes to **updating our pods**

### Deployment Update/Strategy 
This is the action taken to **update** the state of an already deployed replica set (adding the new image to the pods), there are mainly two ways to do so 

#### Recreate
This is the non-common way, it commands k8 when to first **delete all the pods** of the old replica set **then** create a new replica set with the new pods, this obviously doesn't have any **zero down time**, the only case this is advised is when you are sure **two different versions** of your pod can't co-exist. then this is the advisable way to go

#### RollingUpdate
This is more common, as this allows k8 to take down the old replica set pod(s) by pod(s) and at the same time create the new desired replica set pod(s) by pod(s), depending on the configuration you define in the deployment yaml file, but for simplicity let's say we have a replica set with 3 pods and we want to deploy a new version, k8 would take 1 **old pod** down and create a **new one** then we would have two old pods and one new pod, and so on until we have 0 old pods and 3 new pods.

### Common k8 Deployment commands
Now that we know the life cycle of **Deployments** and it's use case here are some common commands

- `kubectl rollout undo -f ./k8-deployment.yaml`: This will rollback the replica set to the **previous version** using the same deployment strategy defined in the yaml file, can be used in case we discovered a bug that might need some time to fix 

## References 
A big thanks for educative for their amazing [course](https://www.educative.io/path/kubernetes-essentials) for k8.