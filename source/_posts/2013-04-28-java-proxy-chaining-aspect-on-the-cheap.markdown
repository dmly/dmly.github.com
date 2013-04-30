---
layout: post
title: "Java Proxy Chaining - Aspect on the Cheap"
date: 2013-04-28 21:01
comments: true
categories: [programming java] 
---

How many of you have written a caching solution for one of your expensive method calls? Or how about retry logic for your service calls?
Living in such a connected world no single application is its own island. For every applications I have worked for, there always external dependencies, either other service APIs or
databases that we need to rely upon. And we can't make the assumption that they are reliable 100% either.

Some service calls are expensive and the underlying data don't change so often. We tend to cache the results. A naive approach would be to find such methods and refactor them 
so that cache can be used before the call to the services are made. The better way is to use AOP where you intercept those calls with 'aspects'. The in those aspects you can make the decision to whether get the results from the cache or to make the expensive service calls.
Another common scenario is retry when you encounter exceptions in your service calls: transient network issues (latency, timed out, spillover in load balancer...) or database hiccups.
Normally, you should at least retry the calls for several times before giving up. Like noted previously, a naive approach would be to go every methods and apply the retry logic. Or you could use AOP.

In this post, I'm going to talk about how to use aspect oriented way to easy the refactoring effort. I will not talk about the full blown bytecode level AOP solution which uses AspectJ
with bytecode weaving. Instead, I will talk about a lighter weight of aspect programming using the Java's dynamic proxy and its reflection mechanism. I think it's pretty similar to the way Spring
AOP works. The only difference is that my code will assume every method calls implement interfaces. Thus, it will not have to use cglib to generate the proxies. Also, I think programming to interface
is a much cleaner and prefer way for your service calls Data access objects.

<!-- more -->

``` java Invocation
protected static final <T> T findRealTarget(T t)
    {
        if (!Proxy.isProxyClass(t.getClass()))
            return t;
        InvocationHandler ih = Proxy.getInvocationHandler(t);
        if (AbstractInvocationHandler.class.isInstance(ih))
        {
            return (T) ((AbstractInvocationHandler) ih).getRealTarget();
        } else
        {
            try
            {
                Field f = findField(ih.getClass(), "target");
                if (Object.class.isAssignableFrom(f.getType())
                        && !f.getType().isArray())
                {

                    f.setAccessible(true); // suppress access checks
                    Object innerTarget = f.get(ih);
                    return (T) findRealTarget(innerTarget);
                }
                return null;
            } catch (NoSuchFieldException e)
            {
                return null;
            } catch (SecurityException e)
            {
                return null;
            } catch (IllegalAccessException e)
            {
                return null;
            } // IllegalArgumentException cannot be raised
        }
    }

```
