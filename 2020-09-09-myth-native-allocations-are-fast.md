---
title: "Myth: Native allocations are fast"
date: "2020-09-09"
categories: 
  - "programming"
tags: 
  - "c"
  - "unmanaged"
---

Lately I have been dipping my toes into unmanaged programming with C#. The goal was to improve the performance of some data structures that I have been working on. I thought that, maybe, I could improve the performance of them by using some unmanaged code. I noticed in the profiler that a lot of the time was just _wasted_ on allocating memory and my initial thought was that this might be faster if I allocate my objects per Marshal.AllocHGlobal instead of good old new().

For the sake of the argument and that you don't have to read papers and papers on the data structures, let's just assume we are talking about a linked list.

For those who are not familiar with linked lists: A linked list is a data structure to hold a sequence of elements. The idea is that each element in the list also holds a reference to the next element in the list. This allows us to traverse the list in order just by following these references. In its simplest form a linked list is just a container holding the reference to the first element. We augment this by also holding a pointer to the last element. This makes adding an item a contant time operation or O(1).

Something like this in managed code (simplified to hold a int instead of a generic T):

    class MLinkedList
    {
        private ListItem \_first;
        private ListItem \_last;

        private class ListItem
        {
            internal ListItem Next;
            internal int Item;
        }

        public MLinkedList()
        {
            \_first = null;
            \_last = null;
        }

        public void Add(int i)
        {
            var newItem = new ListItem();
            newItem.Next = null;
            newItem.Item = i;

            if (\_first == null)
            {
                \_first = newItem;
                \_last = newItem;
            }
            else
            {
                \_last.Next = newItem;
                \_last = newItem;
            }
        }
    }

And here is basically the same in unmanaged code:

    unsafe class ULinkedList : IDisposable
    {
        private ListItem\* \_first;
        private ListItem\* \_last;

        private struct ListItem
        {
            internal ListItem\* Next;
            internal int Item;
        }

        public ULinkedList()
        {
            \_first = null;
            \_last = null;
        }

        public void Add(int i)
        {
            var newItem = (ListItem\*) Marshal.AllocHGlobal(sizeof(ListItem)).ToPointer();
            newItem->Next = null;
            newItem->Item = i;

            if (\_first == null)
            {
                \_first = newItem;
                \_last = newItem;
            }
            else
            {
                \_last->Next = newItem;
                \_last = newItem;
            }
        }

        private void ReleaseUnmanagedResources()
        {
            var item = \_first;
            while (item != null)
            {
                var itemToFree = item;
                item = item->Next;
                Marshal.FreeHGlobal(new IntPtr(itemToFree));
            }
        }

        public void Dispose()
        {
            ReleaseUnmanagedResources();
            GC.SuppressFinalize(this);
        }

        ~FLinkedList()
        {
            ReleaseUnmanagedResources();
        }
    }

They are pretty close to each other, right? Well let's take a look at the Benchmarks.

    public class Program
    {
        public static void Main() => BenchmarkRunner.Run<Program>();

        \[IterationSetup\]
        public void IterationSetup()
        {
            \_unmanagedList = new FLinkedList();
            \_managedList = new MLinkedList();
        }

        \[IterationCleanup\]
        public void IterationCleanup() => \_unmanagedList.Dispose();

        private ULinkedList \_unmanagedList;
        private MLinkedList \_managedList;

        \[Benchmark\]
        public void Unmanaged()
        {
            for (int i = 0; i < 1000; i++)
            {
                \_unmanagedList.Add(i);
            }
        }

        \[Benchmark(Baseline = true)\]
        public void Managed()
        {
            for (int i = 0; i < 1000; i++)
            {
                \_managedList.Add(i);
            }
        }
    }

And here are the results:

|    Method |      Mean |     Error |    StdDev | Ratio | RatioSD |
|---------- |----------:|----------:|----------:|------:|--------:|
| Unmanaged | 48.050 us | 0.9426 us | 1.5749 us |  7.76 |    1.24 |
|   Managed |  6.714 us | 0.3142 us | 0.9014 us |  1.00 |    0.00 |

As you can tell, the managed version seems to be more than 7 faster than the unmanaged version! And I already factored out freeing the memory after each run. What about you, is this the result you expected? For me, I certainly expected it to be much closer with the unmanaged version beating the managed version.

But this is, of course, only half of the truth. Lets examine this a bit further in the [next blog post](https://wholesomeprogrammer.eu/2020/09/10/native-allocations-are-fast-part-2/)! Why does simple allocations seem to be a lot faster in managed code? We will see just how great dotnet memory allocation actually is and get a deeper insight which strategies there are to allocate memory in an efficient manner.
