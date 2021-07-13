- We shouldn't use Bind mounts in production!
- Some apps require a build step before going to production. 

## AWS EC2
- Allows you to spin up your own hosting machine in which you can download and install your own custom software tools 
1. Create and launch EC2 instance, virtual public cloud and secuity group to control who has access to the instance.
2. Configure security grou to expose all required ports to the web so we can have incoming traffic to this EC2 instance
3. Connect to instance via SSH, install Docker and run container - terminal based approach allows us to run commands on that remote computer

## Build app 
1. Dockerfile is already there. So just build the node image using the terminal navigated to the project directory
```
docker build -t node-dep-example .
```
2. Run a container named node-dep based on that image 
```
docker run -d --rm --name node-dep -p 80:80 node-dep-example
```

&nbsp;

## Bind mounts in production
- No volumes or bind mounts for production -
- In development we are encapsulating the environment but not the code because we're fine if the code is coming from outside the container
- For production, we need standalone so it doesn't have any surrounding setup on the remote machine - image is the single source of truth. We don't need to move source code onto the container b/c that container and image should have everything we need. 
    - For building in production we use COPY to copy the code snapshot into the image

&nbsp;

## Intro. to AWS EC2
1. sign up and make an account on aws.amazon.com
2. Go to AWS Management Console and type in EC2 in search bar
3. Click 'Launch Instance' to launch a new cloud based computer
4. Choose 'Amazon Linux 2 AMI' -> Select -> t2.micro -> Configure Instance Details
5. Click 'Review and Launch' -> 'Launch' button
6. Choose 'Create a new key pair' -> type in a custom 'Key pair name' like 'sem-example' -> 'Download Key Pair'
7. Put the downloaded key file into the project root directory but DON'T share with anyone -> click 'Launch Instances' -> click 'View Instances' and wait till Instance State is no longer pending 
- Now we'll connect our local machine to the cloud server via SSH using terminal 
8. Click 'Connect' -> choose 'A standalone SSH client' radio option -> 'SSH Client'
9. Open terminal and paste in this code then run: `chmod 400 sem-example.pem` 
10. Then paste this in terminal and run: `ssh -i "sem-example.pem" ec2-user@ec2-18-116-71-81.us-east-2.compute.amazonaws.com` -> type 'yes' 
11. You're connected when your terminal has something like '[ec2-user@ip-172-31-19-112 ~]$'

## Install Docker on our above virtual EC2 machine
12. In our connected terminal where our local machine terminal is connected to EC2 machine, type in `sudo yum update -y` and run
13. Then type into the terminal, `sudo amazon-linux-extras install docker` to install docker on the remote machine -> y
14. Type into the terminal, `sudo service docker start` to start docker there so now we can run docker commands in that terminal

## Bring up our image from our local machine to the AWS remote machine
- Option 1 - deploy our source code to remote machine then build it there - More complex b/c lot of unnecessary work on remote machine
- Option 2 - build our image on local machine then deploy to AWS EC2
- So we build our image locally, then push to Docker Hub then pull from remote server and run it on remote server
- So to pull and run from docker hub
15. Log in to docker hub -> 'Create Repository' -> type in a name for the repository, 'node-example-sem-1'  -> click 'Create'
16. Now push our source code from our local machine to the newly created docker hub repo by opening a new terminal that's not connected to EC2
    - Make a .dockerignore file in the project root directory and put in these file names: node_modules, Dockerfile, *.pem
Now build the image w/
```
docker build -t node-dep-example-1 .
```
17. Push the image to docker hub w/ a new tag
```
docker tag node-dep-example-1 semie/node-example-sem-1
```
then,
```
docker push semie/node-example-sem-1 
```

