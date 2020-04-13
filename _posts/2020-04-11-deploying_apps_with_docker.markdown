---
layout: post
title:  "Deploying highly scalable apps calmly, comfortably and efficiently with Docker Swarm [part 1]"
date:   2020-04-11 18:11:57 +0800
categories: jekyll update
tags: docker devops golang redis mysql
author: jeshocarmel
---

![release_delivered](/assets/images/release_delivered.jpg){:class="img-responsive"}

Have you ever had this moment? or have you ever had this moment as a developer? 

# **Introduction / My Story**

Last two weeks I've been a part of team that is working all round the clock literally to get a PHP application to production!! The institution in this picture is still in the midst of establishing a devops practice. At the moment they are still traditional where they run apps on VMs with the usual two tier or three tier architecture and manual installation of applications, dbs and the rest.

The developer of this PHP application passed the completed codebase to the release team. The release team that has been put in charge of deploying the app is highly proficient and experienced but still had to go through this agony of **setting up the web server, the app server, the database, the backup servers, the backup databases, the firewalls, the security configurations, etc.**. They are not done yet. The testing team takes over then with the **security testing, the penetration testing, the system testing, integration testing and all the available software testing approaches in this planet**. Once the test reports are published and the findings made available, the developer makes the changes and the part of the process (or the whole) starts over again. I've summarised the process above not taking into account the **additional port openings requests** that the team had to make, the **unable to install library** issues, the **encryption of communication** happening between the servers, the god know what...

I was merely an observer in this whole process and I can't wait for this whole thing to end. Imagine the headaches the developer and the release team would have had during this whole two weeks time. On the contrary, it was just a clean, plain, smooth PHP application which worked perfectly in the development environment and now the **release of this app presumably took more time that it took to write the application itself.** Arghhh! 

![ops problem](/assets/images/ops_problem.png){:class="img-responsive"}



The chaos ended one fine day and then a question popped my mind. What if they had to scale? Could they do it without a downtime? I think I better stop writing now.

I can put down what I picked up from my above experience in 3 points.

1. **Releases are tiring, long and a strain if we are still following traditional release practises.**
2. **It's not just the release team but the developers also have sleepless nights until their apps go to production.**
3. **Scaling is near to impossible without a downtime in a traditionally released application.**

# **Contents**

In this two-part post, I'm going to walkthrough how to get an application to run on **containers** and make our release process simple and easy. 


1. **`Requisites`**
2. **`SimpleApp- A sample application`**
3. **`Deploying the application in development`.**
    - `Docker Compose`
    - `Cache`
    - `DB`
    - `App`
    - `Starting the Application`
    - `Summary`

4. **`Deploying the application in production [part 2]`.** 
5. **`Scaling our application in production [part 2]`.**

>This blog post covers the simpler way of all the above 4 topics with docker compose, stack and swarm and not Kubernetes.

# **Requisites**

