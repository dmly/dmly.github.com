---
author: Doug Ly
layout: post
title: "RIA Portlet Made Possible With Flex Remoting"
date: 2009-03-15 13:50
comments: true
categories: programming
---

Even though portlet development standard has been in existence for a while its popularity and advanced functionality hasn't got a lot of changes comparing to traditional web application development. Up until now, the *only* way to develop a portlet application is to use the dated (but obviously dominant) JSP technology. I know there have have been several attempts to bring advanced view engines into JSR-168/JSP-286 such as JSF prolet bridge or even the Wicket Portlet bridge the lanscape overall is pretty naive and dificult to comprehense. I can refer to [this artical][1] when it comes to the reasons why portlet has not become dominant as if it should be by now.

<!-- more -->

In this story I will demonstrate my attempt to bring RIA development to a portlet environment. The technology stack I am using includes: Sun's OpenPortal portlet container, Flex RIA with blazeDS remoting. One of the most challenging obstacles developers face in developing RIA in a portlet environment is the asynchronous communication channel between the UI and the portlet itself. Portlet development somewhat is very similar to that of servlet development most of the time. But when it comes to technologies that tightly integrated into servlet, things start to break: try to put DWR/Wicket/Tapestry... in a portlet environment you would feel the pain!

My attempt to put the BlazeDS remoting into a portlet environment was a success. But it doesn't mean I didn't run into any bumps along the way.
Here how I did it: I have a regular portlet applicaption which provides nothing but a single JSP view. Then I create a Flex component which will be embedded in the JSP view. The Flex component has a single combo box widget which will make a remote RPC call via BlazeDS remoting to get the list data for itself. Now in the portlet application (which is nothing more than regular web app), I add blazeDS and all of the remote services support. To describe the setup is beyond the scope of this article but you can find many tutorials/articles online regarding the config. So far this is what we have:

   * A portlet app with blazeDS remoting support
   * A Flex component which will get data from remoting services
   * A JSP view to embed the Flex component

Now comes the critical section: making Flex component calling the remoting services from a portlet app.
Normally when you build a flex application with remoting services, you provide a fixed services-config.xml containing the channel/endpoint configuration.
This won't work in a portlet environment since you don't know the hostname, port numbers, context root, etc... before hand. All of those configurations depend on the portlet container implementation. To solve this issue I configure the Flex remoting services at runtime. In the JSP view embedding the Flex component I add a flashvars parameter to hold the BlazeDS remoting URL context. I advice that you use the standard portlet API to generate this URL instead of hardcoding it. For example in the JSP view, I get the remote service URL using this:

``` html JSP
<p><span style="font-family: Courier New;"><object ...></span><span style="font-family: Courier New;"><br>
...<br>
<span style="color: rgb(255, 0, 0);"><param name="flashvars" value='serviceUrl=<%= renderResponse.encodeURL(renderRequest.getContextPath()<br>
    + "/messagebroker/amf") %>' /></span><br>
<embed src='<%= renderResponse.encodeURL(renderRequest.getContextPath()<br>
      + "/portlet.swf") %>'<br>
    ...<br>
    <span style="color: rgb(255, 0, 0);">flashVars='serviceUrl=<%= renderResponse.encodeURL(renderRequest.getContextPath()<br>
      + "/messagebroker/amf") %>'</span><br>
            ><br>
        </embed><br>
</object></span></p>
```

Now in the flex component, you need to create the remote channel/endpoint programmatically:

``` actionscript Channel
var serviceUrl:String = Application.application.parameters.serviceUrl;
var _channelSet:ChannelSet = new ChannelSet();
var _amfChannel:AMFChannel = new AMFChannel("my-amf", serviceUrl);
_channelSet.addChannel(_amfChannel);
                
remoteObj = new RemoteObject("remoteService");
remoteObj.channelSet = _channelSet;
remoteObj.addEventListener(ResultEvent.RESULT, onready);
remoteObj.addEventListener(FaultEvent.FAULT, onfault);
```

One of the coolest thing about putting RIA in a portlet via Flex/BlazeDS remoting services is that you get asynchronous services for free! It means no whole page refresh, ever, for your portal.
Another nice thing is that since we use the standard portlet API to expose the BlazeDS services, all communication between your Flex component and the back-end services will be proxied through the portlet container resource servlet. What it means is that you can literally WSRP-enable your Flex portlet application if you want to and things would continue to work: no firewall setting to mess with, no blazeDS services to reconfigure.

{% img /images/flex-portlet.jpeg [Flex Portlet in OpenPortal] %}

Source code and binary files:
[flex-portlet.war][2] (to be deployed in a portlet container)
[flex-portlet.zip][3] (Source)

[1]:http://today.java.net/pub/a/today/2009/01/20/jsr-286-portlet-irrelevance.html
[2]:http://velcrotag.googlecode.com/files/flex-portlet.war
[3]:http://velcrotag.googlecode.com/files/flex-portlet.zip