## Run and publish the app on EC2
18. Go back to the EC2 connected terminal and run docker with our remote docker image name
```
sudo docker run -d --rm -p 80:80 semie/node-example-sem-1
```
- Check the container on remote EC2 is up and running with 
```
sudo docker ps
```
19. To test our app on the remote EC2, get the 'Public IpV4 address'. We can't connect to that address yet from our local machine b/c security group rules ensure no one else can access that EC2 instance. So set up Security Groups from the side menu on AWS running instance dashboard 'Instances'
    - Click on the specific 'Security group ID' 
        - where outbound rules control which traffic is allowed from instance to somewhere else. We have everything allowed so Docker in EC2 was able to download from dockerhub. 
        - Inbound rules allow traffic from somewhere in the world to come to our instance - only 1 port, SSH port 22 is open, so anyone in the world can connect to our instance via port 22 that's why the key file is very important b/c anyone can start an SSH connection to our instance but only I w/ valid key can do so successfully. Or I can also narrow down the source to my local machine IP address. 
        - But it's not just port 22, but we need to allow http traffic to go into this instance like port 80 for node app which is being blocked
    - Click on 'Edit Inbound rules' under 'Inbound rules' tab -> 'Add rule' -> Choose 'http' for type and 'Anywhere' for source, 'port' can be 80 -> 'Save rules'
    - Now enter the public IpV4 address in the browser again to see your node app 
    - We didn't even need to install Node on our EC2 instance, it was all done inside docker inside our EC2 instance

20. To update the code on our remote EC2 server, make a change in the node app code but it's not automatically reflected on the EC2 instance
    - To reflect the changes we can just rebuild on our local machine, push the updated code to docker hub, and use the latest code from the image on our remote server

```
docker build -t node-example-sem-1 .
```

- tag it appropriately,
```
docker tag node-example-sem-1 semie/node-example-sem-1
```

- push rebuilt image to docker hub
```
docker push semie/node-example-sem-1
```

- Now switch back to our terminal connected to our EC2 instance and stop our previously running container
```
sudo docker stop determined_hellman
```

- Pull the latest image from docker hub
```
sudo docker pull semie/node-example-sem-1
```

- Rerun our node app again to bring it up again to see the updated changes 
```
sudo docker run -d --rm -p 80:80 semie/node-example-sem-1
```
21. To stop the EC2 instance, stop docker containers and remove then go to the 'Instances' dashboard -> 'Actions' -> 'Instance State' -> 'Terminate'

&nbsp;

## Disadvantages of our current app
- We had to create the instance manually, configure it manually, connect to it manually, install docker on it manually
- We fully own this machine, including configuration of security, we have to take care that it's powerful enough to handle all our traffic, replace w/ more powerful one if it fails, ensure all system software stays updated, have to keep operation system up to date, manage it's network, it's security groups, firewall
- We want remote host to be automatically managed where ECS, Elastic Container Service comes in

&nbsp;

## ECS managed service 
- Creation, management, updating is handled automaticcaly, monitoring and scaling is simplified 
- But you have to use only services provided by ECS and follow their rules not your own. 

&nbsp;

## Deploying with ECS 
- Thinks in 4 categories - Clusters, containers, tasks, services
1. Go to ECS service from AWS Management Console -> 'Get started'

2. Click 'Configure' for 'Custom' container to specify how ECS should execute docker run like 'Container name' on ECS is '--name' flag on docker run which you can choose interchangeably.  

3. Type in 'node-demo' for 'Container name' -> type 'semie/node-example-sem-1' for 'Image' -> type '80' for 'Port mappings' to expose port 80 to our container
- For 'Environment' option we can overwrite default 'Entry point' and 'Command' for when it starts. Can also specify 'Environment variables. ' Container Timeouts specify when AWS should stop trying to laucnch the container. 'Storage and Logging' defines volumes. 'Log configuration' being checked allows for logging of the container info when we bring it up 
- 'Working directory' option lets us overwrite the working directory path we specified to be used in the docker container eg. overwrite '/app' in docker. Can also do this in the docker run command with the --workdir flag. We leave it empty
- 'Storage AND Logging' option on ECS allows us to specify volumes and mount points equivalent to -v flag on docker run command. We won't use that for this small demo app right now. 
- Check the 'Log configuration' checkbox to allow for logging of docker terminal info. when we bring up environment. 
- Now click 'Update' on dashboard 

