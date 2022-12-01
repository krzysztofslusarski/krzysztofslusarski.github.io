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

All the examples that you are going to see here are synthetic reproduction of real world 
problems that I solved during my carrier.

## Profiled application

For a purpose of that post I've created spring boot application, so you can run following examples
by your own. It's available on
[my GitHub](https://github.com/krzysztofslusarski/async-profiler-demos){:target="_blank"}.
To build the application do the following:

```shell
git clone https://github.com/krzysztofslusarski/async-profiler-demos
cd async-profiler-demos
mvn clean package
```

To run the application you need two terminals where you run (you need 8081 and 8082 ports available):

```shell
java \
-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints \
-Xms1G -Xmx1G \
-jar first-application/target/first-application-0.0.1-SNAPSHOT.jar 

java \
-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints \
-Xms1G -Xmx1G \
-jar second-application/target/second-application-0.0.1-SNAPSHOT.jar
```

I'm using Corretto 17.0.2:

```shell
$ java -version

openjdk version "17.0.2" 2022-01-18 LTS
OpenJDK Runtime Environment Corretto-17.0.2.8.1 (build 17.0.2+8-LTS)
OpenJDK 64-Bit Server VM Corretto-17.0.2.8.1 (build 17.0.2+8-LTS, mixed mode, sharing)
```

And to create simple load tests I'm using very old tool:

```shell
$ ab -V

This is ApacheBench, Version 2.3 <$Revision: 1879490 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/
```

## How to run an Async-profiler

### Command line

One of the easiest way of running Async-profiler is using command line. In directory with the
profiler you just need to execute

```shell
./profiler.sh -e <event type> -d <duration in seconds> \
-f <output file name> <pid or application name>

# Examples
./profiler.sh -e cpu -d 10 -f prof.jfr first-application-0.0.1-SNAPSHOT.jar
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

## Output formats

Async-profiler gives you a choice how the results should be saved:
- default - printing results to terminal
- JFR
- Collapsed stack
- Flame graphs
- ... 

From that list 95% of a time I'm choosing JFR. It's binary format that contain all the information
gathered by the profiler. That file can be postprocess later by some external tool. I'm using my
own open-sourced [JVM profiling toolkit](https://github.com/krzysztofslusarski/jvm-profiling-toolkit){:target="_blank"}, 
which can read JFR files with additional filters and gives me a possibility to add/remove additional
levels during conversion to flame graph.

## Basic resources profiling

Before you start any profiler first thing you need to know is what is your goal. Only after that
you can choose proper mode of Async-profiler. Let's start with basics.

### Wall-clock

If your goal is to optimize time then you should run Async-profiler in wall-clock mode. This is a
most common mistake made by engineers that are starting their journey with profilers. Majority of
applications that I profiled so far were applications that were working in distributed
architecture, using some DBs, MQ, Kafka, ... In such applications majority of time is spent on
IO - waiting for other service/DB/... to respond. During such action Java is not consuming the CPU. 

```shell
# warmup
ab -n 100 -c 4 http://localhost:8081/examples/wall/first
ab -n 100 -c 4 http://localhost:8081/examples/wall/second

# profiling of first request
./profiler.sh start -e cpu -f first-cpu.jfr FirstApplication
ab -n 100 -c 4 http://localhost:8081/examples/wall/first
./profiler.sh stop -f first-cpu.jfr FirstApplication

./profiler.sh start -e wall -f first-wall.jfr FirstApplication
ab -n 100 -c 4 http://localhost:8081/examples/wall/first
./profiler.sh stop -f first-wall.jfr FirstApplication

# profiling of second request
./profiler.sh start -e cpu -f second-cpu.jfr FirstApplication
ab -n 100 -c 4 http://localhost:8081/examples/wall/second
./profiler.sh stop -f second-cpu.jfr FirstApplication

./profiler.sh start -e wall -f second-wall.jfr FirstApplication
ab -n 100 -c 4 http://localhost:8081/examples/wall/second
./profiler.sh stop -f second-wall.jfr FirstApplication
```

In the ```ab``` output we can see that the basic stats are similar for each request:

```shell
# first request
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       0
Processing:   521  654 217.9    528    1055
Waiting:      521  654 217.9    528    1054
Total:        521  654 217.9    528    1055

# second request
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       1
Processing:   522  665 190.2    548    1028
Waiting:      522  665 190.1    548    1028
Total:        522  665 190.2    549    1029
```

To give you a taste how CPU profile can mislead you here are flame graphs for those two
executions in CPU mode.

**First execution:** ([HTML](/assets/async-demos/wall-cpu-first.html){:target="_blank"})
![alt text](/assets/async-demos/wall-cpu-first.png "flames")

**Second execution:** ([HTML](/assets/async-demos/wall-cpu-second.html){:target="_blank"})
![alt text](/assets/async-demos/wall-cpu-second.png "flames")

I agree that they are not identical, but they have one thing in common. They show
that the most CPU consuming method invoked by my controller is ```WallService.calculateAndExecuteSlow()```.
It is not a lie, it consumes CPU. But does it consume most of the request time? 
Look at the flame graphs in wall-clock mode:

**First execution:** ([HTML](/assets/async-demos/wall-wall-first.html){:target="_blank"})
![alt text](/assets/async-demos/wall-wall-first.png "flames")

**Second execution:** ([HTML](/assets/async-demos/wall-wall-second.html){:target="_blank"})
![alt text](/assets/async-demos/wall-wall-second.png "flames")

I highlighted the ```WallService.calculateAndExecuteSlow()``` method. Wall-clock 
mode shows us that this method is responsible just for **~4%** of execution time.

**Thing to remember**: if your goal is to optimize time, and you use external
systems (including DBs, queues, topics, microservices) or locks, sleeps, disk IO, 
you should start with wall-clock mode.

In wall-clock mode we can also see that these flame graphs differ. The first 
execution spends most of its time in ```SocketInputStream.read()```:

![alt text](/assets/async-demos/wall-wall-first-2.png "flames")

Over **95%** of the time is eaten there. But the second execution:

![alt text](/assets/async-demos/wall-wall-second-2.png "flames")

spends just **75%** on the socket. To the right of the method
```SocketInputStream.read()``` you can spot addition bar. Let's zoom it:

![alt text](/assets/async-demos/wall-wall-second-3.png "flames")

It's ```InternalExecRuntime.acquireEndpoint()``` method which executest 
```PoolingHttpClientConnectionManager$1.get()``` from Apache Http Client, which 
in the end executes ```Object.wait()```. What is it? Basically what we are trying
to do in those two executions is to invoke remote REST service. First execution
contains HTTP client instance with ```20``` available connections, so no thread
needs to wait for connection from the pool:

```java
// FirstApplicationConfiguration
@Bean("pool20RestTemplate")
RestTemplate pool20RestTemplate() {
    return createRestTemplate(20);
}
```

```java
// WallService
void calculateAndExecuteSlow() {
    Random random = ThreadLocalRandom.current();
    CpuConsumer.mathConsumer(random.nextDouble(), CPU_MATH_ITERATIONS);

    invokeWithLogTime(() ->
        pool20RestTemplate.getForObject(SECOND_APPLICATION_URL + 
            "/examples/wall/slow", String.class)
    );
} 
```

The time spent on a socket is just waiting fot the REST endpoint to respond.
Second execution uses different instance of ```RestTemplate``` that has just 
**3** connections in the pool. Since the load is generated from **4** threads
by the ```ab``` then one thread needs to wait for a connection from pool.


### CPU

### Allocation

### Allocation - live objects

## Notes to remove
832
xdotool search localhost windowraise windowmove 50 50 windowsize 876 800