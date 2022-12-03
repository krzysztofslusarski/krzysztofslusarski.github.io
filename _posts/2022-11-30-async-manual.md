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

- [Profiled application](#profiled-application)
- [How to run an Async-profiler](#how-to)
  - [Command line](#how-to-cl)
  - [During JVM startup](#how-to-jvm)
  - [From Java API](#how-to-java)
- [Output formats](#out)
- [Flame graphs](#flames)
- [Basic resources profiling](#basic-resources)
  - [Wall-clock](#wall)
  - [CPU - easy-peasy](#cpu-easy)
  - [CPU - a bit harder](#cpu-hard)
  - [Allocation](#alloc)
  - [Allocation - live objects](#alloc-live)

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

All the flame graphs that are attached to that post are generated by mentioned tool.

## Flame graphs
{: #flames }

If you do **sampling profiling** you need to visualize the results. The results are nothing more than a **set of stacktraces**.
My favorite way of visualization is a **flame graph**. The easiest way to understand what flame graphs are is to understand how
they are created.

First part is to draw a rectangle for each frame of each stacktrace. The stacktraces are drawn bottom-up and sorted
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

### CPU - easy-peasy
{: #cpu-easy }

if you know that your application is CPU intensive and you want to decrease CPU consumption, 
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
- fighting with single/tens of milliseconds latency, where even heap allocation matters

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
by a profiler. That's basically how async-profiler samples allocation.

### Allocation - live objects
{: #alloc-live }

## Notes to remove
832
xdotool search localhost windowraise windowmove 50 50 windowsize 876 800