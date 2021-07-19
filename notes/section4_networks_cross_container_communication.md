> Networks mean how to connect containers, how to let them talk to each other, how to connect your application running in a container to your local host machine, send http request to some other service running on your machine, and how to reach out to the world wide web from inside your container

> Sending requests from inside a container to world wide web like calling an API endpoint from inside the container just works without any special setup

&nbsp;

## Connect a container to some software running on local host machine
- replace "localhost" of the application running on your local host machine with `host.docker.internal` ex.
```
mongoose.connect(
  'mongodb://host.docker.internal:27017/swfavorites',
  { useNewUrlParser: true },
  (err) => {
    if (err) {
      console.log(err);
    } else {
      app.listen(3000);
    }
  }
);
```

&nbsp;

## Working with 2 running containers
1. First run mongoDB on a docker container - get the run instructions from official mongo image from docker hub
    - ex. `docker run -d --name mongodb mongo`
```
docker run -d --name <name of mongodb container> mongo
```
2. Get the port the mongodb is running on by running the below command then in the "NetworkSettings" object then "IPAddress"
```
docker container inspect <name of mongodb container>
```
- response object will show: `"IPAddress": "172.17.0.2"`
3. Paste in that IP address into the app.js node file where mongoDB localhost or docker.host.container is specified

&nbsp;

## Container to container network - more elegant solution
- Docker automatically looks up IP addresses of the specified containers in the specified network - docker internal network 
- 1. First create a network with your custom name for the network:
    - help options
    
```
docker network --help
```
    
- create docker network with custom name
```
docker network create <custom network name>
```
    
- list out networks
```
docker network ls
```

- 2. In the docker run command give the --network flag then network name ex. `docker run -d --name mongodb --network favorites-net mongo` we're starting a mongoDB container from the mongo official image 
```
docker run -d --name <name of container> --network <network name> <image name>
```

- 3. To specify which app is using that network specify in the app.js server code for the mongoDB connection the name of the container ex: `mongoDB` is name of container in `'mongodb://mongodb:27017/swfavorites'`
```
mongoose.connect(
  'mongodb://<name of container>:27017/swfavorites',
  { useNewUrlParser: true },
  (err) => {
    if (err) {
      console.log(err);
    } else {
      app.listen(3000);
    }
  }
);
```

- 4. Run the other app.js code that's not the mongodb database by specifying the name of the network with --network flag to put this container into that specific network
ex. `docker run --name favorites-app --network favorites-net -d --rm -p 3004:3000 favorites-node`
```
docker run --name <container name> --network <network name> -d --rm -p 3004:3000 <image name>
```
    
- see that it's running properly
```
docker logs <container name>
```

&nbsp;

> Docker Networks actually support different kinds of "Drivers" which influence the behavior of the Network. The default driver is the "bridge" driver - it provides the behavior shown in this module i.e. Containers can find each other by name if they are in the same Network. The driver can be set when a Network is created, simply by adding the --driver option. `docker network create --driver bridge my-net`


Docker also supports these alternative drivers - though you will use the "bridge" driver in most cases:
- host: For standalone containers, isolation between container and host system is removed i.e. they share localhost as a network
- overlay: Multiple Docker daemons i.e. Docker running on different machines are able to connect with each other. Only works in "Swarm" mode which is a dated / almost deprecated way of connecting multiple containers
- macvlan: You can set a custom MAC address to a container - this address can then be used for communication with that container
- none: All networking is disabled.
- Third-party plugins: You can install third-party plugins which then may add all kinds of behaviors and functionalities