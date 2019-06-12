# Kubernetes

>   **Orchestration** is the automated arrangement, coordination, and management of computer systems, middleware, and services.
>
>   **Kubernetes** is an orchestration system for **containers**.

*   Kubernets uses **manifest files** to describe a system’s entire architecture.

#### Why use Kubernetes instead of docker swarm?

**Docker Swarm**

*   Docker Swarm is simpler
*   It’s also built in to the docker engine

**Kubernetes**

*   has a steeper learning curve
*   Is more flexible
*   Started as a Google project

*   It’s more popular

# Docker

### Images

>   An **image** is a definition of a container.
>
>   A binary file that contains all of the software, environment variables, and general settings that the container will need to run.

*   https://hub.docker.com

### Containers

>   A **container** is a running instance of a docker image.

## Run Docker on Minikube

`minikube docker-env`

Export docker environment variables from qinikube

`eval $(minikube ducker-env)`

Test DOCKER_HOST variable

`echo $DOCKER_HOST`

List docker images

`docker image ls`

*   By using the docker process inside minikube
    *   We avoid running an extra process (more efficient)

#### Download image from Docker Hub

`docker image pull richardchesterwood/k8s-fleetman-webapp-angular:release0-5`

#### Run image in container

`docker container run -p 80:80 richardchesterwood/k8s-fleetman-webapp-angular:release0-5`

*   the pull is optional, because docker will automatically attempt to pull the image if it doesn’t already exist.
*   the port to expose to the outside world and the internal port to accept traffic is specified with the `-p` flag.
    *   `-p 80:80` exposes port 80 and routes to 80 internally.
    *   `-p 8080:80` exposes port 8080 and routes to 80 internally.
*   The container can be run in the background with the `-d` flag.

#### List running containers

`docker container ls`

*   We can see that the container is running, however when we type localhost:8080 in our browser we see nothing.

*   This is because the container is running in the minikube virtual machine, so we first need to find the ip of our vm.

#### Get ip of minikube virtual machine

`minikube ip`

Now we can navigate to port 8080 on our virtual machine in our browser to see our running application.

`192.168.99.100:8080`

#### Kill docker container

get first few characters from id using

`docker container ls`

then to stop the container type

`docker container stop [first characters of id]`

# Pods

>   a **pod** is a group of one or more containers, with shared storage/network, and specification for how to run the containers.
>
>   *    A pod models an application-specific “logical host”.
>   *   A pod’s contents are always co-located and co-scheduled, and run in a shared context.

*   A pod is the most basic unit of deployment in Kubernetes.
*   In most cases there will be only on container in a pod.
*   In some cases **side-car** containers are used as secondary help.

## Writing our first Pod

https://kubernetes.io/docs/reference/

Show everything defined in Kubernetes cluster

`kubectl get all`

create yaml file for **pod definition**

`code first-pod.yaml `

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: webapp
    image: richardchesterwood/k8s-fleetman-webapp-angular:release0
```

deploy new pod to the cluster

`kubectl apply -f first-pod.yaml`

**SchemaError:** invalid object doesn't have additional properties 

**kubectl** is running from docker which is preinstalled when we install Docker. You can check this by using below command

```
ls -l $(which kubectl) 
```

which returns as

>   /usr/local/bin/kubectl -> /Applications/Docker.app/Contents/Resources/bin/kubectlcode.

Now we have to overwrite the symlink with kubectl which is installed using brew

```
rm /usr/local/bin/kubectl

brew link --overwrite kubernetes-cli
```

(optional)

```
brew unlink kubernetes-cli && brew link kubernetes-cli
```

To Verify

```
ls -l $(which kubectl)
```

deploy new pod to the cluster

`kubectl apply -f first-pod.yaml`

Show new pod in cluster

`kubectl get all`

##### The pod is running, but pods are not visible outside of the cluster

*   So we can’t view the pod in our web browser

![image-20190508215936280](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190508215936280.png)

### Pod Commands

describe the pod

`kubectl describe pod webapp`

connect to the pod and execute a command

`kubectl exec webapp ls`

So we can also get a shell into the pod with an interactive terminal

`kubectl it exec webapp sh`

Then we can use wget to connect to the local web container and download the file being served

`wget http://localhost:80`

Now we can look at the contents of the html file

`cat index.html`

So we know that there is a functioning web server even though we can’t access it outside of the pod

# Services

>   Unlike pods, **services** are long running objects.
>
>   *   A service will have an ip address and a stable port.
>   *   Services use key:value pair **selectors** to select a pod inside the cluster by its **label**.

![image-20190508234042281](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190508234042281.png)

## Writing our first Service

restart minikube

