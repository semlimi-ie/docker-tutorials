## Project structure
- project name: kub-network-01-starting-setup
- 1 cluster
- Within the cluster 1 pod contains the Auth api and the Users api
    - Users api communicates with Auth api so pod internal communication 
- 2nd pod contains the Tasks API
- client from outside world like postman or react app reaches out to the different api's in the pods in the cluster

&nbsp;

## Deployment object for Users api 
- We only deal with Users api for now, make sure it can be reached from the outside world. 
- Auth api is not exposed to the public according to docker-compose file so it can't be reached from outside world, only other containers are able to talk to it. 
1. Make sure minikube is running with `minikube status` and if not then run `minikube start --driver=virtualbox`
2. Make sure no current Kubernetes deployments or service running with `kubectl get deployments` and `kubectl get services` 
3. Replace axios calls to the Auth api from the users API with dummy text for now since we're not creating Auth api Deployment and Service object yet
4. On docker hub create a new repository for the Users api and call it 'kub-demo-users'
5. cd into the Users api project and build the image `docker build -t semie/kub-demo-users .`
6. Push the image to docker hub `docker push semie/kub-demo-users`
7. Create a new folder called 'kubernetes' in the root project directory and create a file called 'users-deployment.yaml' inside the kubernetes folder to configure the Deployment for Users api
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: users-deployment
spec: 
  replicas: 1
  selector:
    matchLabels:
      app: users
  template:
    metadata: 
      labels:
        app: users
    spec:
      containers:
        - name: users
          image: semie/kub-demo-users
```

8. cd into the kubernetes folder and create and Apply the Deployment object with `kubectl apply -f=users-deployment.yaml`
    - `kubectl get pods` will show the running pods

&nbsp;

## Service object for Users api 
- Services allow for stable IP address and allow for access to pods from outside world
1. In kubernetes folder make a file called users-service.yaml. LoadBalancer as type is used because gives access from outside world as well as distributing traffic equally across different pods. 
```
apiVersion: v1
kind: Service
metadata:
  name: users-service
spec:
  selector:
    app: users
  type: LoadBalancer
  ports: 
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

2. Apply the Service object with `kubectl apply -f=users-service.yaml`

3. Start the service network in kubernetes with the name of the service to get the url of the app - `minikube service users-service` 
4. Test the app with the url provided by minikube service

&nbsp;

## Multiple containers in one pod
- In the code for docker compose if Users api wants to reach out to Auth api, it references the container name for the Auth api to make the axios request. But for kubernetes we have to replace the container name in the axios request with the url name minikube service puts up for us. We can replace with environment variables

1. Remove the dummy text codes from the axios calls from the backend app.js file because we'll connect our Users api to the Auth api now
2. Replace the container name of Auth api in the axios calls with environment variables so that Users api uses the container name to reach out to the Auth api in docker-compose and uses the load balancer port url to reach out to the Auth api for kubernetes
ex. replace `await axios.get('http://auth/...'` with `await axios.get('http://${process.env.AUTH_ADDRESS}/...'`
3. In the `docker-compose.yaml file`, under 'users' service make an `environment` key and underneath add the `AUTH_ADDRESS` key with a value of 'auth' because that's the name of the Auth api service/container. And rebuild the Users api image and push to docker hub 
```
  users:
    build: ./users-api
    environment:
      AUTH_ADDRESS: auth
    ports: 
      - "8080:8080"
```

4. Build the Auth api image now and push it to docker hub and create a new docker hub repo for Auth api first, name the repo ex. kub-demo-auth
    - cd into the auth-api folder to build the image first ex. `docker build -t semie/kub-demo-auth .`
    - Push the image to docker hub ex. `docker push semie/kub-demo-auth`

5. Since Auth api is in same pod as Users api we don't create a new Deployment object w/ new deployment yaml file for the auth api but add to the users-deployment.yaml file a new container for the template with new image name and new 'containers' name
    - Also add 'latest' tag to both images so they always pull the latest image whenever a newly built image is pushed to docker hub
    - We don't add a Service object or service yaml file for auth b/c we are not exposing the Auth api to outside world
- `deployment.yaml` file
```
spec: 
  replicas: 1
  selector:
    matchLabels:
      app: users
  template:
    metadata: 
      labels:
        app: users
    spec:
      containers:
        - name: users
          image: semie/kub-demo-users:latest
        - name: auth
          image: semie/kub-demo-auth:latest
```

