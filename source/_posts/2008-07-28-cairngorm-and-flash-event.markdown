---
layout: post
author: Doug Ly
title: "Cairngorm and Flash Event"
date: 2008-07-28 18:46
comments: true
published: false
categories: [programming]
---

For my Flex project at work, I use Cairgorm as the MVC framework. Not that I like Cairgorn a lot but at the time our project starts it seems to be the de-facto framework for mid-to-big-mess Flex applications. Adobe also stands behind Cairgorm in supporting and consulting it.

The learning curve is not so steep. I pick up the tutorial material available at Adobe website, specially the animated diagram is very helpful and learn to use the framework in less than a day. The second day I already begin to integrate it into our application.

One thing annoys me most about this framework is that there is no way (sort of) for the UI to get notified when a remote call is done getting data and the result is available. I end up use some hack to get around it. For example, use a ChangeWatcher to monitor the model. When the result ready, the command object updates the model, just triggers the ChangeWatcher's propertyChange event.

<!-- more -->

This is kind of messy and the more I use ChangeWatcher, the more I tend to stay away from it if I could. Until I find a solution...

Remember the event class you must extend from the CairngormEvent (not Flash event)?

You just need to include a function object (yes, a functor) from the UI component you want Cairgorm to notify when the result is ready to the cairgorm event object before dispatching it. The in your command class, when the result come back, just invoke that function object.

 Sound complicated? Not so, here it is in code

In your event class:

``` actionscript Event Class
class SampleEvent extends CairngormEvent
{
  public var someStuff:Object;
  public var callbackFunctionObj:Function;
}
```
In your Command class

``` actionscript Command Class
class SampleCommand implements Command, Responder

{
  private var callbackFunction:Function;
  public void execute(event:CairgormEvent):void
  {
      callbackFunction = event.callbackFunction;
  }

  public void result(event:Event):void
  {
     // setting your model with the result
      ....
     callbackFunction.call(null);
  }
}
```

``` actionscript In The UI
<script>
    var event:SampleEvent = new SampleEvent;
    event.someStuff = new Stuff()
    event.callbackFunctionObj = this.notfyMeWhenDataReady;
   event.dispatch();

    public void notfyMeWhenDataReady():void
    {
       // do whatever you want when the data is ready here
    }
</script>
```

When the result is ready, the command object will invoke the callback function. Thus in the above example notfyMeWhenDataReady will be called!
