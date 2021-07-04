## Multi-container orchestration with Docker Compose
#### 1. Make a `docker-compose.yaml` file in the root directory of the project - outside both the "frontend" and "backend" folders
#### 2. Make a docker-compose file for MongoDB container
- Can find more info. at this link `https://docs.docker.com/compose/compose-file/compose-file-v3/`
- Version refers to version of docker-compose we're using
- `services` key has to be named exactly that way. Then for child keys of "services" key you give your custom container names like `backend`, `frontend`, `mongodb` 
- `image` key under container name used to point to the image we're using, or your custom image published on docker hub ex. `mongo`, `semie/node-hello-world`
- Containers are removed by default when you bring down containers with `docker-compose down`  
- For detached mode you can specify -d flag in run command if you want it in detached mode `docker-compose up -d`
- Use `volumes` key to specify volumes to create for a particular service/container
    - Underneath "volumes" key, make 2 spaces for child keys and start child key with dash/hyphen 
    - Use the exact same syntax as you would in the docker run commands for named and anoynymous volumes ex. `<volume name>:<volume path inside docker container mapped to local machine file path container>`
    - Can add extra options after the volume path like read-only flag ex. `- data:/data/db:ro`
- For environment variables, make `environment` key under the container/services in the same level as "image" and "volumes" keys
     - Underneath environment key specify the environmental variable key and value like so `<ENV VARIABLE KEY>: <env variable value>` or `- <ENV VARIABLE KEY>=<env variable value>`
     - Can also use environmental variables from an .env file. Ex. Add a subfolder called 'env' then add a file inside 'env' folder called 'mongo.env'. Put environmental variables inside the specific .env file 
        - 1. Then in docker-compose.yaml file under the container/services add a key called `env_file` 
        - 2. Underneath the `env_file` key list out all the environment variable files for that container starting with a hyphen/dash by putting in a relative path from that docker-compose file ex. `./env/mongo.env` 
- Can specify all the networks a service/container should belong to with the `networks` key underneath that container. But don't need because docker automatically defines an environment for all of the services within a single docker-compose file and creates a network on it's own. 
    - But if we wanted to we could do add a network named 'goals-net' like `- goals-net` underneath the `networks` key
- For named volumes have to list out a key `volumes` at the same level as `services` key and list out all named volumes underneath followed by a colon, ex `data:`
```
version: "3.8"
services: 
  mongodb:
    image: 'mongo'
    volumes:
      - data:/data/db
    # environment: 
    #     # MONGO_INITDB_ROOT_USERNAME: sem
    #     # MONGO_INITDB_ROOT_PASSWORD: secret
    #     # - MONGO_INITDB_ROOT_USERNAME=sem
    env_file: 
      - ./env/mongo.env

volumes:
  data:
```
&nbsp;

#### 3. Start the service up in the same directory path in the terminal where the docker-compose file sits
```
docker-compose up
```
- Start docker compose in detached mode
```
docker-compose up -d
```
#### 4. Shut everything down and remove all containers 
```
docker-compose down
```
- but to delete volumes as well when shutting down use:
```
docker compose down -v
``` 

&nbsp;

#### 5. Write up the docker compose for backend container
- If we have a pre-existing backend image we can use the `image` key with a value of name of image to use that image for that service/container, ex. `image: goals-node`
- All images were removed so we build an image for the backend container first with `build` key and a value of relative path to the backend folder, ex. `build: ./backend`
- Alternate way to build image - under `build` key, make another key, `context` and specify path to backend folder. Make another key called `dockerfile` underneath build object and specify `Dockerfile`
```
backend:
  build:
    context: ./backend
    dockerfile: Dockerfile
```
- can speficy args with `args` key  
- Specify ports with `ports` key and value underneath of `<local machine port>:<port in docker container from source code>`
```
ports:
  - '8080:80'
```
- Add volumes with `volumes` key and list out named and anonymous volumes with dash/hyphen
- For bind mount, instead of the absolute path to the backend folder of interest use relative path relative to the docker-compose file
- Anonymous volumes don't need to be added to the top level volumes key
```
volumes:
  - logs:/app/logs
  - ./backend:/app
  - /app/node_modules
```
- Make a 'backend.env' file under env folder and put in the backend environment variables in there, ex. MONGODB_USERNAME=sem
- To include environmental variables from the environmental variable file make a `env_file` key and underneath, list out path to all the env files to be used, path relative to the docker-compose file
- `depends_on` key specifies what other container this current container depends on so list out all the containers this current container depends on underneath the depends_on key starting with hyphen/dash
```
depends_on:
  - mongodb
```
- final version of backend container:
```
services:
  ...
  backend:
    # build: ./backend
    build: 
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - '8080:80'
    volumes:
      - logs:/app/logs
      - ./backend:/app
      - /app/node_modules
    env_file:
      - ./env/backend.env
    depends_on: 
      - mongodb
```
- Start the container with `docker-compose up -d`
- Docker will automatically assign a name to containers started by docker compose

#### 6. Write up docker-compose for frontend container
- Make a `build` key and underneath build key, make a `context` key with a value pointing to the relative path to the frontend directory 
```
frontend:
  build:
    context: ./frontend
```
- Specify a `port` key and underneath list out starting with hypen/dash the local machine port to be on and which port from the docker container to use 
```
ports:
  - '3004:3000'
```
- Make a `volumes` key and underneath list out all the volumes to use for persistent data, for bind mount specify the `- <relative path to the bind mount from docker-compose file>:<path mapped to the docker container file/folder>`
```
volumes:
  - ./frontend/src:/app/src
```
- Specify the input interactive `-it` flag.
    - `stdin_open: true` means standard input open is set to true
    - `tty: true` attaching terminal is set to true
- Make a `depends_on` key and underneath list out with hyphen/dash the name of the backend container:
```
depends_on:
  - backend
```
- final version frontend docker-compose code
```
  frontend:
  build:
    context: ./frontend
    dockerfile: Dockerfile
  ports:
    - '3004:3000'
  volumes:
    - ./frontend/src:/app/src
  stdin_open: true
  tty: true
  depends_on:
    - backend
```
> ** NOTE: The backend port exposed on local machine browser from docker container has to be the same otherwise frontend refuses to connect to backend!! 

## Build images every time you bring up a container with docker-compose up
- Just the build key will only build once when you do docker-compose up and won't build again subsequent times you bring up docker compose
```
docker-compose up --build
```
- With `docker-compose up` it will both build and bring up containers
- with docker-compose up docker assigns an automatic name to the services/container - with "name of directory docker-compose is in"_"name of service"_"incrementing number" 

## To give your own custom name to a container/service add the following to the docker-compose file under the corresponding service
```
backend:
...
  container_name: backend
```

## Final version of docker-compose.yaml file
```
version: "3.8"
services: 
  mongodb:
    image: 'mongo'
    volumes:
      - data:/data/db
    # environment: 
    #     # MONGO_INITDB_ROOT_USERNAME: sem
    #     # MONGO_INITDB_ROOT_PASSWORD: secret
    #     # - MONGO_INITDB_ROOT_USERNAME=sem
    env_file: 
      - ./env/mongo.env
    container_name: mongodb
  backend:
    # build: ./backend
    build: 
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - '8080:80'
    volumes:
      - logs:/app/logs
      - ./backend:/app
      - /app/node_modules
    env_file:
      - ./env/backend.env
    depends_on: 
      - mongodb
      container_name: backend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - '3004:3000'
    volumes:
      - ./frontend/src:/app/src
    stdin_open: true
    tty: true
    depends_on:
      - backend
    container_name: frontend


volumes:
  data:
  logs:
```