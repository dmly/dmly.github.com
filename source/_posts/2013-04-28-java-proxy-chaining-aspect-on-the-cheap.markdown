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
is a much cleaner and prefer way for your service calls Data access objects (DAO).

At the end you could decorate your method with annotations/aspects like this:

``` java Example
@Timeit
@Retry(times=3)
@Cache(timeToLiveInSeconds=3600)
public void goGetMyData(String someParam, int anotherOne)
{
    // do something
}
```

<!-- more -->

Let's define an example interface for the DAO and its implementation.

``` java Interface
interface Say
{
    public void say();
}

public void say()
{
    Random rand = new Random();
	int r = rand.nextInt(9);
	if (r > 7)
	{
		throw new RuntimeException("CRAP please retry!!");
	}
	
    System.out.println("Say hello");
}
```

I intentionally throw a RuntimeException 30% of the times this method run to simulation transient error that could be retried.
Now is the fun part: we will add additional functionalities over this method without modifying its code. As the begining I want to time the method performance
and retry if it fails (up to 3 times before I give up).

The easiest way to do this is to use annotation to denote your new aspects.

``` java Timing Aspect Annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@interface Timeit
{  
}
```

And the Retry aspect with maximum of 3 retries before giving up:

``` java Retry Aspect
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@interface Retry
{
    int times() default 3;
}
```

In order to facilitate the annotations in the java dynamic proxy, we need to create an InvocationHandler for each of those annotations.
For this I first borrow the utility class from "Java Reflection in Action". You can get full source at the end of this post.
I then create a base Interceptor on top of this invocation handler to make the dynamic proxy generation handling annoations easier.

``` java AbstractInvocationHandler and The base Invoker
public abstract class AbstractInvocationHandler<T> implements InvocationHandler
{

    protected T nextTarget;
    protected T realTarget = null;

    public AbstractInvocationHandler(T target)
    {
        nextTarget = target;
        if (nextTarget != null)
        {
            realTarget = findRealTarget(nextTarget);
            if (realTarget == null)
                throw new RuntimeException("findRealTarget failure");
        }
    }
    ...
}

interface RealInvoker
{
    public Object invoke() throws Throwable;
}

interface Invoker<A extends Annotation>
{
	public Object execute(final Method method, final Object[] args, A ann, RealInvoker realInvoker) throws Throwable;
}

class Interceptor<T, A extends Annotation> extends AbstractInvocationHandler<T>
{
    ...
	
    public static <T, A extends Annotation> T createProxy(T obj, Class<A> annotationClass, Invoker<A> invoker)
    {
        return (T) Proxy.newProxyInstance(obj.getClass().getClassLoader(), obj
                .getClass().getInterfaces(), new Interceptor(obj, annotationClass, invoker));
    }
    
	@Override
	public Object invoke(Object proxy, final Method method, final Object[] args)
			throws Throwable {
		Object result = null;
		A annotation = (A) this.realTarget.getClass().getMethod(method.getName()).getAnnotation(annotationClass);
        if (annotation != null)
        {
        	return invoker.execute(method, args, annotation, new RealInvoker() {
				@Override
				public Object invoke() throws Throwable {
					return method.invoke(Interceptor.this.nextTarget, args);
				}
			});
        }
        else
        {
            result = method.invoke(this.nextTarget, args);
        }

        return result;
	}
}
```

The nice thing in doing this is that in order to create an aspect based on an annotation you just need to implement the Invoker interface shown above.
Then you can create the dynamic proxy of the targeted object by calling:

``` java Creating the Dynamic Proxy
// Some dao needed to be wrapped in Timeit aspect
SomeDao dao = ...
dao = Interceptor.createProxy(dao, Timeit.class, new TimerAspect());
```

Interceptor.createProxy takes 3 arguments: the targeted object to be proxied, the aspect annotation class and the object to handle the aspect.
For the Timer (or Timeit) aspect, it could be as simple as this:

``` java Timeit Aspect
public class TimerAspect implements Invoker<Timeit>
{
    @Override
	public Object execute(Method method, Object[] args, Timeit ann,
			RealInvoker realInvoker) throws Throwable {
		long start = System.nanoTime();
		Object result = realInvoker.invoke();
		System.out.println(String.format("-- %s tookkkk %s ms", method.getName(), (System.nanoTime() - start) / 1000000.0));
		return result;
	}    
}
```

Here is why this is an aspect: the execute method takes note of the current time. It then invokes the original method call. Finally it calculate
how long this method call takes. I believe in AspectJ this is called "before and around pointcut".

Similarly, I would create the Retry aspect by implementing the Invoker interface and call

``` java Creating Retry aspect
dao = Interceptor.createProxy(dao, Retry.class, new RetryAspect());
```

After we have the aspects to handle those annotations, how do chain them in a correct order, an order which makes sense at all?
It all depends on your aspects' logic but in this case I would make the Timer aspect outside of the Retry aspect. Confused? Here is the order of execution:

    1.  Enter the Timer aspect, take note of the current time
    2.  Enter the Rety aspect, retry count set to 0
    3.  Invoke the actual Dao method
    4.  If it fails, retry aspect catch the exception and retries! It keeps track of the number of retries (up to 3 times by default)
    5.  Either the call fails if retries exceed 3 times or it exits the Retry aspect and yield the command to Timer Aspect again
    6.  Timer aspect calculate how long this Dao method takes
    7.  Return the result to the caller

One thing you need to pay close attention is the order of the execution of those chained aspects influenced by the way you create them.
The inner most aspect will need to be created last. The outer most aspect will need to be created first.
For this example, this is the order of aspect creation:

``` java Order of Aspect creation
SomeDao dao = ...
dao = Interceptor.createProxy(dao, Retry.class, new RetryAspect());
dao = Interceptor.createProxy(dao, Timeit.class, new TimerAspect());
```

Here is the complete code in 2 simple classes. I hope you find this useful.

``` java AbstractInvocationHandler.java
import java.lang.reflect.*;

public abstract class AbstractInvocationHandler<T> implements InvocationHandler
{

    protected T nextTarget;
    protected T realTarget = null;

    public AbstractInvocationHandler(T target)
    {
        nextTarget = target;
        if (nextTarget != null)
        {
            realTarget = findRealTarget(nextTarget);
            if (realTarget == null)
                throw new RuntimeException("findRealTarget failure");
        }
    }

    protected final T getRealTarget()
    {
        return realTarget;
    }

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

    public static Field findField(Class cls, String name)
            throws NoSuchFieldException
    {
        if (cls != null)
        {
            try
            {
                return cls.getDeclaredField(name);
            } catch (NoSuchFieldException e)
            {
                return findField(cls.getSuperclass(), name);
            }
        } else
        {
            throw new NoSuchFieldException();
        }
    }
}
```

``` java Main.java

import java.lang.annotation.Annotation;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.Random;

interface RealInvoker
{
    public Object invoke() throws Throwable;
}

interface Invoker<A extends Annotation>
{
	public Object execute(final Method method, final Object[] args, A ann, RealInvoker realInvoker) throws Throwable;
}

class Interceptor<T, A extends Annotation> extends AbstractInvocationHandler<T>
{
	private Class<A> annotationClass;
	private Invoker<A> invoker;
	
	private Interceptor(T target, Class<A> annotationClass, Invoker<A> invoker)
    {
        super(target);
        this.annotationClass = annotationClass;
        this.invoker = invoker;
    }
	
	public static <T, A extends Annotation> T createProxy(T obj, Class<A> annotationClass, Invoker<A> invoker)
    {
        return (T) Proxy.newProxyInstance(obj.getClass().getClassLoader(), obj
                .getClass().getInterfaces(), new Interceptor(obj, annotationClass, invoker));
    }
	
	@Override
	public Object invoke(Object proxy, final Method method, final Object[] args)
			throws Throwable {
		Object result = null;
		A annotation = (A) this.realTarget.getClass().getMethod(method.getName()).getAnnotation(annotationClass);
        if (annotation != null)
        {
        	return invoker.execute(method, args, annotation, new RealInvoker() {
				@Override
				public Object invoke() throws Throwable {
					return method.invoke(Interceptor.this.nextTarget, args);
				}
			});
        }
        else
        {
            result = method.invoke(this.nextTarget, args);
        }

        return result;
	}
	
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@interface Timeit
{  
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@interface Cache
{  
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@interface Retry
{
	int times() default 3;
}

interface Say
{
    public void say();
}

class ProxyFactory<T>
{
	public static <T> T createProxy(T obj)
	{
		obj = Interceptor.createProxy(obj, Retry.class, new Invoker<Retry>()
		{
			@Override
			public Object execute(Method method, Object[] args, Retry ann,
					RealInvoker realInvoker) throws Throwable {
				Object result = null;
				
				int retries = 0;
				while (retries < ann.times())
				{
					try
					{
						result = realInvoker.invoke();
						break;
					}
					catch (Throwable crap)
					{
						retries++;
						System.out.println(String.format("Crap catched %s times", retries));
					}					
				}
				
				if (retries >= ann.times()) throw new RuntimeException("Can't handle it anymore");
				
				return result;
			}
		});
		
		obj = Interceptor.createProxy(obj, Timeit.class, new Invoker<Timeit>()
		{
			@Override
			public Object execute(Method method, Object[] args, Timeit ann,
					RealInvoker realInvoker) throws Throwable {
				System.out.println("-- Enter the timer");
				long start = System.nanoTime();
				Object result = realInvoker.invoke();
				System.out.println(String.format("-- %s tookkkk %s ms", method.getName(), (System.nanoTime() - start) / 1000000.0));
				return result;
			}
		});
		
		return obj;
	}
}

public class Main implements Say
{
    public static void main(String... args)
    {
        Say main = new Main();
        main = ProxyFactory.createProxy(main);
        
        for (int i = 0; i < 10; i++)
        {
        	main.say();
        	System.out.println(" ##### \n");
        }
    }

    @Timeit
    @Retry
    public void say()
    {
    	Random rand = new Random();
    	int r = rand.nextInt(9);
    	if (r > 5)
    	{
    		throw new RuntimeException("CRAP rety!!");
    	}
    	
        System.out.println("Say hello");
    }
}
```

``` java Cache Aspect
public CacheAspect implements Invoker
{
    ConcurrentMap<String, ConcurrentMap<Object, Object>> cache = new ConcurrentHashMap<>();
    		
    @Override
    public Object execute(Method method, Object[] args, Cache ann,
    		RealInvoker realInvoker) throws Throwable
    {
    	System.out.println("@@ Entering cache proxy with " + method.getName() + " " + args);
        ConcurrentMap<Object, Object> theCache = cache.get(method.getName());
        if (theCache == null)
        {
        	theCache = new ConcurrentHashMap<>();
        	cache.put(method.getName(), theCache);
        }
        
        Object key = args;
        if (args == null)
        {
        	key = "";
        }
        Object result = theCache.get(key);
        if (result != null)
        {
        	System.out.println("@@ Cache hit with " + method.getName() + " " + args);
        }
        else
        {
        	System.out.println("@@ Cache missed with " + method.getName() + " " + args);
        	result = realInvoker.invoke();
        	theCache.put(key, result == null ? "" : result);
        }
        
        System.out.println("@@ Exiting cache proxy");
        
    	return result;
    }
}    
```