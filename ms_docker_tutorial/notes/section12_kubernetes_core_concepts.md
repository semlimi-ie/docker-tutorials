## Kubernetes does not manage your infrastructure
- You as the developer have to create the cluster and master and worker nodes yourself. 
- You setup the API server, kubelet and other software on the nodes. 
- On AWS you'd need to create your load balancer, EFS yourself, kubernetes doesn't do that. 
- Kubernetes only helps in managing and monitoring the pods, replacing failing pods, scaling pods, moving containers around between worker nodes and everything is up and running and reachable by end users. 
- Kubernetes only ensures your application is running 
- Kubernetes does not manage your machines only your pods.  
- Kubernetes creates your objects or pods and manages them - Kubernetes utilizes the resources you create ex. worker nodes and help you achieve your deployment goals for your containers. It won't create the resources for you though. 

&nbsp;

## Kubernetes setup and installation 
- To install Kubernetes and get started on our local machine, we need a cluster with a master node
    - Master node can be distributed across multiple machines to ensure it's always up and running. 
    - to keep simple we need one machine running in a cluster which has the master node set up and we need one or more worker nodes. 
    - We need all these services and software installed on the nodes. 
    - Master node: API server and schedulers should be installed. 
    - Worker node: Docker, kubelet should be installed so there's a communication device for talking to the master node etc. 
- Another toold we need locally on our machine is kubectl 
    - kubectl is a tool you use to send instructions to the cluster ex. to create a new deployment. 
    - w/ kubectl the developer is able to send instructions to the master node which will then do whatever it needs to do with the worker nodes. 
    - With kubectl, if you send instructions to create more pods of a give container, master node will interact with worker nodes to create extra pods. 
- The cluster is the infrastructure, kubectl is your communication device tool for talking with that infrastructure and for talking with this kubernetes setup. 
- If this was setup in the cloud, cluster would be what happens in the cloud and kubectl is a tool we install locally on our machine. 
- For demo, we'll set up a dummy cluster on our machine. 
- To set up dummy cluster locally, we use minikube. 
- minikube is a tool we install locally, it uses a virtual machine on my laptop to create a cluster there - so it simulates another machine on my machine. The virtual machine on my machine holds the cluster.  
- Minikube will create a single node cluster which means the worker node and master node is combined into 1 single virtual machine - not something that's done on production. 
- Install kubectl and minikub with homebrew and intall VirtualBox
```
brew install kubectl 
```

```
brew install minikube
```

- confirm the installation:
```
minikube start --driver=virtualbox
```

- After minikube virtualbox is started we have installed and started master node, worker node, software master and worker nodes need, we now have our local dev cluster we control with kubectl command, apply kubernetes configuration to run our containers inside this demo local cluster.
- After it's done running, confirm it's running with command:
```
minikube status
```

- Bring up a web dashboard of our cluster with this command: 
```
minikube dashboard
```

&nbsp;

## Kubernetes Objects
- Kubernetes works with Objects
- Objects include:
    - pods
    - Deployments
    - Services
    - volume
    - etc. 
- We can create objects by executing certain commands and kubernetes will take that object created by us and do something based on the instructions in that object. 
- These objects can be created in 2 different ways, imperatively or declaratively. 

&nbsp;

## pod object
- smallest unit Kubernetes interacts with 
- typically one container per pod but multiple containers per pod is feasible
- Tell kubernetes it should create a pod, run a container and do that on some worker node on a cluster by creating a pod object in code or with help of some command and send that object to kubernetes. 
- Pods don't only run containers, they hold shared resources like volumes 
- Pod has a cluster internal IP address used to send requests to that pod and the containers in it
- Pods can communicate with other pods. 
- Containers in the same pod can communicate with each other by using the local host address
- Pods are ephemeral, don't persist. If a pod is replaced or removed by Kubernetes, all resources in the pod like data stored and created by a container is lost. 
- Deployment or controller object in kubernetes creates the pods for us

&nbsp;

## Deployment object
- We don't create the pod objects on our own and manually move them to some worker node, we create a Deployment object which we then give instructions about the number of pods and the containers it should create and manage.   
- Deployment object can control one or more pods - so we can use it to create multiple pods at once
- For Deployment object, we set a desired target state ex. we have 2 pods with this container up and running then kubernetes does everything to reach this target state. 
    - Define what we want, kubernetes gets us there. So pod objects are then created by kubernetes, containers started by kubernetes, kubernetes places these pods on worker nodes which have sufficient memory and CPU capacity. Kubernetes picks a remote machine to place the pods for us. 
