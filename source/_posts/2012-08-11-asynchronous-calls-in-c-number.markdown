---
author: Doug Ly
layout: post
title: "Asynchronous Calls in C#"
date: 2010-08-27 14:19
comments: true
categories: programming 
---

Last December I started leading a team develop a Outlook addin for the firm.
I admit that I am no C# developer. The last time I used C# was when I attended college years ago. But the group of consultants
were let go and we have to jump in and take ownership of the project.

I also have to admit that C# has a lot of language features that I really wish Java does too. Among them is the concept of delegate
and function pointer used for asynchronous method calls. I know I can get the same thing with Groovy or Scala.

<!-- more -->

This is how I implement a asynchronous call in C#. The DoAsyncHelper class is an example of extension method, akin to Groovy's category concept.

``` csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading;
using System.Runtime.Remoting.Messaging;
using System.Diagnostics;

namespace AsyncIterator
{
   class Program
   {

       static void Main(string[] args)
       {
           Console.Out.WriteLine("Calling a very slow method...");
           Console.Out.WriteLine("Press Enter to Cancel.");
           Console.Out.WriteLine("Current thread ID {0}", Thread.CurrentThread.GetHashCode());

           int init = 4;

           verySlowMethod.DoAsync(22, resultReady);

           Console.ReadLine();
       }

       public static void resultReady(int result)
       {
           Console.Out.WriteLine("Current thread ID {0} when result ready", Thread.CurrentThread.GetHashCode());
           Console.WriteLine("Result : {0}",  result);
       }

       public static Func<double, int> verySlowMethod = (double n) =>
       {
           Console.Out.WriteLine("Current thread ID {0} in verySlowMethod", Thread.CurrentThread.GetHashCode());
           Thread.Sleep(5000);
           return (int) n *  (int) n;
       };
   }

   static class DoAsyncHelper
   {
       public static void DoAsync<T, V>(this Func<T, V> f, T arg, Action<V> callback)
       {
           f.BeginInvoke(arg, result =>
               {
                   callback(f.EndInvoke(result));
               }, null);
       }

   }
     
 }
```


