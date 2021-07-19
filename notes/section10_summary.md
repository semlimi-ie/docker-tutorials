## Docker containers and data
- Containers are isolated meaning they have their own data and file system, detached from the host machine filesystem. 
    - Use bind mounts to connect host machine folders 
    - We have a certain path on our host machine which we know which we want to mirror into a path in the container. 
    - For volumes, our container folder is also mirrored to some folder on our local machine but we don't know where this folder is necessarily. We don't want to change that data, we just want it to survive after that container was removed

&nbsp;

## Deployment
- Don't use bind mount when you deploy your application, instead use regular volume or COPY to make sure that everything that should be in a container is inside of it. 
- Multi-container projects or even single container projects may need multiple hosts. Or can also run on the same host depending on application.
    - One machine might not have enough power
- 