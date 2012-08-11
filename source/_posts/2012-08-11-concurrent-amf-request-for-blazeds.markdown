---
author: Doug Ly
layout: post
title: "Concurrent AMF Request for BlazeDS"
date: 2009-07-15 14:12
comments: true
categories: programming 
---

Requirement: AMF data service comes with BlazeDS servlet is asynchronous. One of the services provided is RemoteObject. RemoteObject calls are faster than WebServices and HTTPServices because the information is exchanged in binary.
But RemoteObject itself has a not-too-well-known behavior. While it is true that RemoteObject services are asynchronous, they are not parallel.

<!-- more -->

Example: If you make 3 RemoteObject request simultaneously and let assume the 1st request takes 2 seconds, the 2nd requests takes 10, the 3rd takes 30 seconds. You result will come back in 42 seconds. In other word, you get the result when they ALL complete.
This is because AMF gateway takes all the request and queues them up. It processes each in turn and when all have been run, returns back the results. This is not a bug. 
Most of the time, it is fine. In our application the issue shows its limitation. The app uses client-side caching to improve performance except for memos-related service. 
We also fire multiple requests at the same time, which includes the cached services as well as the memos-related services. Memo-related services would take a few seconds to run.
This in turn defeats the purpose of cached services since all the service requests need to be processed sequentially.

Solution:
We define a separate http channel for the memos-related services to use. Other cached services use a different channel.

``` xml service-config.xml
<channel-definition id="my-amf" class="mx.messaging.channels.AMFChannel">
  <endpoint url="http://{server.name}:{server.port}/{context.root}/messagebroker/amf" class="flex.messaging.endpoints.AMFEndpoint"/>
  <properties>
    <add-no-cache-headers>false</add-no-cache-headers>
  </properties>            
</channel-definition>

<channel-definition id="my-secure-amf" class="mx.messaging.channels.SecureAMFChannel">
  <endpoint url="https://{server.name}:{server.port}/{context.root}/messagebroker/amfsecure" class="flex.messaging.endpoints.AMFEndpoint"/>
  <properties>
    <add-no-cache-headers>false</add-no-cache-headers>
  </properties>
</channel-definition>

<channel-definition id="my-amf2" class="mx.messaging.channels.AMFChannel">
  <endpoint url="http://{server.name}:{server.port}/{context.root}/messagebroker/amf2" class="flex.messaging.endpoints.AMFEndpoint"/>
  <properties>
    <add-no-cache-headers>false</add-no-cache-headers>
  </properties>            
</channel-definition>

<channel-definition id="my-secure-amf2" class="mx.messaging.channels.SecureAMFChannel">
  <endpoint url="https://{server.name}:{server.port}/{context.root}/messagebroker/amfsecure2" class="flex.messaging.endpoints.AMFEndpoint"/>
  <properties>
    <add-no-cache-headers>false</add-no-cache-headers>
  </properties>
</channel-definition>
```

``` xml remoting-config.xml
<destination id="coverageService">
  <properties>
    <factory>spring</factory>
    <source>coverageService</source>
  </properties>
  <channels>
    <channel ref="my-secure-amf" />
    <channel ref="my-amf" />
  </channels>
</destination>

<destination id="activityService">
  <properties>
    <factory>spring</factory>
    <source>memosService</source>
  </properties>
  <channels>
    <channel ref="my-secure-amf2" />
    <channel ref="my-amf2" />
  </channels>
</destination>
```

