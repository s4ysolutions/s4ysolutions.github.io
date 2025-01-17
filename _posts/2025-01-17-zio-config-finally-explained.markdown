---
layout: post
title: "ZIO Config end to end sample"
date: 2025-01-17 00:00:00 +0000
tags: [scala, zio, config]
style: fill
color: secondary
comments: false
---

## Defining the configuration

```scala
package gent.config

import zio.Config
import zio.Config.{int, string}
import zio.config.*

// Describes the (schema, blueprint of the) configuration
case class AgentConfig(serviceHostName: String, servicePort: Int, connectionString: String)
  
object AgentConfig:
  // create the (implicit) instance of the `Config[AgentConfig]` previously called ConfigDescriptor
  // but now more often mentioned as Config, making it confusing with AgentConfig itself
  given Config[AgentConfig] = (
    string("serviceHostName") ?? "Service hostname" zip
      int("servicePort") ?? "Service port" zip
      string("connectionString") ?? "DB2 connection string"
    // pay attention to the `to` here: it converts the tuple to the "Config descriptor" of the `AgentConfig`
    ).to[AgentConfig]
```

## Accessing the configuration

Now the instance of the `AgentConfig` can be got with 

```scala
   ZIO.config[AgentConfig].flatMap { agentConfig =>
     // do something with the agentConfig: AgentConfig
     PrintLine(s"Service hostname: ${agentConfig.serviceHostName}")
   }
```

Where and how the instance of the `AgentConfig` was created is kind of that magic that makes Scala hard to understand
and not a popular language.

## Loading the configuration

The [tutorial](https://zio.dev/guides/tutorials/configurable-zio-application) taunting the mediocre minds mentions the
slurred error message:

```
timestamp=2023-04-01T14:00:13.902065Z level=ERROR thread=#zio-fiber-0 message="" cause="Exception in thread "zio-fiber-4" zio.Config$Error$And: ((((Missing data at HttpServerConfig.host: Expected HTTPSERVERCONFIG_HOST to be set in the environment) or (Missing data at HttpServerConfig.host: Expected HttpServerConfig.host to be set in properties)) and ((Missing data at HttpServerConfig.nThreads: Expected HTTPSERVERCONFIG_NTHREADS to be set in the environment) or (Missing data at HttpServerConfig.nThreads: Expected HttpServerConfig.nThreads to be set in properties))) and ((Missing data at HttpServerConfig.port: Expected HTTPSERVERCONFIG_PORT to be set in the environment) or (Missing data at HttpServerConfig.port: Expected HttpServerConfig.port to be set in properties)))
        at dev.zio.quickstart.MainApp.run(MainApp.scala:35)"
```
and adds another (4th!!) use of the term `configuration` : "we have not provided any **configuration** to the application".

In fact, we already provided class `AgentConfig`, type `Config[AgentConfig]` and implicit instance of `AgentConfig`,
but it is still not enough and this is the point where a clever guy should give up with Scala and save his life
with something invented by less exquisite minds.

But if you insist on spending your precious time, then let's proceed.

The magically created instance of the `AgentConfig` is empty at the moment (obviously) and must be loaded from somewhere.
To do it, another secret entity is involved: `ConfigProvider`.

From the tutorial it is known there's some `current` provider that can be changed with a snippet:

```scala
object MainApp extends ZIOAppDefault {
  override val bootstrap: ZLayer[ZIOAppArgs, Any, Any] =
    Runtime.setConfigProvider(
      ConfigProvider.fromResourcePath()
    )
```

What are `ZIOAppDefault`, `ZIOAppArgs`, `bootstrap`, `ZLayer`, `Runtime`, and (the most curious) why `setConfigProvider`
returns some value(Layer) despite `set` in its name is not assumed to be already known so devote a couple of days least just to
set port 8080 of your http server. 

Actually the full story with `ConfigProvider` seems to be either classified or assumed to be born-in knowledge, but
from the source code 

```scala
def setConfigProvider(configProvider: ConfigProvider)(implicit trace: Trace): ZLayer[Any, Nothing, Unit] =
    ZLayer.scoped(ZIO.withConfigProviderScoped(configProvider))
```

we can see what 1) it returns a `Layer` indeed.
(another term `scoped` appears that cry "Get out of here! You do not need Scala! You could already have written MVP in Kotlin".)
and 2) there something so special in the config so `Runtime` module must have dedicated method for `ZConfig` module
(Definitely the scala developers are not required to know about SOLID principles)


Anyway, looking in the source code of `ConfigProvider` we can find `fromMap` method which is best for moving further
during the prototyping phase:

```scala
object Main extends ZIOAppDefault:
  override val bootstrap: ZLayer[ZIOAppArgs, Any, Any] =
    Runtime.setConfigProvider(
      ConfigProvider.fromMap(Map(
        "serviceHostName" -> "localhost",
        "servicePort" -> "8080",
        "connectionString" -> "jdbc:db2://localhost:50000/sample:user=db2inst1;password=password;",
      ))
    )
```

somewhere in the depth this `ConfigProvider.fromMap`  is assumed to be called like this:

```scala
ConfigProvider.fromMap(Map(
  "serviceHostName" -> "localhost",
  "servicePort" -> "8080",
  "connectionString" -> "jdbc:db2://localhost:50000/sample:user=db2inst1;password=password;",
  )).load(agentConfig))
```

**Advanced, you do not need to know this**:
btw this `.load` return the instance of the `AgentConfig` and it can be used further say be provided with a layer:
```scala
object AgentConfig:
     val layer: ZLayer[Any, Config.Error, AgentConfig] = ZLayer(ConfigProvider.fromMap(Map(
        "serviceHostName" -> "localhost",
        "servicePort" -> "8080"
      )).load(agentConfig))
```
and
```scala
object Main extends ZIOAppDefault:
  override def run: ZIO[Any, Throwable, Unit] =
    val program = for
      config <- ZIO.service[AgentConfig]
  ...
    yield ()

    ZIO.scoped(program)
      .provide(
        AgentConfig.layer,
      ...
```