4. Now define 'Task definition' which is blueprint of our application - tell AWS how it should launch my container. Task is how the server that runs my docker container should be configured. Task can include multiple containers so it's one remote server that runs one or more containers. 
- FARGATE launches container in serverless mode so it's not an EC2 instance. It stores my containers and run settings and whenever there's a request that requires the container to do something, it starts the container up, handles that request then stops the container so cost-effective. 
- Click 'Next' for 'Task definition' 

5. 'Define your service' - Service controls how the task or the configured application and container that belongs to it should be executed.  
- In services, you can add a load balancer which does the heavy lifting of redirecting the incoming requests to the up and running containers behind the scenes 
- We have 1 service per task. 
- Click 'Next for 'Define your service'

6. 'Configure your cluster' - overall network in which our services run. For now we just have 1 service, with 1 task with 1 container. But for multi-container apps, we could group multiple containers in 1 cluster so they can all talk to each other and belong together. 
- cluster network is automatically created and configured for us so we just click 'Next' 

7. Get to 'Review' page -> click 'Create' to create and launch container

8. Click 'View service' once container has launched. 
- See our running application by clicking on 'Tasks' tab -> click on the 'Task' id of the container. 
- After clicking on task id, 'Details' tab will have a 'Pulic IP'. Pasting that 'Public IP' address into my browser will show my running application. 

9. How to update our container and reflect those changes on the running server. 
- Change our source code a little 
- docker build again from my terminal 
- docker tag it again to replace docker hub image with my locally updated image `docker tag node-example-sem-1 semie/node-example-sem-1`
- push the updated image to docker hub `docker push semie/node-example-sem-1 `
- Now how do we make ECS aware of this updated image? it's not automatically reflected. To do that go to 'Clusters' from top -> 'default' -> 'Tasks' tab -> the value of 'Task definition' -> 'Create new revision' -> 'Create' to create new image based on the newly pushed docker hub image -> cick on 'Actions' dropdown -> choose 'Update service' -> 'Skip to review' -> 'Update service' -> Now 'View service' to view the service. 
- Now under 'Tasks' tab we'll see the newly revised task -> so click on the newly revised task id -> get the new 'Public IP' -> paste it into our browser to see the latest changed code. 

10. Delete our running container 
- Click on 'Cluster' from top again -> 'default >' -> check the 'Service Name' -> click 'Delete' -> type in 'delete me' -> click 'Delete' -> click on 'Delete Cluster' -> type in 'delete me' -> click on 'Delete' -> Takes a couple minutes to delete. 

&nbsp;

## Preparing a multi-container app with ECS
- We won't use docker compose here for deployment b/c when deploying we may need to define how much CPU service should be provided for our apps etc. So it's great when running on just local machine but compose is too complicated when remote host servers run different apps on multiple different server machines. 
- But we just use docker-compose as inspiration and blueprint for how we set up our AWS service and manually deploy these services to AWS ECS. 
- We can't use the container/service name `monogdb` in our backend app to connect to MonboDb because on our local machine it's the same machine but on AWS it could be different server machines. so replace `mongodb` in app.js file with `localhost` 
    - But if my containers are added in the same task in ECS, then they are guaranteed to run on the same machine. Still, ECS won't create a Docker network for them instead it allows me to use localhost as an address inside of my container application code. ECS give us access to the network on that machine and to the containers part of that task through localhost.  
    - In backend Dockerfile add `ENV MONGODB_URL=mongodb`