`minikube start`

view everything in cluster

`kubectl get all`

get description of webapp pod

`kubectl describe pod webapp`

create yaml file for service definition

`code webapp-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp

spec:
  # This defines which pods are going to be represented by this Service
  # The service becomes a network endpoint for either orhter services
  # or maybe external users to connect to (eg. browser)
  selector:
    app: webapp

  ports:
    - name: http
      port: 80
      # targetPort: 80
      nodePort: 30080 # Port must be > 30000 so it doesn't clash

  # type: ClusterIP
  # a ClusterIP service will only be available from inside the cluster
  # (Not accessible by the browser) -> Private service
  type: NodePort
  # a NodePort exposes the service through the node to the outside world
```

deploy new service to the cluster

`kubectl apply -f webapp-service.yaml`

show new service in cluster

`kubectl get all`

get minikube ip

`minikube ip`

Test in browser

`192.168.99.100:30080`

The site is still not reachable because the pod does not have a label

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    # app: webapp
    mylabelname: webapp
spec:
  containers:
  - name: webapp
    image: richardchesterwood/k8s-fleetman-webapp-angular:release0
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp

spec:
  selector:
    # app: webapp
    mylabelname: webapp

  ports:
    - name: http
      port: 80
      nodePort: 30080 
  type: NodePort

```

reconfigure service

`kubectl apply -f webapp-servie.yaml`

reconfigure pod

`kubectl apply -f first-pod.yaml`

Test in browser again

`192.168.99.100:30080`

It works!

*   Labels can be any key value pair as long as it’s consistent

## Deploying a new version of our web app

*   To deploy another release with zero downtime
    1.  We can add a secondary label ‘’*release: 0*’’ to our first pod
    2.  Add the “*release: 0*” selector to the service
    3.  deploy another pod with our new release image
        1.  specify the second label as “*release: 0-5*”
    4.  Then, just change the service selector to “release: 0-5”

![image-20190509005359993](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190509005359993.png)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    app: webapp
    release: '0'

spec:
  containers:
  - name: webapp
    image: richardchesterwood/k8s-fleetman-webapp-angular:release0

---
apiVersion: v1
kind: Pod
metadata:
  name: webapp-release-0-5
  labels:
    app: webapp
    release: '0-5'

spec:
  containers:
  - name: webapp
    image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp
  
spec:
  selector:
    app: webapp
    release: '0'

  ports:
    - name: http
      port: 80
      nodePort: 30080
      
  type: NodePort
```

reconfigure the pod definition

`kubectl apply -f first-pod.yaml`

then reconfigure the service definition

`kubectl apply -f webapp-servie.yaml`

Show the new running pod

`kubectl get all`

Describe the service to show that it is still pointing to the original pod

`kubectl describe svc fleetman-webapp`

Change the service selector to “*release: 0-5*”

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp
  
spec:
  selector:
    app: webapp
    release: '0-5'

  ports:
    - name: http
      port: 80
      nodePort: 30080
      
  type: NodePort
```

reconfigure service

`kubectl apply -f webapp-service.yaml`

show running pods

`kubectl get pods` or `kubectl get po`

show running pods with labels

`kubectl get po --show-labels`

filter results with `-l` followed by selector

`kubectl get po --show-labels -l release=0-5`

## Adding our queue Service

rename webapp-service.yaml to services.yaml

`mv webapp-service.yaml services.yaml`

rename first-pod.yaml to pods.yaml

`mv first-pod.yaml pods.yaml`

Remove release0 pod definition and add queue pod definition to pods.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-release-0-5
  labels:
    app: webapp
    release: '0-5'
spec:
  containers:
  - name: webapp
    image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5

---
apiVersion: v1
kind: Pod
metadata:
  name: queue
  labels:
    app: queue
spec:
  containers:
  - name: queue
    image: richardchesterwood/k8s-fleetman-queue:release1
```

Add fleetman-queue service to services.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp
spec:
  selector:
    app: webapp
    release: '0-5'
  ports:
    - name: http
      port: 80 # nginx Web Server
      # targetPort: 80
      nodePort: 30080
  type: NodePort

---
apiVersion: v1
kind: Service
metadata:
  name: fleetman-queue
spec:
  selector:
    app: queue
  ports:
    - name: http
      port: 8161 # Apache MQ
      nodePort: 30010
  type: NodePort
