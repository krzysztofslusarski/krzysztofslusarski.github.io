---
layout: default
title:  "[Java][JVM][Tuning][Profiling][G1] How can an inefficient algorithm degrade performance with innocent G1GC in the middle? _Humongous object_ strikes back"
date:   2021-01-14 09:51:30 +0100
---

# [Java][JVM][Tuning][Profiling][G1] How can an inefficient algorithm degrade performance with innocent G1GC in the middle? _Humongous object_ strikes back
## Profiled application

* **JDK 11u6** from Amazon, deployed on Linux
* The application started consuming **huge amount of CPU** without any reason
* There were **no changes** in that application when problem started
* **Number of requests** processed by the application **didn't change**

## What are _humongous objects_?

If you don't know what are the _humongous objects_ check my [previous article](https://krzysztofslusarski.github.io/2020/11/10/humongous.html){:target="_blank"}.

## Where to start?

In such a situation I like to start with checking how my JVM works at the operation system level. We have plenty of software that can be helpful:
* ```mpstat```
* ```top```
* ```htop```
* ```pidstat```
* ...

Let's focus on the last one. After running ```pidstat -t 1``` I can see which **Java thread consumes CPU** in 1 second interval. Because this is 
JVM in **version 11.x** the thread name set in Java is visible from OS level. If you want such a feature in **version 8.x** you have to compile it by your own, 
here is [the tutorial](https://www.infobip.com/blog/fixing-the-tool). Output of the mentioned command (I removed irrelevant part):

```
[root@prod******* ~]# pidstat -t 1
21:35:02      UID      TGID       TID    %usr %system  %guest   %wait    %CPU   CPU  Command
21:35:03     1000     10089         -    0,00    3,00    0,00    0,00    3,00     2  pidstat
21:35:03     1000         -     10089    0,00    3,00    0,00    0,00    3,00     2  |__pidstat
21:35:03     1000     10122         -  220,00    1,00    0,00    0,00  221,00    13  java
...
21:35:03     1000         -     10129    0,00    1,00    0,00    0,00    1,00     4  |__GC Thread#0
21:35:03     1000         -     10131   91,00    0,00    0,00    0,00   91,00     3  |__G1 Conc#0
21:35:03     1000         -     10135   25,00    0,00    0,00    0,00   25,00    20  |__G1 Refine#0
...
```

A little output explanation:
* **%CPU** - how much one thread consumed CPU in the last second
* **Command** - from our point of view this is a Java thread name  

What I could see is that ```G1 Conc#0``` thread was consuming **91%** of the CPU (it means **91% of one core**, not the whole CPU). This thread is responsible for 
marking live objects in _concurrent mark_ phase of the G1 algorithm. This phase should be run when the G1 decides that the _old generation_ should be cleaned soon.
So why does G1 believe that it needs to clean it all the time? Let's look at the **GC logs**:

## GC logs

This time the issue is so simple, that we don't need any charts. Let's just look at the logs at info level from ```gc,start``` logger:

```
[info ][gc,start             ] GC(1491517) Pause Young (Concurrent Start) (G1 Humongous Allocation)
[info ][gc,start             ] GC(1491518) Pause Remark
[info ][gc,start             ] GC(1491518) Pause Cleanup
[info ][gc,start             ] GC(1491519) Pause Young (Concurrent Start) (G1 Humongous Allocation)
[info ][gc,start             ] GC(1491520) Pause Remark
[info ][gc,start             ] GC(1491520) Pause Cleanup
[info ][gc,start             ] GC(1491521) Pause Young (Concurrent Start) (G1 Humongous Allocation)
[info ][gc,start             ] GC(1491522) Pause Remark
[info ][gc,start             ] GC(1491522) Pause Cleanup
[info ][gc,start             ] GC(1491523) Pause Young (Concurrent Start) (G1 Humongous Allocation)
[info ][gc,start             ] GC(1491524) Pause Remark
[info ][gc,start             ] GC(1491524) Pause Cleanup
...
``` 

The reason of starting _concurrent mark_ phase is **"G1 Humongous Allocation"**. Yes, again. If you want to know how to find Java code that is responsible
for _humongous allocation_, check my [previous article](https://krzysztofslusarski.github.io/2020/11/10/humongous.html){:target="_blank"}. In this specific case
I used **async-profiler 1.8.3** with options:

```shell
./profiler.sh -d 10 -f /tmp/humongous.svg -e G1CollectedHeap::humongous_obj_allocate Application
```

## Reason? Inefficient algorithm 

It occurred that the _humongous objects_ are created because of the inefficient implementation of the method:

```java
    static <T> boolean containsAny(Set<T> setToCheck, Collection<T> valuesToFind) {
        // source code
    }
```  

This method should check if any value of the ```valuesToFind``` collection is present in ```setToCheck```. Here is the inefficient implementation:

```java
    static <T> boolean containsAny(Set<T> setToCheck, Collection<T> valuesToFind) {
        return intersect(setToCheck, valuesToFind).size() > 0;
    }
```

It doesn't look scary at all, but let's look at the ```intersect``` method:

```java
    static <T> Collection<T> intersect(Collection<T> first, Collection<T> second) {
        Set<T> copy = new HashSet<>(first);
        copy.retainAll(second);
        return copy;
    }
```

The problem was that ```Collection<T> first``` was a _humongous object_ and ```itersect``` method simply cloned it, so it creates another _humongous object_. 

## Solution

**Implement the ```containsAny``` method the right way** or just use a library that already has done it.  

## Why did the problem start without any changes in the application? 

The value passed to the ```containsAny``` method as the ```setToCheck``` was the **cache key set**. The cache simply grew in the runtime and that was the
reason why without any changes new _humongous allocation_ occurred. 

## Simplified application

You can play with such a scenario with this simplified application:

```java
/**
 * Run with -Xms2G -Xmx2G -Xlog:gc -XX:ConcGCThreads=1
 */
class NewHumongous {
    private static final Set<String> CACHE = new HashSet<>();
    private static final Set<String> CACHE_2 = new HashSet<>();

    static {
        for (int i = 0; i < 50_000; i++) {
            CACHE.add(NewHumongous.class.getName() + NewHumongous.class.getName() + i);
        }

        for (int i = 0; i < 5_000_000; i++) {
            CACHE_2.add(i + "");
        }
    }

    static <T> Collection<T> intersect(Collection<T> first, Collection<T> second) {
        Set<T> copy = new HashSet<>(first);
        copy.retainAll(second);
        return copy;
    }

    static <T> boolean containsAny(Set<T> setToCheck, Collection<T> valuesToFind) {
        return intersect(setToCheck, valuesToFind).size() > 0;
    }

    public static void main(String[] args) {
        while (true) {
            containsAny(CACHE, new ArrayList<>());
        }
    }
}

```

To fix the problem you have to change ```containsAny``` method to the implementation that doesn't clone the original set:

```java
    static <T> boolean containsAny(Set<T> setToCheck, Collection<T> valuesToFind) {
        for (T t : valuesToFind) {
            if (setToCheck.contains(t)) {
                return true;
            }
        }
        return false;
    }
```

## Simplified application - CPU usage with different cache size

If you run a simplified application with arguments mentioned in the javadoc you get a following **CPU** consumption:
```
pasq@pasq-MS-7C37:~$ pidstat -p `pgrep -f NewHumongous` 1
Linux 5.4.0-60-generic (pasq-MS-7C37) 	15.01.2021 	_x86_64_	(24 CPU)

08:24:25      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
08:24:26     1000     12352  219,00    0,00    0,00    0,00  219,00     4  java
08:24:27     1000     12352  211,00    1,00    0,00    0,00  212,00     4  java
08:24:28     1000     12352  224,00    1,00    0,00    0,00  225,00     4  java
08:24:29     1000     12352  214,00    1,00    0,00    0,00  215,00     4  java
08:24:30     1000     12352  212,00    4,00    0,00    0,00  216,00     4  java
08:24:31     1000     12352  213,00    2,00    0,00    0,00  215,00     4  java
```

Funny part starts when you change ```CACHE``` size from ```50_000``` to ```45_000```. Here is a **CPU** consumption for such a case:

```
pasq@pasq-MS-7C37:~$ pidstat -p `pgrep -f NewHumongous` 1
Linux 5.4.0-60-generic (pasq-MS-7C37) 	15.01.2021 	_x86_64_	(24 CPU)

08:26:00      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
08:26:01     1000     12539  102,00    0,00    0,00    0,00  102,00     9  java
08:26:02     1000     12539  102,00    0,00    0,00    0,00  102,00     9  java
08:26:03     1000     12539  102,00    0,00    0,00    0,00  102,00     9  java
08:26:04     1000     12539  103,00    0,00    0,00    0,00  103,00     9  java
08:26:05     1000     12539  101,00    0,00    0,00    0,00  101,00     9  java
08:26:06     1000     12539  102,00    0,00    0,00    0,00  102,00     9  java
```  
 
Decreasing ```CACHE``` size made it not _humongous_ and **CPU** consumption dropped from **215%** to **100%**. 