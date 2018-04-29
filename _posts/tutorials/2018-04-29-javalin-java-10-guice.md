---
layout: tutorial
title: "Java 10 and Google Guice"
author: <a href="https://www.linkedin.com/in/kristapsvitolins/" target="_blank">Kristaps Vītoliņš</a>
date: 2018-04-29
permalink: /tutorials/javalin-java-10-google-guice
github: https://github.com/alzuma/javalin-java-10-guice.git
summarytitle: Java 10 and Google Guice
summary: Learn how to create Javalin application with Google Guice
language: java
---

## What You Will Learn
In this tutorial we will learn how to create modular application. \\
\\
We will use [Google Guice](https://github.com/google/guice/wiki/Motivation) to enable modularity 
and [Java 10](http://www.oracle.com/technetwork/java/javase/downloads/jdk10-downloads-4416644.html) to do

~~~java
var amazingFramework = "Javalin";
//vs
String amazingFramework = "Javalin";
~~~

## Dependencies

Lets create a Maven project with our dependencies [(→ Tutorial)](/tutorials/maven-setup).\\
We will be using Javalin for our web-server, slf4j for logging, jackson to render respons in JSON and Guice for dependency injection:

```xml
<dependencies>
	<dependency>
		<groupId>io.javalin</groupId>
		<artifactId>javalin</artifactId>
		<version>1.6.0</version>
	</dependency>
	<dependency>
		<groupId>org.slf4j</groupId>
		<artifactId>slf4j-simple</artifactId>
		<version>1.7.13</version>
	</dependency>
	<dependency>
		<groupId>com.google.inject</groupId>
		<artifactId>guice</artifactId>
		<version>4.2.0</version>
	</dependency>
	<dependency>
		<groupId>com.google.inject.extensions</groupId>
		<artifactId>guice-multibindings</artifactId>
		<version>4.2.0</version>
	</dependency>
	<dependency>
		<groupId>com.fasterxml.jackson.core</groupId>
		<artifactId>jackson-databind</artifactId>
		<version>2.9.5</version>
	</dependency>
</dependencies>
```

And add properties for Java 10

```xml
<properties>
	<maven.compiler.source>10</maven.compiler.source>
	<maven.compiler.target>10</maven.compiler.target>
</properties>
```

## High level architecture

* Controller
	* Responsible for request handling. I always see it as bouncer or face control if you wish, nothing more
* Service
	* Actual business logic executor, may or may not require other services to done its job
* Repository
	* Communication with any data storage nothing more

## The Java application


First, lets create a controller in `io.kidbank.user` package.

`UserController` is responsible for handling the request, business logic we implement inside `UserService`. As the method name 
allready states, it returns all user names in uppercase. How that is done, it is up to the service.

~~~java
package io.kidbank.user;

import io.javalin.Context;
import io.kidbank.user.services.UserService;

import javax.inject.Inject;
import javax.inject.Singleton;

@Singleton
class UserController {
    private UserService userService;

    @Inject
    public UserController(UserService userService) {
        this.userService = userService;
    }

    public void index (Context ctx) {
        ctx.json(userService.getAllUsersUppercase());
    }
}
~~~

Now that we have controller, we should bind endpoints to `UserController`. There is `Routing` class 
which helps us to get the `UserController` from Google Guice. And it guarantees that there is method 
`bindRoutes()`, which we will use later on.

~~~java
package io.kidbank.user;

import io.alzuma.Routing;
import io.javalin.Javalin;

import javax.inject.Inject;
import javax.inject.Singleton;

import static io.javalin.ApiBuilder.get;
import static io.javalin.ApiBuilder.path;

@Singleton
class UserRouting extends Routing<UserController> {
    private Javalin javalin;
    @Inject
    public UserRouting(Javalin javalin) {
        this.javalin = javalin;
    }

    @Override
    public void bindRoutes() {
        javalin.routes(() -> {
            path("api/kidbank/users", () -> {
                get(ctx -> getController().index(ctx));
            });
        });
    }
}
~~~

Install and bind all dependencies for `io.kidbank.user` package. To enable more than one `Routing`, we use Google Guice extension `Multibinder`.

~~~java
package io.kidbank.user;

import com.google.inject.AbstractModule;
import com.google.inject.multibindings.Multibinder;
import io.alzuma.Routing;
import io.kidbank.user.repositories.UserRepositoryModule;
import io.kidbank.user.services.UserServiceModule;

public class UserModule extends AbstractModule {
    @Override
    protected void configure() {
        bind(UserController.class);
        install(new UserServiceModule());
        install(new UserRepositoryModule());
        Multibinder.newSetBinder(binder(), Routing.class).addBinding().to(UserRouting.class);
    }
}
~~~

Connect `Javalin` with routes and strart the web-server.

~~~java
package io.kidbank;

import com.google.inject.Inject;
import io.alzuma.AppEntrypoint;
import io.alzuma.Routing;
import io.javalin.Javalin;

import javax.inject.Singleton;
import java.util.Collections;
import java.util.Set;

@Singleton
class WebEntrypoint implements AppEntrypoint {
    private Javalin app;

    @Inject(optional = true)
    private Set<Routing> routes = Collections.emptySet();

    @Inject
    public WebEntrypoint(Javalin app) {
        this.app = app;
    }

    @Override
    public void boot(String[] args) {
        bindRoutes();
        app.port(7000);
        app.start();
    }

    private void bindRoutes() {
        routes.forEach(r -> r.bindRoutes());
    }
}
~~~ 

Create `WebModule` for our `Kid bank` project. Inside module define that project, can be run as web-server. 
Similar trick we did with `Routing`,but instead of `Multibinder` we use a `MapBinder`. So that we can store 
"Run As" into `HashMap<EntrypointType, AppEntrypoint>`

~~~java
package io.kidbank;

import com.google.inject.AbstractModule;
import com.google.inject.multibindings.MapBinder;
import io.alzuma.AppEntrypoint;
import io.alzuma.EntrypointType;
import io.javalin.Javalin;
import org.jetbrains.annotations.NotNull;

class WebModule extends AbstractModule {
    private Javalin app;

    private WebModule(Javalin app) {
        this.app = app;
    }

    @NotNull
    public static WebModule create() {
        return new WebModule(Javalin.create());
    }

    @Override
    protected void configure() {
        bind(Javalin.class).toInstance(app);
        MapBinder.newMapBinder(binder(), EntrypointType.class, AppEntrypoint.class).addBinding(EntrypointType.REST).to(WebEntrypoint.class);
    }
}
~~~

Create `Startup` which will serve as entrypoint resolver. Entrypoints are injected by Guice, rember `WebModule` where we used `MapBinder` that is why  
there is something to inject.

~~~java
import com.google.inject.Inject;
import io.alzuma.AppEntrypoint;
import io.alzuma.EntrypointType;

import javax.inject.Singleton;
import java.util.Collections;
import java.util.Map;
import java.util.Optional;

@Singleton
public class Startup {
    @Inject(optional = true)
    private Map<EntrypointType, AppEntrypoint> entrypoints = Collections.emptyMap();

    public void boot(EntrypointType entrypointType, String[] args) {
        var entryPoint = Optional.ofNullable(entrypoints.get(entrypointType));
        entryPoint.orElseThrow(() -> new RuntimeException("Entrypoint not defined")).boot(args);
    }
}
~~~

As for the last module we define `AppModule`

~~~java
import com.google.inject.AbstractModule;
import io.kidbank.KidBankModule;

public class AppModule extends AbstractModule {
    protected void configure() {
        bind(Startup.class);
        install(new KidBankModule());
    }
}
~~~

Now we are ready to start our web-server. Create injector from `AppModule` which will trigger all binding and installations 
down the path. And resolve `Startup` and boot the REST with Javalin.

~~~java
public class App {
    public static void main(String[] args) {
        var injector = Guice.createInjector(new AppModule());
        injector.getInstance(Startup.class).boot(EntrypointType.REST, args);
    }
}
~~~

Open in browser `http://localhost:7000/api/kidbank/users` and wait for response `["BOB","KATE","JOHN"]`

## Conclusion
We created modular web application, with capabilities to run it self not only as a web-server. It takes 
time to get use to Guice modules, but once you get the idea the sky is your limit!

Most important part. Have fun!
