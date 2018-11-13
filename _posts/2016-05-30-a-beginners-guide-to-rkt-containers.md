---
id: 6
title: A Beginner’s Guide to rkt Containers
date: 2016-05-30T18:40:40+00:00
author: johnpettigrew404
layout: post
guid: http://107.170.124.236/2016/05/30/a-beginners-guide-to-rkt-containers/
permalink: /2016/05/30/a-beginners-guide-to-rkt-containers/
tumblr_john-pettigrew_permalink:
  - http://john-pettigrew.tumblr.com/post/145164704657/a-beginners-guide-to-rkt-containers
tumblr_john-pettigrew_id:
  - "145164704657"
categories:
  - rkt
---
If you know me, then you know I am a <a href="https://www.docker.com/" target="_blank">Docker</a> fan-boy. Docker offers a lot in terms of tooling. As an attempt to avoid vendor lock-in, I chose to research rkt and some of the features it offers. Rkt is maintained by <a href="https://coreos.com" target="_blank">Coreos</a>. Coreos has also developed several other tools to assist in managing containers. (Some of which I might talk about in future blog posts.) Note: These scripts are available on <a href="https://github.com/john-pettigrew/rkt-blog-post-data" target="_blank">my Github page</a>.

I won’t go much into the installation of Rkt, there are great instructions available on the website. Once installed, we can run a simple

    rkt
    

to see some of the available options.

# Images

Rkt uses ACI files for its images. However, it also has the ability to download and convert docker images from Docker registries and Docker Hub! Let’s try running the nginx image from Docker Hub.

<pre><code class="bash">
rkt fetch --insecure-options=image docker://nginx
</code></pre>

Note: the “–insecure-options=image” is needed because Docker images can’t be verified by rkt like ACI files. We use the “docker://” prefix to tell rkt that this is a docker image. See <a href="https://coreos.com/rkt/docs/latest/running-docker-images.html" target="_blank">this link</a> Now, let’s list our local images.

<pre><code class="bash">
rkt image list
</code></pre>

You should see nginx listed. We can also export this image to an ACI file by running..

<pre><code class="bash">
rkt image export registry-1.docker.io/library/nginx:latest nginx.aci
</code></pre>

There should now be an “nginx.aci” in your current directory. Now, lets create a container.

# Creating containers

<pre><code class="bash">
rkt run docker://nginx
</code></pre>

Note: Exit a running container by pressing ‘Ctrl’ and ’]’ three times.
  
You should see some output. Let’s try pointing our web browser to “localhost”…