- Deployment object can also pause or delete deployments and roll them back and go back to previous deployment that worked
- Deployments can be scaled - we can tell Kubernets we now want to have more pods or less pods, we can also use the autoscaling feature where we set certain metrics, ex. incoming traffic, CPU utilization and if these metrics are exceeded increase the number of pods. And when traffic goes down, unnecessary pods will be removed again. 
- When we create deployments, we manage one pod with a deployment but we can also create multiple deployments to have multiple pods being managed by Kubernetes

&nbsp;

## Deployment using imperative approach
- We stil need to create docker images for an application since Kubernetes needs an image to run containers. 
- The only difference with Kubernetes is that we don't run our containers on our own anymore
- We could run the Dockerfile image locally but now we want to run it on the cluster

1. Build the image
```
docker build -t kub-first-app .
```

2. Send image to Kubernets cluster once image is built but as part of a pod or rather as part of a deployment which will then create the pod and the container running in the pod which will then be run and managed for us. 
- We first need to ensure the cluster is up and running before sending to cluster. 
```
minikube status
```

- If cluster is not running, we can restart it with 
```
minikube start --driver=virtualbox
```

- or could also run docker for restart:
```
minikube start --driver=docker
```

- List out kubectl help options:
```
kubectl --help
```

- Stop the minikube cluster
```
minikube stop
```

- Use kubectl to create a new deployment object - and send to the master node in the cluster
    - `kubectl create` will bring up documentation for everything you can create with kubectl like deployment object. 
    - Creating a deployment object also automatically sends the object to kubernetes cluster so no need to send on our end ex. `kubectl create deployment first-app --image=semie/kub-first-app`
    - We can't use the locally built image to send to kubernetes cluster, we have to send dockerhub image - image isn't taken from local machine and sent to cluster. Image is sent to cluster then the cluster looks for the image. And since the cluster runs in the virtual machine the image that's on our local machine but not on virtual machine will not be found. So we push the image to docker hub first.   

```
kubectl create deployment <deployment-name> --image=<remote image from registry to be used for the container of the pod created by this deployment>
```

- Get deployments sent to kubernetes cluster: kubectl automatically configured to connect to minikube. 
    - READY 0/1 means deployment didn't run the container yet so 1 of 1 deployment failed. 

```
kubectl get deployments
```

- Get all pods created by our deployments - READY 0/1 pod wasn't created but did get deployed.
```
kubectl get pods
```

- Delete a deployment, ex. `kubectl delete deployment first-app`
```
kubectl delete deployment <deployment-name>
```

- minikube dashboard will bring up dashboard and more info. about pods and deployments and can find IP address of the pod but it's only internal IP address cannot be reached by my local machine. 

- On the master node, the scheduler analyzes currently running pods and finds the best node for the new pods
    - and then the newly created pod will be moved to one of our worker nodes 
    - On that worker node, we have this kubelet service which manages the pod, starts the container in the pod, monitors the pod and checks it's health. 

&nbsp;

## Reach a pod with Service object
- Service object is another object Kubernetes knows, responsible for exposing pods to other pods or to visitors outside the cluster
- Pod's internal IP address changes whenever pod is replaced - so finding pods is challenging. 
- Service groups pods together and gives them a shared IP address that won't change
- We can also tell the service to expose this address both inside and outside the cluster so that pods can be accessed from outside the cluster. 

&nbsp;

