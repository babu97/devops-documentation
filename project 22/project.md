## install eksctl tool 

using the below guide 

[link ](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/setting-up-eksctl.html)


create  kubernetes cluster using the below code : 

cluster 

```
eksctl create cluster \
> --name project-22 \
> --region us-east-1 \
> --nodegroup-name linux-nodes \
> --node-type t2.micro \
> --nodes 2
```

![](/eks.PNG)

## UNDERSTANDING THE CONCEPT

## create the nginx pod 
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels: 
    app: nginx-pod
spec:
  containers:
  - image: nginx:latest
    name: nginx-pod
    ports:
    - containerPort: 80
      protocol: TCP
```

```
kubectl get pod nginx-pod --show-labels
```
![](/nginx%20pod%20status.PNG)

```
kubectl get service nginx-service -o wide
```
![](/nginx%20service.PNG)



```
kubectl get pod nginx-pod -o wide

```
**Notice that the IP address of the Pod, is NOT the IP address of the server it is running on. Kubernetes, through the implementation of network plugins assigns virtual IP adrresses to each Pod.**

![](/nginx%20wide.PNG)

Therefore, Service with IP 10.100.29.154 takes request and forwards to Pod with IP 172.50.197.236


## access the nginx webpage

To access the service, you must:

- Allow the inbound traffic in your EC2’s Security Group to the NodePort range 30000-32767
- Get the public IP address of the node the Pod is running on, append the nodeport and access the app through the browser.
You must understand that the port number 30080 is a port on the node in which the Pod is scheduled to run. If the Pod ever gets rescheduled elsewhere, that the same port number will be used on the new node it is running on. So, if you have multiple Pods running on several nodes at the same time – they all will be exposed on respective nodes’ IP addresses with a static port number.

![](/nginx%20webpage.PNG)

## Toling app 

create the manifest files  

- **POD**
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels: 
    app: nginx-pod
spec:
  containers:
  - image: nginx:latest
    name: nginx-pod
    ports:
    - containerPort: 80
      protocol: TCP


```
-  **SERVICE**
```
apiVersion: v1
kind: Service
metadata:
  name: tooling-service
 
spec:
  type: NodePort
  selector:
    app: tooling-pod
  
  ports:
   - protocol: TCP
     port: 80
     nodePort: 30081
```

- Run kubectl apply -f ./service and pod  

![](/tooling%20pod.PNG)

![](/tooling%20service.PNG)

![](/tooling%20website.PNG)

### How Kubernetes ensures desired number of Pods is always running?

When we define a Pod manifest and appy it – we create a Pod that is running until it’s terminated for some reason (e.g., error, Node reboot or some other reason), but what if we want to declare that we always need at least 3 replicas of the same Pod running at all times? Then we must use an **[ResplicaSet (RS)](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)** object – it’s purpose is to maintain a stable set of Pod replicas running at any given time. As such, it is often used to guarantee the availability of a specified number of identical Pods.

### COMMON KUBERNETES OBJECTS 
- Pod
- Namespace
- ResplicaSet (Manages Pods)
- DeploymentController (Manages Pods)
- StatefulSet
- DaemonSet
- Service
- ConfigMap
- Volume
- Job/Cronjo

The very first concept to understand is the difference between how Docker and Kubernetes run containers – with Docker, every docker run command will run an image (representing an application) as a container. The running container is a Docker’s smallest entity, it is the most basic deployable object. Kubernetes on the other hand operates with pods instead of containers, a pods encapsulates a container. Kubernetes uses pods as its smallest, and most basic deployable object with a unique feature that allows it to run multiple containers within a single Pod. It is not the most common pattern – to have more than one container in a Pod, but there are cases when this capability comes in handy.

