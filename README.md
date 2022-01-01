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


## References 
A big thanks for educative for their amazing [course](https://www.educative.io/path/kubernetes-essentials) for k8 