## Expose a deployment with a service
- We can create a service with `kubectl service` command but a better command is `kubectl expose`
- `kubectl expose` exposes a pod created by a deployment by creating a service - `kubectl expose deployment first-app 
- the --type flag specifies type of service we're exposing. ex. `kubectl expose deployment first-app --type=LoadBalancer --port=8080`
    - type 'ClusterIP' is default and means only reachable from inside the cluster
    - type 'NodePort' means the deployment should be exposed w/ help of IP address of the worker node on which it's running - accessible from outside
    - type 'LoadBalancer' utilizes a load balancer and this load balancer creates a unique address for this service. It will also evenly distribute incoming traffic across all pods that are part of the service.   

```
kubectl expose deployment <deployment-name> --type=<type of service we're exposing> --port=<port of the app we wanna expose>
```

- Check that a service was created - EXTERNAL-IP is something provided by cloud provider
```
kubectl get services
```

- minikube command to get access to the service by mapping a special port to an IP which we can reach from our local machine and which identifies the virtual machine running on our local machine ex. `minikube service first-app` 
    - for cloud service provider you wouldn't need this command b/c they provide an 'EXTERNAL-IP' anyway

```
minikube service <service-name>
```

&nbsp;

## Restarting containers
- If a container crashes and errors it'll stop running and we can't see our app and `kubectl get pods` shows it's not running but after a few seconds that command will show that it's running if we go back to a good error-free page
- That's because with deployment it's automatically managed and scaled

&nbsp;

## Scaling
- If we don't have autoscaling in place, so kubernetes won't automatically scale up we can do it ourselves with ex. `kubectl scale deployment/first-app --replicas=3`
    - It means the same container and pod running how many times
    - Because we have a load balancer traffic will be distributed evenly across these pods
    - We can test scaling works by visiting an error page so it crashes, then immediately visiting another non-error page and it'll immediately show the functioning page without having to wait. 

```
kubectl scale deployment/<deployment-name> --replicas=<same pod running how many times>
```

- We can manually scale it down from 3 to 1 pod again: `kubectl scale deployment/first-app --replicas=1`

&nbsp;

## Updating deployments
- We can change our code, update our deployment or roll back deployment
1. Once code is changed rebuild the image - `docker build -t semie/kub-first-app .`
2. Push updated image to docker hub - `docker push semie/kub-first-app`
3. Update our deployment to take the new updated image
    - check first if the deployment is still there: `kubectl get deployments`
    - use the kubectl set image command to set a new image for a specific deployment `kubectl set image deployment/first-app kub-first-app=semie/kub-first-app:2`
    - can find current container name from dashboard under 'Containers'
    - But this won't update unless we build with a different tag then push with that different tag, ex. `docker build -t semie/kub-first-app:2 .` ; `docker push semie/kub-first-app:2`

    ```
    kubectl set image deployment/<deployment-name> <which current container and image to update>=<newly updated image from docker hub>:<new tag>
    ```
4. See the updated status of the deployment with: `kubectl rollout status deployment/first-app` - Can check on dashboard under 'pods' then 'Events'
```
kubectl rollout status deployment/<deployment-name>
```

&nbsp;

## Deployment rollbacks and history
1. If we set an image and deploy to the cluster with a tag that doesn't exist it'll try to update but fail: `kubectl set image deployment/first-app kub-first-app=semie/kub-first-app:3`
2. Inspect the progress of your rollouts to see that the above deployment failed: `kubectl rollout status deployment/first-app`
    - it tells you the old replica is terminating and we're waiting for new rollout to finish. Get out with `control + c` command
    - it doesn't shut down the old pod before the new pod is up and running

```
kubectl rollout status deployment/<deployment name>
```
3. So now we want to roll back this update - undo the latest deployment ex. `kubectl rollout undo deployment/first-app`
    - and check the problematic pod is gone with `kubectl get pods`
```
kubectl rollout undo deployment/<deployment name>
```

4. View history of all deployments - `kubectl rollout history deployment/first-app`
```
kubectl rollout history deployment/<deployment-name>
```

5. More details about a specific deployment w/ revision tag - `kubectl rollout history deployment/first-app --revision=3`
```
kubectl rollout history deployment/first-app --revision=<revision number>
```

6. Specify the specific revision we wanna roll back to - `kubectl rollout undo deployment/first-app --to-revision=1`
```
kubectl rollout undo deployment/first-app --to-revision=<revision-number>
```

7. Delete the service or worker node - `kubectl delete service first-app`
```
kubectl delete service <service-name>
```

8. Delete the deployment - `kubectl delete service first-app`
```
kubectl delete deployment <deployment-name>
``` 

&nbsp;

## Imperative vs. declarative approach
- Kubernetes allows us to create resource definition files with declarative approach. 
- Declarative approach makes use of this command: `kubectl apply -f config.yaml`
- The config yaml file is used to define our target state. 
- If we change the config file and re-apply it, Kubernetes looks at the file again to see what changed & make appropriate changes on our running cluster

&nbsp;

## Creating a deployment config file
- Make sure there are no current deployments with command: 
```
kubectl get deployments
```

- If there is a deployment running, delete it with:
```
kubectl delete deployment <deployment-name>
```

- Also make sure there are no pods with command: `kubectl get pods`
- Also make sure there are no services except the Kubernetes service with command: `kubectl get services`
- But make sure minikube is still running and dashboard is up. 

1. Make a new config file in the root of the project directory, name it any name you wannt ex. 'deployment.yaml'
    - need to define api version, google 'kubernetes deployment yaml' for `apiVersion`
    - `kind` specifies the kubernetes object we create which is 'Deployment' object or could be 'Service' object
    `metadata` defines name of deployment ex. 'first-app'
        - `name` defines name of deployment
    - `spec` is specification of deployment
        - `replicas` define number of pods to launch - equal pods containing the same container
        - `template` - pods that should be created as part of deployment. Kind of like --image flag
            - `metadata` 
                - `labels` is the name of pod with custom key and value like 'app: second-app'
            - `spec` - spec of pod inside replica template defines how pod should be configured
                - `containers` key specifies containers. We can list and have multiple containers per pod
                    - `- name` specifies name of our container
                    - `image` specifies name of image - needs to be image from a registry
        - `selector` of the deployment selects pods to be controlled
            - `matchLabels` - below it we define key value pairs of the pod labels we wanna match with this deployment - so the same key value pairs from the 'labels' section of the pods
            - Kubernetes sees and manages all the pods that are there in cluster. 

```
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: second-app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: second-app
      tier: backend
  template: 
    metadata:
      labels:
        app: second-app
        tier: backend
    spec: 
      containers: 
        - name: second-node
          image: semie/kub-first-app:2
        # - name: ...
        #   image: ...
