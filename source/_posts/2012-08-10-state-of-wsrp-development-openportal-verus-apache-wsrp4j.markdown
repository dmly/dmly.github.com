---
author: Doug Ly
layout: post
title: "State of WSRP Development: OpenPortal verus Apache wsrp4j"
date: 2009-01-14 20:27
comments: true
categories: programming
---

with wsrp4j project has become more and more dormant, I am seeking for another wsrp framework. 
To the best of my knowledge, beside the commercial framework out there from BEA Portal and IBM Websphere, Apache's wsrp4j used to be the 
only open source solution until Sun steps in and sponsors java.net's OpenPortal. 
 My intention is to only replace the wsrp producer, but it is not a trivial task to just simply swap one producer and use it with the existing portlet container implementation. 
So go with OpenPortal container and its wsrp I am right now. 

<!-- more -->

So far, my feeling with OpenPortal implementation is nothing but impressed. I'm impressed from 
how easy it is to setup the container/producer in Tomcat, the ease of portlet deployment to the active support of OpenPortal's community. 
Of course, I encounter a few glitches here and there but they all could be overcome within a matter of hours by looking at the code or requesting support of the community. 

What about performance, how does OpenPortal perform compare to Apache portal suite. 
I gear up and setup the two framework to go head to head for a benchmark test. 

What I use:
<table width="100%" cellspacing="1" cellpadding="1" border="1" align="center">
    <tbody>
        <tr>
            <td align="center"><strong>OpenPortal</strong></td>
            <td align="center"><strong>Apache Portal</strong></td>
        </tr>
        <tr>
            <td>OpenPortal portlet container milestone 4 (July 2008) <br />
            OpenPortal wsrp implementation milestone 4 (July 2008)</td>
            <td>Apache Portlet Container Pluto 1.0.1 Release (some time 2006??) <br />
            Apache wsrp4j Revision 327501 on August 28 2005 (they have not a stable release yet)</td>
        </tr>
        <tr>
            <td>Web Services stack: JAX-WS Sun's implementation</td>
            <td>Web Services stack: Apache Axis 1.3</td>
        </tr>
    </tbody>
</table>

All run from JDK 1.6 and tomcat 6 servlet. 

How do I conduct the test: 
-  In the consumer side I setup a servlet filter to capture the time to complete a request: this can be used to benchmark the web services stacks. 
-  in the producer side I also setup a filter to capture the time the producer take to process the request: this can be used to benchmark the portlet containers. 
I'm making one assumption here: producers' performance difference from both implementations can be ignored since it delegates most of its work to the portlet 
container. 

-  I create a simple portlet JSP generating a huge content: table with 500 rows full of text, roughly 3 MB in its final rendered HTML. 
-  Make 10 requests and take the average: discard the result of the very first request due to overhead of initialization of the wsrp engine from both framework 
I notice that OpenPortal takes less time to initialize. When both engines are ready, their performance are pretty identical:
<table width="100%" cellspacing="1" cellpadding="1" border="1">
    <tbody>
        <tr>
            <td align="center"><strong>Benchmark Item</strong></td>
            <td align="center"><strong>OpenPortal Consumer</strong></td>
            <td align="center"><strong>Apache wsrp4j Consumer</strong></td>
            <td align="center"><strong>OpenPortal Producer </strong></td>
            <td align="center"><strong>Apache wsrp4j Producer</strong></td>
        </tr>
        <tr>
            <td># of Requests</td>
            <td>10</td>
            <td>10</td>
            <td>10</td>
            <td>10</td>
        </tr>
        <tr>
            <td>Min Time (ms)</td>
            <td>1406</td>
            <td>1641</td>
            <td>438</td>
            <td>468</td>
        </tr>
        <tr>
            <td>Avg Time (ms)</td>
            <td>1807.8</td>
            <td>1736.3</td>
            <td>611.2</td>
            <td>666.9</td>
        </tr>
        <tr>
            <td>Max Time (ms)</td>
            <td>1985</td>
            <td>2032</td>
            <td>1047</td>
            <td>859</td>
        </tr>
    </tbody>
</table>

*Conclusion*
If we continue to choose open source solution for our portal, OpenPortal would be the best choice at this moment. 
wsrp4j has not had any activity recently (3 years). Its community support is weak. The code is unstable: latest unstable release (0.5) won't even compatible with any Pluto container. 
While Apache pluto/jetspeed container already supports portlet spec 2.0 wsrp4j is still clinging at spec 1.0 

from the last couple of weeks working with OpenPortal I feel the support is pretty strong. Whenever I report a bug, a response always follows the next day. Sometimes 
it comes with patches as well. OpenPortal will not work out of the box at this moment with our existing consumer but it would not take much time or effort to fix it. 
best of all, OpenPortal comes with the support of portlet 2.0 spec and wsrp version 2 spec!