6. When one container needs to talk to another container within the same pod, it can refer to 'localhost' to communicate.
- So in the `users-deployment.yaml` file for the `users` container in the template, give a `env` key 
    - under `env` key -> list out a key of `name` with a value of `AUTH_ADDRESS` 
    - under name key have a key called `value` with a value of `localhost`

```
spec: 
  replicas: 1
  selector:
    matchLabels:
      app: users
  template:
    metadata: 
      labels:
        app: users
    spec:
      containers:
        - name: users
          image: semie/kub-demo-users:latest
          env:
            - name: AUTH_ADDRESS
              value: localhost
        - name: auth
          image: semie/kub-demo-auth:latest
```

7. cd into the kubernetes folder and apply the latest users-deployment.yaml file changes with `kubectl apply -f=users-deployment.yaml` then `kubectl get pods` will show 2 containers running in a pod. Test the minikube provided url with Insomnia to make sure everything is working properly. 

&nbsp;

## Create multiple deployments with separate pods in the same cluster - pod-pod communication
- Tasks API which is in different pod but same cluster has to access Auth api in the different pod
- Then we make a brand new pod for the Users api so in the end we have 3 different pods so 3 different deployments, 1 pod/deployment for Auth api, 1 pod/deployment for Users api and 1 pod/deployment for Tasks api.
- We need to ensure 3 separate pods can communicate with each other
- Auth api should have a service but it can not reachable from outside client, only reachable from other pods in the same cluster 

1. Make a new Deployment object for Auth api with a `auth-deployment.yaml` file in the kubernetes folder 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth
  template:
    metadata:
      labels:
        app: auth
    spec:
      containers:
        - name: auth
          image: semie/kub-demo-auth:latest
```

2. Since Users and Auth container no longer run in the same pod also remove the 'auth' container from the Users Deployment object or users-deployment.yaml file. 
    - So the IP address for this pod can change all the time if we scale it up or down for example
3. Add an `auth-service.yaml` file in the kubernetes folder 
    - for service `type` we change it to `ClusterIP` rather than LoadBalancer b/c LoadBalancer exposes pod to outside world outside of cluster and ClusterIP does automatic load balancing but it can be reached from inside our cluster
    - Now for Users api to reach Auth api with axios call the AUTH_ADDRESS for url address cannot be localhost anymore b/c they're not in the same pod anymore. It can be reached only through it's internal IP address. 
- `auth-service.yaml` file
```
apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  selector:
    app: auth
  type: ClusterIP
  ports: 
    - protocol: TCP
      port: 80
      targetPort: 80
```

4. Get the internal IP address of the separated Auth pod and it's service. 
    1. apply the Auth app's deployment file and service file `kubectl apply -f=auth-deployment.yaml -f=auth-service.yaml`
    2. Run `kubectl get services` and get the CLUSTER-IP of the 'auth-service' from the response and paste it into the users-deployment.yaml file under the `env` key for the value of `value` ex. '10.106.190.87' 
```
spec: 
  replicas: 1
  selector:
    matchLabels:
      app: users
  template:
    metadata: 
      labels:
        app: users
    spec:
      containers:
        - name: users
          image: semie/kub-demo-users:latest
          env:
            - name: AUTH_ADDRESS
              value: "10.106.190.87"