```
2. Run our config file with - applies a configuration file to the connected cluster 
    - `-f=<config filename>` flag specifies the config yaml file we are using, `kubectl apply -f=deloyment.yaml`
```
kubectl apply -f=<config file name>
```

- Check that the deployment is running with `kubectl get deployments`
- Check the pod is running - `kubectl get pods`

&nbsp;

## Creating a Service object with config file
1. Make another yaml file with a custom name like, 'service.yaml' in the project root directory
    - `apiVersion` is just v1
    - `kind` - object kind which is `Service`
    - `metadata` data of the service object
        - `name` of service object
    - `spec` defines and configures the service
        - `selector` identifies which other resources should be controlled or connectd to this resource. Or which pods should be part of this service. Copy the pod labels from the deployment yaml file and copy them under this key. 
        - `ports` specifies ports to expose. Can define multiple ports in a list
            - `- protocol` by default is 'TCP'
            - `port` we want to visit on our local machine to access the app
            - `targetPort` - port our app specified in the code 
        - `type` - options are 'ClusterIP', 'NodePort' or 'LoadBalancer'

```
apiVersion: v1
kind: Service
metadata: 
  name: backend
spec:
  selector:
    app: second-app
  ports:
    - protocol: 'TCP' 
      port: 80
      targetPort: 8080
    #  - protocol: 'TCP' 
    #    port: 80
    #    targetPort: 8080
  type: LoadBalancer
```

2. Run our file with:
```
kubectl apply -f service.yaml
```

3. Now we expose the created service with ex. `minikube service backend`
```
minikube service <service name>
```

&nbsp;

## Updating and deleting resources
1. Make changes to the object yaml files, Deployment object or Service object, rerun the files and the changes will be reflected in the cluster. 
2. Ex. change the number of replicas to 3 in deployment.yaml file and rerun - `kubectl apply -f=deployment.yaml`
3. then `kubectl get pods` will show 3 pods running
4. Delete a resource, won't delete the file just the resource
    - Can delete imperatively with `kubectl delete deployment <name of deployment>`

```
kubectl delete -f=deployment.yaml
```
- Another way to delete multiple resources: `kubectl delete -f=deployment.yaml,service.yaml`
- Another way to delete multiple resources: `kubectl delete -f=deployment.yaml -f=service.yaml`

&nbsp;

## Multiple vs. single config files
1. To merge 2 or more files, make another file of custom name ex. 'master-deployment.yaml' in project root directory 
2. copy and paste the contents of deployment yaml file into the above newly created file 
3. Separate out this object configuration from the next object configuration ex. Service vs. Deployment object with `---` to indicate a new object is started. 
```
apiVersion: v1
kind: Service
metadata: 
  name: backend
