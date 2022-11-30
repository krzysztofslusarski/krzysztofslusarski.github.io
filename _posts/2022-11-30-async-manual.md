---
layout: default
title:  "[Java][Profiling] Async-profiler manual by use cases"
date:   2022-11-30 02:51:30 +0100
---

# [Java][Profiling] Async-profiler manual by use cases

This blog post contains examples of Async-profiler usages that I have found useful in my job.
Some content of this post is copy-pasted from previous entries, I just wanted to avoid unnecessary
jumps between articles. 

The goal of that post is to give examples. It's not a replacement of project README. If you
haven't read it, simply do it.

## How to run an Async-profiler

### Command line

One of the easiest way of running Async-profiler is using command line. In directory with the
profiler you just need to execute

```shell
./profiler.sh -e <event type> -d <duration in seconds> -f <output file name> <pid or application name>

# Examples
./profiler.sh -e cpu -d 10 -f prof.jfr MyApplication
./profiler.sh -e wall -d 10 -f prof.html 1234 # where 1234 is a pid of Java process
```

There are a lot of additional switches that are explained in README. 

You can also do in such a way:

```shell
./profiler.sh start -e <event type> -f <output JFR file name> <pid or application name>
# do something
./profiler.sh stop -f <output JFR file name> <pid or application name>
```

For formats different from JFR you need to pass the file name during ```stop```, but for JFR 
it is needed during ```start```.

**WARNING**: this way of attaching any profiler is vulnerable to
[JDK-8212155](https://bugs.openjdk.org/browse/JDK-8212155){:target="_blank"}. That issue can crash 
your JVM during attach, it was fixed in JDK 17.

If you are attaching profiler this way it is recommended to use ```-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints```
JVM flags.

### During JVM startup

You can add a parameter when you are starting ```java``` process:

```shell
java -agentpath:/path/to/libasyncProfiler.so=start,event=cpu,file=prof.jfr
```

The parameters passed this way differs from the switches used in command line approach.
Mapping between those you can find in
[profiler.sh](https://github.com/jvm-profiling-tools/async-profiler/blob/master/profiler.sh#L149){:target="_blank"}
source code.

### From Java API

The Java API is published to maven central. All you need to do is to include a dependency:

```xml
<dependency>
  <groupId>tools.profiler</groupId>
  <artifactId>async-profiler</artifactId>
  <version>2.9</version>
</dependency>
```

That gives you an API where you can use Async-profiler from Java code. Example of usage:

```java
// If the profiler is loaded with agentpath or the libasyncProfiler.so is 
// placed in one of directory: /usr/java/packages/lib /usr/lib64 /lib64
// /lib /usr/lib
AsyncProfiler profiler = AsyncProfiler.getInstance(); 

// If the profiler is placed elsewhere:
AsyncProfiler profiler = AsyncProfiler.getInstance("/path/to/libasyncProfiler.so");
```

That gives you an instance of ```AsyncProfiler``` object, where you can send orders to the profiler:

```java
profiler.execute(String.format("start,jfr,event=wall,file=%s.jfr", fileName));
// do something, like sleep
profiler.execute(String.format("stop,file=%s.jfr", fileName));
```

## Basic resources profiling

Before you start any profiler first thing you need to know is what is your goal. Only after that
you can choose proper mode of Async-profiler. Let's start with basics.

### Wall-clock

If your goal is to optimize time then you should run Async-profiler in wall-clock mode. This is a
most common mistake made by engineers that are starting they journey with profilers. Majority of
applications that I profiled so far were applications that were working in microservice
architecture, using some DBs, MQ, Kafka, ... In such applications majority of time is spent on
IO - waiting for other service/DB/... to respond. During such action Java is not consuming the CPU. 

### CPU

### Allocation

### Allocation - live objects