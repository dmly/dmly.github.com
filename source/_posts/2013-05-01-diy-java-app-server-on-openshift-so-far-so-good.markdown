---
layout: post
title: "DIY Java App Server on OpenShift: So Far So Good"
date: 2013-05-01 14:36
comments: true
categories: [programming cloud]
---

I have been playing with RedHat OpenShift platform for several days. So far the experience has been very good. Several nuiances encoutered but I was able to get my java server application
running in less than an hour. My application is not using any preconfigured OpenShift's cartrige or application server as I know.
My application is a simple Gomoku game with WebSocket support. I use DropWizard as the light-weight server to bootstrap an embedded version of Jetty. Most of the issues I encoutnered were just
network interface permission, build, deploy and starting the app.

<~-- more -->
