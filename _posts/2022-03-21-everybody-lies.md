---
layout: default
title:  "[Java][Profiling][JVM] Everybody lies, profilers too"
date:   2022-03-21 02:51:30 +0100
---

# [Java][Profiling][JVM] Everybody lies, profilers too

There is an excellent presentation on YouTube called [Profilers are lying hobbitses](https://www.youtube.com/watch?v=7IkHIqPeFjY){:target="_blank"}
by **Nitsan Wakart**. I strongly recommend you to watch it. Here is another examples of lie from profilers.

## Profiled application

Simple class:

```java
import java.util.ArrayList;
import java.util.List;

/** Andrei Pangin is an author of that reproduction. Details later. */
public class ArrayListGrow {
    static final int SIZE = 2048;
    static volatile List<Object> tmp;

    static void runTest() {
        List<Object> list = new ArrayList<>(SIZE);
        fill(list);
        tmp = list;
    }

    static void fill(List<Object> list) {
        for (int i = 0; i < SIZE; i++) {
            list.add(i);
        }
    }

    static void spoil() {
        for (int i = 0; i < 1000000; i++) {
            new ArrayList<>(0).add("");
        }
    }

    public static void main(String[] args) {
        spoil();

        while (true) {
            runTest();
        }
    }
}
```

Let's run that code using Amazon Corretto 1.8.0_312, and run the **Async-profiler** in CPU mode.

```shell
./profiler.sh -d 10 ArrayListGrow
```

The beginning of the **output** is:

```
Profiling for 10 seconds
Done
--- Execution profile ---
Total samples       : 1020

--- 8199255026 ns (80.37%), 820 samples
  [ 0] java.util.Arrays.copyOf
  [ 1] java.util.ArrayList.grow
  [ 2] java.util.ArrayList.ensureExplicitCapacity
  [ 3] java.util.ArrayList.ensureCapacityInternal
  [ 4] java.util.ArrayList.add
  [ 5] ArrayListGrow.fill
  [ 6] ArrayListGrow.runTest
  [ 7] ArrayListGrow.main
```

That means that our ```ArrayList``` executes method ```grow()``` while adding an element. That ```add``` is invoked
from ```ArrayListGrow.fill```. But **how is it possible**? That list is created with ```new ArrayList<>(SIZE)``` and we are adding
exactly ```SIZE``` elements. The ```ArrayList``` shouldn't grow in that case. Let's run it in the debugger. Lest create
a breakpoint at ```runTest()``` line to skip the ```spoil()``` part first:

![alt text](/assets/everybody-lies/1.png "1")

Now let's remove that one and create one in ```ArrayList.grow()```:

![alt text](/assets/everybody-lies/2.png "2")

Let's hit continue, so debugger could stop in the new breakpoint... and it **doesn't want to stop**. So the profiler shows us a huge 
usage of ```ArrayList.grow()```, and the debugger shows us no usage of that method. Who should we trust?

This time the debugger has right, but we cannot blame any profiler for that lie since this is a ...

## JVM bug

That bug was originally found by me, the original discussion over that topic is
[here](https://github.com/jvm-profiling-tools/async-profiler/discussions/541){:target="_blank"}. My reconstruction was
much more complicated than Andrei's, that's why I prefer to show his version. The bug is registered with number
[JDK-8281677](https://bugs.openjdk.java.net/browse/JDK-8281677){:target="_blank"}.

This bug hurts all profilers that use ```AsyncGetCallTrace``` or ```PerfMapAgent```, so basically every modern profilers
(Async-profiler, JProfiler in async mode, Perf + PMA, eBpf profiler + PMA, ...).

This bug **is not just an** ``ArrayList``` issue. I encountered multiple parts of my code where profiler lied to me because of that.