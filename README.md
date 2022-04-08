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

## Ingress

### Motivation 
Since we now know how to create **pods** that contain our apps in containers and **services** to expose those ports to the cluster and the outside world, we need to know how to **communicate** with those expose pods, we can use the IP address of any our nodes in the cluster and the **targetPort** specified in our service ("http://IP:PORT/hello") but this isn't very efficient we need to use http and https using the default port (80 and 443). This means we have to **Know every target port we define** which isn't user friendly plus it's not scalable. Second problem is some of our apps might need a secure connection using https which means we need somewhere to store our SSL certificate.

### Responsibilities 
The ingress takes cares of SSL certificate maintenance. Plus the ingress acts as our cluster **gateway** or **load balancer** all incoming requests could go through it and given certain **URL prefixes** it can determine which service is should forward the request too.

### Types
K8 by default doesn't not provide an ingress controller like the replica set controller, deployments controller and so on. But it provides an API to create one but usually you won't need to create one manually there are different ingress controllers maintained by the community and different ones for each cloud provider you might be using (GCP, AWS, AZURE) the one used in the example is base on the google. We will be using the **nginx** **ingress**, you can find all the configuration for it [here](https://github.com/kubernetes/ingress-nginx/blob/main/README.md). 

### Ingress Creation
After running the command `kubectl create -f ./k8-ingress.yaml` a request is sent to the **API server** to create a new ingress object, the ingress controller is listening to the API server events and sees there is a new ingress creation request. It configures the load balancer (can be nginx or HAproxy, etc in our case we used nginx) according to the yaml [file](./k8-ingress.yaml).

### Multiple ingress creation
We can create multiple ingress files each with it's own rules and k8 will take both configurations and apply them to our load balancer.

### Ingress with default back-end
Like we said we can create multiple ingress configurations in our cluster, we can also define an ingress with no **rules**, k8 will understand this is the default back-end to forward the request to when it doesn't match any of the previous rules, here is an example:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  backend:
    serviceName: devops-toolkit
    servicePort: 80
```
## Volumes

### Motivation 
As we know a container might crash or the **pod** running it even might crash so if we for example had the logs saved **in a file** locally in that container as soon as it restarts or crashes the information is gone. So we need some where to save such data, k8 **volumes** are used to for such cases, volumes can also be used to read data from the host node that is running the pods.

### Types 

#### host path
The `hostPath` creates a volume from the container **host** which in our case is the node running the pod, this isn't necessarily the best option to keep state, as we discussed pod might fail and crash or updated so this host path isn't the most common case for volume mounting.

It has only two options **path** which is the path to the entity (more on that later) that will be available through this volume. 

and **type** which indicates the type of entity the volume will map to could be a file, directory socket, char device or block device, in our case (k8-pod-with-volumes.yaml) we used socket since we ran a container that needed access to the docker server, so we created a volume to mount the socket (on the host a.k.a the node) so our docker container can communicate with the docker server on the node running the pod.

#### Git repo
There is also another type that mounts a directory to a **github repo** directly it doesn't add much value as maybe as part of our image creation can have a `git clone` but it seems more declarative and let's you rely less on ambiguous commands

#### Empty directory 
This type create an empty volume on the node on which the pod resides it will continue to exist **as long as the pod keeps running on the same node** if the container mounting the volume fails and k8 has to recreate it, the volume will keep the old data that was written to it if any. Once the pod is destroyed or fails the volume goes with it.
#### Advanced types
there are other volumes types like **config maps** but it will have it's own section.

## ConfigMaps

### Motivation
Usually our apps need some kind of **configuration** to run for a back-end server for example that might be the port to listen for incoming requests on, the database connections and so on..., it's a very common practice to inject those values as **env** **variables** as sometimes we might deploy the same application in different environments (development, staging and production). This is where **config** **maps** come into a play by allowing us to **inject** configurations into our containers

### Different ways of creating configurations maps

#### From files
One way to create a configuration map is from a file, for instance you might need to run a container for prometheus and you have your configuration file ready on your local machine. using the command 
```js
kubectl create cm my-config \
    --from-file=cm/prometheus-conf.yml
```
 Using the `from-file` option allows you to create a config object on your cluster from the given file.
 #### From key/value literal
 Another way is to explicitly define your configuration using key value pairs, which might be suitable for small configurations
 ```js
 kubectl create cm my-config \
    --from-literal=something=else \
    --from-literal=weather=sunny
 ```
This will create a config object with two keys  (something=else, weather=sunny) named `my-config`

#### From env file
A pretty common practice is to have your env variables defined in a file like so,
```js
something=else
weather=sunny
```
From the given file we can create a config object in a similar way we created one from a normal file
```js
kubectl create cm my-config \
    --from-env-file=cm/my-env-file.yml
```
The main difference between from env file is an env value has to follow a certain format which is known for env files (key value pairs) unlike the normal file which can be anything

### Mounting config maps
Now that we now how to make config objects in different ways we need to **connect** them somehow to our running container inside our pods

#### Mounting as a volume
One way is to update the spec of the pod to add a volume from the config map object and mount it like so 
```js
 spec:
  containers:
  - name: alpine
    image: alpine
    command: ["sleep"]
    args: ["100000"]
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: my-config
```

In the volumes section we define our volume which is created from our previously created config object. Then in the volume mount section we mount that volume and give it the path 

#### Defining env variables 
Another way is to define the needed env variables for our pods in the spec section and **fetch** their values from a predefined config object,
```js
apiVersion: v1
kind: Pod
metadata:
  name: alpine-env
spec:
  containers:
  - name: alpine
    image: alpine
    command: ["sleep"]
    args: ["100000"]
    env:
    - name: something
      valueFrom:
        configMapKeyRef:
          name: my-config
          key: something
    - name: weather
      valueFrom:
        configMapKeyRef:
          name: my-config
          key: weather
```
We added a new **env** section which defines the needed env variables for our container, each env variable has name and we fetch it's value from a pre defined config object and since the config file has more than one variable we need to define which one we need.

### Using a config map YAML file
As with everything in k8 we usually can create it from a YAML file, this applies also to the config map 
#### Defining the whole config object
We can as well take the whole config object as is and have all it's defined variables being read in our pod spec,
```js
apiVersion: v1
kind: Pod
metadata:
  name: alpine-env
spec:
  containers:
  - name: alpine
    image: alpine
    command: ["sleep"]
    args: ["100000"]
    envFrom:
    - configMapRef:
        name: my-config
```
Using the **envFrom** we can read all the env variables from our config object and add them to our pod spec the difference between this and the `env.name` is you don't have to add each env variable manually like [so](./k8-config-map.yaml) by typing the command `kubectl create cm ./k8-config-map.yaml`

## Secrets

### Motivation
Sometime we might need to save configurations but they are supposed to be hidden safely (like passwords, tokens...) since we can view the config map objects data if we have access to the cluster this isn't our best option, we can instead create secrets which are pretty much like config maps

### Default secrets
By default k8 creates a secret for all our containers that is used to communicate with the cluster **API** **server** you can view the secrets by typing the command `kubectl get secrets` you should find the default secret under the type 
`kubernetes.io/service-account-token`

### Types

#### Docker registry 
The first type is `docker-registry` which registers a secret that can be used by k8 to pull images from a private registry 

#### tls
`tls` type is used to store certificates

#### Generic
`generic` type is what resembles config map and is what we will be discussing 

### Different ways of creating secrets

#### From files
One way to create a secrets is from a file
```js
kubectl create secret generic my-cred \
    --from-file=cm/prometheus-conf.yml
```
 Using the `from-file` option allows you to create a config object on your cluster from the given file.
 #### From key/value literal
 Another way is to explicitly define your secrets using key value pairs
 ```js
 kubectl create secret generic my-cred \
    --from-literal=something=else \
    --from-literal=weather=sunny
 ```
This will create a secret object with two keys  (something=else, weather=sunny) named `my-config`

#### From env file
```js
something=else
weather=sunny
```
From the given file we can create a config object in a similar way we created one from a normal file
```js
kubectl create secret generic my-cred \
    --from-env-file=cm/my-env-file.yml
```

### Keeping secrets a secret
You will notice that we created secrets **the same way** we created config maps, so what is the difference ??. Let's assume we created a secrete from literal values like so 
```js
kubectl create secret \
    generic my-creds \
    --from-literal=username=jdoe \
    --from-literal=password=incognito
```
This will create a secret object in our cluster which we can view using the following command
```js
kubectl get secret my-creds -o json
```
The output is should look something like so
```js
{
    "apiVersion": "v1",
    "data": {
        "password": "aW5jb2duaXRv",
        "username": "amRvZQ=="
    },
    "kind": "Secret",
    "metadata": {
        ...
    },
    "type": "Opaque"
}
```
As you can tell we can view our secrets keys but we **can't view** the values are encoded

### Mounting secrets
Given that we create a secret with keys `username` and `password` we need ot mount it somehow into our containers
by running the command `kubectl apply -f ./k8-secrets.yml` on [this file](./k8-secrets.yml), two new files are created by the name `jenkins-user` and `jenkins-password` in the `/etc/secrets` folder 

## Comparison between config maps and secrets

| - | Secrets | Config maps |
|---|--- |---| 
| Creation | From literal values, files and env files | From literal values, files and env files
| Mounting | Can be mounted as a volume or read as env variables | Can be mounted as a volume or read as env variables
|Persistance | Creates actual files on the host system | Saved as files in memory known as tmpfs (temporary file storage) |

A quick note that secrets don't provide total security over config maps, we can still view the encoded values for the secrets from the terminal, but it's a step in the right direction.

## Name spaces

### Motivation 

When developing apps we sometimes need different environments, a dev environment and a production environment and so on, or if we have micro services we might have different teams working on different services, so each team **can** **not** update (or view for that matter) anything apart from their services. A simple answer to all of the above is we create a k8 cluster for each environment or service, while that works managing multiple clusters will require more resources (CPU, memory...) plus it's a headache if the number of clusters get too big to keep up with and maintain. Name spaces allow us to have **virtual** **clusters** inside one k8 cluster by creating k8 objects inside a certain name space **only** other objects will be able to interact with it.

### K8 Auto Created Name Spaces
By default k8 has 3 namespaces, which you can view using the command `kubectl get ns` :

- `default`
- `kube-public`
- `kube-system`

#### default
This is the default namespace where **user created objects** will reside so any pods, services, deployments etc we create **with out** specifying the name space will be created here

#### kube-public

This is the name space that will contain public object that will be available for **all** your namespace, this is a good place for objects that you might need regardless of what name space you are operating in, for example **config maps** can be created in this name space

#### kube-system
Is where k8 related objects lay like pods for `kube-dns` and the ingress controller. 

#### Viewing name space related objects
We can define the name space in our commands using the `--namespace` option, `kubectl --namespace kube-system get pods`

### Creating a name space
We can use the command `kubectl create ns testing` for creating a new name space named `testing`, then for every object we create we need to specify the namespace flag with value `testing` or we can create a new context which adds any new objects into the testing name space by default

```js
kubectl config set-context testing \
    --namespace testing \
    --cluster c1 \ # The name of our cluster
    --user minikube # The name of the user for this context
```
and then type `kubectl config use-context testing`

### Inter Name Space communication 
Like we said by default containers can't communicate to each other if each one lies in a different name space but that's not entirely true.

Let's say we have a pod labeled `app-1` in name space `ns-1` and we have another pod labeled `app-2` in name space `ns-2` and each has it's own `NodePort` service, usually if we needed to make a request to `app-2` if they were in the same namespace it would look something like this `https://app-2/apis/user` but since they are in different namespace we have to append the namespace as a suffix to the pod name `https://app-2.ns-2/apis/user`. 

## Resources

### Motivation
Given that we now know how to create pods and scale them in different ways, what about managing their **resources**, how much memory and cpu each pod is allowed or even expected usage ??. By using resource definitions we can mitigate information to k8 about the expected resource usage and limits of a given pod.

### Creating A Resources Limited Pod
When typing `kubectl apply -f ./k8-resource-pod,yaml`  k8 will try and create this new pod and assign it to a node with **resources more than or equal to** the specified resources in the `request` section (200 megabytes of free memory). 

If there isn't such a node that exists that can accommodate the total sum of the containers resources then the pod state will be **pending** until a node has the required amount of resources.

#### Requests
Memory requests can be assigned a value using kilo byte, mega bytes, giga byte and so on ...
While one cpu unit might differ on where the cluster is deployed on virtual machine or bare-metal but we can view it as relative processing time, as in if one container has  `cpu` request value 2 and another container has request value 1, that means the first container will get double the processing time.

### Passing Allowed Resources Limits
So what happens if a certain container passes the memory `limit`?. The pod will be destroyed and the scheduler will try to redeploy it again on a node that has the free resources needed.

When it passes the cpu limit it's a bit of different story, the pod doesn't get destroyed but it's gets throttled (less cpu time).

### Resource Consumption
We need an actual way of determining the resources (memory and cpu) allocation for each pod any numbers placed will almost be random, as we can not really tell the load in an actual production environment, we can stress test but still those won't be actual numbers.

So we can begin by allocating resources a bit more than what we expect the pod to need and then **monitor** the pod using technologies like **prometheus** and **alert manager** to constantly monitor our resources at peak time and such, and have alerts set to when the pods come close to the defined limit so we can either re-adjust the limits or figure what the problem was (memory-leak, a cpu intensive task..)

### Quality Of Service
Now that we created our pods with resource limitations there is still a possibility of having the total sum of required resources **surpass** the total resources available on the node (we might have a node with 1GB of memory and three pods each with maximum allowed memory of 500 megabyte so total is 1.5 gb). Then the pod will have kill some of the pod(s). This isn't by any means random, it's done according to the **quality of service class**  assigned to the pod 

#### Guaranteed Quality Of Service
A pod has guaranteed quality of service only if:
- The limit for memory is set
- The limit for cpu is set
- The request for memory and cpu **equals** the limit

This means the pod will get guaranteed Quality of service class. (If we don't set the request it's set automatically with values equal to the limit).

#### Burstable Quality Of Service
A pod has Busrtable quality of service if it doesn't meet the requirements for guaranteed quality of service and has a request for either memory or cpu.

#### Best Effort Quality Of Service
A pod that doesn't meet the requirements for Guaranteed or burstable quality of service is considered best effort and can use any available resources if needed.

Usually when there is contention over resources, k8 gets rid off best effort first (lowest priority), if there isn't any best effort it moves on too burstable then guaranteed (highest priority).

### Namespace resources

### Motivation
That's all good but someone might just create a pod with a very large amount of memory and cpu time. so it will hog all the node resources. we might need to set some limits and guidelines for defining maximum and minium allowed resources.

### Creating Namespace Resource Defaults And Limits

If we type the command `kubectl --namespace test create \
    -f ./k8-namespace-resources.yaml \
    --save-config --record`
The new limit object will be created for the `test` namespace so any pods created in that namespace will have to commit by those rules.

### Resource Quotas

### Motivation
Now that we set rules for the resource limitation we still have a small problem, users might stick to the defined rules per pod but they can still create hundreds and hundreds of pod which will hog down our cluster and cause contention for resources. So we need ot define **resource quotas** to limit the overall resource usage per namespace. 

After running the command `kubectl create \
    -f ./k8-namespace-resource-quotas.yaml \
    --record --save-config` The resource quota for the specified namespace, The summation of required pods belonging to that namespace cannot exceed the defined limits.

## Persistent Volumes

### Motivation
As we discovered we can mount volumes inside our containers to be able to read data from file system, repos..., the highest level of durability we reached was mounting a local file/directory on the node so if the container fails and restarts the data still persists on the node, but **if the node fails** we lose all our data which isn't desirable in some case (hosting a db on your cluster needs a durable place to write and save the data).

**Persistent volumes** can solve the above problem by providing a mountable file system that is guaranteed to be durable

### Creating Persistent Volumes

Persistent volume act as a bridge between k8 and the actual file system we want to access so usually we can create a cloud provider specific file system and have a persistent volume that points to it. The common types of file systems are:
- `EBS` and `EFS` for amazon
- `azureDisk` or `azureFile` for microsoft
- `GCEPersistentDisk` for GCP
- If for some reason you are running the k8 cluster on-prem data center we can use `nfs`

In our example we will use the `EBS` for our file system option.

checkout the [persistent volume yaml](k8-persistent-volumes.yaml)

### Access modes
We have multiple mode for accessing our file system:
- `ReadWriteOnce`: Only one pod is allowed to use the file system at any given time/
- `ReadOnlyMany`: Any number of given pods might use the file system as read only.
- `ReadWriteMany`: Any number of given pods might use the file system for reading and write. 

### Persistent Volume Claim
The Persistent volume alone is useless, we only defined the resource we need a way to be able to **claim it**.

By creating a [persistent volume claim](k8-persistent-claim.yaml).

Using the persistent volume claim the scheduler tries to find a matching persistent volume (using the storage class name and storage request) if it does the state of the claim and persistent volume will change to **bound** as in someone is using that volume. If it doesn't the status for the claim will be unbound until a volume is created that matches the specified class name and storage requests.

### Mounting Persistent Volume To Pod
Now that we have our persistent volume definition and claim definition we need to be able to use them in our [pod definition](k8-persistent-volume-pod.yaml).

### Deleting Persistent Volume Claims
If we delete our pod and the persistent volume claim objects, the persistent volume remains in **retain** policy. Which is one of two policies:

- `retain`: The volume needs to be cleared and have all the previous data written on it to be deleted before it can be used again by another pod.
- `Delete`: When a claim for a persistent volume is removed the underlying file system is removed as well (in our case the EBS is removed).
### Dynamic Volumes Provisioning  
What we have just done is called manual volume provisioning, where we first created the EBS on aws then we created a persistent volume with 10Gi and a claim corresponding to it and then a pod mounted that claim, if another pod comes along that needs some kind of persistent volumes we have to go through the whole process **manually**.

Using the storage classes `storage.k8s.io` we can define storage classes that can be used by persistent claims to dynamically create file system.

Depending on the cloud provider or if you are running an on-prem cluster you define the type of file system. in our [storage class](./k8-persistent-volume-storage-class.yaml) our **provisioner** which is responsible for creating the needed file system is `kubernetes.io/aws-ebs` and the file system type is io1 since we want to create a EBS of type io1 on aws. And the reclaim type is `delete` so when the claim is gone the file system is removed.

We just need to create a claim using the same storage class name and a pod using that claim, this is the sequential breakdown of the process:
- We create a pod that mounts the persistent volume claim 
- The claim requests a persistent volume of the same storage class
- our storage class object talks to the aws API to create a file system EBS of type io1
- The EBS volume is mounted on our pod

And if the pod is removed the EBS volume will be deleted as well, since our reclaim policy is delete.

You can check out the 
## References 
A big thanks for educative for their amazing [course](https://www.educative.io/path/kubernetes-essentials) for k8.