1. Build the backend image
```
docker build -t goals-node ./backend/
```
2. To push the created backend image to docker hub create a docker hub image first using the same name as the local built image, 'goals-node'
3. Re-tag the locally built image with the [docker hub username]/[docker hub image-name] to bring the image from docker hub to local machine
```
docker tag goals-node semie/goals-node
```
4. Push the newly tagged image to docker hub
```
docker push semie/goals-node
```
5. Create an ECS cluster on AWS for and add backend container
    1. Go to ECS Cluster page -> click 'Create cluster' 
    2. Choose 'Networking only cluster' -> 'Next step' . We don't choose networking with windows or linux EC2 for this.  
    3. type a 'Cluster name' of my choice like 'goals-app'
    4. Check 'Create VPC' which ensures AWS creates a private cloud for all the containers in the cluster -> click 'Create'
    5. click 'View cluster'. Now we'll add 'Tasks' and 'Services here from the tabs options. Services are based on tasks so crete 'Task' first
    6. click 'Task Definitions' from side menu -> 'Create new task definition' -> Choose 'FARGATE' option where your containers can scale infinitely & you only pay for you need -> 'Next Step'
    7. Type a custom 'Task Definition Name' like 'goals' -> for 'Task Role' choose 'ecsTaskExecutionRole' 
    8. Choose smallest sizes of 'Task memory' and 'Task CPU' like 0.5GB & 0.25vCPU, respectively
    9. Under 'Container Definitions' click 'Add container' -> Type a custom 'Container name' like 'goals-backend' -> type the 'Image' by giving the dockerhub image name, ex. 'semie/goals-node' -> on 'Port mappings' type in port '80'
    10. Still in the 'Add Container' drawer, under 'ENVIRONMENT' then 'Command' type in 'node,app.js' b/c we want to start with node not nodemon. 
    11. Still under the 'ENVIRONMENT' list out the environment variables from the backend .env file but change `MONGODB_URL` to value of  `localhost`
    12. click 'Add' to add the container

6. Add MongoDB container to the cluster
    1. Click on 'Add container' under 'Container Definitions'
    2. Type custom 'Container name' like 'mongodb' -> type 'Image' name as 'mongo' which is official docker image
    3. Type 'Port mappings' of '27017' which is default mongodb port 
    4. under 'ENVIRONMENT' type is the environment variables of mongo from the mongo.env file 
    5. Click 'Add' to add container

7. click 'Create' to create the new 'Task Definition' -> 'View Task Definition'
8. Now launch a service based on that task 
    1. click on 'Clusters' from side menu -> click 'goasl-app >' cluster 
    2. Under 'Services' tab click 'Create'
    3. Choose 'FARGATE' as 'Launch type' -> under 'Task Definition' choose 'goals' task we just created above -> type in a custom 'Service name' like 'goals-service'
    4. Still in create services drawer, type in '1' for 'Number of tasks -> 'Next step'
    5. in 'Configure Network' Choose the 'Cluster VPC' that was created when you created your task - I don't know what this means so I chose a random one
    6. still in 'Configure Network' Choose both 'Subnets' from dropdown
    7. Make sure 'Auto-assign public IP' is 'ENABLED'
    8. For 'Load Balancer type' choose, 'Application Load Balancer'. This helps us make sure incoming traffic is handled smoothly and also helps us assign a custom domain later. It won't find the Load balance by default so click on 'EC2 console' -> under 'Application Load Balancer' click 'Create' -> type a custom 'Name' of your choice like 'ecs-lb' -> choose 'Internet-facing' for 'Scheme' -> make sure 'Load Balancer Port' is '80' -> choose the same 'VPC' as the one I chose for my service under 'Avaialbility Zones' -> check both boxes, 'us-east-2a' and 'us-east-2b' for 'Availability Zones' -> click 'Next: Configure Security Settings' 
    9. click 'Next: Configure Security Groups'
    10. check 'Select an existing security group' -> and choose the security group that's there -> click 'Next: configure routing' 
    11. For 'Configure Routing', under 'Target group' type in a custom 'Name' like 'tg' -> choose 'IP' as 'Target type' -> cick 'Next: Register Targets' 
    12. click 'Next: review' -> 'Create' to create the load balancer and close tab browser
    13. Back on ECS service refresh 'Load balancer name' and choose the one I just created -> choose 'goals-backend:80:80' 'Container name : port' -> 'Add to load balancer' -> choose the target name created, 'tg' for 'Target group name' -> click 'Next step' then 'Next step' again. We ignore auto scaling for demo 
    14. Click 'Create Service' -> 'View Service' 

