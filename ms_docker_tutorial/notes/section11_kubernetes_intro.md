## Kubernetes
- A framework that helps us with container orchestration and large scale deployments independent of the cloud provider we use
- Kubernetes (k8s) is open-source system for automating deployment, scaling and management of containerized applications. 

&nbsp;

## Problems with deploying containers
- Manually deploying containers like with our own EC2 instances - hard to maintain, error-prone, annoying
- Containers might crash or go down and need to be replaced with new containers on the EC2 instance
    - In this case, we want to replace that failed container with a new one, running our application again. 
- When manually running and deploying on EC2 we also have to manually monitor whether something crashed and we have to manually restart the containers afterwards
- We might need more container instances up on traffic spikes so we may need to scale up or scale down during low traffic
- Up to this point, we only executed every container once on EC2 and ECS, we only had 1 running instance of a container. 
    - We only ran one container per one image
    - But in reality we have multiple containers based on the same image
- For transforming image files for example which are uploaded to some folder, if more and more image files are coming in, container will take longer and longer to finish transforming all these files
- Spinning up multiple containers that work on transforming different files is a solution.
- We also need to ensure, incoming traffic is distributed evenly across the instances
- Multiple instances means, same container running multiple times. 
- These are big problems if we manually deploy containers


&nbsp;

## Why Kubernetes
- ECS helps with autoscaling, container health check
- Load balancer evenly distributes out traffic in addition to having only one DNS url. 
- ECS disadvantage is that we're locked into that service. This is where Kubernetes helps us. 
- With kubernetes we can define our deployment, scaling of our containers, how containers should be monitored and replaced if they fail independent of the cloud service we're using
- Kubernetes helps with automatic deployment, scaling, load balancing and management of our containers, replacing if a container goes down. 
- We write down a kubernetes configuration file like desired architecture, number of running containers etc, then pass that configuration to any cloud provider or to any machine owned by us. The machine then takes the configuration file instructions and create the resources and deployment specified by the file
- We can install some kubernetes software on any machine we own and that software can utilize this config file
- W/ kubernetes config file, we can migrate from one cloud provider to another one without changing the file much only the cloud provider credentials probably - there's a standard way of describing deployments
- Kubernetes is like docker-compose file for multiple machines. 

&nbsp;

## Kubernetes architecture and core concepts
- Kubernetes starts with a container which we want to deploy, managed by a pod. Pod simply holds a container and responsible for running this container
    - A pod can also hold multiple containers. 
- The pod then runs on a 'worker node'. Worker nodes run the containers of your application. 
- "Nodes" are your machines or virtual instances. So an EC2 instance is a worker node. 
- Worker node is a machine or computer somewhere w/ a certain amt. of CPU and memory and on that machine we can run our pods. 
    - Can have more than one pod running on the same worker node
- In addition to a pod on a worker node, kubernetes also needs a proxy 
    - proxy is a tool kubernetes sets up for you on a worker node to control the network traffic of the pods on that worker node. 
    - proxy controls whether these pods can reach the internet and how these pods can be reached from the outside world
- Working with Kubernetes we need at least 1 worker node otherwise there's no place to run your pods and containers. 
- For bigger applications, we'll have more than 1 worker node that runs it's own pods and proxy. Kubernetes will automatically distribute traffic equally across all worker nodes.  
- Someone needs to create and start these containers and pods and someone needs to shut them down or replace if they're failing or not needed anymore. This is done by the master node, the control plane which controls the worker nodes. 
- With kubernetes, we don't directly interact with the worker nodes and pods we let kubernetes and control plane do the heavylifting. We developers only define the end state. 
    - Master node which is another remote machine controls your deployments. 
    - For small applications, we can have the machine that's both master node and worker node. 
- The control plane is a collection tools or services. 
- The master node together with all the worker nodes form a cluster therefore, one network in which all the different parts are connected. 
    - Master node is then able to send instructions to a cloud provider API to tell that cloud provider to create it's cloud provider specific resources to replicate this big picture or end state on that cloud provider. 
    - ex. AWS creates all the required EC2 instances, a load balancer which might be needed, kubernetes tool runs on the master node which controls other EC2 instances. 

&nbsp;

## Kubernetes does NOT manage our infrastructure
- We need to provide an environment in which kubernetes is able to run
- We as developers have to set up the cluster and worker nodes and master node on which kubernetes is installed
- We as developers have to install all the Kubernetes software on these nodes that are part of our cluster, configure all these things kubernetes needs. 
- Depending on cloud provider, developers may also need to set up extra resources Kubernetes cluster may need like load balancer or file systems. 
- Kubernetes manages the pods, creates them, automatically distributes them, monitor and restarts them if they fail, manages running deployment, ensure it runs smoothly and stays up and running, and starts the containers for us. 
- Kubernetes is docker-compose for deploying containers. 

&nbsp;

## Worker nodes
- Inside the worker node we have pods that hosts one or more application containers and their resources like volumes, IP, 
- Pods are managed by kubernetes, kubernetes or master node can create or delete pods
- You can have one or multiple containers in a pod
- We usually have more than one pod on a worker node and it can be a copy of another pod if we're scaling up. Or it can be a totally different pod containing a totally different container to do another task. 
- Docker needs to be installed on the worker nodes computer machine. 
- Worker node also has the software kubelet running there which is the communication device between the worker node and master node. 
- Kube-proxy service on worker node handles incoming and outgoing traffic to ensure everything is working as desired and only allowed traffic is able to reach the pods 

&nbsp;

## Master node
- Most important service in master node is API server - API for kubelets to communicate
- scheduler service running on master node is responsible for watching for new pods and choosing worker nodes to run them on
- Kube-controller-manager service watches and controls the worker nodes overall and responsible for ensuring we have correct number of pods up and running 
- cloud-controller-manager which is doing same this as kube-controller-manager but cloud provider specific and knows how to tell that cloud provider what to do - translates instructions to AWS

&nbsp;

## Terms
- Cluster: A set of node machines which are running the containerized application or worker nodes or control other nodes like the master node
- Nodes: Physical or virtual machines with a certain hardware capacity which hosts one or more pods and communicates with the cluster.
- Master node: Control plane that manages the pods across worker nodes
- Pods: hold the actual running app containers and their required resources like volumes. So a shell around your container which starts and manages that container. 
- Services: A logical group of pods with a unique pod- and container-independent IP address. Services are important for reaching our pods and the containers inside them. Services expose pods to the outside world. Ensure certain pods are reachable from outside world with a certain IP address or domain. 