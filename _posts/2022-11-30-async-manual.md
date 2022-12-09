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
  - [AP Loader](#how-to-apl)
  - [IntelliJ Idea](#how-to-idea)
- [Output formats](#out)
- [Flame graphs](#flames)
- [Basic resources profiling](#basic-resources)
  - [Wall-clock](#wall)
  - [Wall-clock - filtering](#wall-filter)
  - [CPU - easy-peasy](#cpu-easy)
  - [CPU - a bit harder](#cpu-hard)
  - [Allocation](#alloc)
  - [Allocation - humongous objects](#alloc-ha)
  - [Allocation - live objects](#alloc-live)
  - [Locks](#locks)
- [Time to safepoint](#tts)
- [Methods profiling](#methods)
  - [Exceptions](#methods-ex)
  - [G1GC humongous allocation](#methods-g1ha)
  - [Thread start](#methods-thread)
  - [Classloading](#methods-classes)
- [Perf events](#perf)
  - [Page faults](#perf-pf)
  - [Cycles](#perf-cycles)
- [Filtering single request](#single-req)
  - [Why aggregated results are not enough](#single-req-why)
  - [Real life example - DNS](#single-req-dns)
- [Continuous profiling](#continuous)
  - [Command line](#continuous-cli})
  - [Java](#continuous-java)
  - [Spring Boot](#continuous-spring)
- [Contextual profiling](#context-id)
  - [Spring Boot microservices](#context-id-spring)
  - [Distributed systems](#context-id-hz)
- [Random thoughts](#random)

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
java -Xms1G -Xmx1G -XX:+AlwaysPreTouch \
-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints \
-Duser.language=en-US \
-Xlog:safepoint,gc+humongous=trace \
-jar first-application/target/first-application-0.0.1-SNAPSHOT.jar 

java -Xms1G -Xmx1G -XX:+AlwaysPreTouch \
-jar second-application/target/second-application-0.0.1-SNAPSHOT.jar

java -Xms1G -Xmx1G -XX:+AlwaysPreTouch \
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
from ```first-application-0.0.1-SNAPSHOT.jar``` to ```FirstApplication```.

## How to run an Async-profiler 
{: #how-to }

### Command line
{: #how-to-cl }

One of the easiest way of running Async-profiler is using command line. In directory with the
profiler you just need to execute

```shell
./profiler.sh -e <event type> -d <duration in seconds> \
-f <output file name> <pid or application name>

# examples
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

It's worth mentioning that async-profiler is supported in JMH benchmarks. If you have one you just need to run:

```shell
java -jar benchmarks.jar -prof async:libPath=/path/to/libasyncProfiler.so\;output=jfr\;event=cpu
```

JMH will take care about every magic, and you get nice JFR output from async-profiler.

### AP Loader
{: #how-to-apl }

There is a pretty fresh project called [AP Loader](https://github.com/jvm-profiling-tools/ap-loader){:target="_blank"},
that can be also helpful to you. This project packages the native distribution to a single JAR, so
if you deploy on different CPU architectures it may be really handy. With this loader you can also use
the Java API without carrying where the binary of the profiler is located. I really recommend to read
the README of that project, it may be suitable to you.

### IntelliJ Idea
{: #how-to-idea }

If you are using IntelliJ Idea then you have build-in async-profiler already. You can profile any JVM
that is running on your machine and get the results visualized in many forms. Honestly I don't use it that
much. Most of the time I'm running profiler on remote machine, and I've got used to it so badly, that I 
run profiler same way on my localhost.

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
levels during conversion to flame graph. In that post I will use naming of filters from my viewer.
There are other products that can visualize the JFR output, you should use the tool that works
for you. There was none that worked for me, so I wrote my own, but it doesn't mean that it would
be the best choice to everybody.

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
that the most CPU consuming method invoked by my controller is ```CpuConsumer.mathConsumer()```.
It is not a lie, it consumes CPU. But does it consume most of the request time? 
Look at the flame graphs in wall-clock mode:

**First execution:** ([HTML](/assets/async-demos/wall-wall-first.html){:target="_blank"})
![alt text](/assets/async-demos/wall-wall-first.png "flames")

**Second execution:** ([HTML](/assets/async-demos/wall-wall-second.html){:target="_blank"})
![alt text](/assets/async-demos/wall-wall-second.png "flames")

I highlighted the ```CpuConsumer.mathConsumer()`` method. Wall-clock 
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

### Wall-clock - filtering
{: #wall-filter}

If you open the wall-clock flame graph for the first time you may be confused, since
it usually look like this:

![alt text](/assets/async-demos/wall-filter.png "flames")

You see all the threads even if they are sleeping or waiting on some queue for a job.
Wall-clock shows you all of them. Most of the time you want to focus on the frames where
your application is doing something, not when it is waiting. All you need to do is to filter
proper stack traces. If you are using Spring Boot with embedded Tomcat you can filter
stack traces that contains ```SocketProcessorBase.run``` method. In my viewer you can just
paste it to _stack trace filter_, and you are done. If you want to focus on one controller,
class, method, ..., it's just a matter of proper filtering.

### CPU - easy-peasy
{: #cpu-easy }

if you know that your application is CPU intensive, or you want to decrease CPU consumption, 
then the CPU mode is suitable. 

Let's prepare our application:

```shell
# execute it once
curl -v http://localhost:8081/examples/cpu/prepare

# little warmup
ab -n 5 -c 1 http://localhost:8081/examples/cpu/inverse

# profiling time:
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
What you see in the flame graph is exactly Hibernate looking for a dirty entities that
should be flushed.

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
# little warmup
ab -n 10 -c 1 http://localhost:8081/examples/cpu/matrix-slow

# profiling time
./profiler.sh start -e cpu -f matrix-slow.jfr FirstApplication
ab -n 10 -c 1 http://localhost:8081/examples/cpu/matrix-slow
./profiler.sh stop -f matrix-slow.jfr FirstApplication

# little warmup
ab -n 10 -c 1 http://localhost:8081/examples/cpu/matrix-fast

# profiling time
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
# little warmup
ab -n 2 -c 1 http://localhost:8081/examples/alloc/

# measuring a heap allocation of a request
jcmd FirstApplication GC.run
jcmd FirstApplication GC.heap_info
ab -n 10 -c 1 http://localhost:8081/examples/alloc/
jcmd FirstApplication GC.heap_info

# profiling time
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

Let's use such a definition for a purpose of this article:
> A memory leak occurs when a _Garbage Collector_ cannot collect Objects that are no longer needed by the Java application.

A heap memory leak is only one kind of problem that may occur on Java heap. The top three kinds are:
* **a memory leak** - the definition above
* **not enough space on a heap** - sometimes Java application may work fine on a heap it has, but there is a possibility to run a part of an application
  that needs more heap than **-Xmx** - it is not a memory leak
* **gray area between** - these are cases when we allocate memory indefinitely, but our application needs these Objects

How we can detect memory leak? At the end of each _GC cycle_ you can find such an entry in GC logs at _info_ level:

```
GC(11536) Pause Young (Normal) (G1 Evacuation Pause) 6746M->2016M(8192M) 40.514ms
```

You can find **three** sizes in such an entry **A->B(C)** that are:
* **A** - used size of a heap before _GC cycle_
* **B** - used size of a heap after _GC cycle_
* **C** - current size of a whole heap

If we take the **B** value from each collection and put it on a chart we can generate _Heap after GC_ chart. From such a chart we can detect if we have a
memory leak on a heap. If a chart looks like those (those are charts from **7 days** period):

![alt text](/assets/monday-2/1.jpg "1")

![alt text](/assets/monday-2/4.jpg "4")

then there is **no memory leak**. The _garbage collector_ can clean up the heap to the same level every day. The chart with memory leak looks like this one:

![alt text](/assets/monday-2/2.jpg "2")

These spikes to the roof are _to-space exhausted_ situations in the **G1** algorithm, those are not _OutOfMemoryErrors_. After each of those spikes there was
**Full GC** phase that is a **failover** in that algorithm.

Here is an example of **not enough space on a heap** problem:

![alt text](/assets/monday-2/3.jpg "3")

This one spike is an _OutOfMemoryError_. One service was run with arguments that needed **~16GB** on a heap to complete. Unfortunately **-Xmx** was set to
**4GB**. **It is not a memory leak**.

We have to be careful if your application is completely stateless, and we use GC with **young/old generations** (like G1, parallel, serial and CMS).
We need to remember that Objects from **memory leak** live in the **old generation**. In stateless application that part of the heap can be cleared even once a
week. Here is an example - **3 days** of the stateless application:

![alt text](/assets/monday-2/5.jpg "5")

It looks like memory leak, the ```min(heap after gc)``` increasing every day, but if we look at the same chart with one additional day:

![alt text](/assets/monday-2/6.jpg "6")

The GC cleared the heap to the previous level. This was done by **old genereation** cleanup that didn't happen in previous days.

The _Heap after GC_ chart can be generated by probing through JMX. The JVM gives that information by mBeans:

* ```java.lang:type=GarbageCollector,name=G1 Young Generation```
* ```java.lang:type=GarbageCollector,name=G1 Old Generation```

Both mBeans provide you attribute with name ```LastGcInfo``` from where we can extract needed information. 

Most memory leaks that I discovered in last years in enterprise applications were either in
frameworks/libraries or on some kind of bridge between them. Recreating such issue in our example
application would require introducing a lot of strange dependencies, so I figured out that
I would recreate one heap memory leak that I discovered few years ago that was custom-made.

```shell
# preparation
curl http://localhost:8081/examples/leak/prepare
ab -n 1000 -c 4 http://localhost:8081/examples/leak/do-leak

# profiling
jcmd FirstApplication GC.run 
jcmd FirstApplication GC.heap_info
./profiler.sh start -e alloc --live -f live.jfr FirstApplication
ab -n 1000000 -c 4 http://localhost:8081/examples/leak/do-leak
jcmd FirstApplication GC.run
jcmd FirstApplication GC.heap_info
./profiler.sh stop -f live.jfr FirstApplication
```

Let's see at the output of ```GC.heap_info``` commands that were invoked soon after running GC:
```shell
 garbage-first heap   total 1048576K, used 50144K [0x00000000c0000000, 0x0000000100000000)
  region size 1024K, 1 young (1024K), 0 survivors (0K)
 Metaspace       used 68750K, committed 69376K, reserved 1114112K
  class space    used 10054K, committed 10368K, reserved 1048576K

 garbage-first heap   total 1048576K, used 237806K [0x00000000c0000000, 0x0000000100000000)
  region size 1024K, 1 young (1024K), 0 survivors (0K)
 Metaspace       used 68841K, committed 69440K, reserved 1114112K
  class space    used 10063K, committed 10368K, reserved 1048576K
```

So invoking our ```do-leak``` request created **~183MB** of objects that couldn't be freed by GC.

Let's see at the flame graph of allocation with ```--live``` option enabled: ([HTML](/assets/async-demos/live.html){:target="_blank"})

![alt text](/assets/async-demos/live.png "flames")

Most of the leak is created in ```JdbcQueryProfiler``` class, let's look at the sources:

```java
class JdbcQueryProfiler {
    private final Map<String, ProfilingData> profilingResults = new ConcurrentHashMap<>();
  
    <T> T runWithProfiler(String queryStr, Supplier<T> query) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        T ret = query.get();
        stopWatch.stop();
        profilingResults.computeIfAbsent(queryStr, ProfilingData::new).nextInvocation(stopWatch.getTotalTimeMillis());
        return ret;
    }
    // ...
}
```

So that class simply calculates the query execution time for each query and remembers that in
some ```ProfilingData``` structure that is placed in ```ConcurrentHashMap```. That doesn't look scary, 
as long as we are using parametrized queries that are under our control the map should have finite size.
Let's see at the usage of the ```runWithProfiler()``` method:

```java
String getValueForKey(int key) {
    String sql = "select a_value from LEAKY_ENTITY where a_Key = " + key;

    return jdbcQueryProfiler.runWithProfiler(sql, () -> {
        try {
            return jdbcTemplate.queryForObject(sql, String.class);
        } catch (EmptyResultDataAccessException e) {
            return null;
        }
    });
}
```

So, well, we are not using parametrized queries, we create new query string for every ```key```. This way
mentioned ```ConcurrentHashMap``` is growing with every new ```key``` passed to the ```getValueForKey``` method.

If you have a heap memory leak in your application then you have two groups of objects:

![alt text](/assets/async-live/leak-1.png "leak")

* **Live set** - this is a group of objects that are still needed by your application
* **Memory leak** - this is a group of objects that are no longer needed

Garbage collectors cannot free the second group if there is at least one strong reference from **live set** to **memory
leak**. The biggest problems with diagnosis memory leaks are:

* the fact that the object was created **is not an issue** - it was created because it was needed for something
* the fact that the mentioned reference was created **is not an issue** - it had some purpose too
* we need to understand why that reference was not removed by our application

The last one is not trivial. All the observability/profiling tools give us a great possibility to understand why
some event has happened, but with memory leaks we need to understand why something hasn't happened.
We can use two additional tools that can help us:

* **heap dump** - that shows us current state of a heap - we can find out what kind of objects are, but shouldn't be
  there
* **profiler** - that shows us where those objects were created

In this simple example any of those tools in enough. In more complicated ones I needed both to find the root cause
of the problem. It is nice to finally have a tool that can profile memory leak on production system.

It is worth mentioning that ```--live``` option is available since **async-profiler 2.9** and it needs 
**JDK >= 11**.

### Locks
{: #locks }

When we need better knowledge about lock contention in our application the lock mode may be useful.
Let's try to use it and understand internals of ```ConcurrentHashMap```. It is commonly known that
```get()``` method is lock free, but what about ```computeIfAbsent()```? Let's profile such a code:

```java
class LockService {
    private final Map<String, String> map = new ConcurrentHashMap<>();
  
    LockService() {
        String a = "AaAa";
        String b = "BBBB";
        log.info("Hashcode equals: {}", a.hashCode() == b.hashCode()); // true
        map.computeIfAbsent(a, s -> a);
        map.computeIfAbsent(b, s -> b);
    }
  
    void withLock(String key) {
        map.computeIfAbsent(key, s -> key);
    }
  
    void withoutLock(String key) {
        if (map.get(key) == null) {
            map.computeIfAbsent(key, s -> key);
        }
    }
}
```

Let's use lock mode to profile that:

```shell
# preparation
ab -n 100 -c 1 http://localhost:8081/examples/lock/with-lock
ab -n 100 -c 1 http://localhost:8081/examples/lock/without-lock

# profiling
./profiler.sh start -e lock -f lock.jfr FirstApplication
ab -n 100000 -c 100 http://localhost:8081/examples/lock/with-lock
ab -n 100000 -c 100 http://localhost:8081/examples/lock/without-lock
./profiler.sh stop -f lock.jfr FirstApplication
```

The flame graph: ([HTML](/assets/async-demos/lock.html){:target="_blank"})

![alt text](/assets/async-demos/lock-1.png "flames")

I highlighted the ```LockController``` occurance. Let's zoom it:

![alt text](/assets/async-demos/lock-2.png "flames")

So we see only locking in ```withLock()``` method. You can study the internals of ```computeIfAbsent()``` method,
when you have a hash collision it may lock. If you have a huge lock contention on this method, and most of the 
time a key is already in the map, then you may consider approach used in ```withoutLock()``` method.


## Time to safepoint
{: #tts }

The common knowledge in the Java developers world is that _garbage collectors_ need Stop-the-world (STW) phase to clean dead objects.
First of all, **not only GC needs it**. There are other internal mechanisms that need to do some work, that require application threads to be hanged.
For example JIT compiler needs STW phase to _deoptimize_ some compilations and to revoke _biased locks_. Let's get a closer look on
how STW phase works.

On our JVM there are running some application threads:

![alt text](/assets/stw/1.png "chart 1")

While running those threads from time to time JVM needs to do some work in the STW phase. So it start it with
_global safepoint request_, which is an information for every thread to go to "sleep":

![alt text](/assets/stw/2.png "chart 2")

Every thread has to find out about this information. Checking if it needs to fall asleep is simply a line on assembly code
generated by the JIT compiler and a simple step in the interpreter. Of course every thread can now execute a different method/JIT compilation,
so time in which threads are going to be aware of STW phase is different for every thread.   
Every thread has to wait for the slowest one. Time between starting STW phase, and the slowest thread finding that information is called
_time to safepoint_:

![alt text](/assets/stw/3.png "chart 3")

Only after every thread is asleep, JVM threads can do the work that needed STW phase. A time when application threads were sleeping
is called _safepoint operation time_:

![alt text](/assets/stw/4.png "chart 4")

When JVM finishes its work application threads are waken up:

![alt text](/assets/stw/5.png "chart 5")

If the JVM application suffers from long STW phases most of the time those are GC cycles, and that information can be found
in GC logs. But if the application has one thread that slows down every other from reaching the safepoint then 
the situation is more tricky.

```shell
# preparation
curl http://localhost:8081/examples/tts/start
ab -n 100 -c 1 http://localhost:8081/examples/tts/execute

# profiling
./profiler.sh start --ttsp -f tts.jfr FirstApplication
ab -n 100 -c 1 http://localhost:8081/examples/tts/execute
./profiler.sh stop -f tts.jfr FirstApplication
```

In safepoint logs (you need to run your JVM with ```-Xlog:safepoint``` flag) we can see:
```shell
[105,372s][info ][safepoint   ] Safepoint "ThreadDump", Time since last: 156842 ns, Reaching safepoint: 13381 ns, At safepoint: 120662 ns, Total: 134043 ns
[105,372s][info ][safepoint   ] Safepoint "ThreadDump", Time since last: 157113 ns, Reaching safepoint: 14738 ns, At safepoint: 120252 ns, Total: 134990 ns
[105,373s][info ][safepoint   ] Safepoint "ThreadDump", Time since last: 157676 ns, Reaching safepoint: 13700 ns, At safepoint: 120487 ns, Total: 134187 ns
[105,402s][info ][safepoint   ] Safepoint "ThreadDump", Time since last: 159020 ns, Reaching safepoint: 29524545 ns, At safepoint: 160702 ns, Total: 29685247 ns
```

The _Reaching safepoint_ contains time to safepoint. Most of the time it is **>15k ns** but there is some much longer:
**29ms**. Async-profiler in ```--ttsp``` mode collects samples between:

- ```SafepointSynchronize::begin```, and
- ```RuntimeService::record_safepoint_synchronized```

During that time our application threads are trying to reach safepoint:
([HTML](/assets/async-demos/tts.html){:target="_blank"})

![alt text](/assets/async-demos/tts.png "flames")

You can see that most of the gathered samples are executing ```arraycopy``` invoked from ```TtsController```.
The time to safepoint issues that I approach so far:

- **arraycopy** - as in our example
- **old JDK + loops** - since JDK 11u4 we have _loop strip mining_ optimization working correctly, before that if you had
  a counted loop, it could be compiled without any check for safepoint
- **swap** - when your application thread executes some work in _thread_in_vm_ state (after calling some native method),  
  and during that execution it waits for some pages to be swapped in/out, that can slow down reaching the safepoint

For the **arraycopy** issue the solution is to copy the array by some custom method. It will be a bit slower,
but will not slow down all the threads but one.

For the **old JDK + loops** you can upgrade or change the counted loop int uncounted one. More information about that
[here](http://psy-lob-saw.blogspot.com/2016/02/wait-for-it-counteduncounted-loops.html){:target="_blank"}.

For the **swap** issue, just disable swap.

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
you can use async-profiler in method mode with 
event ```Java_java_lang_Throwable_fillInStackTrace```. 

Let's start our application with profiler enabled from a start to see also how many
exceptions are thrown during startup of Spring Boot application, just for fun:

```shell
java \
-agentpath:/path/to/libasyncProfiler.so=start,jfr,file=exceptions.jfr,event="Java_java_lang_Throwable_fillInStackTrace" \
-jar first-application/target/first-application-0.0.1-SNAPSHOT.jar
```

After the startup let's do:

```shell
ab -n 100 -c 1 http://localhost:8081/examples/exc/
./profiler.sh stop -f exceptions.jfr first-application-0.0.1-SNAPSHOT.jar
```

The flame graph is too large to post it here as an image, sorry. Spring Boot in that case
threw **12478** exceptions. You can play with [HTML](/assets/async-demos/exceptions.html){:target="_blank"}.
Let's focus on our controller, if those were also caught:

![alt text](/assets/async-demos/exceptions.png "flames")

Source code of the controller:
```java
@GetMapping("/")
String flowControl() {
    ThreadLocalRandom random = ThreadLocalRandom.current();
    try {
        if (!random.nextBoolean()) {
            throw new IllegalArgumentException("Random returned false");
        }
    } catch (IllegalArgumentException e) {
        return "EXC";
    }

    return "OK";
}
```

If you care about performance, don't use exception-control-flow approach. If you really need such a code,
you have an exception constructor that doesn't fill the stack trace:

```java
/**
 * Constructs a new exception with the specified detail message,
 * cause, suppression enabled or disabled, and writable stack
 * trace enabled or disabled.
 *
 * @param  message the detail message.
 * @param cause the cause.  (A {@code null} value is permitted,
 * and indicates that the cause is nonexistent or unknown.)
 * @param enableSuppression whether or not suppression is enabled
 *                          or disabled
 * @param writableStackTrace whether or not the stack trace should
 *                           be writable
 * @since 1.7
 */
protected Exception(String message, Throwable cause,
                    boolean enableSuppression,
                    boolean writableStackTrace) {
    super(message, cause, enableSuppression, writableStackTrace);
}
```

Just set ```writableStackTrace``` to ```false```. I will be also ugly, but faster.

### G1GC humongous allocation
{: #methods-g1ha }

I've already mentioned how to detect humongous object with allocation mode. Since 
async-profiler can also instrument JVM code, and allocation of humongous object is 
nothing else as invocation of part of a code in C, we can take advantage of that.
If you want to check where humongous objects are allocated you can use method mode
with event ```G1CollectedHeap::humongous_obj_allocate```. This approach may have lower
overhead, but won't give you sizes of allocated objects.

```shell
# little warmup
ab -n 2 -c 1 http://localhost:8081/examples/alloc/

# profiling time
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
# little warmup
ab -n 100 -c 1 http://localhost:8081/examples/thread/

# profiling time
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
than ```LoadedClassCount```, you can check who is creating those classes with method mode and 
event: ```Java_java_lang_ClassLoader_defineClass1```.

To be honest I saw such an issue once in my life and I cannot reproduce it now. Making 
synthetic example for this use-case seems wrong, so I will just keep you with the knowledge
that there is such possibility.

## Perf events
{: #perf }

Async-profiler can also help you with low-level diagnosis where you want to correlate perf
events with Java code. For example you can use events:

- ```context-switches``` - to find out which parts of your Java code does context switching
- ```cache-misses``` - which part of your code can stall due to cache misses - if you have many
  context switches that information is harder to analyze
- ```LLC-load-misses```- which part of your code needs something from RAM memory
- ...

I want to describe two in more details.

### Page faults
{: #perf-pf }

![alt text](/assets/async-demos/page-fault-1.png "page")

Every process that is running on Linux contains its own virtual memory. If a process needs more
memory it invokes function like ```malloc``` or ```mmap```. Returning from that function gives
a guarantee from OS that the process can now read/write from/to that memory.
But that returning doesn't mean that eny byte from RAM was reserved by the process. 

Whe we are touching a page of memory for the first time, in the kernel ```page fault``` happens.
Now OS is smart enough to detect if that fault should be converted to SEGFAULT or should kernel
map RAM do virtual memory, because it was previously promised to the process.

Java is a process from OS perspective, nothing less, nothing more. Knowing that we can trace
```page fault``` event to detect why our application consumes more RAM. I may be native memory 
leak, or some framework/library/JVM bug.

For now, it is not perfect for tracing leaks since it shows every need of additional RAM,
including the one that may be freed in the future. I know that Andrei Pangin is working
on native memory leak detector that will trace allocations that hasn't been freed, but for
now that feature is not in the latest release.

As an example let's run our application with and without ```-XX:+AlwaysPreTouch``` and see 
where Java needs more RAM after startup. We will use our heap memory leak that we used
before:

```shell
java -Xmx1G -Xms1G -XX:+AlwaysPreTouch \
-jar first-application/target/first-application-0.0.1-SNAPSHOT.jar
```

In the other console lets do:

```shell
ab -n 10000 -c 4 http://localhost:8081/examples/leak/do-leak

./profiler.sh start -e page-faults -f page-faults-apt-on.jfr first-application-0.0.1-SNAPSHOT.jar
ab -n 1000000 -c 4 http://localhost:8081/examples/leak/do-leak
./profiler.sh stop -f page-faults-apt-on.jfr first-application-0.0.1-SNAPSHOT.jar
```

Now let's do the same without ```-XX:+AlwaysPreTouch```: 

```shell
java -Xmx1G -Xms1G \
-jar first-application/target/first-application-0.0.1-SNAPSHOT.jar
```

In the other console lets do:

```shell
ab -n 10000 -c 4 http://localhost:8081/examples/leak/do-leak

./profiler.sh start -e page-faults -f page-faults-apt-off.jfr first-application-0.0.1-SNAPSHOT.jar
ab -n 1000000 -c 4 http://localhost:8081/examples/leak/do-leak
./profiler.sh stop -f page-faults-apt-off.jfr first-application-0.0.1-SNAPSHOT.jar
```

Flame graph without ```-XX:+AlwaysPreTouch```: ([HTML](/assets/async-demos/page-faults-apt-off.html){:target="_blank"})
![alt text](/assets/async-demos/page-faults-apt-off.png "flames")

Most of the need of addition RAM is done in GC threads, but there are some in our Java code
(big green flame). All of that page faults can hurt your performance and make your latency less
predictable.

Flame graph with ```-XX:+AlwaysPreTouch```: ([HTML](/assets/async-demos/page-faults-apt-on.html){:target="_blank"})
![alt text](/assets/async-demos/page-faults-apt-on.png "flames")

Almost all the frames needed additional RAM are now one by compiler threads. This is done
because code heap is growing with creation of new compilation.

With that mode I was able to isolate and recreate memory leak that I described in
[JDK-8240723](https://bugs.openjdk.org/browse/JDK-8240723){:target="_blank"}.

### Cycles
{: #perf-cycles }

If you need a better visibility of what your kernel is doing then you may consider choosing
```cycles``` event instead of ```cpu``` one. This may be useful for low latency applications
or while chasing bug in kernel (those also exist). Let's see the difference: 

```shell
# warmup
curl -v http://localhost:8081/examples/cycles/

# profiling
./profiler.sh start -e cpu -f cycles-cpu.jfr FirstApplication
curl -v http://localhost:8081/examples/cycles/
./profiler.sh stop -f cycles-cpu.jfr FirstApplication
./profiler.sh start -e cycles -f cycles-cycles.jfr FirstApplication
curl -v http://localhost:8081/examples/cycles/
./profiler.sh stop -f cycles-cycles.jfr FirstApplication
```

The flame graph for ```cpu``` profiling:
([HTML](/assets/async-demos/cycles-cpu.html){:target="_blank"})

![alt text](/assets/async-demos/cycles-cpu.png "flames")

Corresponding profile for ```cycles``` event:
([HTML](/assets/async-demos/cycles-cycles.html){:target="_blank"})

![alt text](/assets/async-demos/cycles-cycles.png "flames")

As we can see, the ```cycles``` profile is more detailed. 

## Filtering single request
{: #single-req }

### Why aggregated results are not enough
{: #single-req-why }

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
 
Let's load the JFR to m viewer and look at the flame graph of a whole application:
([HTML](/assets/async-demos/filtering-1.html){:target="_blank"})

![alt text](/assets/async-demos/filtering-1.png "flames")

It's hard to guess from that why some requests are slower than the others. We can see two different
methods executed at the top of the flame graph:

- ```matrixMultiplySlow()```
- ```matrixMultiplyFaster()```

We cannot conclude from that which one is responsible for worse latency. Let's add filters to that
graph to understand latency of second request from pasted access log:

```
[05/Dec/2022:18:41:57 +0100] [GET /examples/filtering/ HTTP/1.0] [200] [2779 ms] [http-nio-8081-exec-5]
```

- _Access log filter_:
  - _End date_ - ```05/Dec/2022:18:41:57 +0100```
  - _End date format_ - let's keep the default one
  - _Duration_ - ```2779```
  - _Locale language_ - ```EN```
- _Thread filter_ - ```http-nio-8081-exec-5```

Now the flame graph is obvious:
([HTML](/assets/async-demos/filtering-2.html){:target="_blank"})

![alt text](/assets/async-demos/filtering-2.png "flames")

We can check this way few more requests, and we can figure our that:

- In every slow request we executed ```matrixMultiplySlow()```
- In every fast request we executed ```matrixMultiplyFaster()```

That technique is great to fight with the tail of the latency. We can focus our work on longest
operations. That may lead us to some nice fixes.

### Real life example - DNS
{: #single-req-dns }

The point of previous example was to show you why aggregated results can be useless for tracing
a single request. Now I want to show you very common issue that I diagnosed few times already.

Spring Boot has an addition called Actuator. It is commonly used in enterprise world. One of the
feature of Actuator is health check endpoint. Under URI ```/actuator/healh``` you can get JSON
with information about health of your application. That endpoint is sometimes used as a load
balancer probe. Let's consider a multi node cluster of our example application with load
balancer in front of the cluster which:

- probes actuator if application is alive, expecting ```"status" : "UP"``` in the response JSON
- timeouts the probe after **1 second**

Now I will do one hack in my local configuration to make this example works. It will be explained in the end
of this example.

Let's find out what our IP is:

```shell
$ ifconfig 

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 9001
        inet 172.31.36.53  netmask 255.255.240.0  broadcast 172.31.47.255
```

Let's probe an actuator by this IP, not a ```localhost``` (without the hack described later you cannot get same results):

```shell
./profiler.sh start -e wall -f actuator.jfr FirstApplication
ab -n 1000 http://172.31.36.53:8081/actuator/health # check your IP
./profiler.sh stop -f actuator.jfr FirstApplication
```

The result of ```ab```:

```shell
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.0      0       0
Processing:     0    5 158.1      0    5000
Waiting:        0    5 158.1      0    5000
Total:          0    5 158.1      0    5000

Percentage of the requests served within a certain time (ms)
  50%      0
  66%      0
  75%      0
  80%      0
  90%      0
  95%      0
  98%      0
  99%      1
 100%   5000 (longest request)
```

So almost all the actuator endpoints were returned in **0ms**, but there was at least one that lasted **5s**.
In access logs we can see one longer request:

```shell
[08/DEC/2022:08:56:24 +0000] [GET /actuator/health HTTP/1.0] [200] [0 ms] [http-nio-8081-exec-7]
[08/DEC/2022:08:56:24 +0000] [GET /actuator/health HTTP/1.0] [200] [0 ms] [http-nio-8081-exec-10]
[08/DEC/2022:08:56:24 +0000] [GET /actuator/health HTTP/1.0] [200] [0 ms] [http-nio-8081-exec-3]
[08/DEC/2022:08:56:24 +0000] [GET /actuator/health HTTP/1.0] [200] [0 ms] [http-nio-8081-exec-2]
[08/DEC/2022:08:56:29 +0000] [GET /actuator/health HTTP/1.0] [200] [4999 ms] [http-nio-8081-exec-8]
[08/DEC/2022:08:56:29 +0000] [GET /actuator/health HTTP/1.0] [200] [0 ms] [http-nio-8081-exec-5]
```

That **5s** response would make our load balancer remove that node (for some time) from a cluster. Let's
use the same technique to find out what was the reason of that latency:
([HTML](/assets/async-demos/actuator.html){:target="_blank"})

![alt text](/assets/async-demos/actuator.png "flames")

I highlighted usage of ```RemoteIpFilter``` class. Time for some explanations. When your requests are hitting 
your application you can check the IP of the requester with basic ```HttpServletRequest``` API. But if you have
a load balancer before your application, well, you get an IP of load balancer, not the original requester.
To avoid such a confusion load balancers usually add HTTP headers to the request. The original IP is sent
in ```X-Forwarded-For``` header. The ```RemoteIpFilter``` is a tool that makes our lives easier and make the
```HttpServletRequest``` API returns proper IP and so on.

Let's get back to the flame graph. We can see that this filter create an instance of ```XForwardedRequest``` 
that executes ```RequestFacade.getLocalName()``` that in the end executes ```Inet6AddressImpl.getHostByAddr()```.
The last method is trying to identify host name by IP address. How can it be done? Well, we just need a 
request to DNS, nothing more. In that case DNS protocol uses UDP, not TCP. UDP is a protocol that by design
can lose packets. In Linux the ```resolv.conf``` is responsible for probing DNS and the whole retransmission stuff.
Here is a part of manual:

```
timeout:n
       Sets  the amount of time the resolver will wait for a re‐
       sponse from a remote  name  server  before  retrying  the
       query  via  a different name server.  This may not be the
       total time taken by any resolver API call and there is no
       guarantee  that a single resolver API call maps to a sin‐
       gle  timeout.   Measured  in  seconds,  the  default   is
       RES_TIMEOUT (currently 5, see <resolv.h>).  The value for
       this option is silently capped to 30.
```

Long story short - the default timeout is **5s**. If your DNS request is lost ```resolv.conf``` will probe next
_nameserver_ after **5s**. That's what is happening in our example. That what was happening in multiple
Java systems I saw in the past. DNS is commonly used for DDoS attacks. You can easily have firewall in your 
infrastructure that can drop some DNS packets by design. 

The funny thing about ```RemoteIpFilter``` is that the result of that DNS probing is set to field ```localName``` that 
is not used later. So we are just doing DNS request for nothing. To avoid that problem you can simply write
your own filter that won't fire DNS requests. ```RemoteIpFilter``` is open-sourced, so it's an easy job.
There is also ```RemoteIpValve``` that can be enabled by just an entry in Spring Boot properties. It used
to have the same issue. I didn't check if in Spring Boot 3 it exists, it might be fixed accidentally
with [this bug fix](https://bz.apache.org/bugzilla/show_bug.cgi?id=57665){:target="_blank"} which introduced
```changeLocalName``` property. If you want to be sure you need to check it by yourself.

This is not the only DNS request that can be done 
by Actuator health check. That endpoint also probes your databases, queues and many other things. If that probing
is done by DNS name, the same thing may happen. 
 
About the hack. If you want to simulate not responsive DNS you can add 72.66.115.13 (blackhole.webpagetest.org) 
as your nameserver. That one is designed to drop all the packets. On different Linux distribution it is done differently. 
I just use AWS instance with Amazon Linux distribution and there I could just edit ```/etc/resolv.conf``` file, but 
in distros like Ubuntu that file is generated by other services.

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

### Spring Boot microservices
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
@Profile("context")
ObservedAspect observedAspect(ObservationRegistry observationRegistry) {
    observationRegistry.observationConfig().observationHandler(new AsyncProfilerObservationHandler());
    return new ObservedAspect(observationRegistry);
}
```

That will also register us the ```@Observed``` aspect. In the end I didn't use it, but such a configuration
remained.

And that's it. Let's try it out. We need to rerun the Spring Boot applications with additional profile:

```shell
java -Xms1G -Xmx1G \
-Dspring.profiles.active=context \
-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints \
-jar first-application/target/first-application-0.0.1-SNAPSHOT.jar 

java -Xms1G -Xmx1G \
-Dspring.profiles.active=context \
-jar second-application/target/second-application-0.0.1-SNAPSHOT.jar

java -Xms1G -Xmx1G \
-Dspring.profiles.active=context \
-jar third-application/target/third-application-0.0.1-SNAPSHOT.jar
```

And now the applications will look for async-profiler in ```/tmp/libasyncProfiler.so```.

```shell
# little warmup
ab -n 24 -c 1 http://localhost:8081/examples/context/observe

# profiling time - this time we start profiler from Java
curl -v http://localhost:8081/examples/context/start
curl -v http://localhost:8082/examples/context/start
curl -v http://localhost:8083/examples/context/start

ab -n 24 -c 1 http://localhost:8081/examples/context/observe

# stopping the profiler
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
called _Correlation ID filter_ in my viewer, let's put a value ```-3264552494855344825``` which took ```4650ms``` 
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

### Distributed systems
{: #context-id-hz }

At first let's differentiate distributed architecture from distributed system. For a purpose of this post let's assume
following:

- distributed architecture is a set of applications that works together - like microservices
- distributed system is one application that is deployed on more than one JVM and to service some request
  it distributes the work to more than one instance

One example of distributed system may be Hazelcast. I've applied the context ID functionality to trace tail of
the latency in SQL queries.

Sample benchmark details:

* Hazelcast cluster size: **4**
* Machines with **Intel Xeon CPU E5-2687W**
* Heap size: **10 GB**
* **JDK17**
* SQL query that is benchmarked: ```select count(*) form iMap```
* iMap size – **1 million** java serialized objects
* Benchmark duration: **8 minutes**

The latency distribution for that benchmark:

![alt text](/assets/distributed/dist.png "dist")

The **50th** percentile is **1470 ms**, whereas the **99.9th** is **3718 ms**. Let’s now analyze the **JFR** outputs with my tool.
I’ve created a table with the longest queries in the files:

![alt text](/assets/distributed/cid.png "cid")

Let’s analyze a single query using context ID functionality. The full flame graph:

![alt text](/assets/distributed/1.png "1")

Let’s focus on the bottom rows:

![alt text](/assets/distributed/2.png "2")

Let’s start with timestamps and highlight them one by one:

![alt text](/assets/distributed/3.png "3")
![alt text](/assets/distributed/4.png "4")
![alt text](/assets/distributed/5.png "5")
![alt text](/assets/distributed/6.png "6")
![alt text](/assets/distributed/7.png "7")

Summing that up:

* **41.91%** of samples were gathered between 14:47:19 and 14:47:20
* **35.79%** of samples were gathered between 14:47:20 and 14:47:21
* **6.68%** of samples were gathered between 14:47:21 and 14:47:22
* **12.37%** of samples were gathered between 14:47:22 and 14:47:23
* **3.34%** of samples were gathered between 14:47:23 and 14:47:24

Let’s highlight the filenames (which are named with the **IP** of the server) one by one:

![alt text](/assets/distributed/8.png "8")
![alt text](/assets/distributed/9.png "9")
![alt text](/assets/distributed/10.png "10")
![alt text](/assets/distributed/11.png "11")

A little summary:

* **25.42%** of samples are from node 10.212.1.101
* **23,75%** of samples are from node 10.212.1.102
* **21.07%** of samples are from node 10.212.1.103
* **29.77%** of samples are from node 10.212.1.104

Since brown bars are sorted alphabetically by timestamps, we can conclude that the **10.212.1.104** server is doing any work
in the last seconds of processing.

I checked five more long latency requests and the results were the same, confirming that the **10.212.1.104** server is 
the problem. I spent a lot of time trying to figure out what is wrong with that machine. My biggest suspect was a 
difference in meltdown/spectre patches in the kernel. In the end we reinstalled Linux on those machines which solved
the problem with **10.212.1.104** server.

## Random thoughts
{: #random }

1. You need to remember that EVERY profiler lies in some way. The async-profiler is vulnerable to
[JDK-8281677](https://bugs.openjdk.org/browse/JDK-8281677){:target="_blank"}. There is nothing that profiler
can do, JVM is lying to the profiler, so that lie is passed to the end user. You can change the mechanism
that is used by a profiler, but you will be lied too, maybe in different way.
2. You can run async-profiler to collect more than one event. It is allowed to gather ```lock``` and ```alloc```
together with one of the modes that gathers execution samples, like ```cpu```, ```wall```, ```method```...
3. You can run async-profiler with ```jfrsync``` option that will gather more information, that are exposed 
by the JVM.

## Notes to remove
832
xdotool search localhost windowraise windowmove 50 50 windowsize 876 800

zmiana na jar