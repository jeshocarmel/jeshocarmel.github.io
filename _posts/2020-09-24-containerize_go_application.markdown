---
layout: post
title:  "Containerize a go web application"
date:   2020-09-24 14:21:00 +0800
categories: jekyll update
tags: golang docker devops redis
author: jeshocarmel
comments: true
---

Go is getting more and more popular as the go-to language to build web applications. With this post, I hope to show you how you can containerize a web application written in Go.

{: class="table-of-content"}
* TOC
{:toc}

### The App

  I've written a go web application called [**IP Location Mapper**](https://github.com/jeshocarmel/ip_location_mapper){:target="_blank"} which helps find the location of any IP address. It performs an API call to ipstack.com to fetch the location of the IP address. [ipstack.com](https://www.ipstack.com) provides 10,000 API requests free per month.

  | ![ip location mapper](/assets/images/ip_location_mapper.png){:class="img-responsive"} |
|:--:|
| web application home page |


  Once the API request is made, the response is stored in a redis cache with a TTL of 24 hours for faster retrieval and thereby avoid the expensive API call (also remember, we have only 10,000 free requests).

| ![architecture](https://raw.githubusercontent.com/jeshocarmel/ip_location_mapper/master/architecture.png){:class="img-responsive"} |
|:--:|
| architecture |


You can clone the repository [here](https://github.com/jeshocarmel/ip_location_mapper){:target="_blank"}.

#### Get a free api key from ipstack.com

We would need an API key from ipstack.com for this application. You can sign up for free [here](https://ipstack.com/product){:target="_blank"}. Your API key will be available in the dashboard.

| ![ip stack key](/assets/images/ipstack_api_key.png){:class="img-responsive"} |
|:--:|
| api key in ipstack.com |

> Not don't be cheeky and try to use my api key, I've already reset.

### Dockerfile

  <script src="https://gist.github.com/jeshocarmel/56c51d8b6653d1474597edc9092a8409.js"></script>
  
  The Dockerfile for the application has comments in each line stating their intent. Run the below command inside the project to build an image tagged as jeshocarmel/ip_location_mapper.

  **```docker build -t jeshocarmel/ip_location_mapper:latest .```**

  > The ' . ' at the end of the command indicates that you are building the image from the current directory. So make sure you are  inside the project folder when you run the command.

### Compose

Compose is a tool for defining and running multi-container Docker applications.

<script src="https://gist.github.com/jeshocarmel/7f1b83a6ca70b68afd527d3618c2515c.js"></script>

#### redis service

- line 3 - create a service named **redis**.
- line 4 - pull a **redis image** from docker hub.
- line 5 - Run a **one-off command** on the pulled redis image. Here we pass the command to start the redis server with a password from a .env file (explained later).
- line 6,7 - **open port 6379** for application access.
- line 8,9 - pass environmental variable required for this service.
- line 10 - run the container with name as 'ip_location_mapper_redis'.

#### app service
- line 11 - create a service named app.
- line 12 - the **build** represents that this service an image to be built from the current directory (.)
- line 13 - the image which has been built to be tagged as 'jeshocarmel/ip_location_manager'.
- line 14, 15 - this service depends on the earlier **redis** service.
- line 16, 17 - **open port 8080** for web access.
- line 18 - environmental variables for this service are listed line by line here.
- line 19 - The redis host which the application needs to store/retrieve data is listed here. **In docker-compose you can reach a service by simply mentioning the service name.**
- line 20 - Pass the redis password we used to start the redis service. This will be loaded from a .env file (explained later). The app will use it for authentication with the redis service.
- line 21 - Pass the **API_KEY** from ipstack.com. This will be loaded from a .env file (explained later).
- line 22 - run the container with name as 'ip_location_mapper'.


#### .env file

By default, the docker-compose command will look for a file named .env in the directory you run the command. So create a .env file in your project directory and copy lines from the file below. Replace the ```IPSTACK_API_KEY``` with your API key from ipstack.

<script src="https://gist.github.com/jeshocarmel/96b5edec2b3a88d995ee0423bc6d2330.js"></script>


### Running the application

  **```docker-compose up --build```**

  Thats all you need to start the application. Go to [http://localhost:8080](http://localhost:8080) on your local machine browser and you should be able to see the application up and running.

  | ![ip stack search result](/assets/images/ipstack_app_search_result.png){:class="img-responsive"} |
  |:--:|
  | an ip search from the app |

  run **```docker ps```** in your command line and you should be able to see two containers running.

  ![ip tracker docker ps](/assets/images/ip_tracker_containers.png){:class="img-responsive"}

### Next steps

  In the next post, I'll write on how to deploy the application in kubernetes with minikube.



<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
/*
var disqus_config = function () {
this.page.url = PAGE_URL;  // Replace PAGE_URL with your page's canonical URL variable
this.page.identifier = PAGE_IDENTIFIER; // Replace PAGE_IDENTIFIER with your page's unique identifier variable
};
*/
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://jeshocarmel-github-io.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
                            