```

Reconfigure all yaml files in directory with

`kubectl apply -f .`

```powershell
pod/webapp-release-0-5 unchanged
pod/queue created
service/fleetman-webapp unchanged
service/fleetman-queue created
```

# ReplicaSets

*   In a production system, you will very rarely work with pods directly
    *   You’re more likely to be working with **Deployments** or **ReplicaSets**

>   A **ReplicaSet** is an additional configuration setting that defines how many instances of a pod should be running at any given time.
>
>   *   If Replicas is set to 1, then every time a pod dies another will take its place.

*   The ReplicaSet api is a combination of a pod definition and a ReplicaSet definition
    *   So, you don’t need a separate pod definition
    *   The pod definition will be nested under `template:`

## Writing our first ReplicaSet

*   Kubernetes places no rules on yaml file structure
    *   We could have all of our pods, services, etc. in a single file
*   For this project all pods will be in one file and all services in another

convert pods.yaml to a ReplicaSet definition

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webapp
spec:
  selector:
    matchLabes:
      app: webapp
  replicas: 1
  template: #template for the pods
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5

---
apiVersion: v1
kind: Pod
metadata:
  name: queue
  labels:
    app: queue
spec:
  containers:
  - name: queue
    image: richardchesterwood/k8s-fleetman-queue:release1
```

delete all existing pods

`kubectl delete po --all`

check to see that all pods have been deleted

`kubectl get all`

deploy new ReplicaSet to the cluster

`kubectl apply -f pods.yaml`

describe webapp replicaset

`kubectl describe replicaset webapp` or `kubectl describe rs webapp`

remove release: '0-5’ selector from services.yaml and reconfigure

`kubectl apply -f services.yaml`

#### Simulate a pod crashing to test auto replication

`kubectl delete po webapp-[id]`

The ReplicaSet very quickly replicates the crashed pod, but there is still downtime.

To avoid downtime, we can increase our Replicas count to 2

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  replicas: 2
  template: #template for the pods
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5

```

Now when we delete a pod, there will still be a running instance.
So there is no downtime observed in our browser.

### Recap

Now we understand the need for a **Service**

*   A static networking endpoint for accessing a dynamic pool of pods
*   That pool of pods is defined and managed by the **ReplicaSet**

In many cases it is recommended to use a **Deployment** instead of a ReplicaSet.

# Deployments

>   A **Deployment** enables declarative updates for Pods and ReplicaSets.
>
>   It’s a ReplicaSet with automatic rolling updates ensuring zero downtime.
>
>   Another benefit is the ability to rollback updates if something goes wrong.

*   The API for Deployments is identical to ReplicaSets, but it has some additional fields.

## Writing our first Deployment

Stop the running ReplicaSet

`kubectl delete rs webapp`

Convert the pods.yaml ReplicaSet definition to a Deployment by just changing the type to Deployment.

`type: Deployment`

Apply Deployment definition

`kubectl apply -f pods.yaml`

Now when we run `kubectl get all` we see our new Deployment, but we also see a new ReplicaSet with a generated id on the end.

*   Similar to how ReplicaSets manages pods
*   Deployments manage ReplicaSets
*   The names of the deployments are now
    *   [Deployment name] - [ReplicaSet id] - [Pod id]

## Updating with rolling deployment

Now when we make a change to our Deployment and redeploy

1.  The Deployment automatically creates another ReplicaSet.
    1.  This ReplicaSet contains the updated image.
2.  When the new ReplicaSet is responding to requests the previous ReplicaSet won’t disappear, but the Replicas will be set to 0.
    1.  So if we want to revert to the original ReplicaSet we can just use a command to reset the Replicas to 2.
3.  The Deployment will then direct traffic to the new ReplicaSet.

change image to new release

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  minReadySeconds: 10
  selector:
    matchLabels:
      app: webapp
  replicas: 2
  template: #template for the pods
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5

---
apiVersion: v1
kind: Pod
metadata:
  name: queue
  labels:
    app: queue
spec:
  containers:
  - name: queue
    image: richardchesterwood/k8s-fleetman-queue:release1
```

Apply changes

`kubectl apply -f pods.yaml`

#### Updating with rollout

This command gives live updates while updating to new release

`kubectl rollout status deployment webapp` or
`kubectl rollout status deploy webapp` 

show rollout history 

`kubectl rollout history deploy webapp`

rollback to a previous deployment (*only use if you have to*)

`kubectl rollout undo deploy webapp —to-revision = 1` or 
`kubectl rollout undo deploy webapp`

Kubernetes will remember the last 10 ReplicaSets by default

If an image doesn’t exist, a ReplicaSet will be created but the status will be RollBackoff indefinitely unless deleted or the image is changed in yaml.

## Networking and Service Discovery

### How do we network containers together?

We could put both containers into a single pod, so we could use localhost.

![image-20190510102430358](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190510102430358.png)

This isn’t recommended, because it’s tougher to find points of failure.

