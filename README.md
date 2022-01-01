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

## Replica Set
By default a **pod** doesn't have any fault tolerant behavior (if it's created without any **controllers**), so a replica set is one of the **controllers** that is responsible for making sure the right amount of pods are running in our cluster 

### Assigning pods
The replica set identifies the pods it's responsible for by using the **matchLabels** so if the replica set has `matchLabels` x:1 y:2 **any pod** with at least those 2 labels and same value will be considered as part of the replica set pod pool

### Replica set creation
- After running the command `kubectl create -f ./k8-replica-set.yaml` a request is sent to the **API server** and the **replica set controller** is watching for events on the API server and detects a replica set creation event occurred 

- The replica set controller creates the same number of pods as specified in it's yaml file
  
- The scheduler detects new unassigned pod(s) so it assigns those pods to the available nodes and so on ...
## References 
A big thanks for educative for their amazing [course](https://www.educative.io/path/kubernetes-essentials) for k8 