## Specify data volume to be stored in the docker container in the Dockerfile code
- First quote specifies what data will be stored in what folder inside the docker container
- "/app/feedback" means data inside the "app" directory then inside the "feedback" directory that's in the app directory inside docker container
- Transfer of data between local machine and docker inside the "feedback" folder
```
FROM node:14

WORKDIR /app

COPY package.json .

RUN yarn install

COPY . .

EXPOSE 80

VOLUME [ "/app/feedback"]

CMD ["node", "server.js"]
```
- Then build image again with:
```
docker build -t feedback-node:volmes .
```
- Then can run the container in detached mode and remove once stopped - So if container is removed then started again, volume data will persist
```
docker run -p 3004:80 -d --name feedback-app --rm feedback-node:volumes
```
- If running the above command will keep loading the submit once I submit the data and crashes, run docker logs to see what happened
```
docker logs <container name>
```
- But removing container and starting and running another one still won't persist data yet...

---

&nbsp;

## Dcoker volume help
```
docker volume --help
```

## List docker volumes
```
docker volume ls
```

&nbsp;
> Named volumes persist data on your local machine. The volume path we specified without a name in our Dockerfile ex. /app/feedback is hidden somewhere in our local machine but we don't know where and it's not meant to be directly accessed by us. 

## Remove the `VOLUMES` command from Dockerfile for named volumes and instead specify volume name in the docker run container command:
- -v flag stands for volume
- ex. `docker run -p 3004:80 -d --name feedback-app --rm -v feedback-volume:/app/feedback feedback-node:volumes`
```
docker run -p 3004:80 -d --name feedback-app --rm -v <volume name>:<volme path in the container to connect container to our local file path> feedback-node:volumes
```
- If we stop and remove the container then run again with the same volume flag and volume name and path we'll see the data from previous sessions persist

> Anonymous volumes are removed automatically, when a container is removed. This happens when you start / run a container with the --rm option. If you start a container without that option, the anonymous volume would NOT be removed, even if you remove the container with docker rm ...  Still, if you then re-create and re-run the container i.e. you run docker run ... again, a new anonymous volume will be created. So even though the anonymous volume wasn't removed automatically, it'll also not be helpful because a different anonymous volume is attached the next time the container starts i.e. you removed the old container and run a new one.
> Now you just start piling up a bunch of unused anonymous volumes - you can clear them via `docker volume rm VOL_NAME` or `docker volume prune`

&nbsp;

## Bind mounts
* The problem is that every time we change our source code we have to rebuild the docker image and that gets annoying. We want the image change to take effect automatically without having to build another image. That's were bind mounts come in. 
* Bind mounts are different from volume because for volumes we don't know where in our local machine the volume is stored, it's managed by docker but for bind mounts we do know where the data on our local machine is stored. Because we as a developer set the path to which the container internal path should be mapped on our host machine. So with bind mounts we're fully aware of the path in our local host machines so we can put our source code into such a bind mount to make sure the container is aware of that and the source code is not used from that copied in snapshot but from the bind mount - so from some some connection to some folder on our host machine therefore the container will always have access to the latest code.
* Bind mounts are perfect for persistent and editable data
* Normal volume can help us with persistent data but editing is not really possible since we don't know where it's stored on our host machine

&nbsp;

## Set up Bind mount 
- To add a bind mount we don't do it in the Dockerfile because we're not changing the core image, it's specific to a container we run so we add it during docker run when starting and running a container 
- To add a bind mount we add a 2nd volume in addition to the previous named volume with -v flag w/ 2 arguments separated by colon ":"
    - 1st argument is the path to the foler on my host machine where I have all the code/content that should go into the docker container's mapped folder. It must be absolute path 
    - 2nd argument is the path to the folder in the container where we want the data from our local machine to go so for whole source code it'll be /app
