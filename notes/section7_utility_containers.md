## Utility containers
- Only contain environment like Node, NOT environment and then the app with that environment. 
- Don't start an application when you run them but just executes a command when you run it

1. Make an empty repo
    - Could make a package.json file by hand and manually type in dependencies but that is impractical. Instead we typically manage dependencies by running `npm init -y` first. 
    - problem with `npm init -y` is that it's not readily available on your local machine. It's only available if you download node on your local machine. 
    - And the whole idea behind docker is we don't need to install a whole host of tools on our local machine. We want to use containers that come with readily available tools. This is where utility containers come in. 

2. In terminal root path, run official node image from docker
```
docker run -it node
``` 
- Shut it down by pressing `cntrl + c`  twice 
3. Run node container in detached mode:
```
docker run -it -d node
```
4. Docker exec command allows you to run certain commands inside of a running container, ex. `docker exec -it vigorous_dewdney npm init`
```
docker exec -it <name of container> < language specific environment execution command>
```
OR
```
docker run -it node npm init
```

&nbsp;

## Build first utility container
1. Make an empty directory and make a Dockerfile - don't specify CMD b/c we want flexibility to run any command we want when we run container
```
FROM node:14-alpine

WORKDIR /app
```
2. Build image
```
docker build -t node-util .
```
3. Run the docker container
- By default this container will run in the "app" folder inside the container. 
- We also want to mirror it in the local machine so what I create in the container is also available in my host local machine. 
- The whole point is to create a project on my local host machine with help of a container. 
- We use the container to execute something which has an effect on the host machine without having to install all the extra tools on the host machine. 
- So we use a bind mount for that
- ex. `docker run -it -v /Users/sem/Documents/sem_limi_projects/my_utility_container:/app node-util npm init`
```
docker run -it -v <absolute path to local folder>:<path mapped to container> <image name> <any node environment or JS command>
```
- Enter the necessary commands for `npm init` then we'll see the package.json file appear on our local host machine directory b/c of the bind mount! 
- Can also run `yarn install` command inside the container with -it flag and a bind mount
```
docker run -it -v /Users/sem/Documents/sem_limi_projects/my_utility_container:/app node-util yarn install
``` 

## ENTRYPOINT executable command 
- similar to CMD instruction in Dockerfile but difference is, if we add a command after image name in docker run in the termnal then that terminal command overwrites the instruction in the Dockerfile specified by the CMD command 
- With ENTRYPOINT whatever you put in the docker run instruction command in the terminal comes after what you specify with ENTRYPOINT
```
ENTRYPOINT ["yarn"]
```
- Then run this command so you don't have to type in 'yarn' every time it'll always be pre-pended to any command you put in the terminal during docker run. ex. `docker run -it -v /Users/sem/Documents/sem_limi_projects/my_utility_container:/app myyarn install`
```
docker run -it -v /Users/sem/Documents/sem_limi_projects/my_utility_container:/app <image name> <node command>
```
- Container always shuts down but isn't removed every time you run a command and finish all inputs if the command requires inputs. 
- Can also install dependencies like 'express' package by adding that package name to the command instruction like so:
```
docker run -it -v /Users/sem/Documents/sem_limi_projects/my_utility_container:/app myyarn add express
```

&nbsp;

## Utility containers with docker-compose.yaml file
1. Create a docker-compose.yaml file like so
```
version: "3.8"
services: 
  yarn-container:
    build: ./
    stdin_open: true
    tty: true
    volumes:
      - ./:/app
```
2. Execute the file with this command to execute commands created by docker-compose?
```
docker-compose exec
```

3. But more useful feature is docker-compose run with - run docker compose with the container name then any command that should come after our ENTRYPOINT command, ex. `docker-compose run yarn-container install`
```
docker-compose run <container name> <command after ENTRYPOINT command>
```
- But the container won't be removed once it's finished running from docker compose so add --rm flag to remove containers once it's finished running
```
docker-compose run --rm <container name> <command after ENTRYPOINT command>
```