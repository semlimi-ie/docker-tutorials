## Run node image - uses node image to run a so called container based on this image
```
docker run node
```
## Show all processes of running containers on docker (ps stands for processes) that docker created for us
```
docker ps -a 
```
- response: 
```
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS                           PORTS     NAMES
86f84e4f74ab   node      "docker-entrypoint.s…"   3 minutes ago    Exited (0) 3 minutes ago                   dazzling_panini
```

## Tell docker we wanna expose an interactive session from inside the container to our hosting machine - enter interactive node terminal where we can run basic node commands
- B/c interactive shell exposed by node is not automatically exposed by the container to us.
- But node is not running on our machine here b/c the node version of the container is different from the node version on our machine
```
docker run -it node
```
- To exit container press `control + c` twice

### ** Side Note
> Images are used behind the scenes to hold all the logic and all the code a container needs, then we create instances of an image w/ the run command which creates the concrete containers which are based on an image

## Run `docker ps -a` again and we'll see we have 2 node containers now from running the last node image
- we have 2 containers based on the same image. Containers are running instances of the image or the code

---

&nbsp;  


### ** Note:
> Typically, we build up on base images to then build up our own images. Usually pull in the official base image then add our code on top of that to execute our code with that image, and we do all of that inside of a container. So we need to build our own image in that case b/c our application with our own code doesn't exist on docker hub.

---

&nbsp;


## Dockerfile settings - In an app repo make a `Dockerfile` and input the following code
```
// build your image up on another base image - node is now both a local and dockerhub image since we ran node container previously so it's cached to our local machine
FROM node

// Set working directory of the docker container
WORKDIR /app

// Which files from our local machine should go into the image. First dot or path specifies all the files/folders in the same folder that contains the Dockerfile. 
// Second dot or path is path inside image where those files/folder should be stored - every image has it's own internal file system detached from our local machine file system
// So files/folders transferred over to docker from our local machine will be stored in /app directory since we set the working directory to /app from above
COPY . .

// Alternatively, we can store our files/folders from our local machine onto a folder called app in the docker container
//COPY . /app

RUN yarn install

// Expose port from docker onto our local machine so we can run that port on our local machine
EXPOSE 80

// Not executed when the image is created but when the container is started based on the image - that's why we use CMD and not RUN
CMD ["node", "server.js"]
```

## After the above `Dockerfile` settings create an image first based on the instructions in that file before running:
- And build it in the same path as our repo directory if we're in the same root folder of our repo in the terminal 
```
docker build .
``` 

- Now run docker with the image id from the previous build command: localhost on our local machine still won't run yet though
```
docker run 6d5fb74cdefa829195f65187b070bd0af356fe26f59db2334d328d99084cc989
```

- Run docker ps to list only running containers now
``` 
docker ps
```

- Stop the container by taking the name of the container from the `docker ps` command:
```
docker stop quirky_neumann
``` 

- Now `docker ps` won't show that container anymore b/c it's stopped but `docker ps -a` will
- To run it correctly, expose your local machine port to Docker's localhost port with -p flag which stands for publish - "our local port":"docker's local port"
- "6d5fb..." is the Image id
```
docker run -p 3004:80 6d5fb74cdefa829195f65187b070bd0af356fe26f59db2334d328d99084cc989
```

---

&nbsp;


## Stop all docker containers with this command: 
- -q flag only displays container ID's `docker ps -a -q` will display all containers' ID's only, whether running or not running
```
docker kill $(docker ps -q)
``` 

> If you change something on your node server.js file and run `docker run -p 3004:80 6d5fb74cdefa829195f65187b070bd0af356fe26f59db2334d328d99084cc989` again the change won't be reflected, why? Because when we build we copy the source code into the image we made and take a snapshot of the source code at the point of time we copy it. If we edit the source code, we need to rebuild our image to copy the new source code into the image. So images are locked and finished once you build them. So to update the app w/ the new source code we run `docker build .` again to build a new image then `docker run -p 3004:80 <image ID>` and the changes will take effect
> If source code doesn't change and you stop the docker container and then build again it'll be really fast because docker cached the results of previous build - layer based architecture
> Whenever one layer changes, the previous layers if not changed is used from cache but all subsequent layers are rebuilt even if not changed. 

&nbsp;  

## Unless we change something in package.json we don't need to run yarn install again but when building docker does run yarn install again so we can optimize:
- copy package.json file and run 'yarn install' to docker's app directory BEFORE copying over all the code and files/folders/source code to the app directory 
- because of layer based structure, if source code changes we don't run yarn install again if package.json didn't change
```
FROM node
WORKDIR /app
COPY package.json /app
RUN yarn install
COPY . /app
EXPOSE 80
CMD [ "node", "server.js" ]
```

---

&nbsp;

## Help command to list all available command options on docker:
```
docker --help
```

- help options for docker running prcoesses/containers
```
docker ps --help
```

- help options for docker run
```
docker run --help
```

- help options for docker build
```
docker build --help
```

---
&nbsp;  

## Restart the stopped container without starting a new instance of that container: It's running on the background
- Get the docker container name or id with `docker ps -a` command
```
docker start <name or ID of specific docker container>
```
> When running the container you'll get console outputs on the terminal if you have any console logs. With docker start command you don't get any logs on the terminal if you make some changes on the code. So docker start command starts container in detached mode