- ex. `docker run -d -p 3004:80 --rm --name feedback-app -v feedback-volume:/app/feedback -v "/Users/sem/Documents/sem_limi_projects/docker-tutorials/data-volumes-01-starting-setup:/app" feedback-node:volumes`
- ** Note: can also bind a single file rather than the whole folder if you only want to share and edit the contents of that single file
- Include double quotes in the bind mount paths so any special characters or spaces don't break the command
- Once we run the below command our entire folder in our local machine will be mounted as a volume into the app folder that's inside the container - so everything that was in the app folder in the container is overwritten by what we specified from our absolute path folder from our local machine
```
docker run -d -p 3004:80 --rm --name feedback-app -v feedback-volume:/app/feedback -v <absolute path to the folder on our local machine where we want the edited data to go to the container>/:<path in the docker container where we want the changed source code data to go>
```
- But fails to find express node library if we do run it now because we don't have node-modules folder in our local machine since we never did yarn install on local and all our local source code overwrites what was previously on the container /app folder

&nbsp;

> You should make sure docker has access to the folder which you're sharing as a bind mount
1. From the docker whale choose Preferences
2. Go to Resources side tab -> File Sharing 
3. If File Sharing options show /Users as one of the permitted folders and my source code is in parent directory /Users then I'm good

&nbsp;

> If you don't always want to copy and use the full path, you can use these shortcuts: macOS / Linux: `-v $(pwd):/app`

&nbsp;

## Tell Docker there are certain parts in it's internal file system which should not be overwritten from outside which is from our local machine
- This is achieved with another volume which we add to the container during run command. With an anonymous volume
- The third volume command looks something like this: `-v /app/node_modules
- Docker evaluates all volumes you're setting on a container and if there are clashes the one w/ longest path name wins - node_modules folder survives overwrite
- ex. `docker run -d -p 3004:80 --name feedback-app --rm -v feedback:/app/feedback -v "/Users/sem/Documents/sem_limi_projects/docker-tutorials/data-volumes-01-starting-setup:/app" -v /app/node_modules feedback-node:volumes`
```
docker run -d -p 3004:80 --name feedback-app --rm -v <name of volume>:<path in container for volume data to communicate with local machine data> -v "<absolute path of source code folder from local machine:<destination path in container to put the local machine's source code in>" -v <path of folder in container w/ long path name to preserve that folder/file> feedback-node:volumes
```

---
&nbsp;

## Add nodemon so don't have to start container again if backend server code changes
1. In package.json manually type in nodemon as a `devDependency`
2. In package.json, make a scripts object 
3. Inside scripts object, put in a "start" key with value of "nodemon server.js" so it starts with nodemon
4. inside docker file change the CMD [ "node", "serever.js" ] to instead `CMD [ "yarn", "start" ]`
- Now `docker logs <container name>` will log out what was in the console whenever we change console log

&nbsp;

## Volumes and bind mounts summary:
- #### Anonymous Volume
- volume attached to a container but volume removed if container is removed
- But does survive container shutdown and restart unless --rm is used
- So cannot be reused on same image
- Useful for locking in previous data that already existed in container like node modules 
- W/ this volume docker creates a copy of the volume data inside host local machine as well, so performant and efficient because functionality is outsourced out of docker into our local host machine
```
docker run -v /app/data...
```

- #### Named Volume
- Because we do assign a name to the volume
- Created in general, not tied to any specific container
- Survive container shutdown/restart and container removal
- Can be shared across containers
```
docker run -v data:/app/data...
```

- #### Bind Mount
- We know where the data volume is stored on the local host machine and not tied to any specific container
- Survive container shutdown and removal
- To remove it in the container have to delete the file/folder on the local host machine to delete everything
- can be shared across containers
```
docker run -v /path/to/code:/app/code
```

&nbsp;

## Make a bind mount volume read only from docker - We can read write to that bind mount in the docker container from our local machine but docker can't write to that container
- After the volume path that's in the docker container attach `:ro` , ro stands for read only
```
docker run -d -p 3004:80 --name feedback-app --rm -v <name of volume>:<path in docker container> -v "<absolute path from our local machine project folder>:<path in docker container to put the source code from local machine to>:ro" -v /app/node_modules feedback-node:volumes
```  
- But caveat is that it'll prevent writing to any folder in the directory we specified with :ro flag so have to put another -v flag for other specific file types we do want docker to change like the /temp folder in the node app
```
docker run -d -p 3004:80 --name feedback-app --rm -v <name of volume>:<path in docker container> -v "<absolute path from our local machine project folder>:<path in docker container to put the source code from local machine to>:ro" -v /app/node_modules -v /app/temp feedback-node:volumes
```
&nbsp;

> Bind mount does not show up with `docker volume ls` because bind mount is not a volume managed by docker 

## Docker volume create help command:
```
docker volume create --help
```

## Add a custom named volume first before specifying container's data path for that volume:
- ex. `docker run -d --rm -p 3004:80 --name feedback-app -v feedback-files:/app/feedback`
```
docker volume create <volume name>
```
- Then add the named volume from previous step to a specific docker container path so it'll hold the data we send from local machine to docker container
```
docker run -d --rm -p 3004:80 --name feedback-app -v <volume name created w/ create command>:<docker container's path to put data in>
```
&nbsp;

## Inspect docker volumes:
- run `docker volume ls` command to see the available docker volumes first
```
docker volume inspect <volume name>
```

## Remove a volume:
- ** Note: Can't remove volume of an active container, have to stop the container first to remove that volume
```
docker volume rm < volume name >
```

## Remove all unused volumes:
```
docker volume prune
```

&nbsp;

#### Why do we use `COPY . .` in our Dockerfile if we're just using bind mounts to copy over the entire source code directory to Docker? 
- We use bind mount in development code to reflect changes from our source code onto the app
- Once we're done developing, we don't run it on our local machine anymore, we put it on a server and files are served from a remote server so we're not running it with a bind mount anymore
- In production we only want a snapshot of our code, not source code that's constantly changing. 

&nbsp;

## Restrict what gets copied from our local machine over to docker container with .dockerignore file
- Include node_modules folder in .dockerignore file to make sure node_modules don't get copied over to docker container since we run the yarn install file in the docker container's working directory anyway
- Can also add the Dockerfile and .git file into the .dockerignore file

---
&nbsp;

> Set ENV variables available in Docker and application code w/ ENV in Dockerfile or via --env or docker run

## Set ENV variables into Docker:
1. Remove hard coded port on node server code and replace with `process.env.PORT`
2. In Dockerfile insert a command after COPY . . like so `ENV <environment variable name> <environment variable value>` ex. `ENV PORT 80`
3. To use environment variable values inside Dockerfile name of value has to be preceded with $ ex. `EXPOSE $PORT`
```
FROM node:14