<figure class="tmblr-full">![error page](https://raw.githubusercontent.com/john-pettigrew/rkt-blog-post-data/master/images/browser_error.png)</figure> 

As you can tell, it seems like our service is not running. However, if we run

<pre><code class="bash">
rkt list
</code></pre>

<figure class="tmblr-full">![rkt list](https://raw.githubusercontent.com/john-pettigrew/rkt-blog-post-data/master/images/rkt_list_1.png)</figure> 

We can see that the nginx container is accessable at “172.16.28.3”.

<figure class="tmblr-full">![welcome to nginx](https://raw.githubusercontent.com/john-pettigrew/rkt-blog-post-data/master/images/nginx_welcome.png)</figure> 

We can also run our container with the “–net=host” option to link the host network to the container and access it at “localhost” like so.

<pre><code class="bash">
rkt run --net=host docker://nginx
</code></pre>

# Accessing files

Let’s say we want to access our container throught the shell for debugging purposes similar to how one might access a running Docker container using “docker exec”. We can easily run…

<pre><code class="bash">
rkt enter f2323923
</code></pre>

(f2323923 is the uuid of my container. Yours will differ.)
  
Now, we are running “/bin/bash” in our container. Let’s add some text to our nginx welcome page.

<pre><code class="bash">
apt-get update
#I am using vim here, feel free to use a different text editor.
apt-get install -y vim
vim /usr/share/nginx/html/index.html
</code></pre>

Now we can see our edited page by refreshing our browser.

<figure class="tmblr-full">![updated page](https://raw.githubusercontent.com/john-pettigrew/rkt-blog-post-data/master/images/nginx_welcome_2.png)</figure> 

It would be very tedious to run these commands everytime we wanted to modify our html page. We can solve this by mounting a volume in our container.

<pre><code class="bash">
mkdir html
rkt run --net=host --volume data,kind=host,source=$PWD/html docker://nginx --mount volume=data,target=/usr/share/nginx/html
</code></pre>

If we try to navigate our browser to our new nginx container, we get a 403 page. This is because our new html folder is empty. We can add an “index.html” page on our host machine inside this html directory and reload our browser to see it updated.

<figure>![new page](https://raw.githubusercontent.com/john-pettigrew/rkt-blog-post-data/master/images/nginx_welcome_3.png)</figure> 

# Creating ACIs

Now, let’s talk about building our own ACIs. I will be using a tool called (acbuild)[https://github.com/appc/acbuild]. This tool allows us to write a script file and build images similar to how a Dockerfile works. However, one benefit is that we gain the power and flexibility of the shell. So, I highly recommend you install this tool. Let’s start by creating a script for our container to run.

<pre><code class="bash">
#! /bin/sh
while :
do
  time=`date "+%x %X"`
  echo $time
  echo $time &gt;&gt; /app/log/logs
  sleep 5
done
</code></pre>

This script simply logs the current date and time to the console and to a file every 5 seconds. Next, we can write the script that will build our ACI file.

<pre><code class="bash">
#! /bin/bash
set -e

acbuild begin
acbuild set-name john.pettigrew.rocks/log
</code></pre>

Here, we setup our script. I named mine “john.pettigrew.rocks/log”. Next, we’ll base it on <a href="http://www.alpinelinux.org/" target="_blank">Alpine Linux</a>. An interesting thing to note is that <a href="http://quay.io/coreos/alpine-sh" target="_blank">quay.io/coreos/alpine-sh</a> is also visitable by your web browser.

<pre><code class="bash">
# based on alpine
acbuild dep add quay.io/coreos/alpine-sh
</code></pre>

Now we create a directory and add our script that we created earlier.

<pre><code class="bash">
# add our script
acbuild run -- mkdir -p /app/log
acbuild copy ./log.sh /app/log.sh
</code></pre>

And, finally we tell rkt to run our script when our container starts and write our image to a file.

<pre><code class="bash">
# run our script
acbuild set-exec -- /app/log.sh

# write file
acbuild write --overwrite log-container.aci
acbuild end
</code></pre>

Since this is a normal shell script, we could have done a lot of interesting things, like setting environment variables from our host and conditionally adding things to our final container image. Now, let’s try out our new container image. (I saved mine as “log_container.sh”.)

<pre><code class="bash">
chmod +x ./log_container.sh
./log_container.sh
rkt run --insecure-options=image ./log-container.aci
</code></pre>

Note: We have to add the “–insecure-options=image” because we don’t have an asc file. See <a href="https://coreos.com/rkt/docs/latest/signing-and-verification-guide.html" target="_blank">this page</a>.

We can see that our container is printing to the console and if we use the following, we should also see the logs file being updated.

<pre><code class="bash">
rkt enter [YOUR CONTAINER'S UUID] /bin/sh
tail -f /app/log/logs
</code></pre>

# Multi-Container Pods

Let’s put together some of the information we have learned today and run two containers in the same pod and share a mounted volume between them. We have the option to define mount points while we build our image and also when we run our containers. Since our images are already built, we’ll do the latter.

<pre><code class="bash">
rkt run --debug --insecure-options=image --volume timelog,kind=empty registry-1.docker.io/library/nginx:latest --mount volume=timelog,target=/usr/share/nginx/html  --net=host ./log-container.aci --mount volume=timelog,target=/app/log
</code></pre>

This command seems complicated but when we break it down, it’s really not. We tell rkt to run just like we did before with the insecure option. We then create a volume “timelog” for our containers to share. (We specify “empty” instead of “host” since our volume won’t be on the host.) We run the nginx container specifying our mount point and binding it to the host’s network. Finally, we start our custom time log container and set a mount point for it.
  
The result can be seen by navigating to “localhost/logs” in our browser. A “logs” file should download containing our date logs.

There are many benefits to using rkt. For example, using scripts to create our images gives us a great level of flexibility for our containers. Also, with ACI files, we can setup container storage without having to run a special “registry”. As we’ve now seen, many of the common tasks we use Docker for can be accomplished with rkt, and I have just barely even scratched the surface of how powerful it really is. I highly recommend giving it a try and seeing if it fits in your workflow.

– John P.