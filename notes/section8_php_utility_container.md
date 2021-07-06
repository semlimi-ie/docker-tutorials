## The PHP app setup
- We have a source code folder on our local host machine. 
- We'll use PHP interpreter app container and connect our source code from local host machine to this container
- We'll use Nginx Web Server as another app level container connected to PHP Interpreter container
- We'll use MySQL database as our 3rd app level container connected to the PHP Interpreter container
- The local machine source code will also be connected to 3 different utility containers
    - npm
    - Laravel Artisan
    - Composer

&nbsp;

## Build a PHP Laravel app in docker without any tool installation in host machine
1. Make an empty directory called 'my_php_app'
2. Make a `docker-compose.yaml` file in that directory to set up all app containers and utility containers
    - We'll have 6 different `services` or containers for this
    1. `server` container or Nginx
    2. `php` container
    3. `mysql` container
    4. `composer` container
    5. `artisan` container
    6. `npm` container

3. For first service, Nginx `server`, docker official image for that exists so just use that as the image
    - Nginx `server` should look at incoming requests and funnel them to the `php` container and let that container execute our PHP code. So we provide our own configuration with bind mount where we'll edit the nginx.conf file in our local machine and bind it to the one in docker container.
        - For this create 'nginx' folder then a 'nginx.conf' file inside that.
        - Paste in the 'nginx.conf' code by Max - Listens on port 80, handles requests to index files, specifies a root directory to look for code to respond to incoming requests, redirection location rules so ensuring all incoming requests are redirected to index.php files, or requests that do already target PHP files are then forwarded to our PHP interpreter   
    - Nginx is usually on port 80 in docker containers, we'll expose that port 80 on our local machine to port 8080

```
services:
  server:
    image: 'nginx:stable-alpine'
    ports: 
      - '8000:80'
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
```

4. `php` service/container
    - We need a custom Dockerfile b/c we need custom image on top of base PHP b/c we want PHP and extra extensions. Make 'dockerfiles' folder then 'php.dockerfile' file inside
    - We set working directory inside docker container for php to '/var/www/html' b/c it's a standard path for web servers to serve your websites from. We make sure from now on, this folder always holds our final application - this path is already set as root in our Nginx server
    - We don't need a CMD b/c it'll run a default PHP command
    - We have to ensure the PHP interpreter can reach our source code - we'll have our php laravel application source code later and we'll need a bind mount for that
        - for this make a folder in the project directory called 'src' which will map to the /var/www/html folder inside the php container 
        - For the bind mount, set a `path/in/host/machine:/path/to/container:delegated`, delegated means if the containers should write some data there it's not instantly reflected back to the host machine, instead it's processed in batches and performance is a bit better
    - `ports` are specified by [container name]:[port number] rather than 'localhost' b/c when in the same docker-compose file they're part of the same network and different containers find each other by container name 
        - In the nginx.conf file we specify what port we want to send the request to handle some PHP file, port 3000 initially but we'll change to port 9000 b/c that's the port php image uses according to their github source code Dockerfile
        - In nginx it's going straight to the container, it's not going to be sent from our local host machine so we just specify port 9000 in our nginx.conf file. We don't need to interact with the PHP container from our local host machine so - we have direct container-container communication with nginx setup not through our local host machine  

```
  php:
    build:
      context: ./dockerfiles
      dockerfile: php.dockerfile
    volumes:
      - ./src:/var/www/html:delegated
```

5. `mysql` service/container
    - Docker comes with an official MySQL image so we'll use that as base image 
    - We need environmental variables for MySQL so make a 'env' folder with 'mysql.env' file inside

```
MYSQL_DATABASE=homestead
MYSQL_USER=homestead
MYSQL_PASSWORD=secret
MYSQL_ROOT_PASSWORD=secret
```

```
  mysql:
    image: mysql:5.7
    env_file:
      - ./env/mysql.env
```

6. `composer` utility container
    - Make a dockerfile for composer, import in composer image which is an official image
    - Need `ENTRYPOINT` to make `composer` executable w/ flag `--ignore-platform-reqs`
    - Working directory inside container will again be '/var/www/html' 
    - will have a bind mount connecting 'src' folder on our local host machine to the '/var/www/html' path in docker container

```
  composer:
    build: 
      context: ./dockerfiles
      dockerfile: composer.dockerfile
    volumes:
      - ./src/:/var/www/html
```

&nbsp;

- We can run individual single docker container like only composer for now with `docker-compose run...` command we don't need to run the other services - and it'll have a whole bunch of composer files/folders after running
```
docker-compose run --rm composer create-project --prefer-dist laravel/laravel .
```