9. Again click on 'Clusters' from side menu -> 'goals-app >' -> click on 'goals-service' under 'Services' tab -> we see the tasks associated with that service under 'Tasks' tab and if we click on that task we see the 'Public IP' again. We also see the 'mongodb' and 'goals-backend' containers under that tasks
10. User the 'Public IP' then the endpoint like '3.129.69.15/goals' to see the list of goals and also use that IP to post to the goals endpoint. 

11. Problem is that IP address changes every time we deploy and update a container. If we search EC2 then we can find our 'load balancer' there. Click on 'load balancer' then under 'Description' copy 'DNS name' and paste it into browser with the endpoint extension ex. 'ecs-lb-602584689.us-east-2.elb.amazonaws.com/goals'. BUT it won't work yet because load balancer keeps starting and stopping the task. 
12. To fix, click 'Target Groups' from side menu of EC2 load balancer page -> click on the task 'tg' under 'Name' -> under 'Health Check' tab click 'Edit' -> type in '/goals' under 'Health check path' b/c previously it was sending a request to slash nothing when it should send a request to /goals endpoint 
13. Another thing to fix, for EC2 load balancer page click 'Load Balancers' from side menu ->  'Description' tab click 'Edit Security Groups' under 'Security' -> check both 'default' and 'goals-6224...' security groups -> 'Save'
14. One issue we have, if we change something in the source code, we build again, tag again, and push to docker hub again then go to the cluster -> 'Service' -> click on 'Update' -> check 'Force new deployment' -> 'Skip to review' -> 'Update Service' -> 'View Source' to restart the service. But now the stored goal previously stored on the database is now lost. 

15. To persist data create volume with 'Task Definitions' from the side menu using EFS
    1. click latest task definition -> click 'Create new revision' 
    2. click 'Add volume' under 'Volumes' -> type in custom volume name like 'data' -> choose 'EFS' for 'Volume type' lets us attach a file system to our serverless elastic container, so we attach hard drives to these containers so the data survives even if container is re-deployed
    3. still in 'Volumes' popup click on 'Amazon EFS console' under 'Filse System ID' -> 'Create File System'
    4. Give file system a custom 'Name' like 'db-storage' -> Choose same 'Virtual Private Cloud' you used for ECS before -> 'customize' -> 'next'
    5. For 'Network access' first on a new tab go to 'EC2' page on AWS console -> click 'Security Groups'
    6. click 'Create security group' -> type in a custom 'Security group name' like 'efs-sc' 
    7. Choose the previous vpc under 'VPC' j
    8. Under 'Inbound rule' click 'Add rule' -> choose 'NFS' from 'Type' dropdown -> under 'Source' choose the 'goals-6224...' security group. -> click 'Create Security Group'
    9. Go back to EFS 'Network access' wizard and choose the new efs-sc security group for both 'Availability zone' 
    10. click -> 'Next' -> 'Next' -> 'Create'
    11. Back in ECS console now, refresh 'File system ID' and choose the newly created file system -> click 'Add'
    12. connect volume to mongodb container by clicking on 'mongodb' under 'Container name' -> under 'Storage and Logging' for 'Mount points' option choose the 'data' volume we created
    13. bind it to '/data/db' 'Container path' by typing that in 
    14. click 'Update'
    15. click 'Create' to create new task revision. 
    16. click 'Actions' dropdown -> 'Update Service' -> check 'Force new deployment' 
    17. Click 'Skip to review' -> 'Update service'. Now it's all stored in file system and data persists 
    18. ** Fixed node connecting to MongoDB error by clicking on 'Clusters' -> 'goals-app >' -> click on the service under 'Services' tab -> 'Update' -> 'Force new deployment' -> choose 'Platform version' of '1.4.0'. 
    19. Verify storage in file system, go to 'Clusters' -> 'goals-app >' -> 'goals-service' under 'Services' tab -> 'Update' -> check 'Force new deployment' -> 'Skip to review' -> 'Update Service' 
