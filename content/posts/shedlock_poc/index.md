---
title: "A very easy-to-wire-up distributed lock solution for scheduled tasks"
date: 2020-12-19T21:37:07+02:00
draft: false
tags: ['lock', 'scheduled', 'distributed', 'micronaut', 'elasticsearch']
---

[Shedlock](https://github.com/lukas-krecan/ShedLock)
is a very well documented library and has already integrated with
[SpringBoot](https://spring.io/projects/spring-boot)
and [Micronaut](https://micronaut.io/) way of setting a scheduled task.
Usually, these kind of frameworks can schedule a task with just simply
applying a decorator/annotation like so `@Scheduled` on top of your java
method.

Deploying a `SpringBoot` application or a `Micronaut` (nowadays) into an
orchestration platform like ***Kubernetes*** usually brings some issues;
these scheduled tasks are bound to the container/vm they're running into.
Meaning that, if we have `3 replicas` of that service that executes a
`@Scheduled` task, we will experience that task executing `3` times -- once per `replica` of a container.

On a recent task of mine to implement a scheduled task at my work
which would call a third-party api to:
- fetch some data
- update some other
- and report (after calling again and checking)

Obviously, we wouldn't want our scheduled task to execute these routines
as many times as the `replica number` of the container inside the Kubernetes cluster.
Hence, a colleague of mine introduced me to `Shedlock`.

As mentioned above, ***Shedlock*** already has support on ***SpringBoot***
and ***Micronaut*** so to make it work was no pain at all. The only dependency
required is a `DataSource` -- a DB, cache, etc. The list of all the datastores
that it supports is right [here](https://github.com/lukas-krecan/ShedLock#configure-lockprovider)
```
Mongo
Redis
ElasticSearch
Cassandra
...
and many more!
```

On a high level, ***Shedlock*** creates an index/table where the `id` of the scheduled
task is *locked*, and it will make sure that only one of the instances
will execute the scheduled task. Of course, if one of the instances will fail for any reason,
the lock will be released for the next available!

![high_level_architecture_diagram](./images/shedlock_high_level.png)

***Shedlock*** needs just another `annotation` on top of your scheduled task as follows in my example:
```java
@Scheduled(fixedDelay = "5s")
@SchedulerLock(name = "say-hello-scheduler")
void schedule() {
    assertLocked();
    log.info("Hello from {}", App.class.getName());
}
```
along with some extra configuration in `application.yml`
```yaml
shedlock:
  defaults:
    lock-at-most-for: 8s
    lock-at-least-for: 5s
```
Here's the output logs of a demo PoC I did

![proof](./images/proof.png)

That's it! I hope this blog helped...

The repo can be found right here
```html
https://github.com/pagidas/shedlock-poc
```