## Get docker running in detached mode:
-d flag means detached mode so that terminal you started docker container from isn't frozen and disabled. Returns a long string ID
```
docker run -p 8000:80 -d <image name of container>
```

## Attach yourself to a detached container again: Attached means terminal running the container is read-only
```
docker container attach <container name OR container ID>
```

## Print previous logs in the terminal in a detached container:
```
docker logs < container name >
```

## Keep listening or keep attached when you started in detached mode:
```
docker logs -f <container name>
```

## To start in attached mode with `docker start` command:
- -a flag means attached
```
docker start -a <container name>
```

## Be able to put input something in a container like inp in python app AND create a pseudo terminal with that:
- * Note: running `docker start -a heuristic_buck` may only let you do input once but not twice if you have two options to input
```
docker run -it <image name or container>
```

## To start docker container in attached mode AND allow for all inputs:
```
docker start -a -i < container name >
```

---
&nbsp;  


## Remove or delete container:
- Can't remove a running container you have to stop it first
- can remove multiple containers by passing in the names of the containers you want to remove separated by a white space
```
docker rm <container name>
```

---
&nbsp;  

## List docker images:
```
docker images
```

## Remove docker images:
- can only remove images if they're not being used by any container anymore even if that container is stopped so container needs to be removed first 
- can remove multiple images on same line by passing in multiple image ID's separated by whitespace
```
docker rmi <image ID>
```

## Remove all unused images:
```
docker image prune
```

## Remove a docker container automatically as soon as it's stopped by instructing it at it's run time
```
docker run -p 8000:80 -d --rm <image name>
```

---
&nbsp;  

## Get more info. about an image configuration:
- Gives info. like id, port, when image was created, environmental variables, entry point which is the command to start the node server, docker version, operating system, 
```
docker image inspect <image name>
```

---
&nbsp;  

## Copy files and folders into an already running container or out of a running container:
- cp stands for copy
- suppose we make a folder called "dummy" with a file inside it called test.txt after the project container is already up and running
```
// "dummy" specifies the source or folder/file we want to copy somewhere else, "." says copy everything in that folder or you can specify a specific file like dummy/test.txt
// "quirky_wescoff" is the destination you want to copy folder/file to or container name
// "tests" is the path inside the container you want to copy to

docker cp dummy/. quirky_wescoff:/tests
```
- #### To check the previous folder was copied to the container from localhost:
    - 1. Delete a file ex. "test.txt" within the newly created dummy folder in local machine
    - 2. Run this command so the file to copy is from the running container and destination is now our local folder
    ```
    docker cp quirky_wescoff:/tests dummy
    ``` 
    - 3. You can see the file from the /tests folder from the running container is now copied over to the dummy folder in the local machine
    - Can also copy over a specific file, "test.txt" from running container into a local folder, "dummy" in our local machine w/ this command
    ```
    docker cp quirky_wescoff:/tests/test.txt dummy
    ```

---
&nbsp;  

## Run docker container with your own custom name:
- "goalsapp" is your own custom container name
```
docker run -p 3004:80 -d --rm --name goalsapp <image ID>
```

## Build docker image with your own custom name and tag:
- t flag specifies custom tag with "name":"tag" format
- "goals" is the name
- "1.0" is the tag
- run `docker images` after building to see the latest image w/ your custom name
```
docker build -t goals:1.0 .
```

## Run container with custom name based on the image you just created with your custom image name and tag:
```
docker run -p 3004:80 -d --rm --name <custom container name> <custom image name>:<custom image tag>
```

## Remove ALL images including tagged images:
```
docker image prune -a
```

---
&nbsp;
## Share docker images on Docker hub
#### Push images to docker hub
1. Sign-up for docker hub free plan
2. Click on "Repositories" on top tab -> "Create Repository"
3. Give repository a name like node-hello-world
4. Choose public repository radio button -> create
5. Push our local docker image to docker hub with following command:
```
docker push semie/node-hello-world
```
reponse will show this:
```
An image does not exist locally with the tag: semie/node-hello-world
```
- So to create an image with semie name locally we could just build again with that user name like so:
```
docker build -t semie/node-hello-world .
```
- But we could just rename our existing local app that we already have to use the semie "docker hub username"/"repo-name":"new tag name" 
```
docker tag < old image name or REPOSITORY>:<old tag of image> semie/node-hello-world:latest
```

6. Now push again to docker hub with this command:
```
docker push semie/node-hello-world
```
- If it says "denied: requested access to the resource is denied" then log in with the following command and type in username and password:
```
docker login
```
- can log out with:
```
docker logout
```

&nbsp;

#### Pull images from docker hub
1. Type following command in terminal: ex. `docker pull semie/node-hello-world:latest`
```
docker pull <docker hub username>/<repo name in docker hub>:<image tag>
```
2. Run the pulled in image that you pulled in from docker hub on your local machine exposing to port 8000 on local: ex. `docker run -p 8000:3000 semie/node-hello-world:latest`
```
docker run -p 8000:80 --rm <docker hub username>/<repo name in docker hub>:<image tag>
```
- ** Note: Running docker run command of pulled in image from docker hub will not run latest changes that were pushed to docker hub unless you pull in changes again
- ** Note: Running docker run of docker hub image will run the image that's on docker hub if you don't have a local copy of it on your local machine