16. Problem now is, deploying a 2nd time will cause conflict for mongodb trying to write to the same EFS but we'll remove mongodb later

> We created ECS w/ two ECS task, 1) Node REST API, 2) MongoDB. With a volume of AWS EFS storage. 
> We use load balancer to expose our Node REST API backend to the public so it has a unique URL which doesn't change every time a task restarts. Load balancer forwards the request to Node API which calls out MongoDB to store data and sends data back to client. 

&nbsp;

## Database and containers considerations
- Can maange your own Database containers but scaling and managing availability can be challenging like multiple read write operations can occur simultaneously so we'll need multiple database containers w/ same configuration running simultaneously and this can confuse the databases like who's writing currently what if they'r already done writing but the other database still thinks a user is writing - we need synchronicity
- performance is an issue if we have only one database container and there's a traffic spike. 
- We have to take care about backups and security so roll back to a backup if data is lost. 
- So we switch to managed database service 
- For MongoDB we have MongoDB Atlas provided by AWS. 

&nbsp;

## Move to MongoDB Atlas
1. Go to mongodb.com -> 'Cloud' -> 'Atlas' -> 'Get Started' -> Sign up your info -> 'Get started for free' 
2. choose 'Create Free Cluster' option -> choose 'AWS' option -> 'Create cluster'
3. Once cluster is created click 'Clusters' on side menu -> 'Connect' -> 'Connect your application'
4. I get back a MongoDB connection string, copy it. But we may want to connect to it only in production not in development. 
    - But we have to use the exact same version of Mongo that Atlas uses in our local environment from docker so dev and production environments are same
5. Replace Mongodb connection string in my backend code with the one provided by atlas and store database name in an environment variable so dev and production databases are different. 
    - env database name is MONGODB_NAME which we also include in the Docker file `ENV MONGODB_NAME=goals-dev`
6. Since we launch our MongoDB from cloud we can get rid of 'mongodb' service from docker compose file. 
7. Change to MongoDB atlas URL for MONGODB_URL in env file ex. 'cluster0.dlich.mongodb.net'
8. Add a user on the MongoDB atlas UI and change env variables to my MongoDB Atlas username and password and should be able to bring it up and connect backend to Mongo on local with docker-compose up.

&nbsp;

## Using MongoDB Atlas in production 
- Get rid of the Docker MongoDB and replace with Atlas. 
- Also rebuild image with new credentials and push to docker hub first. 
- Update service again by going to the 'goals-service' from 'Services' tab and click 'Update' -> check 'Force new deployment' -> 'Update service' 
1. click 'Task Definitions' from side menu on AWS ECS page -> click on latest 'goals' task definition -> click 'Create new revision'
2. Delete 'mongodb' container from the 'Container Definitions' list 
3. Get rid of volumes by going to EFS on a new tab on AWS console -> click 'Delete' for that chosen EFS
4. Delete security groups by going to EC2 from a new tab on AWS console -> choose the file system security group and click 'Actions' dropdown -> 'Delete security groups'
5. Back on ECS page, click on 'goals-backend' under 'Container Name' ->  change the environment variable values to match Atlas credentials from backend env file.
6. Click 'update' -> click 'create' to create new task definition. 
7. Choose 'Actions' dropdown -> 'Update Service' 
8. Now choose the 'Force new deployment' box -> 'skip to review' -> 'Update Service' -> 'View Service' . Now our new task will be running connected to MongoDB Atlas. So database scaling and performance in the future will all be automatically managed for us. 

&nbsp;

## We want to add a frontend app
- Problem is, SPA's require a build step. Build means optimize script that needs to be executed after development but before deployment. 
- We'll have to have different environments for dev and production. 
1. Add a 2nd dockerfile to frontend React app called 'Dockerfile.prod' for production
    - We need our own server when we do 'yarn build' because 'yarn start' is only for local and uses unoptimized server 
