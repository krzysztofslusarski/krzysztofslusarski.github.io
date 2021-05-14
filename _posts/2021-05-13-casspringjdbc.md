---
layout: default
title:  "[Java][JVM][Memory leak][Profiling][Spring] Spring Session JDBC + Apereo CAS = Heap memory leak, and it is a documented feature"
date:   2021-05-13 09:51:30 +0100
---

# [Java][JVM][Memory leak][Profiling][Spring] Spring Session JDBC + Apereo CAS = Heap memory leak, and it is a documented feature
## Before you read this article
Make sure you have read my [previous article](https://krzysztofslusarski.github.io/2021/02/26/casboot.html) about a Spring + CAS memory leak. 

## How a _CAS client_ removes a _HTTP Session_ from its storage 

A little reminder for you, the _CAS client_ removes a _HTTP Session_ using an implementation below:

```java
public final class SingleSignOutHttpSessionListener implements HttpSessionListener {
    ...
    @Override
    public void sessionDestroyed(final HttpSessionEvent event) {
        if (sessionMappingStorage == null) {
            sessionMappingStorage = getSessionMappingStorage();
        }
        final HttpSession session = event.getSession();
        sessionMappingStorage.removeBySessionById(session.getId());
    }
    ...
}
```

It is also mentioned in the [Javadoc](https://github.com/apereo/java-cas-client/blob/master/cas-client-core/src/main/java/org/jasig/cas/client/session/SingleSignOutHttpSessionListener.java)
of that class.

Mind that ```HttpSessionListener``` class is a part of HTTP server, not a Spring Framework. If you register such a listener then during the session 
invalidation _Embedded Tomcat at Spring_ is going to emit an event of type ```javax.servlet.http.HttpSessionEvent```. 
Such an event is processed by all the listeners of type ```HttpSessionListener```.

## _Spring_ documentation

As I wrote in the title of that article, all the behavior described below were documented. Let's look at the _Spring Session_ 
[documentation](https://docs.spring.io/spring-session/docs/current/reference/html5/#httpsession-httpsessionlistener):

> Spring Session supports ```HttpSessionListener``` by translating ```SessionDestroyedEvent``` and ```SessionCreatedEvent``` 
> into ```HttpSessionEvent``` by declaring ```SessionEventHttpSessionListenerAdapter```. To use this support, you need to: 
> * **Ensure your ```SessionRepository``` implementation supports and is configured to fire ```SessionDestroyedEvent``` and ```SessionCreatedEvent```**

The ```SessionRepository``` used in _JDBC_ implementation of _Spring Session_ is defined in ```JdbcIndexedSessionRepository```. Let's look at the 
[Javadoc](https://docs.spring.io/spring-session/docs/current/api/org/springframework/session/jdbc/JdbcIndexedSessionRepository.html):

> A ```SessionRepository``` implementation that uses Spring's ```JdbcOperations``` to store sessions in a relational database. This implementation
> **does not support publishing of session events**.

## Why is there a leak?

Long story short:
* The _CAS client_ needs ```HttpSessionEvent``` published to remove sessions from its storage
* The _Spring Session_ needs ```SessionRepository``` implementation to emit Spring ```SessionDestroyedEvent``` event to publish a proper ```HttpSessionEvent``` 
* The _JDBC_ implementation doesn't emit ```SessionDestroyedEvent```

And yes, it is all documented, so **RTFM** :) 

## What can we do about it?

The most proper way of dealing with this memory leak is:
* **Get rid of** _CAS_ - in most companies it is not an easy task
* **Get rid of** _Spring Session_ - doable
* **Change** _Spring Session_ **implementation** from _JDBC_ to _Hazelcast_ or _Redis_
* **Hack** the _JDBC_ implementation

## How can we hack it?

I don't see any good and easy way of hacking the implementation. We need to be prepared for two actions:
* Manual logout - this is an **easy** part, we can register ```LogoutFilter``` that will manually invoke ```SingleSignOutHttpSessionListener```, ugly, but it works
* Logout after a timeout - a **hard** part

The implementation of removing expired session is done by simple JDBC delete:

```java
public void cleanUpExpiredSessions() {
    Integer deletedCount = this.transactionOperations
            .execute((status) -> JdbcIndexedSessionRepository.this.jdbcOperations.update(
                    JdbcIndexedSessionRepository.this.
                    deleteSessionsByExpiryTimeQuery, System.currentTimeMillis()));

    if (logger.isDebugEnabled()) {
        logger.debug("Cleaned up " + deletedCount + " expired sessions");
    }
}
```

Possible hacks that I can think of:
* Create custom implementation of ```SessionRepository``` that extends ```JdbcIndexedSessionRepository``` and change the implementation of that cleaning -
expired sessions - hard in multi-node deployment, but doable
* Create a _trigger_ on _delete_ on DB level, that will store sessions that should be invalidated in a separate table. This also needs some ```@Scheduled```
job to scan that table in order to invalidate such sessions. 

## What will be done in the "original" application, where I've found that leak?

**Nothing :)** Yes, there is a memory leak, but this leak in that application is **not dangerous**. The memory leaks about **100MB/month** with a **3GB** heap and 
deployments are done at least once a month. The size of that leak depends on the number of unique sessions (created with login through _CAS_) in the application.
If that number increases to the level in which that leak becomes dangerous, the profiled application is going to be **switched to another implementation** of 
_Spring Session_.

The most important part is that the team developing the application knows the reason behind the leak and can easily remove it when necessary. For now adding
new technology (Hazelcast/Redis) is not cost-effective.   