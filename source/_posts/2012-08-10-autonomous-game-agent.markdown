---
layout: post
author: Doug Ly
title: "Autonomous Game Agent"
date: 2008-10-23 18:57
comments: true
categories: programming
---

I am always interested in game AI programming, specially autonomous game agents which move around and act on their own will but based on a set of pre-defined rules of course. 
After reading the first chapter of [Programming Game AI by Example] [1] by Matt Buckland, I can understand the magic behind those intelligent behaviors. I am very impressed to know how simple it is  to create those behaviors. But it's definitely not simple if one is not familiar with trigonometry and geometric algebra.

The code presented in this book is all C++. And I have a hard time to make those code built in Visual C++ 2008. It is guaranteed to work with Visual C++ 2005 only!
So I rolled up my sleeves and wrote my own code in Java. And I think this is the best way to test how much I understand the book.

<!-- more -->

So far, I have finished coding the following behaviors: arrive, seek, avoid, wandering and obstacle avoidance.

I put those behaviors in a single applet illustrated here. It's just a bunch of flies flying around trying to avoid their predator, a spider may be!

<object classid="clsid:8AD9C840-044E-11D1-B3E9-00805F499D93" codebase="http://java.sun.com/update/1.4.2/jinstall-1_4_2_12-windows-i586.cab" width="500" height="500" standby="Loading Processing software..."  >	
<param name="code" value="com.lyfam.game.agent.WanderingAvoidance" />
<param name="archive" value="http://autonomous-game-agent.googlecode.com/svn/trunk/SteeringBehavior/wandering.jar" />

<param name="mayscript" value="true" />
<param name="scriptable" value="true" />
	
<param name="image" value="loading.gif" />

<param name="boxmessage" value="Loading Processing software..." />
<param name="boxbgcolor" value="#000000" />
					
<param name="test_string" value="inner" />
				
<p>
<strong>
This browser does not have a Java Plug-in.
<br />
<a href="http://java.sun.com/products/plugin/downloads/index.html" title="Download Java Plug-in">
Get the latest Java Plug-in here.
</a>

</strong>
</p>

</object>

[1]: http://www.amazon.com/Programming-Game-Example-Mat-Buckland/dp/1556220782/ref=sr_1_5?ie=UTF8&s=books&qid=1224810375&sr=8-5
