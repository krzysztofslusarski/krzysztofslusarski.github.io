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
problems that I solved during my carrier. Even if some example looks like "it's too stupid
to happen anywhere", well, it isn't. 

- [Profiled application](#profiled-application)
- [How to run an Async-profiler](#how-to)
  - [Command line](#how-to-cl)
  - [During JVM startup](#how-to-jvm)
  - [From Java API](#how-to-java)
  - [From JMH benchmark](#how-to-jmh)
- [Output formats](#out)
- [Flame graphs](#flames)
- [Basic resources profiling](#basic-resources)
  - [Wall-clock](#wall)
  - [CPU - easy-peasy](#cpu-easy)
  - [CPU - a bit harder](#cpu-hard)
  - [Allocation](#alloc)
  - [Allocation - humongous objects](#alloc-ha)
  - [Allocation - live objects](#alloc-live)
  - [Locks](#locks)
- [Methods profiling](#methods)
  - [Exceptions](#methods-ex)
  - [G1GC humongous allocation](#methods-g1ha)
  - [Thread start](#methods-thread)
  - [Classloading](#methods-classes)
- [Perf events](#perf)
- [Filtering single request](#single-req)
- [Continuous profiling](#continuous)
  - [Command line](#continuous-cli})
  - [Java](#continuous-java)
  - [Spring Boot](#continuous-spring)
- [Contextual profiling](#context-id)
  - [Contextual profiling in Spring Boot microservices](#context-id-spring)

## Profiled application 
{: #profiled-application }

For a purpose of that post I've created spring boot application, so you can run following examples
by your own. It's available on
[my GitHub](https://github.com/krzysztofslusarski/async-profiler-demos){:target="_blank"}.
To build the application do the following:

```shell
git clone https://github.com/krzysztofslusarski/async-profiler-demos
cd async-profiler-demos
mvn clean package
```

To run the application you need three terminals where you run (you need 8081, 8082 and 8083 ports available):

```shell
java -Xms1G -Xmx1G \
-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints \
-Duser.language=en-US \
-Xlog:safepoint,gc+humongous=trace \
-jar first-application/target/first-application-0.0.1-SNAPSHOT.jar 

java -Xms1G -Xmx1G \
-jar second-application/target/second-application-0.0.1-SNAPSHOT.jar

java -Xms1G -Xmx1G \
-jar third-application/target/third-application-0.0.1-SNAPSHOT.jar
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

I will not go through the source code to explain you all the details before I jump into examples.
It's completely not natural. In my work I often profile applications which source code I haven't 
seen. Just consider it as a typical microservice build with Spring Boot and Hibernate.

In all the examples I will assume that the application is started with ```java -jar``` command.
If you are running the application from the IDE then the name of the application is switched 
from ```first-application-0.0.1-SNAPSHOT.jar``` to ```first-application-0.0.1-SNAPSHOT.jar```

## How to run an Async-profiler 
{: #how-to }

### Command line
{: #how-to-cl }

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
{: #how-to-jvm }

You can add a parameter when you are starting ```java``` process:

```shell
java -agentpath:/path/to/libasyncProfiler.so=start,event=cpu,file=prof.jfr
```

The parameters passed this way differs from the switches used in command line approach.
Mapping between those you can find in
[profiler.sh](https://github.com/jvm-profiling-tools/async-profiler/blob/master/profiler.sh#L149){:target="_blank"}
source code.

You can also attach not started profiler using ```-agentpath```, which is the safest way of starting your JVM
if you want to profile it anytime soon.

### From Java API
{: #how-to-java }

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

### From JMH benchmark
{: #how-to-jmh }

TODO

## Output formats
{: #out }

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

All the flame graphs that are attached to that post are generated by mentioned tool from the JFR. The JFR file format
for each sample contains:

- Stack trace
- Java thread name
- Thread state (is the Java thread consumes cpu)
- Timestamp
- Monitor class - for ```lock``` mode
- Waiting for lock duration - for ```lock``` mode
- Allocated object class - for ```alloc``` mode
- Allocated object size - for ```alloc``` mode
- Context ID - if you are using async-profiler with
  [Context ID PR](https://github.com/jvm-profiling-tools/async-profiler/pull/576){:target="_blank"} merged

That information is already there, we just need to extract what we need and present it in nice form.

## Flame graphs
{: #flames }

If you do **sampling profiling** you need to visualize the results. The results are nothing more than a **set of stack traces**.
My favorite way of visualization is a **flame graph**. The easiest way to understand what flame graphs are is to understand how
they are created.

First part is to draw a rectangle for each frame of each stack trace. The stack traces are drawn bottom-up and sorted
alphabetically. For example, such a graph:

![alt text](/assets/hz-sql/flame-1.png "flame")

corresponds to set of stactraces:

* 3 samples - ```a() -> h()```
* 5 samples - ```b() -> d() -> e() -> f()```
* 2 samples - ```b() -> d() -> e() -> g()```
* 2 samples - ```b() -> d()```
* 2 samples - ```c()```

The next step is **joining** the rectangles with the same method name to one bar:

![alt text](/assets/hz-sql/flame-2.png "flame")

The flame graph usually shows you how some resource is utilized by your application. The resource is utilized
**by the top methods** of that graph (visualized with green bar):

![alt text](/assets/hz-sql/flame-3.png "flame")

So in this example method ```b()``` is not utilizing the resource at all, it just invokes methods that do it. Flame graphs
are commonly used to present the **CPU utilization**, but CPU is just one of the resources that we can visualize this way.
If you use **wall-clock mode** then your resource is **time**. If you use **allocation mode** then your resource is
**heap**.

## Basic resources profiling
{: #basic-resources }

Before you start any profiler first thing you need to know is what is your goal. Only after that
you can choose proper mode of Async-profiler. Let's start with basics.

### Wall-clock
{: #wall }

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
./profiler.sh start -e cpu -f first-cpu.jfr first-application-0.0.1-SNAPSHOT.jar
ab -n 100 -c 4 http://localhost:8081/examples/wall/first
./profiler.sh stop -f first-cpu.jfr first-application-0.0.1-SNAPSHOT.jar

./profiler.sh start -e wall -f first-wall.jfr first-application-0.0.1-SNAPSHOT.jar
ab -n 100 -c 4 http://localhost:8081/examples/wall/first
./profiler.sh stop -f first-wall.jfr first-application-0.0.1-SNAPSHOT.jar

# profiling of second request
./profiler.sh start -e cpu -f second-cpu.jfr first-application-0.0.1-SNAPSHOT.jar
ab -n 100 -c 4 http://localhost:8081/examples/wall/second
./profiler.sh stop -f second-cpu.jfr first-application-0.0.1-SNAPSHOT.jar

./profiler.sh start -e wall -f second-wall.jfr first-application-0.0.1-SNAPSHOT.jar
ab -n 100 -c 4 http://localhost:8081/examples/wall/second
./profiler.sh stop -f second-wall.jfr first-application-0.0.1-SNAPSHOT.jar
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
```PoolingHttpClientConnectionManager$1.get()``` from Apache HTTP Client, which 
in the end executes ```Object.wait()```. What is it? Basically what we are trying
to do in those two executions is to invoke remote REST service. First execution
contains HTTP Client instance with ```20``` available connections, so no thread
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
You may think that this is a stupid human error, that someone created a pool with
too low number of connections. In the real world the problem is with defaults, that
are quite low. In our testing application default settings for the thread pool are:

- ```maxTotal``` - **25** connections totally in the pool
- ```defaultMaxPerRoute``` - **5** connections to the same address

That numbers vary between versions. I remember one application with HTTP Client 4.x
with defaults set to **2**.

THere are plenty of tools that log the invocation time of external services. The common
problem in those tools is that the time of waiting on pool for a connection is usually
included in the whole time ov invocation which is a lie. I saw in the past when caller 
had a line in logs that shew execution time of ```X ms```, caller had similar log
that presented ```1/10 * X ms```. What was those teams doing to understand that? They
tried to convinced network department that it's a network issue. Big waste of time.

I also saw plenty of custom logic that traced external execution time. In our application
you can see such pattern:

```java
void calculateAndExecuteFast() {
    // ...
    invokeWithLogTime(() ->
            pool3RestTemplate.getForObject(SECOND_APPLICATION_URL + "/examples/wall/fast", String.class)
    );
}

private <T> T invokeWithLogTime(Supplier<T> toInvoke) {
    StopWatch stopWatch = new StopWatch();

    stopWatch.start();
    T ret = toInvoke.get();
    stopWatch.stop();

    log.info("External WS invoked in: {}ms", stopWatch.getTotalTimeMillis());
    return ret;
}
```

That logs don't trace the time on external service, the time also includes all the magic done by
Spring, including waiting for a connection from the pool. You can easily see that for the second request
logs like:

```
External WS invoked in: 937ms
```

But the second service that is invoked is:

```java
    @GetMapping("/fast")
    String fast() throws InterruptedException {
        Thread.sleep(500);
        return "OK";
    }
```

### CPU - easy-peasy
{: #cpu-easy }

if you know that your application is CPU intensive, or you want to decrease CPU consumption, 
then the CPU mode is suitable. 

Let's prepare our application:

```shell
# Execute it once
curl -v http://localhost:8081/examples/cpu/prepare

# Little warmup
ab -n 5 -c 1 http://localhost:8081/examples/cpu/inverse

# Profiling time:
./profiler.sh start -e cpu -f cpu.jfr first-application-0.0.1-SNAPSHOT.jar
ab -n 5 -c 1 http://localhost:8081/examples/cpu/inverse
./profiler.sh stop -f cpu.jfr first-application-0.0.1-SNAPSHOT.jar
```

You can check during the benchmark what is the CPU utilization of our JVM:

```shell
$ pidstat -p `pgrep -f first-application-0.0.1-SNAPSHOT.jar` 5
 
09:42:49      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
09:42:54     1000     49813  115,40    0,40    0,00    0,00  115,80     1  java
09:42:59     1000     49813  111,40    0,60    0,00    0,00  112,00     1  java
09:43:04     1000     49813  106,80    0,20    0,00    0,00  107,00     1  java
09:43:09     1000     49813  113,00    0,20    0,00    0,00  113,20     1  java 
```

We are using a bit more than one CPU core. Our load generator executes the load with a single
thread so that CPU usage is pretty high. Let's see what our CPU is doing while execution our
spring controller: ([HTML](/assets/async-demos/cpu.html){:target="_blank"})

![alt text](/assets/async-demos/cpu.png "flames")

I know that flame graph is pretty long, but hey, welcome to Spring and Hibernate.
I highlighted the ```existsById()``` method. You can see that it eats **95%** of the
CPU. But why? From the code perspective it doesn't look scary at all:

```java
@Transactional
public void inverse(UUID uuid) {
    sampleEntityRepository.findById(uuid).ifPresent(sampleEntity -> {
        boolean allConfigPresent = true;
        for (int i = 0; i < 100; i++) {
            allConfigPresent = allConfigPresent && sampleConfigurationRepository.existsById("key-" + i);
        }
        sampleEntity.setFlag(!sampleEntity.isFlag());
    });
}
```

We are just executing ```existsById()``` in Spring Dara JPA repository. The answer 
why that method is slow is in the beginning of the method:

```java
sampleEntityRepository.findById(uuid)
```

and in JPA mapping:

```java
public class SampleEntity {
    @Id
    private UUID id;

    @Fetch(FetchMode.JOIN)
    @JoinColumn(name = "fk_entity")
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER, orphanRemoval = true)
    private Set<SampleSubEntity> subEntities;

    private boolean flag;
}
```

What that means is that when we are getting one ```SampleEntity``` by id, we are
also extracting the ```subEntities``` from the database, because of ```fetch = FetchType.EAGER```.
This is not a problem yet. All that JPA entities are loaded into Hibernate session.
That mechanism is pretty cool, because it gives you the _dirty checking_ functionality.
The downside however is that the _dirty_ entities needs to be flushed by Hibernate 
to DB. You have different flush strategies in Hibernate. The default one is ```AUTO```,
you can read about them in the
[Javadocs](https://javadoc.io/doc/org.hibernate/hibernate-core/5.6.14.Final/org/hibernate/FlushMode.html){:target="_blank"}.

What can be done about that? Well, first of all, it should be forbidden to develop
a large Hibernate application without reading the 
[Vlad Mihalcea's book](https://vladmihalcea.com/books/high-performance-java-persistence/){:target="_blank"}.
If you are developing such an application, buy that book, it's great. From my experience
some engineers has tendency to abuse Hibernate. Let's look at the code sample I pasted 
before. We are loading a huge ```SampleEntity```. What are we doing with it?

```java
@Transactional
public void inverse(UUID uuid) {
    sampleEntityRepository.findById(uuid).ifPresent(sampleEntity -> {
        // irrelevant
        sampleEntity.setFlag(!sampleEntity.isFlag());
    });
}
```

So we're basically changing one column in one row in DB. We can do it more efficiently 
with simple ```update``` query, even with Spring Data JPA repository or simple JDBC.
Do we really need to use Hibernate everywhere?

### CPU - a bit harder
{: #cpu-hard }

Sometimes the result of CPU profiler is just beginning of the fun. Let's consider following example:

```shell
# Little warmup
ab -n 10 -c 1 http://localhost:8081/examples/cpu/matrix-slow

# Profiling time
./profiler.sh start -e cpu -f matrix-slow.jfr FirstApplication
ab -n 10 -c 1 http://localhost:8081/examples/cpu/matrix-slow
./profiler.sh stop -f matrix-slow.jfr FirstApplication

# Little warmup
ab -n 10 -c 1 http://localhost:8081/examples/cpu/matrix-fast

# Profiling time
./profiler.sh start -e cpu -f matrix-fast.jfr FirstApplication
ab -n 10 -c 1 http://localhost:8081/examples/cpu/matrix-fast
./profiler.sh stop -f matrix-fast.jfr FirstApplication
```

Let's see the times of ```matrix-slow``` request:

```shell
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.0      0       0
Processing:  1706 1735  29.0   1735    1786
Waiting:     1706 1735  28.9   1735    1786
Total:       1706 1735  29.0   1735    1786
```


The profile looks like that: ([HTML](/assets/async-demos/cpu-hard-slow.html){:target="_blank"})

![alt text](/assets/async-demos/cpu-hard-slow.png "flames")

Whole CPU is wasted in method: 

```java
public static int[][] matrixMultiplySlow(int[][] a, int[][] b, int size) {
    int[][] result = new int[size][size];
    for (int i = 0; i < size; i++) {
        for (int j = 0; j < size; j++) {
            int sum = 0;
            for (int k = 0; k < size; k++) {
                sum += a[i][k] * b[k][j];
            }
            result[i][j] = sum;
        }
    }
    return result;
}
```

If we look at the times of ```matrix-fast``` request:

```shell
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.0      0       0
Processing:   861  888  20.7    890     924
Waiting:      861  887  20.7    890     924
Total:        861  888  20.7    890     924
```

That request is two times faster than the ```matrix-slow```, but if we look at the profile:
([HTML](/assets/async-demos/cpu-hard-fast.html){:target="_blank"})

![alt text](/assets/async-demos/cpu-hard-fast.png "flames")

Whole CPU is wasted in method:

```java
public static int[][] matrixMultiplyFaster(int[][] a, int[][] b, int size) {
    int[][] result = new int[size][size];
    for (int i = 0; i < size; i++) {
        for (int k = 0; k < size; k++) {
            int current = a[i][k];
            for (int j = 0; j < size; j++) {
                result[i][j] += current * b[k][j];
            }
        }
    }
    return result;
}
```

Both methods ```matrixMultiplySlow``` and ```matrixMultiplyFaster``` have the same complexity O(n^3). So
why one is faster than the other? Well, if you want to understand how exactly CPU intensive algorithm is working
you need to understand how CPU is working, which is far away from the topic of this post. Be aware that if
you want to optimize such algorithms you will probably need at least one of:

- Knowledge of CPU architecture
- Top-Down performance analysis methodology
- Looking at the ASM of generated methods  

Many Java programmers forget that all the execution is done in CPU. To talk to CPU Java needs to use ASM. That's
basically what JIT compiler is doing, it converts your hot methods and loops into effective ASM. At the assembly level
you can check if JIT used vectorized instruction for your loops for example. So yes, sometime you need to get dirty 
with such a low level stuff. For now async-profiler can give you a hint which methods you should focus on.

### Allocation
{: #alloc }

The common use cases where that resource should be tracked are:

- decreasing of GC runs frequency
- finding allocation outside the TLAB which are done in slow path
- fighting with single/tens of milliseconds latency, where even heap object creation matters

First let's understand how new objects on a heap are created, so we have a better
understanding of what the async-profiler shows to us.

A portion of our Java heap is called an **eden**. This is a place where new objects are
born. Let's assume for a simplicity that eden is a continuous part of memory. The very efficient
way of allocation in such a case is called **bumping pointer**. We keep a pointer to the first 
address of a free space:

![alt text](/assets/async-demos/alloc-1.png "alloc")

When we do ```new Object()``` we simply can calculate a space needed for it and to locate 
next free address we just need to bump the pointer by ```sizeof(Object)```:

![alt text](/assets/async-demos/alloc-2.png "alloc")

But there is one major problem with that technique, we have more than one thread than can 
create new object at the same time. We need to synchronize those calls somehow or ... we
can give each thread a portion of eden, that is dedicated to only that thread. That portion 
is called **TLAB** - thread local allocation buffer. In such case each thread can use
**bumping pointer** at his own TLAB safely.

Introducing TLABs creates two more issues that JVM needs to deal:

- a thread can allocate an object, but there is not enough space in its TLAB - in that case
  if there is still a space in eden JVM creates new TLAB for that thread
- a thread can allocate a big object, so it's not optimal to use TLAB mechanism - in that case
  JVM will use _slow path_ of allocation that allocates the object directly in eden

What is important to us is that in both this cases JVM emits an event that can be captured
by a profiler. That's basically how async-profiler samples allocation:

- if allocation of an object needed a new TLAB - we see aqua frame for that
- if allocation was done outside the TLAB - we see brown frame

In real world systems the frequency of GC can be monitored by systems like Grafana or Zabbix.
Here we have a synthetic application, so let's measure the allocation size differently:

```shell
# Little warmup
ab -n 2 -c 1 http://localhost:8081/examples/alloc/

# Measuring a heap allocation of a request
jcmd FirstApplication GC.run
jcmd FirstApplication GC.heap_info
ab -n 10 -c 1 http://localhost:8081/examples/alloc/
jcmd FirstApplication GC.heap_info

# Profiling time
./profiler.sh start -e alloc -f alloc.jfr FirstApplication
ab -n 1000 -c 1 http://localhost:8081/examples/alloc/
./profiler.sh stop -f alloc.jfr FirstApplication
```

Let's see at the output of ```GC.heap_info``` commands:

```shell
 garbage-first heap   total 1048576K, used 41926K [0x00000000c0000000, 0x0000000100000000)
  region size 1024K, 2 young (2048K), 0 survivors (0K)
 Metaspace       used 67317K, committed 67904K, reserved 1114112K
  class space    used 9868K, committed 10176K, reserved 1048576K

 garbage-first heap   total 1048576K, used 110018K [0x00000000c0000000, 0x0000000100000000)
  region size 1024K, 8 young (8192K), 0 survivors (0K)
 Metaspace       used 67317K, committed 67904K, reserved 1114112K
  class space    used 9868K, committed 10176K, reserved 1048576K
```

We executed ```alloc``` request ten times and our used heap usage wa increased from **41926K** to **110018K**.
So we are creating over **6MB** of objects per request on a heap. If we look at the controller source code
it's hard to justice that:

```java
@RestController
@RequestMapping("/examples/alloc")
@RequiredArgsConstructor
class AllocController {
    @GetMapping("/")
    String get() {
        return "OK";
    }
}
```

Let's look at the flame graph of allocation size: ([HTML](/assets/async-demos/alloc.html){:target="_blank"})

![alt text](/assets/async-demos/alloc.png "flames")

The class ```AbstractRequestLoggingFilter``` is responsible for over **99%** of recorder allocations. 
How to find where it is created will be done in [Methods profiling](#methods) section, feel free to skip it for now.
Here is an answer:

```java
@Bean
CommonsRequestLoggingFilter requestLoggingFilter() {
    CommonsRequestLoggingFilter loggingFilter = new CommonsRequestLoggingFilter() {
        @Override
        protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
            return !request.getRequestURI().contains("alloc");
        }
    };

    loggingFilter.setIncludeClientInfo(true);
    loggingFilter.setIncludeQueryString(true);
    loggingFilter.setIncludePayload(true);
    loggingFilter.setMaxPayloadLength(5 * 1024 * 1024);
    loggingFilter.setIncludeHeaders(true);
    return loggingFilter;
}
```

I saw that code many times, this ```CommonsRequestLoggingFilter``` is a class that helps you log
the REST endpoint communication. The ```setMaxPayloadLength()``` method sets the max size that we want
to log. You can browse over Spring source code to see that the implementation creates bytes array of
such size in the constructor. No matter how big the payload is, we always create **5MB** array here.

The advice that I gave to users of that code was to create its own filter that will do the same job
but allocate the array lazily.

### Allocation - humongous objects 
{: #alloc-ha }

If you are using G1 garbage collector, which is JVM's default since JDK 9, your heap is divided into
regions. If you are trying to allocate the object that is larger or equal to half of the region size then
you are doing humongous allocation. Long story short it has been, and it is a pain in the ass. 
It is allocated directly in the old generation, but it is also cleared during minor GCs. I saw situations
where G1 GC needed to invoke FullGC phase because of the humongous allocation. If you are doing a 
lot of it G1 will also invoke more concurrent collections, which can waste your CPU.

While running previous example you could spot in ```FirstApplication``` logs:

```shell
...
[70,149s][debug][gc,humongous] GC(34) Reclaimed humongous region 436 (object size 5242896 @ 0x00000000db400000)
[70,149s][debug][gc,humongous] GC(34) Reclaimed humongous region 442 (object size 5242896 @ 0x00000000dba00000)
[70,149s][debug][gc,humongous] GC(34) Reclaimed humongous region 448 (object size 5242896 @ 0x00000000dc000000)
[70,149s][debug][gc,humongous] GC(34) Reclaimed humongous region 454 (object size 5242896 @ 0x00000000dc600000)
...
```

Those are GC logs that tell us that some humongous object of size ```5242896``` was reclaimed. The very nice thing
of JFR files is that they also keep the size of sampled allocations. So with that we should be able to find out
the stack trace that has created that object.

We don't need sophisticated JFR viewer for that. With JDK distribution we've got ```jfr``` command. Let's use it:

```shell
$ jfr summary alloc.jfr
...
 Event Type                          Count  Size (bytes) 
=========================================================
 jdk.ObjectAllocationOutsideTLAB      1013         19220
 jdk.ObjectAllocationInNewTLAB         359          6719
...
```

Let's focus on allocations outside TLAB, it is unlikely to allocate humongous object in the TLAB. 

```shell
$ jfr print --events jdk.ObjectAllocationOutsideTLAB --stack-depth 10 alloc.jfr
...
jdk.ObjectAllocationOutsideTLAB {
  startTime = 2022-12-05T08:56:04.354766183Z
  objectClass = byte[] (classLoader = null)
  allocationSize = 5242896
  eventThread = "http-nio-8081-exec-9" (javaThreadId = 49)
  stackTrace = [
    java.io.ByteArrayOutputStream.<init>(int) line: 81
    org.springframework.web.util.ContentCachingRequestWrapper.<init>(HttpServletRequest, int) line: 90
    org.springframework.web.filter.AbstractRequestLoggingFilter.doFilterInternal(HttpServletRequest, HttpServletResponse, FilterChain) line: 281
    org.springframework.web.filter.OncePerRequestFilter.doFilter(ServletRequest, ServletResponse, FilterChain) line: 116
    org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ServletRequest, ServletResponse) line: 185
    org.apache.catalina.core.ApplicationFilterChain.doFilter(ServletRequest, ServletResponse) line: 158
    org.springframework.web.filter.RequestContextFilter.doFilterInternal(HttpServletRequest, HttpServletResponse, FilterChain) line: 100
    org.springframework.web.filter.OncePerRequestFilter.doFilter(ServletRequest, ServletResponse, FilterChain) line: 116
    org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ServletRequest, ServletResponse) line: 185
    org.apache.catalina.core.ApplicationFilterChain.doFilter(ServletRequest, ServletResponse) line: 158
    ...
  ]
}
...
```

We can easily match ```allocationSize = 5242896``` with the object size from GC logs, so using that 
technique we can find and eliminate humongous allocations. Actually you can filter the allocation
JFR result file with objects that size is larger or equal to half of our G1 region size. All of 
that is humongous allocation.

### Allocation - live objects
{: #alloc-live }

TODO

### Locks
{: #locks }

TODO

## Methods 
{: #methods }

Async-profiler can instrument us a method, so we can see all the stack traces that invoked that method.
To achieve that async-profiler uses instrumentation.

**Big fat warning**: It's already pointed in the README of the profiler that if you are not running
the profiler from ```agentpath``` then the first instrumentation of Java method can result in code
cache flush. It's no a fault of async-profiler, it's a nature of all instrumentation based profilers
combined with JVM's code. Here is a comment from JVM sources:

> Deoptimize all compiled code that depends on this class.
>
> If the can_redefine_classes capability is obtained in the onload
>  phase then the compiler has recorded all dependencies from startup.
>  In that case we need only deoptimize and throw away all compiled code
> that depends on the class.
>
> If can_redefine_classes is obtained sometime after the onload
> phase then the dependency information may be incomplete. In that case
> the first call to RedefineClasses causes all compiled code to be
> thrown away. As can_redefine_classes has been obtained then
> all future compilations will record dependencies so second and
> subsequent calls to RedefineClasses need only throw away code
> that depends on the class.

You can check the [README PR](https://github.com/jvm-profiling-tools/async-profiler/pull/483#discussion_r735019623){:target="_blank"}
discussion for more information on that. For now let's focus on the usage of the mode for our purposes.
In this case we could easily do it with plain IDE debugger, but there are situations where something
is happening only on one environment, or we are tracing some issue that we do not know how to reproduce.

Since Spring beans are usually created during applications startup let's run our application that way:

```shell
java \
-agentpath:/path/to/libasyncProfiler.so=start,event="org.springframework.web.filter.AbstractRequestLoggingFilter.<init>" \
-jar first-application/target/first-application-0.0.1-SNAPSHOT.jar
```

The ```AbstractRequestLoggingFilter.<init>``` is simply a constructor. We are trying to find out
where such an object is created. After our application is started we can execute such a command
in the profiler directory:

```shell
./profiler.sh stop first-application-0.0.1-SNAPSHOT.jar
```

It will print us to standard output one stack trace:

```shell
--- Execution profile ---
Total samples       : 1

--- 1 calls (100.00%), 1 sample
  [ 0] org.springframework.web.filter.AbstractRequestLoggingFilter.<init>
  [ 1] org.springframework.web.filter.CommonsRequestLoggingFilter.<init>
  [ 2] com.example.firstapplication.examples.alloc.AllocConfiguration$1.<init>
  [ 3] com.example.firstapplication.examples.alloc.AllocConfiguration.requestLoggingFilter
  [ 4] com.example.firstapplication.examples.alloc.AllocConfiguration$$SpringCGLIB$$0.CGLIB$requestLoggingFilter$0
  [ 5] com.example.firstapplication.examples.alloc.AllocConfiguration$$SpringCGLIB$$2.invoke
...
  [24] org.springframework.beans.factory.support.AbstractBeanFactory.getBean
...
  [33] org.springframework.boot.web.embedded.tomcat.TomcatStarter.onStartup
...
  [58] org.springframework.boot.web.embedded.tomcat.TomcatWebServer.<init>
...
  [78] org.springframework.boot.loader.JarLauncher.main

       calls  percent  samples  top
  ----------  -------  -------  ---
           1  100.00%        1  org.springframework.web.filter.AbstractRequestLoggingFilter.<init>
```

We have all the information that we need. The object is created in ```AllocConfiguration``` during
creation of ```CommonsRequestLoggingFilter``` bean.

One of the other use-case where I used to use method profiling was finding memory leaks. From heap
dump I knew which object is leaking and with method profiling I could know where objects of that
type were created. That method is obsolete, since now we have [dedicated mode](#alloc-live) to
do it.

There are some methods that are worth mentioning, that those can be traced with that mode, 
let's cover them quickly.

### Exceptions
{: #methods-ex }

If you application works without any outage/downtime, everything is stable, how much 
exception should be thrown? Exception should be thrown if something unexpected happened.
Unfortunately I saw application that used exception-control-flow approach. Throwing new
exception is a CPU consuming operation, since by default it fills the stack trace. 
The number one application in that category that I saw consumed **~15%** of CPU on just
creating new exception. If you want to see where exception with stack trace is created 
you can use async-profiler in method mode with event
```Java_java_lang_Throwable_fillInStackTrace```

TODO example

### G1GC humongous allocation
{: #methods-g1ha }

I've already mentioned how to detect humongous object with allocation mode. Since 
async-profiler can also instrument JVM code, and allocation of humongous object is 
nothing else as invocation of part of a code in C, we can take advantage of that.
If you want to check where humongous objects are allocated you can use method mode
with event ```G1CollectedHeap::humongous_obj_allocate```. This approach may have lower
overhead, but won't give you sizes of allocated objects.

```shell
# Little warmup
ab -n 2 -c 1 http://localhost:8081/examples/alloc/

# Profiling time
./profiler.sh start -e "G1CollectedHeap::humongous_obj_allocate" -f humongous.jfr FirstApplication
ab -n 1000 -c 1 http://localhost:8081/examples/alloc/
./profiler.sh stop -f humongous.jfr FirstApplication
```

Flame graph is almost the same as in ```alloc``` mode, but this time we also
can see some JVM yellow frames:
([HTML](/assets/async-demos/humongous.html){:target="_blank"})

![alt text](/assets/async-demos/humongous.png "flames")

### Thread start
{: #methods-thread }

Starting a platform thread is also an expensive operation. The numbers of started 
threads can be easily monitored with any JMX based monitoring tool. Here is the MBean
with the value of all the created threads: 

```shell
java.lang:type=Threading 
attribute=TotalStartedThreadCount
```

If you monitor that value and the chart like that:

![alt text](/assets/async-demos/threads-2.png "threads")

Then you may check who is creating those short living threads. You can find out that
with method mode of async-profiler using ```JVM_StartThread``` event.

```shell
# Little warmup
ab -n 100 -c 1 http://localhost:8081/examples/thread/

# Profiling time
./profiler.sh start -e "JVM_StartThread" -f threads.jfr FirstApplication
ab -n 1000 -c 1 http://localhost:8081/examples/thread/
./profiler.sh stop -f threads.jfr FirstApplication
```

The flame graph:
([HTML](/assets/async-demos/threads.html){:target="_blank"})

![alt text](/assets/async-demos/threads-1.png "flames")

You may see that such a flame graph is not really complicated. In real life such use cases 
like thread creation looks similar to this one. The code responsible for that thread creation:

```java
@SneakyThrows
@GetMapping("/")
String doInNewThread() {
    ExecutorService threadPool = Executors.newFixedThreadPool(1);
    return threadPool.submit(() -> {
        return "OK";
    }).get();
}
```

And yes, I saw such a thing in real production application. The intention was to have
fixed thread pool and delegate tasks to it, but by mistake someone created that pool
every request.

### Classloading
{: #methods-classes }

Similar to creating short living threads I saw an application that created plenty of
short living class definitions. I know that there are some use cases for such a behavior,
but in mentioned case it has been done by accident. You can monitor number of loaded classes
with JMX:

```shell
java.lang:type=ClassLoading
attribute=TotalLoadedClassCount
```

There are some internals of JVM (like reflection) or some frameworks that can generate you
some new class definitions during runtime, so increasing of that number (even after warmup) 
doesn't mean that we have a problem already. But if ```TotalLoadedClassCount``` is much higher 
than ```LoadedClassCount```, you can check who is creating those classes with method mode and event:
```Java_java_lang_ClassLoader_defineClass1```.

TODO example?

## Perf events
{: #perf }

TODO

## Filtering single request
{: #single-req }

So far we were looking at the profile of a whole application. But what if app is working well
but from time to time there is some slower requests, and we want to know why? In such an application,
where one request is handled by one thread, we can extract profile of a single request. The JFR file contains
all the information needed, we just need to filter them out. To do it, we need to have a log
that will tell us which thread was responsible for execution of the request with time of the
execution. Tomcat that is embedded into Spring Boot has access logs that have all that information.

I configured our example application with access logs in format:

```shell
[%t] [%r] [%s] [%D ms] [%I]
```

Short explanation of that magic:
- ```%t``` - time of finishing handling of the request
- ```%r``` - requested URI
- ```%s``` - response status code
- ```%D``` - duration time in milliseconds
- ```%I``` - thread that handled request

Let's see it in action

```shell
# warmup
ab -n 20 -c 4 http://localhost:8081/examples/filtering/

# profiling of first request
./profiler.sh start -e wall -f filtering.jfr FirstApplication
ab -n 50 -c 4 http://localhost:8081/examples/filtering/
./profiler.sh stop -f filtering.jfr FirstApplication
```

In the access logs we can spot faster and slower requests:

```shell
[05/Dec/2022:18:41:57 +0100] [GET /examples/filtering/ HTTP/1.0] [200] [1044 ms] [http-nio-8081-exec-2]
[05/Dec/2022:18:41:57 +0100] [GET /examples/filtering/ HTTP/1.0] [200] [2779 ms] [http-nio-8081-exec-5]
[05/Dec/2022:18:41:58 +0100] [GET /examples/filtering/ HTTP/1.0] [200] [1048 ms] [http-nio-8081-exec-8]
[05/Dec/2022:18:41:58 +0100] [GET /examples/filtering/ HTTP/1.0] [200] [1052 ms] [http-nio-8081-exec-9]
[05/Dec/2022:18:41:59 +0100] [GET /examples/filtering/ HTTP/1.0] [200] [2829 ms] [http-nio-8081-exec-7]
[05/Dec/2022:18:41:59 +0100] [GET /examples/filtering/ HTTP/1.0] [200] [1058 ms] [http-nio-8081-exec-1]
```

TODO dokończyć 
## Continuous profiling
{: #continuous }

Let's now focus on a such a problem. We had some kind of performance degradation/outage in our system one hour ago.
What can we do? The problem is gone, so attaching profiler now won't help us much. We can start profiling 
and wait for the problem to occur again, but maybe we can inspire ourselves with concept that is working
in aviation business. 

![alt text](/assets/async-demos/cont-1.png "cont")

In case of airplane disaster, what is the plane owner doing? Is he adding logs or attach other tools and waits
for next crash? No, aviation business has flight recorder on every plane. 

![alt text](/assets/async-demos/cont-2.png "cont")

That box records every data it can during the flight. After any disaster that data are ready to be analyzed.
Can we do similar approach with Java profiling? Yes, we can. We can have profiler attach 24/7 dumping the data
every fixed interval of time. If anything bad happens to our application, we have the data that we can analyze.

From my personal experience the continuous profiling is the best technique to diagnose degradations
and outages efficiently. It is also very useful if you want to understand why the performance differs
between two versions of the same application. You just need to get previous version profiling results
from some kind of archive and compare it to current one. 

Here are the ways of enabling async-profiler in continuous mode:

### Command line
{: #continuous-bash }

Here is the simplest way to run async-profiler in continuous mode (dump output every **60 seconds** in **wall** mode):

```shell
while true
do
	CURRENT_DATE=`date +%F_%T`
	./profiler.sh -e wall -f out-$CURRENT_DATE.jfr -d 60 ClassName
done
```

It looks completely dumb, but I used that loop many times. From some time there is a build-in option to do that
loop with just ```profiler.sh``` script:

```shell
./profiler.sh -e wall --loop 1m -f profile-%t.jfr ClassName
```

That is pretty much equivalent to the loop above.

### Java
{: #continuous-java }

I already introduced how to use Java api [here](#how-to-java). To do it continuously you can simply create a thread
that is executing:

```java
AsyncProfiler asyncProfiler = AsyncProfiler.getInstance();

DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd_HH:mm:ss");

while (true) {
    String date = formatter.format(LocalDateTime.now());
    asyncProfiler.execute(
        String.format("start,jfr,event=wall,file=out-%s.jfr", date)
    );
    Thread.sleep(60 * 1000);
    asyncProfiler.execute(
        String.format("stop,file=out-%s.jfr", date)
    );
}
```

### Spring boot
{: #continuous-spring }

If you have a Spring/SpringBoot application you can just use a starter written by me and Michał Rowicki:

```xml
<dependency>
    <groupId>com.github.krzysztofslusarski</groupId>
    <artifactId>continuous-async-profiler-spring-starter</artifactId>
    <version>1.6</version>
</dependency>
```

Read **[readme](https://github.com/krzysztofslusarski/continuous-async-profiler){:target="_blank"}** first.

## Contextual profiling
{: #context-id }

Continuous profiling together with possibility to extract a profile of a single request is very
powerful. Unfortunately there are applications where that is not enough. Some examples:

- Any work that is delegated to different thread will be missed in that profile
- If one request is computed by multiple threads/JVMs we need to combine multiple profiles
- Applications in distributed architecture, even if single request is processed by single thread
  usually there are remote calls to other services

In the last example we can extract profile for each microservice to understand behaviour of the
request processing in such a distributed architecture. This is doable, but consumes a lot of time.

All the problems mentioned above can be covered by **contextual profiling**. The concept is pretty trivial.
Every time when any thread is executing any work, that work is done is some context (eg. in context
of a single request). Instead of just doing that work we do:

```java
asyncProfiler.setContextId(contextId);
actualWork();
asyncProfiler.clearContextId();
```

If there is any sample gathered during ```actualWork()``` the profiler can add ```contextId``` to 
the sample in JFR file. Such a functionality is introduced in
[Context ID PR](https://github.com/jvm-profiling-tools/async-profiler/pull/576){:target="_blank"}.
For now that PR is not merged to the master, but it's a matter of paperwork, I hope it will be
merged soon.

### Contextual profiling in Spring Boot microservices
{: #context-id-spring }

Let's try to join Spring Boot microservices with the contextual profiling. In Spring Boot 3.0
we have included **Micrometer Tracing**. One of its functionality is generating **context ID** 
(called ```traceId```) for every request. That ```traceId``` is passed during execution to
other Spring Boot microservices. We just need to pass that ```traceId``` to the async-profiler
and we are done. 

Since that PR is not merged to the master you need to compile async-profiler from sources.
I compiled it on my Ubuntu x86 with glibc
[here](https://github.com/krzysztofslusarski/async-profiler-demos/blob/master/libasyncProfiler.so){:target="_blank"}.
It may not work on every linux on every machine. If it's your case, just compile the
profiler from sources. It's really easy. 

Ok, let's integrate it with async-profiler. This time I will use Java API. For a start simple
utility class:

```java
public abstract class AsyncProfilerUtils {
    private static volatile AsyncProfiler asyncProfiler;
    private static final Object MUX = new Object();

    public static AsyncProfiler load() {
        if (asyncProfiler == null) {
            synchronized (MUX) {
                if (asyncProfiler == null) {
                    asyncProfiler = AsyncProfiler.getInstance("/tmp/libasyncProfiler.so");
                }
            }
        }
        return asyncProfiler;
    }

    public static void start(String filename) throws IOException {
        load().execute("start,jfr,event=wall,file=" + filename);
    }

    public static void stop(String filename) throws IOException {
        load().execute("stop,jfr,event=wall,file=" + filename);
    }
}
```

I load the profiler from ```/tmp/libasyncProfiler.so```and use **wall-clock** mode, I believe it is the
most suitable mode for most of the enterprise applications.

To integrate profiler with the Micrometer Tracing we need to implement ```ObservationHandler```:

```java
public class AsyncProfilerObservationHandler implements ObservationHandler<Observation.Context> {
    private static final ThreadLocal<TraceContext> LOCAL_TRACE_CONTEXT = new ThreadLocal<>();

    @Override
    public boolean supportsContext(Observation.Context context) { return true; }

    @Override
    public void onStart(Observation.Context context) {
        TracingContext tracingContext = context.get(TracingContext.class);
        TraceContext traceContext = tracingContext.getSpan().context();
        TraceContext currentTraceContext = LOCAL_TRACE_CONTEXT.get();

        if (currentTraceContext == null || 
                !currentTraceContext.traceId().equals(traceContext.traceId())) {
            LOCAL_TRACE_CONTEXT.set(traceContext);
            AsyncProfilerUtils.load().setContextId(lowerHexToUnsignedLong(traceContext.traceId()));
        }
    }

    @Override
    public void onError(Observation.Context context) { }

    @Override
    public void onEvent(Observation.Event event, Observation.Context context) { }

    @Override
    public void onStop(Observation.Context context) {
        TracingContext tracingContext = context.get(TracingContext.class);
        TraceContext traceContext = tracingContext.getSpan().context();
        TraceContext currentTraceContext = LOCAL_TRACE_CONTEXT.get();
        
        if (currentTraceContext != null && 
                currentTraceContext.spanId().equals(traceContext.spanId())) {
            LOCAL_TRACE_CONTEXT.remove();
            AsyncProfilerUtils.load().clearContextId();
        }
    }
}
```

**Big fat warning**: don't treat that class as production ready. It's suitable for that example,
but for sure it will not work with any asynchronous/reactive calls. Mind that ```onStart/onStop```
can be called multiple times with the same ```traceId``` and different ```spanId```.

Now we need to register that implementation:

```java
@Bean
ObservedAspect observedAspect(ObservationRegistry observationRegistry) {
    observationRegistry.observationConfig().observationHandler(new AsyncProfilerObservationHandler());
    return new ObservedAspect(observationRegistry);
}
```

That will also register us the ```@Observed``` aspect. In the end I didn't use it, but such a configuration
remained.

And that's it. Let's try it out.

TODO spring boot profile

```shell
# Little warmup
ab -n 24 -c 1 http://localhost:8081/examples/context/observe

# Profiling time - this time we start profiler from Java
curl -v http://localhost:8081/examples/context/start
curl -v http://localhost:8082/examples/context/start
curl -v http://localhost:8083/examples/context/start

ab -n 24 -c 1 http://localhost:8081/examples/context/observe

# Stopping the profiler
curl -v http://localhost:8081/examples/context/stop
curl -v http://localhost:8082/examples/context/stop
curl -v http://localhost:8083/examples/context/stop
```

Let's look at the times during profiling (I've cut output to 12 rows):

```shell
04/gru/2022:20:54:25 +0100 [GET /examples/context/observe HTTP/1.0] [200] [1538 ms] [http-nio-8081-exec-8]
04/gru/2022:20:54:26 +0100 [GET /examples/context/observe HTTP/1.0] [200] [1537 ms] [http-nio-8081-exec-9]
04/gru/2022:20:54:29 +0100 [GET /examples/context/observe HTTP/1.0] [200] [3039 ms] [http-nio-8081-exec-10]
04/gru/2022:20:54:33 +0100 [GET /examples/context/observe HTTP/1.0] [200] [3218 ms] [http-nio-8081-exec-1]
04/gru/2022:20:54:34 +0100 [GET /examples/context/observe HTTP/1.0] [200] [1538 ms] [http-nio-8081-exec-2]
04/gru/2022:20:54:37 +0100 [GET /examples/context/observe HTTP/1.0] [200] [3038 ms] [http-nio-8081-exec-3]
04/gru/2022:20:54:39 +0100 [GET /examples/context/observe HTTP/1.0] [200] [1538 ms] [http-nio-8081-exec-4]
04/gru/2022:20:54:42 +0100 [GET /examples/context/observe HTTP/1.0] [200] [3270 ms] [http-nio-8081-exec-5]
04/gru/2022:20:54:45 +0100 [GET /examples/context/observe HTTP/1.0] [200] [3038 ms] [http-nio-8081-exec-6]
04/gru/2022:20:54:47 +0100 [GET /examples/context/observe HTTP/1.0] [200] [1536 ms] [http-nio-8081-exec-7]
04/gru/2022:20:54:48 +0100 [GET /examples/context/observe HTTP/1.0] [200] [1536 ms] [http-nio-8081-exec-8]
04/gru/2022:20:54:53 +0100 [GET /examples/context/observe HTTP/1.0] [200] [4756 ms] [http-nio-8081-exec-9]
```

We can see that we have three groups of times:
- ```~1500ms```
- ```~3000ms```
- ```~4500ms```

After we executed script above we have three JFR files in ```/tmp``` directory. When we load all three files 
together to my viewer and check the _Correlation ID stats_ section, we can see:

![alt text](/assets/async-demos/context-1.png "context-1")

So we have similar times from our JFR files. Looking good. Let's filter all the samples by context ID. I's
called _ECID filter_ in my viewer, let's put a value ```-3264552494855344825``` which took ```4650ms``` 
according to records in the JFR. Let's also add additional _filename level_. Filename is correlated to
application name. Here comes the flame graph: ([HTML](/assets/async-demos/context-1.html){:target="_blank"})

![alt text](/assets/async-demos/context-2.png "flames")

All three application on the same flame graph. This is beautiful. Just a reminder: it's not a whole application,
it's a **single request** presented here. I highlighted the ```slowPath()``` method
executed in second and third app, which causes higher latency. You can play with HTML flame graph by your own
to see what is happening there, or you can just jump into the code. I would like to focus on what context ID 
functionality gives us. Because, there is more. We've already added additional _filename level_. We can also
add timestamps as another level. Let's do that. I will present you only the bottom of the graph, since
that's what is important here: ([HTML](/assets/async-demos/context-2.html){:target="_blank"})

![alt text](/assets/async-demos/context-3.png "flames")

The article image resolution may look unclear, you can check [HTML](/assets/async-demos/context-2.html){:target="_blank"}
version for clarity. In the bottom you can see five brown rectangles. Those are timestamps trimmed to seconds.
So basically from left to right we can see what was happening to our request second by second. Let's highlight
when the second application was running during that request:

![alt text](/assets/async-demos/context-4.png "flames")

And the third:

![alt text](/assets/async-demos/context-5.png "flames")

First application is always running since it's entry point to our distributed architecture. The second application
code that is invoked is:

```java
@RequiredArgsConstructor
class ContextService {
    // ...
    void doSomething() {
        if (counter.incrementAndGet() % 3 == 0) {
            slowPath();
            return;
        }

        fastPath();
    }

    private void fastPath() {
        // ...
    }

    private void slowPath() {
        // ...
    }
}
```

And the third application:

```java
class ContextService {
    // ...
    void doSomething() {
        if (counter.incrementAndGet() % 4 == 0) {
            blackhole = slowPath();
            return;
        }
        blackhole = fastPath();
    }

    private int slowPath() {
        // ...
    }

    private int fastPath() {
        // ...
    }
}
```

So the second application is slower every third request, and the third application is slower every forth request.
That means that every twelfth request we are slower in both of them.

I strongly believe that contextual profiling is the future in that area. Everything you see here take with a grain of
salt. My Spring integration and my JFR viewer are far away to something professional. I want to inspire you to
search for the new possibilities like that. If you find any, share it with the rest of Java performance community.

## Notes to remove
832
xdotool search localhost windowraise windowmove 50 50 windowsize 876 800

zmiana na jar