2. Add the following to the production Dockerfile. But now we need a server that serves it so we need multi-stage builds
```
FROM node:14-alpine

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

CMD [ "npm", "run", "build" ]
```
&nbsp;

## Multi-stage builds
- One dockerfile, multiple build or setup steps. so stages can copy files and folders from each other and another stage to serve them. 
- We can build the entire dockerfile image or select individual stages up to which we wanna build skipping all stages that'd come after. 
- So the build step is first stage - We grab and install our dependencies, grab the source code then build that source code
1. We change the CMD to instead `RUN npm run build` so we can continue thereafter
2. We want to switch to a different base image after the RUN command, once we have these optimized files. Because we need node only for the build optimizing step then we don't need node anymore. We can instead use nginx which is lightweight web server.   
- Every FROM instruction creates a new stage in your Dockerfile even if you use the same image as in the previous step. 
```
FROM nginx:stable-alpine 
```
3. Now we don't want to discard our files from before the RUN command, we want to optimize them. So add a special instruction after every FROM instruction, the 'as' keyword and then any name of my choice
```
FROM node:14-alpine as build
```
4. In our second stage, Copy all source code and dependencies from the 1st stage using `COPY --from=<name of previous stage>` command
    - But we also don't want to copy everything so specify the source path after the copy commmand and name of stage. And we are copying from the docker container not from our local machine. 
    - We specify '/app/build' b/c the RUN build command creates a 'build' folder which holds the final serveable files. 
    - So we want to copy all files and folders from the /app/build path into the /usr/share/nginx/html folder