In the world of docker, or docker compose, to run the Tooling app, you must deploy separate containers for the application and the database. But in the world of Kubernetes, you can run both: application and database containers in the same Pod. When multiple containers run within the same Pod, they can directly communicate with each other as if they were running on the same localhost. Although running both the application and database in the same Pod is NOT a recommended approach.
A Pod that contains one container is called single container pod and it is the most common Kubernetes use case. A Pod that contains multiple co-related containers is called multi-container pod. There are few patterns for multi-container Pods; one of them is the sidecar container pattern – it means that in the same Pod there is a main container and an auxiliary one that extends and enhances the functionality of the main one without changing it.

- **kind:** Represents the type of kubernetes object created. It can be a Pod, DaemonSet, Deployments or Service.
- **version:** Kubernetes api version used to create the resource, it can be v1, v1beta and v2. Some of the kubernetes features can be released under beta and available for general public usage.
- **metadata:**provides information about the resource like name of the Pod, namespace under which the Pod will be running,
labels and annotations.
- **spec:** consists of the core information about Pod. Here we will tell kubernetes what would be the expected state of resource, Like container image, number of replicas, environment variables and volumes.
**status:** consists of information about the running object, status of each container. Status field is supplied and updated by Kubernetes after creation. This is not something you will have to put in the YAML manifest.



### Deploying a random Pod

Lets see what it looks like to have a Pod running in a k8s cluster. This section is just to illustrate and get you to familiarise with how the object’s fields work. Lets deploy a basic Nginx container to run inside a Pod.

- **apiVersion** is v1
- **kind** is Pod
- **metatdata** has a name which is set to nginx-pod
- The **spec** section has further information about the Pod. Where to find the image to run the container – (This defaults to Docker Hub), the port and protocol.

1. Create a Pod yaml manifest on your master node
```
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
name: nginx-pod
spec:
containers:
- image: nginx:latest
name: nginx-pod
ports:
- containerPort: 80
  protocol: TCP
EOF
```
2. Apply the manifest with the help of kubectl

```
kubectl apply -f nginx-pod.yaml
```
3. Get an output of the pods running in the cluster

```
kubectl get pods
```
![](/kubectl%20get%20pods.PNG)



### Understanding the common YAML fields for every Kubernetes object

### ACCESSING THE APP FROM THE BROWSER

The ultimate goal of any solution is to access it either through a web portal or some application (e.g., mobile app). We have a Pod with Nginx container, so we need to access it from the browser. But all you have is a running Pod that has its own IP address which cannot be accessed through the browser. To achieve this, we need another Kubernetes object called Service to accept our request and pass it on to the Pod.

A service is an object that accepts requests on behalf of the Pods and forwards it to the Pod’s IP address. If you run the command below, you will be able to see the Pod’s IP address. But there is no way to reach it directly from the outside world.

```
kubectl get pod nginx-pod  -o wide 
```
![](/nginx%20pod%202.PNG)

Let us try to access the Pod through its IP address from within the K8s cluster. To do this,

1. We need an image that already has curl software installed

use this docker image 
```
dareyregistry/curl
```


2. Run kubectl to connect inside the container

```
kubectl run curl --image=dareyregistry/curl -i --tty

```
connect to the pod using 
```
kubectl exec -it nginx-pod bash
```

3. Run curl and point to the IP address of the Nginx Pod (Use the IP address of your own Pod)

```
curl -v 192.168.25.215:80

```
![](/output.PNG)

If the use case for your solution is required for internal use ONLY, without public Internet requirement. Then, this should be OK. But in most cases, it is NOT!


Assuming that your requirement is to access the Nginx Pod internally, using the Pod’s IP address directly as above is not a reliable choice because Pods are ephemeral. They are not designed to run forever. When they die and another Pod is brought back up, the IP address will change and any application that is using the previous IP address directly will break

To solve this problem, kubernetes uses Service – An object that abstracts the underlining IP addresses of Pods. A service can serve as a load balancer, and a reverse proxy which basically takes the request using a human readable DNS name, resolves to a Pod IP that is running and forwards the request to it. This way, you do not need to use an IP address. Rather, you can simply refer to the service name directly.

Let us create a service to access the **Nginx Pod**
1. Create a Service yaml manifest file:

