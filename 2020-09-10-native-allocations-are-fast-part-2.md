---
title: "Native allocations are fast - Part 2"
date: "2020-09-10"
categories: 
  - "programming"
tags: 
  - "c"
  - "unmanaged"
---

In the [last blogpost](https://wholesomeprogrammer.eu/2020/09/09/myth-native-allocations-are-fast/) we talked about me, being puzzled that my linked list is slower if I use Marshal.AllocHGlobal instead of managed allocations with new. I might not be able to give a definitive answer to this, but I certainly can give some educated guesses!

There are lots of things that need to happen when I create an object with the new keyword, right? Like allocating memory, registering it for garbage collection, calling its constructor and so on. But is that actually correct? Or to put it in another way: What does allocating memory in the .NET Framework actually mean? And when we are already at it let's ask the one question nobody I know would have a proper answer for: Why is the stack faster than the heap and what are those concepts anyway, when looking at the .NET Framework.

I'd like to start by making the observation that when I allocate a small managed .NET object, my program will _not_ have to go to the operating system and ask it for some more memory. Instead we already have a lot of memory available! Ever heard of GC generations? What these are is basically just a bunch of continous memory regions. Heck, actually they can actually be the same continous memory region, the distinction in generations is more of a logical view on the memory.

So think about this: Instead of asking the operating system for new memory every time we need to allocate a new tree node, we just allocate a continous memory region once and every time someone asks for memory we already have it! Basically we start by giving out the first few bytes and remember how many bytes are already used. The next new, the next few bytes gone, and so on. Sounds pretty efficient, right?

> "Hey operating system, do you have some memory for me of this size? Please write zeros into every byte! k thx!".

Now please do not think that this is what is happening under the hood when we allocate a new object on the heap. But you know what works this way? The stack! And this is _one_ of the reasons why the stack is supposedly faster than the heap, as its allocation and garbage collection is trivial (just move the pointer). Of course this description is a little simplified, but if you are interested in all the gory details: It is called a Bump Pointer Allocator.

So how does it help in our quest to find the reason for our slow allocations? Let's try to put these things together and formulate a theory. We learned that one reason the stack is so fast because basically our program already have the memory and only has to move pointers back and forth (and fill the memory with zeros). I also talked about how the GC generations basically are just a bunch of continuous memory segments. My theory is that it is much faster to allocate a decently sized continuous block of memory than lots of small memory blocks and that the .NET Framework hides away all the complexity of that! And think about how much more it does with this memory, all the garbage collection stuff that we haven't even started talking about. When I think about how amazing this is, from a performance and a usability point of view, I am simply in awe.

Now let's try to put this knowledge into practice and implement our own little bump pointer allocator. If our theory is correct this shoud be faster than the .NET new implementation of our linked list. In its simplest form it could look like this:

public unsafe class BumpPointerAllocator : IDisposable
    {
        private readonly IntPtr \_memory;
        private byte\* \_current;

        public BumpPointerAllocator()
        {
            \_memory = Marshal.AllocHGlobal(sizeof(FLinkedList.ListItem) \* 1024);
            \_current = (byte\*) \_memory.ToPointer();

        }

        public byte\* Alloc(int size)
        {
            var result = \_current;
            \_current += size;
            return result;
        }

        private void ReleaseUnmanagedResources()
        {
            Marshal.FreeHGlobal(\_memory);
        }

        public void Dispose()
        {
            ReleaseUnmanagedResources();
            GC.SuppressFinalize(this);
        }

        ~BumpPointerAllocator()
        {
            ReleaseUnmanagedResources();
        }
    }

Now let´s run our benchmark from the previous post again, with the LinkedList using our BumpPointerAllocator-implementation as well:

        public void Add(int i)
        {
            ListItem\* newItem = (ListItem\*) \_allocator.Alloc(sizeof(ListItem));
            ...
        }

|               Method |      Mean |     Error |    StdDev | Ratio | RatioSD |
|--------------------- |----------:|----------:|----------:|------:|--------:|
|            Unmanaged | 47.119 us | 0.9431 us | 2.1858 us |  7.04 |    1.09 |
|              Managed |  6.910 us | 0.3795 us | 1.0579 us |  1.00 |    0.00 |
| BumpPointerAllocator |  2.972 us | 0.0649 us | 0.1913 us |  0.44 |    0.07 |

Now we are finally there, our code is more than twice as fast as the managed version! Is it usable? Most likely not, as we can only add stuff to our list, not delete items. For this we would have to put in a lot more effort and this might take us to basically reimplementing .NET memory management or live with a lot of allocated data in memory we are not referencing anymore. I think the only use-case for this right now is when you have an append-only data structure, know just how much memory you need and have a lot of small objects to allocate (or extend the implementation to allocate multiple segments on demand), AND want to ditch all the advantages that managed code has (like safety), then... well... go for it ;-)