*   Did the pod fail because of the application container or the database?

Most often we will deploy the application and the database in separate pods and expose their functionalities as separate services.

![image-20190510102902784](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190510102902784.png)

*   Each service is given it’s own private ip address
    *   So we can reference a specific ip address to access a service from another container.
    *   However, we aren’t going to know the ip address, because they are dynamically allocated by kubernetes
*   Kubernetes maintains its own private DNS service automatically
    *   So we can access services by their names instead of their dynamic ip addresses
    *   ![image-20190510103446931](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190510103446931.png)

The container can lookup the database in the dns service, so that it can access it directly using the databases ip address.

![image-20190510103838472](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190510103838472.png)

### Namespaces

>   A **Namespace** is a way of partitioning your resources.
>
>   By default, a resource is put into the default namespace if a namespace is not specified.

List all namespaces that have been defined on the system.

`kubectl get ns`

List all pods in the default namespace

`kubectl get po`

List all pods in the kube-system namespace.

`kubectl get po -n kube-system`

List all resources in cube-system namespace.

`kubectl get all -n kube-system`

There is nothing in tube-public.

`kubectl get all -n kube-public`

Describe kube-dns system.

`kubectl describe svc kube-dns -n kube-system`

### Service discovery

Open a shell into a running webapp pod

`kubect exec -it webapp-[replicaset id]-[pod id]`

print the resolution configuration file

`cat /etc/resolv.conf`

