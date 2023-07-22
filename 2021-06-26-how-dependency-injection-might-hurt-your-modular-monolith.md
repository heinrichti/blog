---
title: "How dependency injection might hurt your modular monolith"
date: "2021-06-26"
categories: 
  - "programming"
tags: 
  - "net"
  - "architecture"
---

I have seen many legacy applications with no clear architecture that have some maintainability issues. What I noticed then is that someone wants to "modernize" it, so a few developers have a big meeting and decide that it is time to make our monolith a little more modular. These modules should be clearly separated and located within their own assembly. If you want to do DDD you are probably thinking about bounded contexts now.

So far this all sounds about right, but this is the point where I think most of them have gone wrong: How should the modules communicate with each other? Clearly the modules have to talk to each other and not only in one direction, but then you cannot have circular references between assemblies. It seems that the golden hammer answer to this specific problem is dependency injection. We could just have the contracts to all modules in its own assembly per module (or one contracts-assembly for all modules) and reference it from everywhere. The DI container will take care of providing the implementation, so I don't have to care about referencing other modules anymore. So all is good, right?

Fast forward a year or two. Suddenly we have hundreds of modules, developers seem to like the idea of having micro-modules, where some only contain a handful of classes (yes, this is a problem in itself and should not have happened). Our now totally majestic modular monolith needs up to 30 seconds for a cold startup on a fast computer with a SSD and using ngen. Management is not happy how the application doesn't feel snappy anymore and customers complain that they "have to fetch a coffee" every time they start our application. How did this happen, and what does it has to do with anything I talked about earlier?

Think about how our dependency container can resolve those interfaces: We have to register the concrete implementation to the interface. Usually I see this done by using reflection on all referenced assemblies, which already takes time by itself and gets slower the more assemblies we are referencing. But let´s just assume that we are not using reflection at all, but somewhere have a really long list of handwritten registrations, like this, which gets ugly real quick:

container.Register<IType1, MyType1>();
container.RegisterSingleton<IType2, MyType2>();
...

What we have to know is that not all assemblies are loaded from disk as soon as we start an application. Instead the assembly is loaded as soon our program flow stumbles across code from that assembly. Usually we would not use the stuff from one of our modules until the user executes some kind of feature that is located in the module. But now as soon as we register our classes in the DI-container all of our assemblies are loaded from disk to memory. Each assembly has a little overhead in load time and memory footprint and boom, our application starts so slow that even our loading screen that we build because our application was getting slow does not seem to help the issue anymore.

Let´s face it, even developer motivation and productivity is going down, as every time we compile and start the application we have to wait way longer than we would wish we had to.

Another problem now is testing. Whenever we want to test something, we would have to either initialize our DI-container properly, mock away all the dependencies or instantiate them ourself. This is either much more work than I want to invest in my tests (leading to worse test coverage) or makes the tests pretty slow (postponing valuable feedback). This will lead to lower quality overall.

Now that we are in this situation, what can we do to fix this? Or maybe even more important, what could we have done to avoid the issue from the start?

First off, an easier fix if you already have hundreds of modules and it would be too much work to completely change how the modules communicate with each other: Source generators are a thing now and you can replace your DI-container by generated code. [StrongInject](https://github.com/YairHalberstadt/stronginject) does a good job at this. This way you do not register your dependencies at runtime, therefore loading all your modules, but instead write code to get your dependencies at compile time. The module is loaded only when someone requests a dependency to it at runtime then. No registrations at runtime, no startup overhead.

But what I also want to promote is using the mediator pattern to communicate between modules. Further do proper inversion of control: Define the interfaces you need yourself inside your own module and have a mediator implement them. This way your module does not have to have any knowledge about the rest of the modules. No references to any kind of contract assembly, if someone changes an interface in their module my module does not have to change, only the mediator(s) that call code from the module might have to change. I have truly decoupled my module from the other modules. Of course, my mediator(s) have to be up and running during runtime and tests, and somehow provided to my modules. But now we are talking one assembly that gets loaded and has to be registered in the DI-container instead of hundreds. This also makes the test setup so much simpler, just instantiate the mediator you need instead of hooking up all registrations in the DI-container.

If you need some kind of initialization logic inside your modules, I would therefore strongly advice against doing this at startup, instead you might want to call it from your mediator, lazy at runtime when the feature is needed. AOP might come in handy here as otherwise you might have to write some kind of initialized-check in each method of the mediator.

What do you think of all this? Do I describe a non-issue as this would has never happened on the projects you have faced or is your experience similar to mine, that you see this kind of pitfall in basically every second legacy project you inherit? How many modules does your application load on startup? You can see the loaded modules in [Process Explorer](https://docs.microsoft.com/en-us/sysinternals/downloads/process-explorer).