```

    
5. Apply the users-deployment file change with `kubectl apply -f=users-deployment.yaml` and `kubectl get pods` will show 2 pods rather than 2 containers in 1 pod now.
    - Test by making a request from the Insomnia client and see it's all working. 
6. A more convenient way to get the internal pod address of auth-service is that kubernetes gives automatically generated environment variables with information about all the services which are running in the cluster like IP addresses
    1. In the backend endpoint code in users-app.js file where axios calls are being made, change to the service name of the service/container we're calling out to ex. 'auth-service', but now in all caps and hyphens replaced by underscore followed by underscore and then SERVICE_HOST so <SERVICE_NAME_SERVICE_HOST>, ex. `AUTH_SERVICE_SERVICE_HOST` which is now an internal Kubernetes env variable
    2. Replace the AUTH_ADDRESS variable in docker-compose.yaml file with kubernetes generated environment variable 
7. Rebuild the users-app image and push to docker hub and delete the old users-deployment.yaml file and reapply because nothing changed in that deployment file test again with Insomnia client to make sure it is working
```
kubectl delete -f=users-deployment.yaml
```

```
kubectl apply -f=users-deployment.yaml
```

&nbsp;

## Using DNS for pod-pod communication 
- Kubernetes clusters come with a built-in service called core DNS which is a domain name service which helps to create domain names - cluster internal domain names
- cluster internal domain names are not domains you could enter into the browser b/c they won't be registered with global domain name services but domains known inside of your cluster. 
- Just like IP addresses are known inside of your cluster there are domain names known inside your cluster
- Core DNS installed inside of the cluster automatically creates domain names which are avialable

1. In `users-deployment.yaml` file under `containers` -> `env` -> `value` replace the IP address with automatically generated domain in cluster which is your service name ex. `"auth-service.default"`
    - so this will be the address which can be used in the code running in your cluster to send requests to other services of that cluster
    
```
  template:
    metadata: 
      labels:
        app: users
    spec:
      containers:
        - name: users
          image: semie/kub-demo-users:latest
          env:
            - name: AUTH_ADDRESS
              value: "auth-service.default"
```

2. You can get the default namespace with the following command which will give a 'default' as one of the 'NAME' in the response to that command
```
kubectl get namespaces
```

3. Apply the changes made to users-deployment.yaml file - `kubectl apply -f=users-deployment.yaml`
4. Now making a request to the url provided by minikube should work as long as the axios is referring to the name of the env which in this case is AUTH_ADDRESS

&nbsp;
- In most cases you don't want to have 2 containers in the same pod

&nbsp;

## Create and connect Tasks pod in the cluster
- Start up minikube and apply the previous Users and Auth deployments and services object with their respective yaml files with kubectl apply -f... command

1. Make a docker hub repo for the tasks-api call it 'kub-demo-tasks'
2. cd into the tasks-app folder and build the image with `docker build -t semie/kub-demo-tasks .`
3. Push the image to docker hub with `docker push semie/kub-demo-tasks`
4. Createa Deployment object for the tasks app api with a file called `tasks-deployment.yaml` in the kubernetes folder
- For the value of AUTH_ADDRESS env variable get it by running `kubectl get namespaces`
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tasks-deployment
spec: 
  replicas: 1
  selector:
    matchLabels:
      app: tasks
  template:
    metadata:
      labels:
        app: tasks
    spec:
      containers:
        - name: tasks
          image: semie/kub-demo-tasks:latest
          env:
            - name: AUTH_ADDRESS
              value: "auth-service.default"
            - name: TASKS_FOLDER
              value: tasks
```

5. In the tasks-app.js backend api endpoint code, replace the axios call referring to 'auth' container with `await axios.get('http://${process.env.AUTH_ADDRESS}/verify-token/' + token);` so add `process.env.AUTH_ADDRESS` which will have a value of the Kubernetes internal core DNS address for auth service which is just the <nameOfAuthService.default> and rebuild the image and re-push.
6. In the docker-compose.yaml file make sure to add the AUTH_ADDRESS environment variable with a value of the auth container name under the tasks container `environment` key so that if we run the app with docker-compose it can reach out to the container for Auth api
```
  tasks:
    build: ./tasks-api
    ports: 
      - "8000:8000"
    environment:
      TASKS_FOLDER: tasks
      AUTH_ADDRESS: auth
```

7. Create a Service object for the tasks app api with a file called `tasks-service.yaml` in the kubernetes folder
```
apiVersion: v1
kind: Service
metadata:
  name: tasks-service
spec: 
  selector:
    app: tasks
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
```

8. Apply the tasks Deployment and Service objects with `kubectl apply -f=tasks-deployment.yaml -f=tasks-service.yaml`
9. Start up the tasks service with minikube to get a browser port url to test out with outside clients like Insomnia or React app - `minikube service tasks-service`

&nbsp;

## Add containerized frontend - Run with docker only not Kubernetes
1. Paste the url we get back from `minikube service tasks-service` command into the fetch requests of the frontend
2. cd into the frontend service and build the docker image with `docker build -t semie/kub-demo-frontend .`
3. Run docker command `docker run -p 80:80 --rm -d semie/kub-demo-frontend`
  - 'nginx.conf' file runs frontend on port 80 and the Dockerfile exposes port 80 so we use -p 80:80 and we pull up the browser frontend on localhost:80