```powershell
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

The nameserver is the same as the cube-dns service ip address (ClusterIp)

```powershell
service/kubernetes  ClusterIP  10.96.0.1  <none>  443/TCP
```

Do a namespace lookup on google.com (external facing address)

`nslookup google.com`

Do a namespace lookup on an internal service fleetman-webapp

`nslookup fleetman-webapp`

This gives us the same ip of the service as when we use `kubectl get all`

*   So this shows how the dns service is routing to services given a service name.

# Microservice Architecture 

### Microservice vs Monolith

>   **Monolithic Architecture**: Entire system is deployed as a singe unit.

*   **Example**: a java application being deployed as a singe WAR file
    *   That application would access a single global database known as an **Integration Database**.
    *   The Integration Database may also be used by other monolith applications.

![image-20190510200604671](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190510200604671.png)

The big problem with monoliths is that they get bloated to the point that they become too big to handle.

*   A little change becomes a big change, and puts the rest of the system at risk.
*   One broken feature could break the whole application.
*   Development takes longer because of the communication needed and the extreme complexity involved.

>   **Microsystem Architecture**: System is a set of self-contained components or subsystems that can function, be deployed, and be developed independent of each other.

*   Microsystem components have no shared code, so they communicate with each other through well defined interfaces.
*   This modular approach allows for each subsystem to be developed in their own workspaces and on their own stand-alone hardware.

![image-20190510200653165](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190510200653165.png)

*   Ideally, each microservice would be on its own server, but since that would be very expensive it has become common practice to use **containers** that act as dedicated virtual servers.
*   Each microservice should be responsible for one and only one business requirement. 

*   A microservice should have only one reason to change.
    *   Highly Cohesive - All functions really belong together
    *   Loosely Coupled - Minimize interfaces between services

### What about databases?

*   Integration databases are a definite no!
    *   They aren’t cohesive, because they include information from many different business areas
    *   They are also not loosely coupled, because any part of the system can read and write to the database.
        *   Other systems might even be able to read and write to it.
    *   Even with multiple access roles, privileges, and multiple schemas there is still a major point of integration causing a lot of coupling.
*   We still need databases, but each microservice needs to have its own datastore.
    *   It should be the only part of the system able to read and write.

![image-20190510203233290](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190510203233290.png)

### Fleetman Microservice Architecture

![image-20190510205011026](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190510205011026.png)

*   There’s a transport company that needs to track its vehicles
*   Each vehicle is constantly reporting its location
*   The system is implemented in Java using Spring Boot.
*   The first microservice is called **Position Simulator**. It simulates the movement of vehicles to show functionality.
    *   When it starts it reads in a number of data files
    *   There is a file for each vehicle
    *   At a set interval the microservice will read from the files (that’s all it does)
    *   It must then hand that data to another service.
*   The read in positions are passed to a queue (**Active Mq**)
    *   Implemented as a single docker file that loads an image from Apache.org.
*   Next is the **Position Tracker**. The most important microservice.
    *   This service reads from the queue and performs calculations on them including the speeds.
    *   It also forms a repository to store the history of the vehicles in.
*   Between the frontend and the other microservices there is the **API Gateway** [*](https://microservices.io/patterns/apigateway.html).
    *   The API gateway is the single point of entry to the application.
    *   It accepts requests from the frontend and delegates them to the underlying microservices. 
    *   This isolates the frontend from any changes as the microservices inevitably need to be changed overtime.

## Deploying the queue

Delete all resources defined in pods.yaml

`kubectl delete -f pods.yaml`

Clear all running pods and services

`kubectl delete -f .`

>   **Workload**: Term used in Kubernetes for pods, deployments, replicates, etc.

Rename pods.yaml to workloads.yaml

`mv pods.yaml workloads.yaml`

The only complicated part about the queue is knowing which ports we need to expose.

*   **Image**: richardchesterwood/k8s-fleetman-queue:release1
*   The admin console is broadcast on port **8161**
    *   This can be exposed to the browser on port **30010**
*   Port **61616** is used to send and receive messages
    *   This will need to be declared in our kubernetes configuration
    *   But it won’t need to be exposed to browsers or the outside world.

*   If the queue goes down for any reason, we will definitely want it to be restarted as quickly as possible.
    *   This means we should use a ReplicaSet or a Deployment
    *   Since we may want to upgrade this service in the future, it will be best to use a Deployment for automatic rolling deployments.

Reconfigure workloads.yaml with queue Deployment

``````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  minReadySeconds: 10
  selector:
    matchLabels:
      app: webapp
  replicas: 2
  template: #template for the pods
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release0-5

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue
spec:
  selector:
    matchLabels:
      app: queue
  replicas: 1
  template:
    metadata:
      labels:
        app: queue
    spec:
      containers:
      - name: queue
        image: richardchesterwood/k8s-fleetman-queue:release1
``````

Deploy/apply workloads.yaml

`kubectl apply -f workloads.yaml`

Show all running resources

`kubectl get all`

Describe queue service using it’s pod name

`kubectl describe po queue-[replicaset id]-[pod id]`

*   We can see that the image has been pulled
*   The container has been created
*   The container has been started

Reconfigure services.yaml

``````yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-webapp

spec:
  # This defines which pods are going to be represented by this Service
  # The service becomes a network endpoint for either orhter services
  # or maybe external users to connect to (eg. browser)
  selector:
    app: webapp
    # release: '0-5'
    # mylabelname: webapp

  ports:
    - name: http
      port: 80 # nginx Web Server
      # targetPort: 80
      nodePort: 30080 # Port must be > 30000 so it doesn't clash

  # type: ClusterIP
  # a ClusterIP service will only be available from inside the cluster
  # (Not accessible by the browser) -> Private service
  type: NodePort
  # a NodePort exposes the service through the node to the outside world

---
apiVersion: v1
kind: Service
metadata:
  name: fleetman-queue
spec:
  selector:
    app: queue
  ports:
    - name: http
      port: 8161 # Apache MQ
      nodePort: 30010

    - name: endpoint
      port: 61616
      # nodePort: doesn't need to be exposed
  type: NodePort
``````

Deploy/apply services.yaml

`kubectl apply -f services.yaml`

*   We can see that the external port **30010** has been mapped to the internal port **8161** and the internal endpoint **31527** has been given a random port.
    *   `8161:30010/TCP,61616:30156/TCP`

*   This allows us to access the ActiveMQ dashboard by typing the relative address into our browser ([kubernetes vm address]:[open port] )
    *   `192.168.99.100:30010`
    *   login by clicking ‘***Manage ActiveMQ broker***’
        *   user: admin, pass: admin
    *   If we go to the **Queues** tab there are no queues, but there will be once we start our position simulator.

## Deploying the position simulator

*   **Image**: richardchesterwood/k8s-fleetman-position-simulator:release1
*   No ports needed - it’s isolated!
*   It needs an environment variable:
    *   `SPRING_PROFILES_ACTIVE` : `production-microservice`

This is a Spring Boot application

*   An application can run in different profiles
*   under /src/main/resources
    *   there are three properties files
        1.  application-local-microservice.properties
        2.  application-localhost.properties
        3.  application-production-microservice.properties
    *   At runtime Spring will read in one of those files depending on the environment variable that has been passed in.
    *   This allows us to easily change between settings files by changing the environment variable for different environments.

Define a deployment for the position simulator service in workloads.yaml

``````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-simulator
spec:
  selector:
    matchLabels:
      app: position-simulator
  template:
    metadata:
      labels:
        app: position-simulator
    spec:
      containers:
      - name: position-simulator
        image: richardchesterwood/k8s-fleetman-position-simulator:release1
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: producakkdfjagjhasgjh
        # resources:
        #   limits:
        #     memory: "128Mi"
        #     cpu: "500m"
        # ports:
        # - containerPort: <Port>

``````

Here the environment variable SPRING_PROFILES_AVTIVE has been misassigned. 

So when we attempt to apply our new Deployment

`kubectl apply -f workloads.yaml`

`kubectl get all`

We see that our position simulator pod is stuck at `ContainerCreating`

To better see what’s happening we can describe the position-simulator

`kubectl describe po position-simulator`

Under the Events we see that the container is failing to start with the reason `BackOff` and kubernetes continues in a loop trying to start the container.

### Inspecting Pod Logs

View logs for a specific pod

`kubectl logs [pod id]`

*   displays whatever the application is outputting
*   only pods issue logs, so we don’t have to specify type (ex. po)

Can add -f tag to freeze the console, so the screen will update as the log updates.

`kubectl logs -f [pod id]`

*   Besides the fact that the active profile shown doesn’t makes sense we can also see in the logs that Spring Boot encountered an exception.

    `Error creating bean with name 'journeySimulator': Injection of autowired dependencies failed; nested exception is java.lang.IllegalArgumentException: Could not resolve placeholder`

    *   So we know to look at the dependencies defined in our profiles, our environment variable being passed, or the code in the journeySimulator bean.
    *   In this case, we know it’s an incorrect profile in our environment variable.

Change value of SPRING_PROFILES_ACTIVE to production-microservice

`value: production-microservice`

Apply changes to workloads.yaml

`kubectl apply -f workloads.yaml`

`kubectl get all`

Now there is a new ReplicaSet with the correct application profile running.

Remove the stopped ReplicaSet using it’s id

`kubectl delete rs position-simulator-[replicaset id]`

Get logs on new replicaset to check that it is running correctly now

`kubectl logs position-simulator-[replicaset id]`

Now if we login to our ActiveMQ admin dashboard in our browser we should see a new queue called positionQueue.

`192.168.99.100:30010`

## Deploying the position tracker

-   The most important microservice so far is the position tracker
    -   The position tracker reads messages from the queue and exposes a REST interface so the clients can access the details of any vehicle.
-   **Image**: richardchesterwood/k8s-fleetman-position-tracker:release1
-   Port **8080** can be exposed to test the REST interface
-   environment variable needed:
    -   `SPRINT_PROFILES_ACTIVE` : `production-microservice` 

Define the position-tracker Deployment in workloads.yaml

``````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-tracker
spec:
  selector:
    matchLabels:
      app: position-tracker
  replicas: 1
  template:
    metadata:
      labels:
        app: position-tracker
    spec:
      containers:
      - name: position-tracker
        image: richardchesterwood/k8s-fleetman-position-tracker:release1
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
        # resources:
        #   limits:
        #     memory: "128Mi"
        #     cpu: "500m"
        # ports:
        # - containerPort: <Port>
``````

Apply changes to workloads.yaml

`kubectl apply -f workloads.yaml`

Show running resources

`kubectl get all`

Describe pod if there is an issue starting the Deployment

`kubectl describe po position-tracker-[replicaset id]-[pod id] `

Show logs with

`kubectl logs -f position-tracker-[replicaset id]-[pod id]`

Now when we check back in the ActiveMQ console we should see that the new position-tracker microservice is taking the messages off the queue just as quickly as they are being added to the queue.

*   The result should be zero pending messages and an equal number of messages enqueued and dequeued.

![image-20190528231716092](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190528231716092.png)

The developer tells us that we can also access the REST interface of the position-tracker microservice using the URI scheme */vehicles/{vehicle name}*. If we issue a get request to that URI we will get the current status data for that vehicle.

*   /vehicles/City Truck $\rightarrow$ /vehicles/City%20Truck

First we must identify a port for that service before we can request data from it.

Define the fleetman-position-tracker service in services.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-position-tracker
  
spec:
  selector:
    app: position-tracker

  ports:
  - port: 8080
    nodePort: 30020
  type: NodePort
```

