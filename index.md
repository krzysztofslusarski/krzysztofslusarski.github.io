---
layout: default
title: "JVM/Java profiling and tuning"
---

## Manuals

[**Async-profiler** - manual by use cases](2022/12/12/async-manual.html)

## My articles about JVM performance and tuning

[**\[2025.04\]** Java/JVM native memory leaks](2025/03/31/native.html)

[**\[2022.11\]** Finding heap memory leaks with Async-profiler](2022/11/27/async-live.html)

[**\[2022.08\]** Performance tuning of Hazelcast SQL engine](2022/08/25/hz-sql.html)

[**\[2022.04\]** Tracing a Single Operation in Distributed Systems](2022/04/26/distributed.html)

[**\[2022.03\]** Everybody lies, profilers too](2022/03/21/everybody-lies.html)

[**\[2022.03\]** JVM thread state changes by example](2022/03/21/cont-longtts-addition.html)

[**\[2021.08\]** Continuous profiling with async-profiler - long _time to safepoint_ - logs vs swap](2021/08/22/cont-longtts.html)

[**\[2021.08\]** Continuous profiling with async-profiler](2021/08/17/cont-async.html)

[**\[2021.05\]** Full GC every hour after deploying new application version](2021/05/16/fullgc.html)

[**\[2021.05\]** Spring Session JDBC + Apereo CAS = Heap memory leak, and it is a documented feature](2021/05/13/casspringjdbc.html)

[**\[2021.04\]** Spring Boot + Spring Security + Apereo CAS + Concurrent Session Control = Heap memory leak](2021/02/26/casboot.html)

[**\[2021.01\]** How can inefficient algorithm degrade performance with innocent G1GC in the middle? _Humongous object_ strikes back](2021/01/14/inefficient.html)

[**\[2020.12\]** From one JDK bug to another?](2020/12/14/jdkbugs.html)

[**\[2020.11\]** Can G1GC cause outage?](2020/11/29/g1outage.html)

[**\[2020.11\]** _Time to safepoint_ - finally there is a tool to find long ones, and yes it is the async-profiler](2020/11/14/tts.html)

[**\[2020.11\]** JVM internals basics - Stop-the-world phase (safepoints) - how it works?](2020/11/13/stw.html)

[**\[2020.11\]** To tune or not to tune? G1GC vs humongous allocation](2020/11/10/humongous.html)

[**\[2020.11\]** Biased lock disabled, again](2020/11/09/biased.html)

## Other articles

[**\[2021.11\]** Core dump - the last line of defence during outages](2021/11/14/coredump.html)

[**\[2021.09\]** The ultimate monitoring of the JVM application - out of the box](2021/09/04/jmx.html)

[**\[2021.04\]** Spring Boot + good old JSP/Tags = disaster](2021/04/04/bootjsp.html)

## Monday with JVM logs

[**\[Safepoints\]** Introduction and safepoints logs](2021/07/16/monday-intro.html)

[**\[JIT\]** JIT logs](2021/08/25/monday-jit.html)

[**\[Classloader\]** Classloader logs - can attaching the JFR (and other profilers) cause the outage?](2021/09/02/monday-class.html)

[**\[GC\]** Heap after GC - do I have a memory leak?](2021/07/17/monday-hagc.html)

[**\[GC\]** Allocation rate](2021/08/02/monday-alloc.html)

[**\[GC\]\[G1GC\]** Heap before GC - is my heap wasted?](2021/07/28/monday-hbgc.html)

[**\[GC\]\[G1GC\]** G1GC Stop-the-world phases](2021/08/10/monday-phases.html)

[**\[GC\]\[G1GC\]** Efficiency of the _old generation_ cleanup](2021/08/16/monday-mixed.html)

## My open source projects
### Aync-profiler utils
I created some tools that help me with profiling big application with [Async-profiler](https://github.com/jvm-profiling-tools/async-profiler): 
* [JFR/Collapsed stack viewer](https://github.com/krzysztofslusarski/jvm-profiling-toolkit)
* [Continuous async-profiler](https://github.com/krzysztofslusarski/continuous-async-profiler)

### JVM/GC logs analyzer
Here is my open source JVM and GC logs analyzer:
* Source code is available [here](https://github.com/krzysztofslusarski/jvm-gc-logs-analyzer) 
* Web part is deployed on [gclogs.com](http://gclogs.com/)

Krzysztof Ślusarski
