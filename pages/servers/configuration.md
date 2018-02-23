---
title: Configuration
caption: Configuring the Server   
section: Servers
permalink: /servers/configuration.html
---

Ktor uses [HOCON (Human-Optimized Config Object Notation)](https://github.com/lightbend/config/blob/master/HOCON.md)
format as external configuration file.
This format is similar to JSON, but it is optimized to be read and written by humans,
and supports additional features like environment variable substitution.

Ktor also uses a set of lambdas with a typed DSL (Domain Specific Language) to configure the application,
and to configure the server engine when using `embeddedServer`.

**Table of contents:**

* TOC
{:toc}

## The HOCON file
{: #hocon-file}

Ktor requires you to specify which module or modules does it have to load
when starting the server.

When using HOCON, you define the initial modules with `ktor.application.modules`.

Modules are just functions that receive an `Application` and configure the server:
registering routes, handling requests...
{: .note}

A typical, simple HOCON file for Ktor would look like this:

```kotlin
ktor {
    deployment {
        port = 8080
    }

    application {
        modules = [ io.ktor.samples.metrics.MetricsApplicationKt.main ]
    }
}
```

Using dot notation it would be equivalent to:

```kotlin
ktor.deployment.port = 8080
ktor.application.modules = [ io.ktor.samples.metrics.MetricsApplicationKt.main ]
```

Ktor allows you to configure much more: from additional core configurations,
to Ktor features and even custom configurations for your applications:

```kotlin
ktor {
    deployment {
        environment = development
        port = 8080
        sslPort = 8443
        autoreload = true
        watch = [ httpbin ]
    }

    application {
        modules = [ io.ktor.samples.httpbin.HttpBinApplicationKt.main ]
    }

    security {
        ssl {
            keyStore = build/temporary.jks
            keyAlias = mykey
            keyStorePassword = changeit
            privateKeyPassword = changeit
        }
    }
}

jwt {
    domain = "https://jwt-provider-domain/"
    audience = "jwt-audience"
    realm = "ktor sample app"
}

youkube {
  session {
    cookie {
      key = 03e156f6058a13813816065
    }
  }
  upload {
    dir = ktor-samples/ktor-samples-youkube/.video
  }
}
```
{: .compact}

There is a [list of available core configurations](#available-config) in this document.

## Command Line
{: #command-line}

When using [`commandLineEnvironment`](https://github.com/ktorio/ktor/blob/master/ktor-server/ktor-server-host-common/src/io/ktor/server/engine/CommandLine.kt)
(any `DevelopmentEngine` main) there are several switches and configuration parameters you can use to configure
your application module.

Using switches, you can, for example, change the binded port by:

`java -jar myapp-fatjar.jar -port=8080`

There is a [list of available command line switches](#available-config) in this document.

## Configuring the embeddedServer
{: #embedded-server}

`embeddedServer` includes an optional parameter `configure` that allows you to set the configuration for the
engine specified in the first parameter.
Independently to the engine used, you will have some available properties to configure:

```kotlin
embeddedServer(AnyEngine, configure = {
    // Size of the event group for accepting connections
    connectionGroupSize = parallelism / 2 + 1
    // Size of the event group for processing connections,
    // parsing messages and doing engine's internal work 
    workerGroupSize = parallelism / 2 + 1
    // Size of the event group for running application code 
    callGroupSize = parallelism 
}) {
    // ...
}.start(true)
```

### Netty
{:.no_toc}

When using Netty as engine, in addition to common properties, you can configure some other properties:

```kotlin
embeddedServer(Netty, configure = {
    // Size of the queue to store [ApplicationCall] instances that cannot be immediately processed
    requestQueueLimit = 16 
    // Do not create separate call event group and reuse worker group for processing calls
    shareWorkGroup = false 
    // User-provided function to configure Netty's [ServerBootstrap]
    configureBootstrap = {
        // ...
    } 
    // Timeout in seconds for sending responses to client
    responseWriteTimeoutSeconds = 10 
}) {
    // ...
}.start(true)
```

### Jetty
{:.no_toc}

When using Jetty as engine, in addition to common properties, you can configure the Jetty server.

```kotlin
embeddedServer(Jetty, configure = {
    // Property to provide a lambda that will be called during Jetty
    // server initialization with the server instance as argument.
    configureServer = {
        // ...
    } 
}) {
    // ...
}.start(true)
```

### CIO
{:.no_toc}

When using CIO (Coroutine I/O) as engine, in addition to common properties, you can configure the `connectionIdleTimeoutSeconds` property.

```kotlin
embeddedServer(CIO, configure = {
    // Number of seconds that the server will keep HTTP IDLE connections open.
    // A connection is IDLE if there are no active requests running.
    connectionIdleTimeoutSeconds = 45
}) {
    // ...
}.start(true)
```

### Tomcat
{:.no_toc}

When using Tomcat, in addition to common properties, you can configure the Tomcat server.

```kotlin
embeddedServer(Tomcat, configure = {
    // Property to provide a lambda that will be called during Tomcat
    // server initialization with the server instance as argument.
    configureTomcat { // this: Tomcat ->
        // ...
    }
}) {
    // ...
}.start(true)
```

## Available configuration parameters
{: #available-config}

**Switch** refers to arguments that you pass to the application, so you can, for example, change the binded port by:

`java -jar myapp-fatjar.jar -port=8080`

**Parameter paths** are paths inside the HOCON `application.conf` file:

```
ktor.deployment.port = 8080
```

```
ktor {
    deployment {
        port = 8080
    }
}
```

General switches and parameters:

| Switch          | Parameter path                         | Default               | Description |
|-----------------|:---------------------------------------|:----------------------|:------------|
| `-jar=`         |                                        |                       | Path to JAR file |
| `-config=`      |                                        |                       | Path to config file (instead of `application.conf` from resources) |
| `-host=`        | `ktor.deployment.host`                 | `0.0.0.0`             | Binded host |
| `-port=`        | `ktor.deployment.port`                 | `80`                  | Binded port |
| `-watch=`       | `ktor.deployment.watch`                | `[]`                  | Package paths to watch for reloading |
|                 | `ktor.application.id`                  | `Application`         | Application Identifier used for logging |
|                 | `ktor.deployment.callGroupSize`        | `parallelism`         | Event group size running application code |
|                 | `ktor.deployment.connectionGroupSize`  | `parallelism / 2 + 1` | Event group size accepting connections |
|                 | `ktor.deployment.workerGroupSize`      | `parallelism / 2 + 1` | Event group size for processing connections, parsing messages and doing engine's internal work |
{: .styled-table}

Required when SSL port is defined:

| Switch          | Parameter path                         | Default          | Description |
|-----------------|:---------------------------------------|:-----------------|:------------|
| `-sslPort=`     | `ktor.deployment.sslPort`              | `null`           | SSL port    |
| `-sslKeyStore=` | `ktor.security.ssl.keyStore`           | `null`           | SSL key store |
|                 | `ktor.security.ssl.keyAlias`           | `mykey`          | Alias for the SSL key store |
|                 | `ktor.security.ssl.keyStorePassword`   | `null`           | Password for the SSL key store |
|                 | `ktor.security.ssl.privateKeyPassword` | `null`           | Password for the SSL private key |
{: .styled-table}

You can use `-P:` to specify parameters that don't have a specific switch. For example:
`-P:ktor.deployment.callGroupSize=7`.

## Reading the configuration from code
{: #accessing-config}

If you are using a `DevelopmentEngine` instead of an `embeddedServer`, the HOCON file is loaded,
and you are able to access to its configuration properties.

You can also define arbitrary property paths to configure your application.

```kotlin
val port: String = application.environment.config
    .propertyOrNull("ktor.deployment.port")?.getString()
    ?: "80"
```

It is possible to access the HOCON `application.conf` configuration too, by using a custom main with commandLineEnvironment:

```kotlin
embeddedServer(Netty, commandLineEnvironment(args + arrayOf("-port=8080"))).start(true)
```

Or by redirecting it to the specific `DevelopmentEngine.main`:

```kotlin
val moduleName = Application::module.javaMethod!!.let { "${it.declaringClass.name}.${it.name}" }
io.ktor.server.netty.main(args + arrayOf("-port=8080", "-PL:ktor.application.modules=$moduleName"))
```

Or with a custom `applicationEngineEnvironment`:

```kotlin
embeddedServer(Netty, applicationEngineEnvironment {
    log = LoggerFactory.getLogger("ktor.application")
    config = HoconApplicationConfig(ConfigFactory.load()) // Provide a Hocon config file

    module {
        routing {
            get("/") {
                call.respondText("HELLO")
            }
        }
    }

    connector {
        port = 8080
        host = "127.0.0.1"
    }
}).start(true)
```

You can also access the configuration properties by manually loading the default config file `application.conf`:

```kotlin
val config = HoconApplicationConfig(ConfigFactory.load())
``` 

**Note:** Bear in mind that when using `application.environment.config` you are accessing the HOCON document,
so switches from the command line are not included there.
{: .note} 


## Custom configuration systems
{: #custom}

Ktor provides an interface that you can implement the configuration in, available at `application.environment.config`.
You can construct and set the configuration properties inside an `applicationEngineEnvironment`.

```kotlin
interface ApplicationConfig {
    fun property(path: String): ApplicationConfigValue
    fun propertyOrNull(path: String): ApplicationConfigValue?
    fun config(path: String): ApplicationConfig
    fun configList(path: String): List<ApplicationConfig>
}

interface ApplicationConfigValue {
    fun getString(): String
    fun getList(): List<String>
}

class ApplicationConfigurationException(message: String) : Exception(message)
```

Ktor provides two implementations. One based on a map (`MapApplicationConfig`), and other based in HOCON (`HoconApplicationConfig`).

You can create and compose config implementations and set them at `applicationEngineEnvironment`, so it is available to all of the
application components.