Apply changes

`kubectl apply -f services.yaml`

Now we see our new position-tracker service mapped to port **30020**.

We can then request make a request using the /vehicles URI

`[kubernetes ip]:30020/vehicles/City Truck`

![image-20190528233758872](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190528233758872.png)

Then, to hide our service from the outside world we can change the service type from **NodePort** to **ClusterIp** and remove the **nodePort** value.

``````yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-position-tracker
  
spec:
  selector:
    app: position-tracker

  ports:
  - port: 8080
    # nodePort: 30020
  type: ClusterIP
``````

So the service is still being broadcast at port **8080**, but it isn’t accessible to the outside world.

## Deploying the API Gateway

-   **Image**: richardchesterwood/k8s-fleetman-api-gateway:release1
-   Port **8080** can be exposed to test the REST interface
    -   can test this port with **/**
-   environment variable needed:
    -   `SPRINT_PROFILES_ACTIVE` : `production-microservice` 

Define the api-gateway Deployment in workloads.yaml.

``````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: richardchesterwood/k8s-fleetman-api-gateway:release1
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
``````

Define the api-gateway Service in services.yaml.

``````yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-api-gateway
spec:
  selector:
    app: api-gateway # referring to the pod label
  ports:
  - name: http
    port: 8080
    nodePort: 30020
  type: NodePort
``````

The nodePort **30020** has been temporarily exposed for testing.

Apply changes.

`kubectl apply -f workloads.yaml`

`kubectl apply -f services.yaml`

List all running resources

`kubectl get all`

**Hide** external access to the api-gateway by changing the Service type to ClusterIp and removing the **nodePort** value.

``````yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-api-gateway
spec:
  selector:
    app: api-gateway # referring to the pod label
  ports:
  - name: http
    port: 8080
    # nodePort: 30020
  type: ClusterIP
``````

## Deploying the Webapp

*   **Image**: richardchesterwood/k8s-fleetman-webapp-angular:release1
*   Expose port **80** to the outside world!
*   Same environment variable:
    -   `SPRINT_PROFILES_ACTIVE` : `production-microservice` 

Just upgrade the release number to **release1** in workloads.yaml

``````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  minReadySeconds: 10
  selector:
    matchLabels:
      app: webapp
  replicas: 1
  template: #template for the pods
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release1
``````

Deploy changes.

`kubectl apply -f workloads.yaml`

View running resources.

`kubectl get all`

Visit the webapp in the browser on port **30080**

You may need to do a clear cache refresh (**Cmd+R**)

![image-20190529143624679](/Users/Evan/Documents/Spring_2019/Kubernetes Microservices/Introduction.assets/image-20190529143624679.png)

# Persistence

Without persistence, when a pod is destroyed all the data contained inside the pod is lost. However, just like in docker, we can store the data in a persistent volume such as a flat file or an actual hard drive in the cloud or on a local server.

### Release 2

*   We are asked to update all images to the **:release2** tag.
*   **New feature**: vehicle tracking now shows the history of each vehicle.

Modify all images in workloads.yaml with the **:release2** tag

``````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  minReadySeconds: 10
  selector:
    matchLabels:
      app: webapp
  replicas: 1
  template: #template for the pods
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue
spec:
  selector:
    matchLabels:
      app: queue
  replicas: 1
  template:
    metadata:
      labels:
        app: queue
    spec:
      containers:
      - name: queue
        image: richardchesterwood/k8s-fleetman-queue:release2

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-simulator
spec:
  selector:
    matchLabels:
      app: position-simulator
  template:
    metadata:
      labels:
        app: position-simulator
    spec:
      containers:
      - name: position-simulator
        image: richardchesterwood/k8s-fleetman-position-simulator:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-tracker
spec:
  selector:
    matchLabels:
      app: position-tracker
  replicas: 1
  template:
    metadata:
      labels:
        app: position-tracker
    spec:
      containers:
      - name: position-tracker
        image: richardchesterwood/k8s-fleetman-position-tracker:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: richardchesterwood/k8s-fleetman-api-gateway:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

``````

Apply changes.

`kubectl apply -f .`

Now when we click on a vehicle in the list we can see the path that it took. However, since the position tracker is responsible for saving the vehicle history, every time the position tracker service stops the history is lost.

*   A microservice should be responsible for one thing only
*   We should really split the position-tracker service into a history persistence service and a vehicle telemetry service.
*   Instead we are going to add a history database to persist past position data.

## Upgrading to a mongo pod

We need to setup a database service that matches the fully-qualified domain name defined in Release 3 of the position tracker service (fleet man-mongodb).

### Release 3

The position-tracker service in release 3 relies on a mongo database to persist data. So, when the new position-tracker service is deployed, it will continue trying to connect to the database until a database pod matching the fully-qualified domain name is deployed.

*   We want the deployment to succeed even if the position-tracker service won’t work until a database is deployed.
*   The services shouldn’t need to be deployed in any specific order since this isn’t a production server.
    *   If services need to be deployed in a specific order, then there is a coupling issue.

Upgrade position-tracker to release 3.

``````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-tracker
spec:
  selector:
    matchLabels:
      app: position-tracker
  replicas: 1
  template:
    metadata:
      labels:
        app: position-tracker
    spec:
      containers:
      - name: position-tracker
        image: richardchesterwood/k8s-fleetman-position-tracker:release3
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
``````

Apply changes.

`kubectl apply -f workloads.yaml`

The webapp will stop updating the vehicle positions.

Define a Deployment with a mongo database image from Docker Hub in a new file named **mongo-stack.yaml**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  replicas: 1
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:3.6.12-xenial
```

Deploy mongo database.

`kubectl apply -f mongo-stack.yaml`

We are asked to update all images to the **:release2** tag

**New feature**: vehicle tracking shows the history of each vehicle.

Update workloads.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  minReadySeconds: 10
  selector:
    matchLabels:
      app: webapp
  replicas: 1
  template: #template for the pods
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: richardchesterwood/k8s-fleetman-webapp-angular:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue
spec:
  selector:
    matchLabels:
      app: queue
  replicas: 1
  template:
    metadata:
      labels:
        app: queue
    spec:
      containers:
      - name: queue
        image: richardchesterwood/k8s-fleetman-queue:release2

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-simulator
spec:
  selector:
    matchLabels:
      app: position-simulator
  template:
    metadata:
      labels:
        app: position-simulator
    spec:
      containers:
      - name: position-simulator
        image: richardchesterwood/k8s-fleetman-position-simulator:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-tracker
spec:
  selector:
    matchLabels:
      app: position-tracker
  replicas: 1
  template:
    metadata:
      labels:
        app: position-tracker
    spec:
      containers:
      - name: position-tracker
        image: richardchesterwood/k8s-fleetman-position-tracker:release3
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: richardchesterwood/k8s-fleetman-api-gateway:release2
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
```

Apply changes

`kubectl apply -f workloads.yaml`

Define the mongo service in mongo-stack.yaml.

The port that needs to be exposed can be determine the port needed by viewing the logs for the new position-tracker pod. Or look at the documentation for the official MongoDB Docker image on [Docker Hub](https://hub.docker.com/_/mongo/) to find that the *standard MongoDB port* is **27017**.

``````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  replicas: 1
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:3.6.12-xenial

---
apiVersion: v1
kind: Service
metadata:
  name: fleetman-mongodb
spec:
  selector:
    app: mongodb
  ports:
  - name: mongoport
    port: 27017
    type: ClusterIP
``````

*   The type is set to ClusterIP, because we will not need to access our datastore outside of the cluster.

Apply changes to mongo-stack.yaml.

`kubectl apply mongo-stack.yaml`

We can now see in the browser that the paths are being saved even if we delete our position tracker pod and a new pod replaces it.

However, if we delete the pod that our mongo container is running in, the data will be lost.

We need a **persistent volume** outside of the container to store our position history permanently.

*   This is similar to Docker [**bind mounts**](https://docs.docker.com/storage/bind-mounts/) which are used to map/mount a file or directory on the *host machine* into a Docker container.

## Persistent Volumes

https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Managing storage is a distinct problem from managing compute. The `PersistentVolume` subsystem provides an API for users and administrators that abstracts details of how storage is provided from how it is consumed.

### Concepts

1.  #### PersistentVolume (PV)

    1.  A piece of storage in the cluster that has been provisioned by an administrator. 
    2.  A resource in a cluster just like a node is a cluster resource. PVs are volume plugins like Volumes, but have a lifecycle independent of any individual pod that uses the PV. 
        1.  This API object captures the details of the implementation of the storage, be that NFS, iSCSI or a cloud-provider-specific storage system.

2.  #### PersistentVolumeClaim (PVC)

    1.  A request for storage by a user.
    2.  Pods consume node resources and PVCs consume PV resource.
        1.  Pods can request specific levels of resources (*CPU and Memory*)
        2.  Claims can request specific size and access modes (*e.g., can be mounted once read/write or many times read-only*).

3.  #### StorageClass

    1.  An abstraction from the PersistentVolume resources that differ in more ways than just size and access modes.
        1.  This avoids exposing the users to the details of how the volumes are implemented.

### Mounting a Volume

Modify mongo-stack.yaml to mount to an external Volume.

``````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  replicas: 1
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:3.6.12-xenial
        volumeMounts:
          - name: mongo-persistent-storage
            mountPath: /data/db
      volumes:
      	- name: mongo-persistent-storage
      	  hostPath:
      	    # directory location on host
      	    path: /data
      	    # this field is optional
      	    type: DirectoryOrCreate
``````

Now the mount path has been set. 

However, the volume must be configured elsewhere to avoid definition cupeling across pods incase we want to change the volume later (*we will*).

First we will be configuring our container to use a pre-existing file on the host machine to persist data to. The file on the host machine will be directly exposed to the container using a **HostPath**.

### Persistent Volume Claim

