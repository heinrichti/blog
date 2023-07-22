---
title: "Be mindful with those shiny things"
date: "2020-12-26"
categories: 
  - "programming"
tags: 
  - "architecture"
  - "patterns"
---

There are a lot of shiny things in the programming world. It might be a paradigm, pattern, technology or anything else really. And not only that, it promises us that it will solve our problems or fix our productivity or whatever they might claim. Oftentimes these technologies, solutions, paradigms or patterns are fantastic for very specific use cases, but get promoted as if it is the perfect [golden hammer](https://sourcemaking.com/antipatterns/golden-hammer) that actually works. Suddenly everyone uses it, sometimes even management hears of it and orders developers to use it, so everyone is happy, right? I mean, it has a really good reputation as everyone else seems to be using it as well, so what could go wrong?

Well I am not happy. Let me make a case against all of those shiny things by saying this: Everything you add to your software will most likely increase the complexity of your software exponentially.

Let´s start of by combining a few things and see where this breaks. I guess these technologies are not too far-fetched and everyone has heard about them: Dependency injection, CQRS and micro services.

Starting off with the shiny parts: Our application is sliced vertically by domain responsibility. These slices are organized in micro services so that we don't mix responsibilities and will be able to scale and deploy every service individually. We separate reads and writes with CQRS so that we don´t mix reads and writes and could have different data stores for reads and writes to optimize for reads. Then we use dependency injection so that we don't have to take care about instantiating our complex object graphs.

From my perspective the last paragraph sounds quite nice, but if these are your motivations to use any of those technologies then this is already deeply broken.

Starting off with micro services: What motivation did I have to using them and what are the alternatives? Comparing to just using a DLL the only thing I see is a massively increased complexity of...  
...deployment  
...managing source code (if stored in different repositories)  
...dev machine setup  
...calling anything in the micro service  
...debugging interaction of micro services  
...keeping performance acceptable

Without any other motivation I would certainly prefer slicing my application with shared libraries instead of micro services, but let´s add more stuff to make it worse and have a look at CQRS. CQRS promises improved performance and scalability and wants to deliver those by simply segregating reads (queries) and writes (commands). Apart from the fact, that most of what we write won´t need this, this is where the fun begins, because most online resources want you to use some kind of framework to achieve this. Oh and when we are already at it we should also add event sourcing, but let´s not go into that rabbit hole. To make it simple just assume a message bus where you throw your reads or writes to and some kind of handler for each read/write. What I see now is the added complexity of finding the right code pieces that belong together. When reasoning about the code it makes it much harder to first have to search for the event handler than just directly following a function call in your IDE. While this is not a problem of CQRS as a pattern at all, it is a problem of most of the suggested CQRS frameworks/implementations.

Now forward with dependency injection (with DI container). The reason to use dependency injection is the question of responsibility. Object creation can get complex and the knowledge how to create a certain object (let´s call it a service, but might be anything really) should not spread to all the places where the service is needed. The more dependencies a service has the worse the problem gets. A common solution is to not create the service where it is used but instead make it a constructor or method argument, so someone else has to create the service, therefore dependency injection. An alternative could have been to create a factory and instead of relying on someone else to provide the dependency just call the appropriate factory method to get the instance. This already adds a little more complexity, because whoever now creates the service has to manage its lifetime, which might be not as easy as it seems at first glance. To manage the creation of those service usually a DI container is added. Some only support injecting interfaces, so let´s use one of those. Which classes should be added to the DI container? Who knows, I usually see added about everything and even had discussions with coworkers that it might be a bad idea to inject an IList with the DI container. To sum it up: We suddenly don´t know anything about the concrete implementation of the service anymore nor about its lifetime. Usually not a problem, but can lead to some unexpected bugs and makes it harder to reason about the code, therefore more complexity.

Each of those "things" are not bad, but not necessarily good. But I spoke of exponential complexity, but by now each of these patterns only added a little complexity, right? Ok looking at them together now, from the perspective of someone who recently inherited the application, where a certain web request is too slow:

We have a web request handler which has a `IMessageBus` as a parameter. All it does is send a new `TraceRequestCommand` to the message bus, and after that a `StoreSomethingCommand`. Alright, let´s just assume that the message bus does what it should do (which we might have to double check later, mental note), so where is the message handled? Searching for usages of the `TraceRequestCommand` yields about a thousand usages. If we are lucky our IDE let´s us filter out instantiations. After a few minutes we find the corresponding handler. This handler has a `ITraceService` injected and calls the `TraceRequest` method on it. Searching for the implementation of this service we find three different implementations, but also which one of them is registered in the DI container, so this should be our implementation (if no one did something "clever", so might have to double check later, mental note). In the `TraceService` we see a web request to another micro service. After we found the correct repository to clone we can now have a look at the code there. Let´s say the team for this micro service followed the same structure, then we will have to do everything again, and are finally, a few minutes later, in our handler. If we are lucky we find the problem there (like a `Thread.Sleep` someone forgot), if we are unlucky we have to follow our `StoreSomethingCommand`, go to another micro service, which calls two other micro services and so on. All in all maybe three lines of code I would describe "not-plumbing code", but an hour lost searching for these three lines of code, which are distributed to three repositories and for which I had to read about a hundred lines of code during which I have to keep a mental model of our network, how many network calls I already made and how this might affect performance. Combine this mental model with the mental notes I mentioned previously and we already don´t have a lot of free mental resources to take care of the problem itself: The request is slow.

Compare this to not using any of these patterns: We still have our web request handler:

public void HandleWebRequest(SomethingParams params)
{
    Trace.Writeline(nameof(HandleWebRequest));
    Thread.Sleep(1000);
    using (var service = new SomethingService())
        service.Store(params);
}

Found the problem yet? If not we can just hit F12 on `Store` to follow the reference and read on. If our code stays consistently simple we should be done in a minute or two.

Yes, the code won´t stay simple and yes, all of the patterns described above are useful when used in moderation. All I tried to say is this: Don´t mindlessly follow the latest trends. Be mindful when introducing anything to your project. If you don´t need it in 99,9% of the cases you probably don´t need it as a generalized pattern in your codebase that everyone should use. Ask which problem are you solving by introducing this shiny thing. If it doesn´t solve a real-world problem that you actually have then don´t use it at all!