For this activity we need some basic working skills in the following.
- [`Docker`](https://www.docker.com/){:target="_blank"} - [`Containers`](https://www.docker.com/resources/what-container){:target="_blank"}

In case you are not familiar with **Docker** or **Containers** you can get this book [Docker Deep Dive](https://www.amazon.com/Docker-Deep-Dive-Nigel-Poulton-ebook/dp/B01LXWQUFF) by [Nigel Poulton](https://nigelpoulton.com/).


# **SimpleApp- A sample application**

SimpleApp is a web based application that has a form to collect data about your favourite food. The app uses MySQL as a database and Redis as a cache to keep track of how many visits the application has received.

- Language: **`Go`** 
- Database: **`MySQL`**
- Cache: **`Redis`**
- Frontend: **`Bootstrap 4.0`**, **`JQuery`**
- Repository: [**```https://github.com/jeshocarmel/mysimpleapp```**](https://github.com/jeshocarmel/mysimpleapp){:target="_blank"} 

| ![simpleapp_architecture](/assets/images/mysimpleapp_architecture.png){:class="img-responsive"} |
|:--:|
| SimpleApp's Architecture |


| ![simpleapp_home](/assets/images/simple_app_home.png){:class="img-responsive"} |
|:--:|
| SimpleApp's home screen. Page visits loaded from cache |

| ![simpleform](/assets/images/simple_form.png){:class="img-responsive"} |
|:--:|
| Simple Form within the app |

| ![simpleform_success](/assets/images/simple_form_success.png){:class="img-responsive"} |
|:--:|
| Form submissions are stored in MySQL |


I hope the above pictures clearly explains the application flow and the purpose. Since this post is about deploying application, we won't go much in detail about the codebase.


# **Deploying the application in development**

To deploy a multi-layer application using container in a dev environment, we have to write a docker-compose file.

>Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration.
>
> -- <cite>Docker official documentation</cite>

So we need a .yml file to start our 3 services namely the `App`, `MySQL` and `Redis`. But first let's have a look at the folder structure of our codebase.


![simpleapp_folder_structure](/assets/images/simpleapp_folder_structure.png){:class="img-responsive"}

Now that the folder structure is clear, we are going to disect the docker-compose.yml file.

#### **Docker Compose**


``` yml
version: '3.1'
services: 
    cache:
        <content here>
    db:
        <content here>
    app:
        <content here>

volumes:
        <content here>
networks:
        <content here>
secrets:
        <content here>
```
- `version` represents the version of docker-compose used.
- `services` listed here 1-by-1 are the services/layers that are part of this whole application. 
- `volumes` listed here are the persistent storage file systems that will be used by this project i.e. they are not erased even after shutdown.
- `networks` listed here are the network channels through which communication should happen betweek the respective layers.
- `secrets` is a feature of docker swarm which helps to send/store passwords among layers with auto-encryption.

Let's go deep into each of these `services`.

#### **Cache**

```yml
version: '3.1'
services: 
    cache:
        build: 
            context: ./redis_db
        image: jeshocarmel/mysimpleapp_cache
        networks:
            - cache-tier
        volumes: 
            - cache_data:/dump.rdb
        container_name: mysimpleapp_cache
```

- `build-context` represents the directory in which all the relevant files for this service / layer is available.
- `image` represents the intended name for this image. I chose this service image name to be mysimpleapp_cache.
- `networks` is the **network layer in which other services can reach this service**. Other services can reach redis only if they are a part of this 'cache-tier' network. 
- `volumes` creates a **persistent storage** that exists out of the container. Hence the data inside this storage is not deleted in case of failure / shutdown.
- `container_name` mentions the name the container which launches the image should have.

#### **Database**

```yml
db:
        build: 
            context: ./mysql_db
        image: jeshocarmel/mysimpleapp_db
        volumes: 
            - db_data:/var/lib/mysql
        networks: 
            - back-tier
        environment: 
            TZ: Asia/Singapore
            MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_password
            MYSQL_DATABASE: simple_db
        command: 
            --explicit_defaults_for_timestamp
            --default-authentication-plugin=mysql_native_password
        secrets:
            - mysql_password
        container_name: mysimpleapp_db
```

- `build-context` definition same as cache.
- `image` definition same as cache. I chose this service image name to be mysimpleapp_db.
- `networks` definition same as cache. Other services **can reach MySQL only if they are a part of this 'back-tier' network.** 
- `volumes` definition same as cache
- `environment` - environmental variables are passed through this parameter as **Key: Value**. The password for MySQL DB is also passed through the **MYSQL_ROOT_PASSWORD_FILE** where the file location is mentioned as _/run/secrets/mysql_password_. We will revisit this later.
- `secrets` - we will cover secrets later in this post.
- `container_name` definition same as cache.

#### **App**

```yml
app:
        build: 
            context: ./app
        image: jeshocarmel/mysimpleapp_app
        environment: 
            WAIT_HOSTS: db:3306
            MYSQL_DATABASE: simple_db
        depends_on: 
           - cache
           - db
        networks:
            - back-tier
            - cache-tier
        ports: 
            - "80:80"
        container_name: mysimpleapp_app
        secrets:
            - mysql_password
            - redis_password
```

- `build-context` definition same as cache.
- `image` definition same as cache. I chose this service image name to be mysimpleapp_app.
- `networks` definition same as cache. **This app is part of back-tier and cache-tier networks.** 
- `environment` - environmental variables are passed through this parameter as **Key: Value**. WAIT_HOSTS is an environmental variable is passed for a component in 'app' service to wait for the database server to be up before starting the application.
- `ports` - By default, when you create a container, it does not publish any of its ports to the outside world. To make a port available to services outside of Docker, or to Docker containers which are not connected to the container’s network, use the --publish or -p flag. This creates a firewall rule which maps a container port to a port on the Docker host.
- `secrets` - we will cover secrets later in this post.
- `container_name` definition same as cache.

#### **Networks, Volumes & Secrets**

```yml
volumes:
    cache_data:
    db_data:

networks: 
    cache-tier:
    back-tier:

secrets:
    mysql_password:
        file: ./devsecrets/mysql_password
    redis_password:
        file: ./devsecrets/redis_password
```

- `volumes` - As said before, volumes are the preferred mechanism for **persisting data** generated by and used by Docker containers. The volumes used in mysimpleapp are listed here.

- `networks` - Simply put, Docker networking is the **native container SDN solution** you have at your disposal when working with Docker. Docker networking allows you to attach a container to as many networks as you like. The networks used in SimpleApp are listed here.

- `secrets` - In terms of Docker Swarm services, **a secret is a blob of data, such as a password, SSH private key, SSL certificate, or another piece of data that should not be transmitted over a network or stored unencrypted in a Dockerfile or in your application's source code**. But in a development environment, it is not necessary or to rather say its not possible to use docker secrets. Hence we **redirect our secrets to our local folder 'devsecrets' for the containers** to fetch and load these secrets. However in docker swarm i.e. in production docker secrets it's a whole other ball game. 

If you would like to know more about docker volumes, please do watch this short video below. 

<a href="http://www.youtube.com/watch?feature=player_embedded&v=p2PH_YPCsis
" target="_blank"><img src="http://img.youtube.com/vi/p2PH_YPCsis/0.jpg" 
alt="IMAGE ALT TEXT HERE" width="240" height="180" border="10" /></a>


#### **Starting the Application**

Now that we have got our `docker-compose.yml` file ready. Let's build and start our application.

The command for this would be **`docker-compose up --build`**

console output when running the command:

```
Building cache
Step 1/3 : FROM redis
 ---> de25a81a5a0b
Step 2/3 : COPY redis.conf /usr/local/etc/redis/redis.conf
 ---> Using cache
 ---> 79ac6d2c56d9
Step 3/3 : CMD [ "redis-server", "/usr/local/etc/redis/redis.conf" ]
 ---> Using cache
 ---> 708de55d1ae7
Successfully built 708de55d1ae7
Successfully tagged jeshocarmel/mysimpleapp_cache:latest
Building db
Step 1/3 : FROM mysql:latest
 ---> d435eee2caa5
Step 2/3 : EXPOSE 3306
 ---> Using cache
 ---> 9f60f83c18d7
Step 3/3 : COPY schema.sql /docker-entrypoint-initdb.d/schema.sql
 ---> Using cache
 ---> 42103796fb81
Successfully built 42103796fb81
Successfully tagged jeshocarmel/mysimpleapp_db:latest
Building app
Step 1/10 : FROM golang:latest
 ---> a1072a078890
Step 2/10 : WORKDIR /app
 ---> Using cache
 ---> 9f59226f1eed
Step 3/10 : COPY go.mod go.sum ./
 ---> Using cache
 ---> 3b99000a0622
Step 4/10 : RUN go mod download
 ---> Using cache
 ---> 35e92d59b211
Step 5/10 : COPY . .
 ---> 052fe1397154
Step 6/10 : RUN go build -o main .
 ---> Running in c86a7218a47b
go: finding github.com/go-sql-driver/mysql v1.5.0
go: downloading github.com/go-sql-driver/mysql v1.5.0
go: extracting github.com/go-sql-driver/mysql v1.5.0
Removing intermediate container c86a7218a47b
 ---> 04d630491c85
Step 7/10 : EXPOSE 80
 ---> Running in 91d18e1fe667
Removing intermediate container 91d18e1fe667
 ---> aecf7314ad86
Step 8/10 : ADD https://github.com/ufoscout/docker-compose-wait/releases/download/2.2.1/wait /wait

 ---> 082172241487
Step 9/10 : RUN chmod +x /wait
 ---> Running in 3b9ea369cf81
Removing intermediate container 3b9ea369cf81
 ---> a6aaf6a8996e
Step 10/10 : CMD /wait && ./main
 ---> Running in cc329f404219
Removing intermediate container cc329f404219
 ---> 47a1d14ddd03
Successfully built 47a1d14ddd03
Successfully tagged jeshocarmel/mysimpleapp_app:latest
Starting mysimpleapp_cache ... done
Starting mysimpleapp_db    ... done
Recreating mysimpleapp_app ... done
Attaching to mysimpleapp_cache, mysimpleapp_db, mysimpleapp_app
app_1    | Checking availability of db:3306
app_1    | Host db:3306 not yet available
cache_1  | 1:C 13 Apr 2020 02:46:54.484 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
cache_1  | 1:C 13 Apr 2020 02:46:54.484 # Redis version=5.0.6, bits=64, commit=00000000, modified=0, pid=1, just started
cache_1  | 1:C 13 Apr 2020 02:46:54.484 # Configuration loaded
cache_1  |                 _._                                                  
cache_1  |            _.-``__ ''-._                                             
cache_1  |       _.-``    `.  `_.  ''-._           Redis 5.0.6 (00000000/0) 64 bit
cache_1  |   .-`` .-```.  ```\/    _.,_ ''-._                                   
cache_1  |  (    '      ,       .-`  | `,    )     Running in standalone mode
cache_1  |  |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
cache_1  |  |    `-._   `._    /     _.-'    |     PID: 1
cache_1  |   `-._    `-._  `-./  _.-'    _.-'                                   
cache_1  |  |`-._`-._    `-.__.-'    _.-'_.-'|                                  
cache_1  |  |    `-._`-._        _.-'_.-'    |           http://redis.io        
cache_1  |   `-._    `-._`-.__.-'_.-'    _.-'                                   
cache_1  |  |`-._`-._    `-.__.-'    _.-'_.-'|                                  
cache_1  |  |    `-._`-._        _.-'_.-'    |                                  
cache_1  |   `-._    `-._`-.__.-'_.-'    _.-'                                   
cache_1  |       `-._    `-.__.-'    _.-'                                       
cache_1  |           `-._        _.-'                                           
cache_1  |               `-.__.-'                                               
cache_1  | 
cache_1  | 1:M 13 Apr 2020 02:46:54.488 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
cache_1  | 1:M 13 Apr 2020 02:46:54.488 # Server initialized
cache_1  | 1:M 13 Apr 2020 02:46:54.488 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
cache_1  | 1:M 13 Apr 2020 02:46:54.489 * DB loaded from disk: 0.000 seconds
cache_1  | 1:M 13 Apr 2020 02:46:54.489 * Ready to accept connections
db_1     | 2020-04-13 10:46:54+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.18-1debian9 started.
db_1     | 2020-04-13 10:46:54+08:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
db_1     | 2020-04-13 10:46:54+08:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.18-1debian9 started.
db_1     | 2020-04-13T02:46:55.073998Z 0 [Warning] [MY-011070] [Server] 'Disabling symbolic links using --skip-symbolic-links (or equivalent) is the default. Consider not using this option as it' is deprecated and will be removed in a future release.
db_1     | 2020-04-13T02:46:55.074169Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.18) starting as process 1
db_1     | 2020-04-13T02:46:55.683128Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
db_1     | 2020-04-13T02:46:55.687729Z 0 [Warning] [MY-011810] [Server] Insecure configuration for --pid-file: Location '/var/run/mysqld' in the path is accessible to all OS users. Consider choosing a different directory.
db_1     | 2020-04-13T02:46:55.733007Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.18'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server - GPL.
db_1     | 2020-04-13T02:46:55.809000Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Socket: '/var/run/mysqld/mysqlx.sock' bind-address: '::' port: 33060
app_1    | Host db:3306 is now available
app_1    | 2020/04/13 02:46:56 PONG <nil>
app_1    | 2020/04/13 02:46:56 DB connection successful !! <nil>
```

That's it. Our cache is up, our DB is up and then our App is up and they all come together and sing **'Kumbaya'**

Run `docker ps` in cmd and you will see this. Three containers running in perfect harmony.

![docker ps output](/assets/images/docker_ps_screenshot.png){:class="img-responsive"}

To bring down the application, the command would be **`docker-compose down`**

### **Summary (Part 1)**

- We were able to bring our whole architecture up in a single command. 
- We have configured networks for communication between app with cache and vice versa with cache-tier and app with db and vice versa with back-tier. 
- We created persisted storage for both redis and MySQL.
- We configured our passwords and passphrases via devsecrets.
- We monitor all our containers by a single command i.e. `docker ps`

In part 2, we will explore [docker swarm](https://docs.docker.com/engine/swarm/){:target="_blank"} where we can easily spin off replicas of our components and also explore on docker secrets where we can store our passwords securely in a production environment. We will end part 2 by dynamically scaling components of SimpleApp without any downtime.