```
sudo cat <<EOF | sudo tee ./nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-pod 
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF
```
2. Create a nginx-service resource by applying your manifest

```
kubectl apply -f nginx-service.yaml

```
check the created service

```
kubectl get service

```
![](/all%20services.PNG)


**Observation:**

- ClusterIP
- NodePort
- LoadBalancer &
- Headless Service


Now that we have a service created, how can we access the app? Since there is no public IP address, we can leverage kubectl's port-forward functionality.

```
kubectl  port-forward svc/nginx-service 8089:80

```
8089 is an arbitrary port number on your laptop or client PC, and we want to tunnel traffic through it to the port number of the nginx-service 80


To make this work, you must reconfigure the Pod manifest and introduce labels to match the selectors key in the field section of the service manifest.


1. Update the Pod manifest with the below and apply the manifest:

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx-pod  
spec:
  containers:
  - image: nginx:latest
    name: nginx-pod
    ports:
    - containerPort: 80
      protocol: TC
```

Notice that under the metadata section, we have now introduced labels with a key field called app and its value nginx-pod. This matches exactly the selector key in the service manifest.

The key/value pairs can be anything you specify. These are not Kubernetes specific keywords. As long as it matches the selector, the service object will be able to route traffic to the Pod.

Apply the manifest with:

```
kubectl apply -f nginx-pod.yaml

```
2. Run kubectl port-forward command again

```
kubectl  port-forward svc/nginx-service 8089:80

``
![](/port%20forwarding.PNG)

## CREATE A REPLICA SET 
Let us create a `rs.yaml` manifest for a ReplicaSet object:

```
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod


  template:
    metadata:
      name: nginx-pod
      labels:
         app: nginx-pod
    spec:
      containers:
      - image: nginx:latest
        name: nginx-pod
        ports:
        - containerPort: 80
          protocol: TCP
```

```
kubectl apply -f rs.yaml

```

The manifest file of ReplicaSet consist of the following fields

- apiVersion: This field specifies the version of kubernetes Api to which the object belongs. ReplicaSet belongs to **apps/v1** apiVersion.
- kind: This field specify the type of object for which the manifest belongs to. Here, it is **ReplicaSet**.
- metadata: This field includes the metadata for the object. It mainly includes two fields: name and labels of the **ReplicaSet**.
- spec: This field specifies the **label selector** to be used to select the Pods, number of replicas of the Pod to be run and the container or list of containers which the Pod will run. In the above example, we are running 3 replicas of nginx container.

Let us check what Pods have been created:

```
kubectl get pods

```
![](/load%20balancing.PNG)

Here we see three ngix-pods with some random suffixes (e.g., -j784r) – it means, that these Pods were created and named automatically by some other object (higher level of abstraction) such as ReplicaSet

if you try and delete the pods still gets re-created 

Explore the ReplicaSet created:


![](/replicasetss.PNG)




Notice, that ReplicaSet understands which Pods to create by using **SELECTOR** key-value pair.

#### Get detailed information of a ReplicaSet

To display detailed information about any Kubernetes object, you can use 2 differen commands:
```
- kubectl describe %object_type% %object_name% (e.g. `kubectl describe rs nginx-rs`)
- kubectl get %object_type% %object_name% -o yaml (e.g. `kubectl describe rs nginx-rs -o yaml`)
```
Try both commands in action and see the difference. Also try `get` with `-o` json instead of `-o yaml` and decide for yourself which output option is more readable for you.


```
kubectl describe rs nginx-rs
```





### OUTPUT
![](/selector%20status.PNG)

### Scale ReplicaSet up and down:
In general, there are 2 approaches of Kubernetes Object Management: imperative and declarative.

Let us see how we can use both to scale our Replicaset up and down:

#### Imperative:

We can easily scale our ReplicaSet up by specifying the desired number of replicas in an imperative command, like this:
 
```
 kubectl scale rs nginx-rs --replicas=5
```
![](/scaled%20replicas.PNG)

