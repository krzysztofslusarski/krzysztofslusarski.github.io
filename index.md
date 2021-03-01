---
layout: default
---
## My articles about JVM performance and tuning
[Biased lock disabled, again](2020/11/09/biased.html)

[To tune or not to tune? G1GC vs humongous allocation](2020/11/10/humongous.html)

[JVM internals basics - Stop-the-world phase (safepoints) - how it works?](2020/11/13/stw.html)

[_Time to safepoint_ - finally there is a tool to find long ones, and yes it is the async-profiler](2020/11/14/tts.html)

[Can G1GC cause outage?](2020/11/29/g1outage.html)

[From one JDK bug to another?](2020/12/14/jdkbugs.html)

[How can inefficient algorithm degrade performance with innocent G1GC in the middle? _Humongous object_ strikes back](2021/01/14/inefficient.html)

[Spring Boot + Spring Security + Apereo CAS + Concurrent Session Control = Heap memory leak](2021/02/26/casboot.html)

## My open source projects
### Aync-profiler utils
I created some tools that help me with profiling big application with [Async-profiler](https://github.com/jvm-profiling-tools/async-profiler): 
* [Collapsed stack viewer](https://github.com/krzysztofslusarski/collapsed-stack-viewer)
* [Collapse JFR](https://github.com/krzysztofslusarski/collapse-jfr)
* [Continuous async-profiler](https://github.com/krzysztofslusarski/continuous-async-profiler)

### JVM/GC logs analyzer
Here is my open source JVM and GC logs analyzer:
* Source code is available [here](https://github.com/krzysztofslusarski/jvm-gc-logs-analyzer) 
* Web part is deployed on [gclogs.com](http://gclogs.com/)

Krzysztof Ślusarski