- Change the DB_ username, password, host etc in the local machine laravel .env file generated by previous command matching the MySQL credentials we had in the mysql.env file. 
    - DB_HOST should say mysql rather than localhost because we're accessing MySQL database from inside the container not through our local host machine
- But problem is our server doesn't have access to our source code only the php service interpreter does, so PHP files need to be exposed to the server so we need to add an extra bind mount, binding our source folder to the /var/www/html folder inside the `server` docker container
```
  server:
    image: 'nginx:stable-alpine'
    ports: 
      - '8000:80'
    volumes:
      - ./src:/var/www/html 
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
```

- Now start the services up with docker-compose up where PHP interpreter will talk to mysql database because they're in same network. We only want to bring up 3 of the services for now though, `server`, `php` and `mysql`. We can specify only certain services
```
docker-compose up -d server php mysql
```
- But nginx `server` crashes and stops although not removed because need to change the nginx.conf path inside docker container to
```
./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
``` 
- Visit `localhost:8000` to see the starting screen app
- But now starting each specific service by typing out their individual name in docker-compose up gets annoying - it'd be better if we could just run `docker-compose up -d server` and it'll start up all other services. We can let docker-compose know that that should happen by adding dependencies to the `server` service
```
depends_on: 
  - php
  - mysql
```

- ** Note: Another improvement we can make - docker-compose looks at if an image exists if we bring it up and if it exists then we use that. It never rebuilds that image if something gets edited in source code that's not mapped to a bind mount for example the dockerfile. We'd have to run 'docker build .' again. To force docker-compose to re-evaluate our image and re-build when we bring it up add the `--build` flag to docker-compose up    
```
docker-compose up -d --build server
```

6. `artisan` service/container - tool to run certain laravel commands like populate the database with initial data
    - For build command use the same dockerfile as php.dockerfile  
    - Needs volumes with our source code /src directory mapped to /var/www/html folder
    - We need to run our own command also in addition to what's specified in the php.dockerfile in the form of ENTRYPOINT so add an `entrypoint:` key in docker-compose file with value of commands to run. 

```
  artisan:
    build: 
      context: ./dockerfiles
      dockerfile: php.dockerfile
    volumes:
      - ./src:/var/www/html
    entrypoint: ["php", "/var/www/html/artisan"]
```

7. `npm` service/container 
    - use node image
    - set a working directory with `working_dir: /var/www/html`

```
  npm:
    image: node:14
    working_dir: /var/www/html
    entrypoint: ["npm"]
    volumes:
      - ./src:/var/www/html
```
- Now run artisan only to make database migrations
```
docker-compose run --rm artisan migrate
```

&nbsp;
8. Make nginx dockerfile 
- Add an nginx.conf file in dockerfiles folder - 
    - import nginx from docker official image
    - set working directory in docker container to /etc/nginx/conf.d
    - copy our local machine nginx.conf file to the working directory in docker container
    - Rename nginx.conf file in docker container to default.conf file in that docker container using `RUN mv` command
    - switch working directory to instead /var/www/html
    - Then copy our source code folder, src into the docker container /var/www/html directory 
    - nginx has a default command to start the web server so no need to specify any commands
    - Tweak docker-compose file to user the nginx.dockerfile


```
FROM nginx:stable-alpine

WORKDIR /etc/nginx/conf.d

COPY nginx/nginx.conf .

RUN mv nginx.conf default.conf

WORKDIR /var/www/html

COPY src .
```
- Tweaked docker-compose file for `nginx` server 
    - context matters here because the docker file will both run, build AND set some files/folders in the directory we specify so we set context to root directory path rather path where dockerfile is so all our files/folders are in same directory  
```
  server:
    # image: 'nginx:stable-alpine'
    build:
      context: .
      dockerfile: dockerfiles/nginx.dockerfile
    ports: 
      - '8000:80'
    # volumes:
    #   - ./src:/var/www/html 
    #   - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on: 
      - php
      - mysql
```

- Also have to tweak php.dockerfile 
    - because what we RUN won't be in production code only in development so, we copy src folder into working directory of php.dockerfile which is /var/www/html so comment out the `volumes` for php service/container in docker-compose
    - build in the current root directory now instead of referring to dockerfiles folder path

```
  php:
    build:
      context: .
      dockerfile: dockerfiles/php.dockerfile
    # volumes:
    #   - ./src:/var/www/html:delegated
```

```
docker-compose up -d --build server
```

- Will give permission denied error now because php image denies read/write access to our src folder in the container by the container so add this to the phd.dockerfile
    - chown means change ownership of folder/file, change who's allowed to read/write to folders

```
RUN chown -R www-data:www-data /var/www/html
```
