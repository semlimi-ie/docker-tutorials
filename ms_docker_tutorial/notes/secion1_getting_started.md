## Start docker after copying over files to Dockerfile, exposing port and running node app.mjs
```
docker build . 
```
&nbsp;  


## Run docker image after building by copying over the image id from SHA256 & expose port 3000 from docker onto our own local machine port 3000
```
docker run -p 3000:3000 e6788b2c6c6abb0555af9ed619432b1121897c632534161092ab
```


## To stop container open up a new terminal and type the following code
- ### List all running containers first
```
docker ps
```


- ### Grab name of container from the response from 1st command & type in following command 
```
docker stop jolly_galois
```