WORKDIR /app

COPY package.json .

RUN yarn install

COPY . .

ENV PORT 80

EXPOSE $PORT

CMD [ "yarn", "start" ]
```
4. Alternative can set environmental variable in run command with `--env <env variable name>=<env variable value>` ex. `docker run -d --rm -p 3004:8000 --env PORT=8000 --name feedback-app -v feedback:/app/feedback -v "/Users/sem/Documents/sem_limi_projects/docker-tutorials/data-volumes-01-starting-setup:/app:ro" -v /app/temp -v /app/node_modules feedback-node:env`
    - Or can set environment variables in docker run command with -e flag too ex. `docker run -p 3004:8000 -e PORT=8000 ...`
5. Alternatively, can run environment variables from a .env file from source code and use that in the run command
ex. `--env-file ./.env`
```
--env-file <file path route from the folder I'm currenlty in on local><name of environmental variable file>
```

&nbsp;

## Build time arguments 
- Allows to plug in different values into our Dockerfile or into our image when we build that image withou having to hardcode the values into the Dockerfile
- But you can't use default arguments in CMD command
- Can have Dockerfile like this
```
FROM node:14

WORKDIR /app

COPY package.json .

RUN yarn install

COPY . .

ARG DEFAULT_PORT=80

ENV PORT $DEFAULT_PORT

EXPOSE $PORT

CMD [ "yarn", "start" ]
```
- Then can build image based on the Dockerfile like this: We can have 2 of the same images w/ 2 different tags that use different ports 
```
docker build -t feedback-node:dev --build-arg DEFAULT_PORT=8000 .
```