**Declarative:**
Declarative way would be to open our rs.yaml manifest, change desired number of replicas in respective section

```
spec:
  replicas: 3
```
and applying the updated manifest:

```
kubectl apply -f rs.yaml

```
There is another method – ‘ad-hoc’, it is definitely not the best practice and we do not recommend using it, but you can edit an existing ReplicaSet with following command:
```
kubectl edit -f rs.yaml

```
As Kubernetes mature as a technology, so does its features and improvements to k8s objects. ReplicationControllers do not meet certain complex business requirements when it comes to using selectors. Imagine if you need to select Pods with multiple lables that represents things like:

- Application tier: such as Frontend, or Backend
- Environment: such as Dev, SIT, QA, Preprod, or Prod
So far, we used a simple selector that just matches a key-value pair and check only ‘equality’:

```
  selector:
    app: nginx-pod
```
But in some cases, we want ReplicaSet to manage our existing containers that match certain criteria, we can use the same simple label matching or we can use some more complex conditions, such as:

```
 - in
 - not in
 - not equal
 - etc...
```
lets look at the following manifest file :

```
apiVersion: apps/v1
kind: ReplicaSet
metadata: 
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      env: prod
    matchExpressions:
    - { key: tier, operator: In, values: [frontend] }
  template:
    metadata:
      name: nginx
      labels: 
        env: prod
        tier: frontend
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
          protocol: TCP
```
In the above spec file, under the selector, matchLabels and matchExpression are used to specify the key-value pair. The matchLabel works exactly the same way as the equality-based selector, and the matchExpression is used to specify the set based selectors. This feature is the main differentiator between ReplicaSet and previously mentioned obsolete ReplicationController.

## USING AWS LOAD BALANCER TO ACCESS YOUR SERVICE IN KUBERNETES.

You have previously accessed the Nginx service through ClusterIP, and NodeIP, but there is another service type – Loadbalancer. This type of service does not only create a Service object in K8s, but also provisions a real external Load Balancer (e.g. Elastic Load Balancer – ELB in AWS)

To get the experience of this service type, update your service manifest and use the LoadBalancer type. Also, ensure that the selector references the Pods in the replica set.

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    tier: frontend
  ports:
    - protocol: TCP
      port: 80 # This is the port the Loadbalancer is listening at
      targetPort: 80 # This is the port the container is listening at
```

Apply the configuration:

```
kubectl apply -f nginx-service.yaml

```

Get the newly created service :

```
kubectl get service nginx-service

```

![](/rs%20nginx-service.PNG)

An ELB resource will be created in your AWS console.


![](/alb--.PNG)
![](/kube%20tag.PNG)


Get the output of the entire yaml for the service. You will some additional information about this service in which you did not define them in the yaml manifest. Kubernetes did this for you.

```
kubectl get service nginx-service -o yaml
```
## OUTPUT

![](/wewe.PNG)

1. A clusterIP key is updated in the manifest and assigned an IP address. Even though you have specified a Loadbalancer service type, internally it still requires a clusterIP to route the external traffic through.
2. In the ports section, nodePort is still used. This is because Kubernetes still needs to use a dedicated port on the worker node to route the traffic through. Ensure that port range 30000-32767 is opened in your inbound Security Group configuration.
3. More information about the provisioned balancer is also published in the .status.loadBalancer field.

```
status:
  loadBalancer:
    ingress:
    - hostname: ac12145d6a8b5491d95ff8e2c6296b46-588706163.eu-central-1.elb.amazonaws.com
```

## USING DEPLOYMENT CONTROLLERS

Do not Use Replication Controllers – Use Deployment Controllers Instead
Kubernetes is loaded with a lot of features, and with its vibrant open source community, these features are constantly evolving and adding up.

Previously, you have seen the improvements from ReplicationControllers (RC), to ReplicaSets (RS). In this section you will see another K8s object which is highly recommended over Replication objects (RC and RS).

A Deployment is another layer above ReplicaSets and Pods, newer and more advanced level concept than ReplicaSets. It manages the deployment of ReplicaSets and allows for easy updating of a ReplicaSet as well as the ability to roll back to a previous version of deployment. It is declarative and can be used for rolling updates of micro-services, ensuring there is no downtime.

Officially, it is highly recommended to use Deplyments to manage replica sets rather than using replica sets directly.

Let us see Deployment in action.

1. Delete the ReplicaSet


```
kubectl delete rs nginx-rs

