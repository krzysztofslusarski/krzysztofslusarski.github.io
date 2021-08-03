---
layout: default
title:  "[Java][JVM Logs][GC Logs] Monday with JVM logs - allocation rate"
date:   2021-08-02 09:51:30 +0100
---

# [Java][JVM Logs][GC Logs] Monday with JVM logs - allocation rate

## GC logs

At the end of each _GC cycle_ you can find such an entry in GC logs at _info_ level:

```
[494592.129s] GC(9073) Pause Young (Normal) (G1 Evacuation Pause) 4195M->1188M(5120M) 29.701ms
[494613.918s] GC(9074) Pause Young (Normal) (G1 Evacuation Pause) 4244M->1184M(5120M) 29.930ms
```

You can find **three** sizes in such an entry **A->B(C)** that are:
* **A** - used size of a heap before _GC cycle_
* **B** - used size of a heap after _GC cycle_
* **C** - current size of a whole heap

The first value in the example entries is the **uptime** of the JVM, let's call it **T**. Now let's take **two** entries:

```
[T1] ... A1->B1(C1) ...
[T2] ... A2->B2(C2) ...
```

The value **(A2 - B1) / (T2 - T1)** is called the **allocation rate**. This value means how many **MB per second** our
application allocates the memory on the heap. If you want you can also   subtract also the pause time, but it usually makes
no difference.

In the example above the **allocation rate** is:
```
(4244 MB - 1188 MB) / (494613.918 s - 494592.129 s) =
3056 MB / 21,789 s =
140,25 MB/s
``` 

We can put that value on the chart (those are **1 day** charts);

![alt text](/assets/monday-4/raw-1.jpg "1")

![alt text](/assets/monday-4/raw-2.jpg "2")

![alt text](/assets/monday-4/raw-3.jpg "3")

Sometimes such a chart is not clear, because spikes of allocation may occur. We can create the **average allocation rate** 
in some time period. Here are the examples:

![alt text](/assets/monday-4/avg-1.jpg "1")

![alt text](/assets/monday-4/avg-2.jpg "2")

![alt text](/assets/monday-4/avg-3.jpg "3")

Let's focus on the last one. That application allocates:

* **10 MB/s** in the morning
* **130-160 MB/s** in the middle of the day
* **200 MB/s** for very short time

## Why is that value useful?

Knowing that value, and the size of _young generation_ you can estimate how often your _garbage collector_ will run. Knowing that and how long your GC STW pauses last
you can calculate responsiveness/SLA of your application.

## How can you decrease the allocation rate?

First thing you need to understand is that _garbage collector_ **has nothing to do** with the allocation rate. 
Your application is responsible for the allocation, not the _GC_. If you want to decrease the allocation rate, you need
to tune your application. To do that, first you need to find what part of your code is responsible for
the majority of the allocation. In other words you need to find **hotspots** of the allocation. At the production
environment you can use the **allocation profiler** based on the **JFR events**:

* Java flight recorder / JDK mission control
* Async-profiler using the _alloc_ event

These profilers can show you which part of the code is responsible for your allocation. The rest is on your hands, 
you need to change your application by yourself to decrease the allocation rate.
