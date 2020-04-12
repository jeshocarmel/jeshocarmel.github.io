---
layout: post
title:  "Deploying highly scalable apps calmly, comfortably and efficiently with Docker Swarm"
date:   2020-04-11 18:11:57 +0800
categories: jekyll update
tags: docker devops golang redis mysql
author: jeshocarmel
---

![release_delivered](/assets/images/release_delivered.jpg){:class="img-responsive"}

Have you ever had this moment? or have you ever had this moment as a developer? 

# **Introduction / My Story**

Last two weeks I've been a part of team that is working all round the clock literally to get a **PHP application to production!!** The institution in this picture is still in the midst of establishing a devops team. At the moment they are still traditional i.e. they run apps on VMs with the usual two tier or three tier architecture and manual installation of applications and dbs on their respective servers.

The developer of the PHP application passed the completed codebase to the ops team and the ops team. The ops team that is doing the release is proficient and skilled but it still had to go through this pain of **setting up the web server, the app server, the database, the backup servers, the backup databases**. Then comes the **security testing, the penetration testing, the system testing, integration testing and all the legion of forces**. Once the reports are published, the developers make the changes and the whole process starts over again. I've somewhat summarised the above not taking into account the **port openings** that the team had to make, the **unable to install library** issues, the **encryption of communication** happening between the servers, the god know what...

I was merely an observer in this whole process and I can't wait for this whole thing to end and start my programming life in peace. It was just a clean, plain, smooth PHP application and the **release presumably took more time that it took to write the application.** 

The chaos ended one day and then a question popped to my mind. What if they had to scale? Could they do it without a downtime? I better go sleep.

I can put down in 3 brief points what I learnt from the above experience.

1. Releases are tiring, long and a pain if we are traditional.
2. It's not just the release team but developers also have sleepless nights in a release.
3. Scaling is near to impossible without a downtime if we are traditional.

# **Contents**

In this post, I'm going to walkthrough how to get an application to run on containers and make our release process easy. 

1. `A sample application`
2. `Deploying the application in development`. 
3. `Deploying the application in production`. 
4. `How to scale our deployed application without any downtime`.

# **Requisites**

For this activity we need some basic working skills in the following.
- [`Docker`](https://www.docker.com/){:target="_blank"} - [`Containers`](https://www.docker.com/resources/what-container){:target="_blank"}

In case you are not familiar with **Docker** or **Containers** you can get this book [Docker Deep Dive](https://www.amazon.com/Docker-Deep-Dive-Nigel-Poulton-ebook/dp/B01LXWQUFF) by [Nigel Poulton](https://nigelpoulton.com/).


# **A sample application**

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
- `image` definition same as cache. I chose this service image name to be mysimpleapp_dn.
- `networks` definition same as cache. Other services can reach MySQL only if they are a part of this 'back-tier' network. 
- `volumes` definition same as cache
- `environment` - environmental variables are passed through this parameter as Key: Value. The password for MySQL DB is also passed through the **MYSQL_ROOT_PASSWORD_FILE** where the file location is mentioned as _/run/secrets/mysql_password_. We will revisit this later.
- `secrets` - we will revisit this later. Keep an open mind for now.
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
        tty: true
        container_name: mysimpleapp_app
        secrets:
            - mysql_password
```