---
layout: default
title:  "[Java][JVM Logs][GC Logs][G1GC] Monday with JVM logs - G1GC Stop-the-world phases"
date:   2021-08-10 06:51:30 +0100
---

# [Java][JVM Logs][GC Logs][G1GC] Monday with JVM logs - G1GC Stop-the-world phases

## GC logs

At the beginning of each _GC cycle_ you can find the name of the collection at _info_ level:

```
Pause Cleanup
Pause Full (G1 Evacuation Pause)
Pause Full (G1 Humongous Allocation)
Pause Full (Heap Dump Initiated GC)
Pause Full (System.gc())
Pause Remark
Pause Young (Concurrent Start) (G1 Evacuation Pause)
Pause Young (Concurrent Start) (G1 Humongous Allocation)
Pause Young (Concurrent Start) (GCLocker Initiated GC)
Pause Young (Concurrent Start) (Metadata GC Threshold)
Pause Young (Mixed) (G1 Evacuation Pause)
Pause Young (Mixed) (G1 Humongous Allocation)
Pause Young (Mixed) (GCLocker Initiated GC)
Pause Young (Normal) (G1 Evacuation Pause)
Pause Young (Normal) (G1 Humongous Allocation)
Pause Young (Normal) (GCLocker Initiated GC)
Pause Young (Prepare Mixed) (G1 Evacuation Pause)
Pause Young (Prepare Mixed) (G1 Humongous Allocation)
Pause Young (Prepare Mixed) (GCLocker Initiated GC)
```

Such a collection names I have found in logs I have. Let's try to understand what they mean. First division is the
collection name. We have:

* ```Pause Full``` - that one we don't want to see, long clear of the whole heap
* ```Pause Cleanup```, ```Pause Remark``` - those are phases done at the end of the _concurrent cycle_
  * ```Pause Remark``` - this phase performs global processing of references, reclaiming empty regions, class unloading, 
    and some other **G1GC** internal cleanup
  * ```Pause Cleanup``` - this phase decides is any _mixed collection_ needed  
* ```Pause Young```
  * ```Normal``` - classic cleanup of the _young generation_
  * ```Prepare Mixed``` - cleanup of the _young generation_ with preparation for cleaning the _old generation_ 
  * ```Mixed``` - cleanup of the _young generation_ and the _old generation_
  * ```Concurrent Start``` - cleanup of the _young generation_ with preparation for _concurrent mark_ phase

In the last brackets you can see **the reason** why the **G1GC** starts the collection:

* ```System.gc()``` - pause triggered by calling ```Runtime.gc()``` method (yes ```Runtime```, it is not a mistake)
* ```Heap Dump Initiated GC``` - pause triggered by request for a _heap dump_
* ```G1 Evacuation Pause``` - classic reason for the GC, request for creating new object failed because there was not enough space in the _eden_.
* ```Metadata GC Threshold``` - it was a time to clean the _metaspace_, GC is also responsible for cleaning that area
* ```G1 Humongous Allocation``` - request for creating new _humongous object_ failed
* ```GCLocker Initiated GC``` - **G1GC** cannot start immediately when any thread is in the _JNI critical section_, GC has
to wait for threads to exit such a section - such a situation is marked as ```GCLocker Initiated GC```
  
## What do we want to see in the log?

What we want to see in such a log is as many ```G1 Evacuation Pause``` caused collections as possible. What we 
don't want to see:
* multiple ```Pause Full``` collections
* multiple collection caused by other reason then ```G1 Evacuation Pause```

## Charts

As usual, you can present your GC performance on a chart. We can generate charts with count of each collection type,
time of each collection type, count of the reason, ...

Here are count charts of the JVM that had a problem with _humongous allocation_:

![alt text](/assets/monday-6/count-1.jpg "1")

![alt text](/assets/monday-6/reason-1.jpg "1")

**Most** of the collections started with ```G1 Humongous Allocation``` reasons. 

Example of a pretty healthy application:

![alt text](/assets/monday-6/count-2.jpg "2")

![alt text](/assets/monday-6/reason-2.jpg "2")

There are **some** collections started with ```G1 Humongous Allocation``` reasons, but there are not a lot of them. 

Another example of a healthy application:

![alt text](/assets/monday-6/count-3.jpg "3")

![alt text](/assets/monday-6/reason-3.jpg "3")
