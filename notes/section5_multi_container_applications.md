## 1. Dockerize MongoDB
- This time we expose a port for mongoDB for backend to access - backend running on local can still access this mongo from a docker container
```
docker run --name mongodb --rm -d -p 27017:27017 mongo
```

&nbsp;

## 2. Dockerize backend node app
- 1. Make a Dockerfile on the backend root directory
```
FROM node

WORKDIR /app

COPY package.json .

RUN yarn install

COPY . .

EXPOSE 80

CMD ["node", "app.js"]
```
- 2. Build a backend node image from the Dockerfile instructions
```
docker build -t goals-node .
```
- 3. Run a backend container based on that built node image 
    - it will crash unless you change the mongodb connection from 'localhost' to `host.docker.internal` and rebuild image

```
docker run --name goals-backend --rm -d -p 80:80 goals-node
```

&nbsp;

## 3. Dockerize Frontend React app
- 1. Make a Dockerfile
```
FROM node

WORKDIR /app

COPY package.json .

RUN yarn install

COPY . .

EXPOSE 3000

CMD ["yarn", "start"]
```
- 2. Docker build the frontend application
```
docker build -t goals-react .
```

- 3. Run the Docker React frontend container - it won't run unless you run it with -it flag b/c there are inputs
```
docker run --name goals-frontend --rm -d -p 3000:3000 -it goals-react
```

&nbsp;

## 4. Add networks for container-container communication
- 1. Create a network first
```
docker network create goals-net
```
- 2. Run the database container MongoDB in the network created above - replace 'localhost' or 'host.docker.internal' with name of MongoDB container, `mongodb` and rebuild image
```
docker run --name mongodb --rm -d --network goals-net mongo
```
- 3. Run the backend app in the above created network:
```
docker run --name goals-backend --rm -d --network goals-net goals-node
```
- 4. On React frontend, initially replace wherever we're making an API call to 'localhost' with name of backend docker container ex. goals-backend then rebuild image
```
await fetch('http://goals-backend/goals)
```

- Run react app with following command: Won't work with axios/fetch being called from backend container name which replaced localhost b/c React runs in browser not inside a container. 
- So replace the backend container name in axios/fetch request with 'localhost'
- And because it's running on browser no need to run it on a network
```
docker run --name goals-frontend --rm -p 3000:3000 -it goals-react
```
- Because frontend React runs on browser not on container, that means we also need to publish our backend container to a port so frontend can access backend through localhost
```
docker run --name goals-backend --rm -d --network goals-net -p 80:80 goals-node
```

&nbsp;

## 5. Add data persistence to MongoDB with Volumes
- 1. So data persists in database even with container shutdown
- Add a volume with -v flag, a named volume. And some random path in the docker container to store MongoDB data? ex. `docker run --name mongodb -v data:/data/db --rm -d --network goals-net mongo`
```
docker run --name mongodb -v <volume name>:<random path in docker container to store database data?> --rm -d --network goals-net mongo
```

- 2. Add security to database with username and password for database
- Add environment variables for username and password with -e flag and mongoDB specified environment names `docker run --name mongodb -v data:/data/db --rm -d --network goals-net -e MONGO_INITDB_ROOT_USERNAME=sem -e MONGO_INITDB_ROOT_PASSWORD=secret mongo`
```
docker run --name mongodb -v data:/data/db --rm -d --network goals-net -e <env variable username key>=<env variable username value> -e <env variable password key>=<env variable password value> mongo
```

- This will fail because our backend initially connected to a mongoDB database without a username and password & now that database is protected. So on the backend mongoDB connection string change to this `mongoose.connect('mongodb://sem:secret@mongodb:27017/course-goals?authSource=admin', {...}` and run above docker run command again
```
mongodb://<username>:<password@><container name>:<port number>
```

&nbsp;

## 6. Add data persistence with bind mounts for source code change to take immediate effect
- 1. Backend - named volume for saving log files, bind mount for the entire backend directory
ex. `docker run --name goals-backend -v /Users/sem/Documents/sem_limi_projects/docker-tutorials/multi-01-starting-setup/backend:/app -v logs:/app/logs --rm -p 80:80 --network goals-net goals-node`
```
docker run --name goals-backend -v <absolute path of backend directory>:<path in backend docker container to the entire directory> -v <name of volume>:<path in docker container that matches the folder path from local machine> --rm -p 80:80 --network goals-net goals-node
```
- 2. Add nodemon as a dev dependency to backend repo and rebuild image for backend
- 3. Add environment variables in the backend Dockerfile for the database connection username and password 
```
// In Dockerfile
ENV MONGODB_USERNAME=root

ENV MONGODB_PASSWORD=secret
```
- Insert those environment variables into the backend app.js mongoDB connection string:
```
mongoose.connect(
  `mongodb://${process.env.MONGODB_USERNAME}:${process.env.MONGODB_PASSWORD}@mongodb:27017/course-goals?authSource=admin`,
  {...}
```
- Run the backend container again this time set another environment variable w/ -e flag w/ key/value of `MONGODB_USERNAME=sem` and no need to add an environment variable for password b/c it's the same as the one set in the dockerfile ex. `docker run --name goals-backend -v /Users/sem/Documents/sem_limi_projects/docker-tutorials/multi-01-starting-setup/backend:/app -v logs:/app/logs -v /app/node_modules --rm -p 80:80 --network goals-net -e MONGODB_USERNAME=sem goals-node`

- 4. Add bind mounts to React frontend code to be able to automatically change source code and changes to take effect without having to rebuild
```
docker run -v /Users/sem/Documents/sem_limi_projects/docker-tutorials/multi-01-starting-setup/frontend/src:/app/src --name goals-frontend --rm -p 3000:3000 -it goals-react
```