```

2. Understand the layout of the deployment.yaml manifest below. Lets go through the 3 separated sections:

```
# Section 1 - This is the part that defines the deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend

# Section 2 - This is the Replica set layer controlled by the deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend

# Section 3 - This is the Pod section controlled by the deployment and selected by the replica set in section 2.
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```
3. putting them altogether

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 8
```
```
kubectl apply -f deployment.yaml
```
Run commands to get the following

1. Get the Deployment

```
kubectl get deployment
```
![](/get%20deployment.PNG)

2. Get the replicaSet
```
kubectl get replicaSet
```
![](/replicaSet.PNG)

3. Get the Pods
```
kubectl get pods
```
![](/pods2.PNG)


4. scale the replicas in the Deployment to 15 pods

```
kubectl scale rs nginx-deployment-b87799947 --replicas=15
```

5. Exec into one of the pod's container to run linux commands
```
 kubectl exec -it nginx-deployment-b87799947-ffwzc bash 

```
List the files and folders in the Nginx directory

![](/listss.PNG)


Check the content of the default Nginx configuration file

```
 cat  /etc/nginx/conf.d/default.conf 
```

![](/default%20nginx%20web.PNG)


Now, as we have got acquaited with most common Kubernetes workloads to deploy applications:

![](/k8s_workloads.png)








```
kubectl delete -f nginx-pod.yaml
```

## PERSISTING DATA FOR PODS

Deployments are stateless by design. Hence, any data stored inside the Pod’s container does not persist when the Pod dies.

If you were to update the content of the index.html file inside the container, and the Pod dies, that content will not be lost since a new Pod will replace the dead one.

Let us try that:

1. Scale the Pods down to 1 replica.

![](/scaled%20replica.PNG)

2. EXEC into the running container

```
kubectl exec  -it nginx-deployment-b87799947-ffwzc bash

```

3. install `vim` to edit files
```
apt-get update
apt-get install vim
```

4. Update the content of the file and add the code below `/usr/share/nginx/html/index.html`
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to DAREY.IO!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to DAREY.IO!</h1>
<p>I love experiencing Kubernetes</p>

<p>Learning by doing is absolutely the best strategy at 
<a href="https://darey.io/">www.darey.io</a>.<br/>
for skills acquisition
<a href="https://darey.io/">www.darey.io</a>.</p>

<p><em>Thank you for learning from DAREY.IO</em></p>
</body>
</html>

```
5. Now, delete the only running Pod

```
kubectl delete pod nginx-deployment-5d6cf97577-92zqs

```
![](/nginx%20paggee.PNG)

6. Refresh the web page – You will see that the content you saved in the container is no longer there. That is because Pods do not store data when they are being recreated – that is why they are called ephemeral or stateless. (But not to worry, we will address this with persistent volumes in the next project)

![](/nginx%203.PNG)


Storage is a critical part of running containers, and Kubernetes offers some powerful primitives for managing it. Dynamic volume provisioning, a feature unique to Kubernetes, which allows storage volumes to be created on-demand. Without dynamic provisioning, DevOps engineers must manually make calls to the cloud or storage provider to create new storage volumes, and then create PersistentVolume objects to represent them in Kubernetes. The dynamic provisioning feature eliminates the need for DevOps to pre-provision storage. Instead, it automatically provisions storage when it is requested by users.

To make the data persist in case of a Pod’s failure, you will need to configure the Pod to use following objects:

- Persistent Volume or pv – is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes.
- Persistent Volume Claim or pvc. Persistent Volume Claim is simply a request for storage, hence the "claim" in its name.