```
COPY --from=build /app/build /usr/share/nginx/html
```
5. Now we expose port 80 after copying to nginx server `EXPOSE 80`
6. Final command is I want to start nginx server 
```
CMD ["nginx", "-g", "daemon off;"]
```
- final file looks like this
```
FROM node:14-alpine as build

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

RUN npm run build

FROM nginx:stable-alpine

COPY --from=build /app/build /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

&nbsp;

## Build multi-stage image
- We send a bunch of http request to localhost which should instead be in an environmental variable. Localhost won't work b/c frontend app sends a request not from inside the server but from inside the browser which runs on the machine of my end users. 
- So completely get rid of the 'http://localhost' and just have the endpoint '/goals/' b/c we will deploy frontend in the same task as our REST backend API so it'll be reachable through the same URL. So when we hit '/goals' where the request will be sent to the same server as was used for serving our website 
1. With the above changes, create a new dockerhub repository, build the image with the frontend in it, push and deploy to ECS
    1. Create a dockerhub repo called 'goals-react'
    2. Build the frontend docker image using production Dockerfile. -f flag specifies the Dockerfile to use to build

```
docker build -f frontend/Dockerfile.prod -t semie/goals-react ./frontend
```

    3. Push image to dockerhub `docker push semie/goals-react` so we can deploy to AWS ECS

2. Deploy our React frontend to ECS
    1. On ECS of the project which now has backend and Mongo, click 'Task Definitions' -> click the 'Task Definition' we want to add a task to -> Click latest task revision
    2. Click 'Create new revision' -> click 'Add container' under 'Container Definitions' 
    3. type in custom 'Container name' like 'goals-frontend' -> type in the dockerhub frontend repo image as the 'Image' -> 'Port mappings' is '80' b/c nginx uses port 80
    4. For 'Startup Dependency Ordering' choose 'goals-backend' so it's starting and running successfully first before frontend. -> 'Add'
    - We can't create yet b/c both frontend and backend are using port 80. We can't have 2 servers on the same host. 
    5. We create a new task definition b/c both frontend and backend are using port 80, so click 'Task Definitions' from side menu -> click 'Create new Task Definition' 
    6. Choose 'FARGATE' -> 'Next step'
    7. on 'Configure task and container definitions' page type custom 'Task Definition Name' of 'goals-react' -> 'Task Role' of 'ecsTaskExecutionRole' -> choose lowest amt. of 'Task memory' and 'Task CPU'
    8. Click 'Add Container' under 'Container Definitions' 
    9. type custom name of 'goals-react' for 'Container name' -> 'Image' from dockerhub frontend image -> '80' for 'Port Mappings' -> 'Add' -> 'Create' -> 'View task definition'
    10. Create a service based on this task. But since we're on different tasks we can't just use the endpoint extensions to fetch http requsts anymore. There's a NODE_ENV default environment variable which holds a value of 'development' if we do 'yarn start' and value of 'production' if we run 'yarn build' script. Make a conditional of if that environment is 'devlopment' then use backendUrl of 'http://localhost' otherwise use the backend load balancer url 

```
const backendUrl = process.env.NODE_ENV === 'development' ? 'http://localhost:80' : 'ecs-lb-602584689.us-east-2.elb.amazonaws.com'
```

    11. Create a new load balancer for frontend app so we can use one constant url by going to EC2 on AWS console -> 'load balancers' -> 'Create load balancer' -> 'HTTP HTTPS Create'
    12. Give a 'Name' of 'goals-react-lb' -> choose 'internet-facing' -> choose samce 'VPC' as the other load balancer b/c part of same cluster -> select both 'Availability Zone' options -> 'Next Configure security settings' -> 'Next: Configure Security groups'
    13. Select all security groups so we're selecting the load balancer the other security group is in -> 'Next: Configure routing'
    14. type a custom target 'Name' like 'react-tg' -> 'Target type' is 'IP' -> 'Next: register targets' -> 'Next: review' -> 'Create' 
    15. Choose the frontend load balancer from the EC2 load balancer page and find the 'DNS name' from the 'Description' tab to visit the frontend. 
    16. Now rebuild image, push to dockerhub and start service based on the task that uses that image. 
    17. To create a service based on that task go to ECS console page, select 'Actions' dropdown -> 'Create Service' 
    18. Choose 'FARGATE' as 'Launch type' -> use 'goals-app' for 'Cluster' -> type custom 'Service name' of 'goals-react' -> 'Number of tasks' is '1' -> 'Deployment type' is 'Rolling updates' -> 'Next step'
    19. Under 'Configure network' choose the two 'Subnets' in this VPC our cluster provides -> click 'Edit' for 'Security groups'
    20. For 'Assigned security groups' select 'Select existing security group' -> check 'goals-6224' for 'Security group ID' -> 'Save' 
    21. For 'Load balancer type' choose 'Application Load Balancer' -> choose the 'goals-react-lb' load balancer we just created for 'Load balancer name' -> 'Add to load balancer' 
    22. For 'Container to load balance', 'Target group name' choose 'react-tg' target we just created -> 'Next step' -> 'Next step' -> 'Create Service' -> 'View Service' 
    23. Get the frontend IP address from the EC2 'DNS name' 

&nbsp;

- See if everything works ok locally by running `docker-compose up` for the project that includes both frontend and bakcend 
- We could also only build the first stage and stop at the second stage by targeting build stages by their name w/ --target flag so it'd build the code but won't spin up the server in that image - `... --target <name of stage>` 
```
docker build --target build -f frontend/Dockerfile.prod ./frontend
```

&nbsp; 

## Summary
- Bind mounts shouldn't be used in production b/c we don't want source code to change in production
- Some apps need a build step
- Some multi-container apps might need to be split across multiple remote machines or multiple tasks like React and node b/c they might use and expose the same port and also different DNS urls.  
- Your own remote machine, EC2 is not always practical if you're not a cloud expert. You can make insecure setups. 
- ECS is the managed approach, alternative to EC2. 
    - Tasks contain your containers
    - Start the tasks as services. 
    - Can have multiple containers per task. 
    - EFS adds a file system to our containers so data not lost if container is re-deployed

- Cloud database is better b/c it's automatically managed for scaling. 
