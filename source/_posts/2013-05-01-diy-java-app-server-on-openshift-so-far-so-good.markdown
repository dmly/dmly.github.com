---
layout: post
title: "DIY Java App Server on OpenShift: So Far So Good"
date: 2013-05-01 14:36
comments: true
categories: [programming cloud]
---

I have been playing with [RedHat OpenShift][1] platform for several days. So far the experience has been very good. Several nuances encountered but I was able to get my java server application
running in less than an hour. My application is not using any preconfigured OpenShift's cartridge or application server as I know.
My application is a simple Gomoku game with WebSocket support. I use DropWizard as the light-weight server to bootstrap an embedded version of Jetty. Most of the issues I encounter were just
network interface permission, build, deploy and starting the app.

<!-- more -->

OpenShift comes with several preconfigured deployment platforms which they call cartridges. There are cartrige for Java app servers (JBoss, Tomcat), Python (Django) or Ruby on Rails.
Instead of deploying my [Gomoku][2] app, I just use the bare-bone DIY OpenShift virtual machine. To my surprise the DIY version of OpenShift already comes with JDK 7 (OpenJDK), Maven 3 and Ant 1.8
pre-installed. That saves me a tons of time. Since my Gomoku app is just a regular Maven project that uses [Dropwizard][3] all I need to do is to push my code into the RedHat OpenShift git repository (resides
inside my VM) to build and deploy the app.

There are some procedures I need to follow to make sure my app can run on OpenShift.
First, if the app needs to open a socket connection, I need to make sure that it binds on the permitted IP address. This is defined as $OPENSHIFT_INTERNAL_IP and the only allowed port is
8080. In Dropwizard I can easily configure that in the http session:

``` yaml Dropwizard connection settings
http:
    bindHost: @OPENSHIFT_INTERNAL_IP@
    adminPort: 8080
```

You need to make sure your deploy script (more below) replace the @OPENSHIFT_INTERNAL_IP@ with the actual permitted IP address. You also want to make sure that the adminPort is the same 8080 as the app server port.
OpenShift free tier only allows one port 8080 open as far as I know.

Second, OpenShift has several activation hooks to automate things when you push commits to git. There are hooks for start, stop, deploy and build the app.

For my application, the build hook is just to build using Maven.

``` bash build
cd $OPENSHIFT_REPO_DIR
mvn package
```

After the build, the deploy hook is called. My deploy script is to replace the @OPENSHIFT_INTERNAL_IP@ string with real permitted IP to bind to in the app config file.

``` bash deploy
cd $OPENSHIFT_REPO_DIR
sed -i 's/@OPENSHIFT_INTERNAL_IP@/'"$OPENSHIFT_INTERNAL_IP"'/g' gomoku.yml
```

Finally, the start hook is called. This time I just run the JVM with my jar file. Dropwizard does a great job making this so easy.

``` bash start
cd $OPENSHIFT_REPO_DIR
java -jar target/gomoku-1.0.jar server gomoku.yml
```

After this my Gomoku app is up and ready to serve requests.
To summarize, the whole workflow is:

    *   Push your changes into your OpenShift Git repo
    *   Commit hooks got call, in turn it: builds, deploys and starts your app
    *   Serve!
    
I was new to OpenShift and got the whole done in under an hour. I bet the pre-defined cartriges (JBoss or Tomcat) can even speed up this process even more.    

[1]:https://www.openshift.com/
[2]:https://gomoku-dmly.rhcloud.com/index.html
[3]:http://dropwizard.codahale.com/