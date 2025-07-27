# Docker tutorial

Learn how to use bind mounts, run multi-container apps, connect to a MySQL database and use Docker Compose.

From the official [Docker tutorial page](https://www.docker.com/101-tutorial/).

## Bind mounts

Bind mounts are used to mount our source code into the container to let it see code changes, respond, and let us see the changes right away.

1. Starting a Dev-Mode Container

To run our container to support a development workflow, we will do the following:
- Mount our source code into the container
- Install all dependencies, including the "dev" dependencies
- Start nodemon to watch for filesystem changes

```
cd app

docker run -dp 3000:3000 \
    -w /app \ 
    -v "$(pwd):/app" \
    node:18-alpine \
    sh -c "yarn install && yarn run dev"
```
    
- `-dp 3000:3000` : run in detached (background) mode and create a port mapping
- `-w /app` : sets the container's present working directory where the command will run from
- `-v "$(pwd):/app"` : bind mount (link) the host's present getting-started/app directory to the container's /app directory. Note: Docker requires absolute paths for binding mounts, so in this example we use pwd for printing the absolute path of the working directory, i.e. the app directory, instead of typing it manually
- `node:18-alpine` : the image to use. Note that this is the base image for our app from the Dockerfile
- `sh -c "yarn install && yarn run dev"` : the command. We're starting a shell using sh (alpine doesn't have bash) and running yarn install to install all dependencies and then running yarn run dev. If we look in the package.json, we'll see that the dev script is starting nodemon.

2. Watch the logs

```
docker ps
docker logs -f <container-id>
```

3. Make some changes in the app 

For example, in the `src/static/js/app.js` file, change the "Add Item" button to simply say "Add" (line 109).

4. Refresh the `localhost` page to see the changes

5. Stop the container and build the new image

```
docker rm -f <container-id>
docker build -t getting-started .
```

## Multi-container apps

Update the application stack with 2 containers : one hosting the app and the other the MySQL database.

![multi-app-architecture](img/multi-app-architecture.png "multi-app-architecture")

### Start the MySQL container

1. Create the network
```
docker network create todo-app
```
2. Start a MySQL container and attach it to the network
```
docker run -d \
    --network todo-app \
    --network-alias mysql \
    -v todo-mysql-data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    -e MYSQL_DATABASE=todos \
    mysql:8.0
```
 
- `-d` : run in detached mode
- `--network todo-app` : the network to connect to
- `--network-alias mysql` : define an alias for the network IP
- `-v todo-mysql-data:/var/lib/mysql` : mount a volume to the MySQL database
- `-e MYSQL_ROOT_PASSWORD=secret` : define environment variable MYSQL_ROOT_PASSWORD
- `-e MYSQL_DATABASE=todos` : define environment variable MYSQL_DATABASE
- `mysql:8.0` : the image tag to run
    
Note that we're using a volume named `todo-mysql-data` here and mounting it at `/var/lib/mysql`, which is where MySQL stores its data. However, we never ran a `docker volume create` command. Docker recognizes we want to use a named volume and creates one automatically for us.

3. Check that the database is up and running

```
docker ps
docker exec -it <mysql-container-id> mysql -p
```

Enter the password `secret`.

List the databases with
```
SHOW DATABASES;
```
You should see an output like this :
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| todos              |
+--------------------+
5 rows in set (0.00 sec)
```
Check that the `todos` database is listed.

Exit the sql terminal with `exit`.

### Identify the host name

1. Start a new container using the `nicolaka/netshoot` image and connect it to the same network.
```
docker run -it --network todo-app nicolaka/netshoot
```
2. Lookup the IP address for the hostname `mysql`.
```
dig mysql
```
3. Look for the host name in the `ANSWER SECTION` :

```
; <<>> DiG 9.18.8 <<>> mysql
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 32162
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;mysql.             IN  A

;; ANSWER SECTION:
mysql.          600 IN  A   172.23.0.2

;; Query time: 0 msec
;; SERVER: 127.0.0.11#53(127.0.0.11)
;; WHEN: Tue Oct 01 23:47:24 UTC 2019
;; MSG SIZE  rcvd: 44
```

The host name is `mysql`.

### Launch the application stack

1. Run the app and connect it to the database

```
docker run -dp 3000:3000 \
  -w /app \
  -v "$(pwd):/app" \
  --network todo-app \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=secret \
  -e MYSQL_DB=todos \
  node:18-alpine \
  sh -c "yarn install && yarn run dev"
```

- `-dp 3000:3000` : run in detached mode and create a port mapping
- `-w /app` : sets the container's present working directory where the command will run from
- `-v "$(pwd):/app"` : bind mount (link) the host's present `app` directory to the container's `/app` directory.
- `--network todo-app` : the network to connect to
- `-e MYSQL_HOST=mysql` : define environment variable
- `-e MYSQL_USER=root` : define environment variable
- `-e MYSQL_PASSWORD=secret` : define environment variable
- `-e MYSQL_DB=todos` : define environment variable
- `node:18-alpine` : the image to run
- `sh -c "yarn install && yarn run dev"` : the command to run.


2. Check that the container is connected to the mysql database.

Watch the logs :

```
docker ps
docker logs -f <container-id>
```

3. Check that the database is being updated.

- Open the app in your browser and add a few items to your todo list.
- Connect to the mysql database.
```
docker exec -it <mysql-container-id> mysql -p todos
```
The password is `secret`.

- In the MySQL shell, print the database items :
```
select * from todo_items;
```
Check that the items are being written to the database.

## Docker Compose

We can simplify this process and define multi-container applications using Docker Compose.

1. Open the file named `docker-compose.yml`.
```
services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:8.0
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment: 
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

2. Close any running container.
```
docker ps
docker rm -f <ids>
```
3. Start up the application stack.
```
docker compose up -d
```
4. Look at the logs
```
docker compose logs -f
```
5. Stop the app
```
docker compose down
docker compose logs -f
```