4. Stop the container when done using `docker stop <container-name>` ; get docker containers that are running with `docker ps` first. 

&nbsp;

## Deploy frontend to kubernetes
1. In the 'kubernetes' folder of the project directory, add a file called `frontend-deployment.yaml` file
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: semie/kub-demo-frontend:latest
```

2. We want users to be able to reach our frontend so in the kubernetes folder also add a `frontend-service.yaml` file
```
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

2. Make a docker hub repository for the frontend app call it 'kub-demo-frontend' and push the built frontend image to that dockerhub `docker push semie/kub-demo-frontend` 
3. cd into kubernetes folder and apply the frontend deployment object and service object with `kubectl apply -f=frontend-deployment.yaml -f=frontend-service.yaml`
4. Expose the frontend service to minikube with `minikube service frontend-service` to get a url provided by kubernetes minikube running on your browser  

&nbsp;

## Using a reverse proxy for the frontend
- Hardcoding a port url into the frontend fetch request is a bit clunky. So we use a reverse proxy
  - proxy means we wanna send a request to ourselves aka to the server which serves our frontend application so in our case to the nginx server which we've set up and start in our Dockerfile. 
  - We want to send a request to ourself
  - Typically when a request reaches that server our frontend application is served 
  - We dive into our nginx.conf file in the conf folder in the frontend project directory to generally serve our frontend but also redirect requests to some other host or other domain if they have a certain structure or target a specific path - `reverse proxy`
1. Enable reverse proxy by going into the frontend `nginx.conf` file in the conf folder, & b/t listen 80 and location / .. add 
```
location /app {

}
```

  - `location /app {}` means that I now wanna set up a dedicated configuration for requests reaching this nginx server where the path starts with '/api' And if it starts with anything else the other location - 'location /' will kick in instead and serve my build react application. 

2. Inside the `location /api {}` object, include an instruction nginx understands which is `proxy_pass`
  - proxy_pass tells nginx that the request to /api should be frowarded to some other address. 
  - Next to proxy_pass we grab and put the auth-service url we got from minikube/kubernetes found in the frontend fetch request
  - This means requests to the server '/api' wlll instead be forwarded to that url for auth service provided by kubernetes/minikube - So they won't be handled by nginx server but rather to that other /api server to which they're passed. 

```
  location /api {
    proxy_pass http://192.168.99.100:32182;
  }
```

3. Go to the frontend code where the fetch/axios requests are being made to the auth api and change the auth-service url that was provided by kubernetes/minikub to instead '/api' so whatever server is on that kubernetes/minikube provided url will be masked as '/api'
```
 fetch('/api/tasks', {...
 }
```
4. Rebuild the image and re-push to docker hub after cd into frontend project. 
5. cd into kubernetes folder and delete the frontend deployment and service and reapply them with kubectl apply command
6. Now on frotend it'll fail to fetch tasks because in the nginx.conf file we are redirecting requests to the auth api url provided by kubernetes/minikube but the problem with that IP url is that it's an IP which we can use on our local machine to access the cluster that's spun up and managed by minikube but the configuration on the ngix.conf file will not be executed on our local machine. 
  - the nginx.conf configuration is instead executed on the server, it will run inside of the container - parsed when the server starts which is inside of the cluster. 
  - So for reverse proxy of proxy_pass we can use cluster internal IP addresses or also use our automatically generated domain names so replace the kubernetes/minikube generated url with the auto-generated tasks service domain name `tasks-service.default` and include a trailing '/' after api and after default to ensure correct path is targeted

```
  location /api/ {
    proxy_pass http://tasks-service.default/;
  }
```

7. Also since our backend tasks app starts on port 8000 and we're exposing port 8000 on docker and on kubernetes service add the port number after a colon to the proxy url
```
server {
  listen 80;

  location /api/ {
    proxy_pass http://tasks-service.default:8000/;
  }
  ...
}
```
8. Rebuild the frontend image and re-push to dockerhub 
9. cd into kubernetes folder and delete and reapply the frontend Deployment and Service objects with kubectl apply command. 
10. Start up the service with a url IP in minikube with `minikube service frontend-service` command and the frontend will work now. 