spec:
  selector:
    app: second-app
  ports:
    - protocol: 'TCP' 
      port: 80
      targetPort: 8080
    #  - protocol: 'TCP' 
    #    port: 80
    #    targetPort: 8080
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: second-app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: second-app
      tier: backend
  template: 
    metadata:
      labels:
        app: second-app
        tier: backend
    spec: 
      containers: 
        - name: second-node
          image: semie/kub-first-app:2
```

4. Apply the newly created yaml file with merged objects: `kubectl apply -f=master-deployment.yaml`
5. Expose the service again with `minikube service backend`

&nbsp;

## matchExpression selector to select your pods in the deployment object
- alternative to matchLabels
- You have more configuration options for matchExpressions
- List of expressions in curly braces.
- In curly braces define key of pod labels with `key` and define values of pod labels with `values`, and values will be list of values in an array
- In curly braces `operator` means values are 'In', 'NotIn', 'Exists' and 'DoesNotExist' in the key of the pod.
```
selector:
  matchExpressions:
    - {key: app, operator: In, values: [second-app]}
```

1. Can delete also by selector using `-l` flag for label
    - To test it out first, add `labels` with custom key value pair in the parent metadata section of the deployment object:

```
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: second-app-deployment
  labels:
    group: example
```

2. Add the same `labels` to the Service object metadata 
```
apiVersion: v1
kind: Service
metadata: 
  name: backend
  labels:
    group: example
```

3. Delete deployments made by 'master-deployment.yaml' file - `kubectl delete -f=master-deployment.yaml` 
4. Apply the changed files: `kubectl apply -f=deployment.yaml -f=service.yaml`
5. Delete by label with the key value pairs specified in the metadata 'labels' section, we want to delete from object deployments and services which have the specified labes: 
```
kubectl delete deployments,services -l group=example
```

&nbsp;

## Liveness probes
- To configure how kubernetes checks if pods are healthy or not or change how the container is checked by the pod to see whether it's up and running 
1. Addd a `livenessProbe` key to to the template -> containers key in the Deployment object or deployment.yaml file
    - `httpGet` key specifies a get http request should be sent by the pod or Kubernetes to the running application, 
        - we wanna send it to a `path` of just / 
        - On `port` 8080 in our container since that's the port our container exposes
    - on the same level as httpGet we add a `periodSeconds` key with a value of 3 meaning how often this should be performed
    - Still same level as httpGet we add `initialDelaySeconds` key which tells kubernetes how long it should wait until it checks the health for the first time, we'll set it to value of 5 seconds 
```
  template: 
    metadata:
      labels:
        app: second-app
        tier: backend
    spec: 
      containers: 
        - name: second-node
          image: semie/kub-first-app:2
          livenessProbe: 
            httpGet:
              path: /
              port: 8080
            periodSeconds: 10
            initialDelaySeconds: 5
```

2. Apply the deployment.yaml and service.yaml files - `kubectl apply -f=deployment.yaml -f=service.yaml`
3. Bring up the application on a browser localhost port, we named the service 'backend' - `minikube service backend`
4. Now if we visit /error page it crashes but if we go back to homepage, it'll immediately restart 

&nbsp;

## configuration options
- Problem is when we updated our image it didn't pick up the image change unless we changed the image tag. It should've picked it up if we specified a tag of 'latest' then it'll always use the latest image. so we can configure that
1. In deployment.yaml file under containers key have a key called `imagePullPolicy` with a value of `Always` means always pull latest image if tag isn't specified
    - `Never` value means never pull latest image. 
2. Change something in the source code and build the image on that tag and push the image to docker hub on that tag
```
  template: 
    metadata:
      labels:
        app: second-app
        tier: backend
    spec: 
      containers: 
        - name: second-node
          image: semie/kub-first-app:2
          imagePullPolicy: Always
          livenessProbe: 
            httpGet:
              path: /
              port: 8080
            periodSeconds: 10
            initialDelaySeconds: 5
```

3. Before, kubernetes wouldn't pull the image again if the tag didn't change but now it will. 
4. Apply the deployment.yaml file it should pull that image - `kubectl apply -f=deployment.yaml`
5. And if you run `kubectl get pods` you'll see one container is being terminated and the new one is being started b/c it detected a change in the image so it's performing a rolling update - Now reloading the app will show the new changes

&nbsp;
## End the services and deployments:
- Delete the Deployment object and Service object by targeting the files
```
kubectl delete -f=deployment.yaml -f=service.yaml
```