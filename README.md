## Building and running the application

Your application is configured to be built with Maven. Every Maven-configured project contains a pom.xml file, which defines the project configuration, dependencies, plug-ins, and so on.

Git clone the project you will be working in:

`git clone https://github.com/openliberty/guide-getting-started.git`

Navigate to the start directory where your pom.xml file is located. Your pom.xml file is configured to include the liberty-maven-plugin, which allows you to install applications into Open Liberty as well as manage the server instances.

`cd guide-getting-started/start`

Install and run the server

`mvn install liberty:run`

The mvn command initiates a Maven build, during which the target directory is created to store all build-related files.

The install argument specifies the Maven install phase. During this phase, the application is built and packaged into a .war file, an Open Liberty server runtime is downloaded and installed into the target/liberty/wlp directory, a server instance is created and configured in the target/liberty/wlp/usr/servers/GettingStartedServer directory, and the application is installed into that server via loose config.

The liberty:run-server argument specifies the Open Liberty run-server goal, which starts an Open Liberty server instance in the foreground.

For more information on the Liberty Maven plug-in, see its GitHub repository.

When the server begins starting up, various messages display in your active shell. Wait for the following message, which indicates that the server startup is complete:

`The server GettingStartedServer is ready to run a smarter planet.`

To access the `system` microservice, access the service endpoint to cause some application measurements to be recorded:

`curl http://localhost:9080/system/properties`
````
{
    "os.name": "Mac OS X",
    "java.version": "1.8.0_151",
    ...
}
````
Later, when you need to stop the server, simply press `CTRL+C` in the shell session where you ran the server. 

Open up a new terminal window.

## Updating the server configuration without restarting the server

We need to export the Java Home JRE

`export JAVA_HOME=$(/usr/lib/jvm/java-8-openjdk-amd64)`

`export PATH=$JAVA_HOME/jre/bin:$PATH`

When you update the server configuration files, you can run the `mvn package` command to invoke the Maven `package` phase that executes various Maven goals that repackage the server.

Try updating the server configuration while the server is running. If you stopped the server, start it again before you proceed. The `system` microservice does not currently include health monitoring to report whether the server and the microservice that it runs are healthy. You can add health reports with the MicroProfile Health feature, which adds a `/health` endpoint to your application. 

If you try to access this endpoint now at the ` curl http://localhost:9080/health` URL, you see a 404 error because the /health endpoint does not yet exist:

`Error 404: java.io.FileNotFoundException: SRVE0190E: File not found: /health`

Open up the `server.xml` file and add the MicroProfile Health feature to the server, include the mpHealth feature in the server.xml.
 
Open `server.xml` file 

`guide-getting-started/start/src/main/liberty/config/server.xml`

Add the mpHealth feature tag between the `<feature manager>` tags:

`<feature>mpHealth-2.0</feature>`

Navigate to the start directory, 

`cd guide-getting-started/start`

and repackage the server:

`mvn package`

When enabled, the `mpHealth` feature automatically adds a `/health` endpoint to the application. You can see the server being updated in the server log that’s displayed in your first shell session:

````
<pre>
[INFO] [AUDIT] CWWKG0016I: Starting server configuration update.
[INFO] [AUDIT] CWWKT0017I: Web application removed (default_host): http://foo:9080/
[INFO] [AUDIT] CWWKZ0009I: The application io.openliberty.guides.getting-started has stopped successfully.
[INFO] [AUDIT] CWWKG0017I: The server configuration was successfully updated in 0.284 seconds.
[INFO] [AUDIT] CWWKT0016I: Web application available (default_host): http://foo:9080/health/
[INFO] [AUDIT] CWWKF0012I: The server installed the following features: [mpHealth-1.0].
[INFO] [AUDIT] CWWKF0008I: Feature update completed in 0.285 seconds.
[INFO] [AUDIT] CWWKT0016I: Web application available (default_host): http://foo:9080/

<pre>
{
    "checks":[],
    "outcome":"UP"
}
</pre>
````
You now have a means of verifying if your server is up and running.

## Updating the source code without restarting the server

The JAX-RS application that contains your `system` microservice is configured as a loose application, meaning that it runs in a server from its `.class` file and other artifacts. Open Liberty automatically monitors these artifacts, and whenever they are updated, it updates the running server without the need for the server to be restarted.

Navigate to the `pom.xml` file under start directory


The loose application support is enabled with the `<looseApplication/>` element in the `liberty-maven-plugin` plug-in.


Try updating the source code while the server is running. At the moment, the `/health` endpoint reports whether or not the server is running, but the endpoint doesn’t provide any details on the microservices that are running inside of the server.


MicroProfile Health offers health checks for both readiness and liveness. A readiness check allows third-party services, such as Kubernetes, to know if the microservice is ready to process requests. A liveness check allows third-party services to determine if the microservice is running.


### Create the SystemReadinessCheck class.

Head to the directory where the SystemReadinessCheck class will be created

`cd src/main/java/io/openliberty/sample/system`

Create SystemReadinessCheck class

`touch SystemReadinessCheck.java`

Open SystemReadinessCheck.java

`guide-getting-started/start/src/main/java/io/openliberty/sample/system/SystemReadinessCheck.java`

Insert code into SystemReadinessCheck class


````
import javax.enterprise.context.ApplicationScoped;

import javax.inject.Inject;
import javax.inject.Provider;

import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.eclipse.microprofile.health.Readiness;
import org.eclipse.microprofile.health.HealthCheck;
import org.eclipse.microprofile.health.HealthCheckResponse;
import org.eclipse.microprofile.health.HealthCheckResponseBuilder;

@Readiness
@ApplicationScoped
public class SystemReadinessCheck implements HealthCheck {

    @Inject
    @ConfigProperty(name = "io_openliberty_guides_system_inMaintenance")
    Provider<String> inMaintenance;

    @Override
    public HealthCheckResponse call() {
        HealthCheckResponseBuilder builder = HealthCheckResponse.named(
                SystemResource.class.getSimpleName() + " readiness check");
        if (inMaintenance != null && inMaintenance.get().equalsIgnoreCase("true")) {
            return builder.withData("services", "not available").down().build();
        }
        return builder.withData("services", "available").up().build();
    }

}

````

The `SystemReadinessCheck` class verifies that the `system` microservice is not in maintenance by checking a config property.

Go to the directory that the `SystemReadinessCheck.java` will be saved

Now we create the SystemLivenessCheck class

Create a new file called `SystemLivenessCheck.java`

`touch SystemLivenessCheck.java`

Open `SystemLivenessCheck.java` and  Insert the following code 

````
package io.openliberty.sample.system;

import javax.enterprise.context.ApplicationScoped;

import java.lang.management.MemoryMXBean;
import java.lang.management.ManagementFactory;

import org.eclipse.microprofile.health.Liveness;
import org.eclipse.microprofile.health.HealthCheck;
import org.eclipse.microprofile.health.HealthCheckResponse;

@Liveness
@ApplicationScoped
public class SystemLivenessCheck implements HealthCheck {

    @Override
    public HealthCheckResponse call() {
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        long memUsed = memBean.getHeapMemoryUsage().getUsed();
        long memMax = memBean.getHeapMemoryUsage().getMax();

        return HealthCheckResponse.named(
            SystemResource.class.getSimpleName() + " liveness check")
                                  .withData("memory used", memUsed)
                                  .withData("memory max", memMax)
                                  .state(memUsed < memMax * 0.9).build();
    }

}
````

The `SystemLivenessCheck` class reports a status of DOWN if the microservice uses over 90% of the maximum amount of memory.

After you make the file changes, Open Liberty automatically reloads its configuration and the system application.
The following messages display in your first shell session:

````
<pre>
[INFO] [AUDIT] CWWKT0017I: Web application removed (default_host): http://foo:9080/
[INFO] [AUDIT] CWWKZ0009I: The application io.openliberty.guides.getting-started has stopped successfully.
[INFO] [AUDIT] CWWKT0016I: Web application available (default_host): http://foo:9080/
[INFO] [AUDIT] CWWKZ0003I: The application io.openliberty.guides.getting-started updated in xx.xx seconds.
</pre>
````

Access the /health endpoint again by visiting the `http://localhost:9080/health` URL. This time you see the overall status of your server as well as the aggregated data of the liveness and readiness checks for the system microservice: 

````
<pre>
{
   "checks":[
      {
         "data":{
            "services":"available"
         },
         "name":"SystemResource readiness check",
         "status":"UP"
      },
      {
         "data":{
            "memory used":81064480,
            "memory max":4294967296
         },
         "name":"SystemResource liveness check",
         "status":"UP"
      }
   ],
   "status":"UP"
}
</pre>
````
You can also access the `/health/ready` endpoint by visiting the `http://localhost:9080/health/ready` URL to view the data from the readiness health check. Similarily, access the /health/live endpoint by visiting the `http://localhost:9080/health/live` URL to view the data from the liveness health check.

## Checking the Open Liberty server logs

Go back to the second shell session

While the server is running in the foreground, it displays various console messages in the shell. 

These messages are also logged to the `target/liberty/wlp/usr/servers/GettingStartedServer/logs/console.log` file. 
You can find the complete server logs in the 

`target/liberty/wlp/usr/servers/GettingStartedServer/logs` directory. The `console.log` and `messages.log` files are the primary log files that contain console output of the running application and the server. More logs are created when run time errors occur or whenever tracing is enabled. You can find the error logs in the `ffdc` directory and the tracing logs in the trace.log file.


In addition to the log files that are generated automatically, you can enable logging of specific Java packages or classes by using the `<logging/>` element:

```
<logging traceSpecification="<component_1>=<level>:<component_2>=<level>:..."/>
```

The `component` element is a Java package or class, and the `level` element is one of the following logging levels: `off`, `fatal`, `severe`, `warning`, `audit`, `info`, `config`, `detail`, `fine`, `finer`, `finest`, `all`.

Try enabling detailed logging of the MicroProfile Health feature by adding the `<logging/>` element to your configuration file.

`src/main/liberty/config/server.xml`

```
<server description="Sample Liberty server">
    <featureManager>
        <feature>jaxrs-2.1</feature>
        <feature>jsonp-1.1</feature>
        <feature>cdi-2.0</feature>
        <feature>mpMetrics-2.0</feature>
        <feature>mpHealth-2.0</feature>
        <feature>mpConfig-1.3</feature>
    </featureManager>

    <applicationManager autoExpand="true" />
    <quickStartSecurity userName="admin" userPassword="adminpwd" />
    <keyStore id="defaultKeyStore" password="mpKeystore" />
    <logging traceSpecification="com.ibm.ws.microprofile.health.*=all" />
    <httpEndpoint host="*" httpPort="${default.http.port}"
        httpsPort="${default.https.port}" id="defaultHttpEndpoint"/>

    <variable name="io_openliberty_guides_system_inMaintenance" value="false"/>

    <webApplication location="getting-started.war" contextRoot="/"/>
</server>
```

Next, repackage the server:

`mvn package`

Now, when you visit the `/health` endpoint, additional traces are logged into the `trace.log` file.

## Starting and stopping the Open Liberty server in the background 

Although you can start and stop the server in the foreground by using the Maven `liberty:run-server` goal, you can also start and stop the server in the background with the Maven `liberty:start-server` and `liberty:stop-server` goals:

`mvn liberty:start-server`

`mvn liberty:stop-server`

## Running the application from a minimal runnable JAR

So far, Open Liberty has been running out of the `target/liberty/wlp` directory, which effectively contains an Open Liberty server installation and the deployed application. The final product of the Maven build is a server package for use in a continuous integration pipeline and, ultimately, a production deployment.


Open Liberty supports a number of different server packages. The sample application currently generates a usr package that contains the servers and application to be extracted onto an Open Liberty installation.


The type of server package is configured with `<packaging.type/>` in the `pom.xml`.

Instead of creating a server package, you can generate a runnable JAR file that contains the application along with a server runtime. This JAR can then be run anywhere and deploy your application and server at the same time. To generate a runnable JAR, invoke the `runnable-package` profile by using the `-P` flag:

`mvn install -P runnable-package`

The `-P` flag specifies the Maven profile to be run during the build. In this case, the `runnable-package` profile is invoked, which temporarily overrides the `packaging.type` property from the usr package to the runnable package. This property then propagates to the liberty-maven-plugin plug-in, which generates the server package that you want.


When the build completes, you can find the runnable `getting-started.jar` file in the `target` directory. By default, this JAR file comes with all the features available in Open Liberty, including the entirety of Java EE and MicroProfile. As a result, this JAR is over 100 MB. To omit the features that you don’t need and package the JAR with only the `features` that you defined in the `server.xml` file, use `minifiy,runnable` as the packaging type. To build a minimal runnable JAR, invoke the `minify-runnable-package` profile by using the `-P` flag:

`mvn install -P minify-runnable-package`

The `minify-runnable-package` profile overrides the `packaging.type` property from the `usr` package to the `minify,runnable` package and generates a runnable JAR file that contains only the `features` that you explicitly enabled in your `server.xml` file. As a result, the generated JAR is only about 50 MB.

To run the JAR, first stop the server if it’s running. Then, navigate to the `target` directory and run the `java -jar` command:

`cd target`

`java -jar getting-started.jar`

When the server starts, visit the http://localhost:9080/system/properties URL to access your application that is now running out of the minimal runnable JAR.

At this point, you can stop the server by pressing `CTRL+C` in the shell session that the server runs in.

## Running the application in a Docker container

To run the application in a container, you need to have Docker installed. For installation instructions, see the Official Docker Docs.

To containerize the application, you need a `Dockerfile`. This file contains a collection of instructions that define how a Docker image is built, what files are packaged into it, what commands run when the image runs as a container, and so on. You can find a complete `Dockerfile` in the `start` directory. This `Dockerfile` packages the `usr` server package into a Docker image that contains a preconfigured Open Liberty server.

Change directories to build the docker containers:

`cd ..`

To build and containerize the application, start your Docker daemon and run the following command:

`docker build -t openliberty-getting-started:1.0-SNAPSHOT .`

The `-t` flag in the `docker build` command allows the Docker image to be labeled (tagged) in the `name[:tag]` format. The tag for an image describes the specific image version. If the optional `[:tag]` tag is not specified, the `latest` tag is created by default. The Docker openliberty-getting-started:1.0-SNAPSHOT image is built from the `Dockerfile`. To verify that the image is built, run the `docker images` command to list all local Docker images:

`docker images`

Your image should appear in the list of all Docker images:

````
REPOSITORY                     TAG             IMAGE ID        CREATED         SIZE
openliberty-getting-started    1.0-SNAPSHOT    85085141269b    21 hours ago    487MB
````


Next, run the image as a container:

`docker run -d --name gettingstarted-app -p 9080:9080 openliberty-getting-started:1.0-SNAPSHOT`{{execute}}

There is a bit going on here, so let’s break down the command:

````
Flag	Description
-d
Runs the container in the background.
--name
Specifies a name for the container.
-p
Maps the container ports to the host ports.
````


The final argument in the docker run command is the Docker image name.
Next, run the docker ps command to verify that your container started:

`docker ps`

````
Make sure that your container is running and does not have Exited as its status:
CONTAINER ID    IMAGE                         CREATED          STATUS           NAMES
4294a6bdf41b    openliberty-getting-started   9 seconds ago    Up 11 seconds    gettingstarted-app
````
To access the application, visit the `http://localhost:9080/system/properties` URL.

To stop and remove the container, run the following commands:

`docker stop gettingstarted-app && docker rm gettingstarted-app`

To remove the image, run the following command:

`docker rmi openliberty-getting-started:1.0-SNAPSHOT`


## Running the application from a minimal runnable JAR

So far, Open Liberty was running out of the target/liberty/wlp directory, which effectively contains an Open Liberty server installation and the 
deployed application. The final product of the Maven build is a server package for use in a continuous integration pipeline and, ultimately, a 
production deployment.
Open Liberty supports a number of different server packages. The sample application currently generates a `usr` package that contains the servers and 
application to be extracted onto an Open Liberty installation.
Instead of creating a server package, you can generate a runnable JAR file that contains the application along with a server runtime. This JAR file 
can then be run anywhere and deploy your application and server at the same time. To generate a runnable JAR file, override the `include` property:

`mvn liberty:package -Dinclude=runnable`


The packaging type is overridden from the `usr` package to the `runnable` package. This property then propagates to the `liberty-maven-plugin` plug-in, 
which generates the server package based on the `openliberty-kernel` package.

When the build completes, you can find the minimal runnable `guide-getting-started.jar` file in the target directory. This JAR file contains only the features 
that you explicitly enabled in your `server.xml` file. As a result, the generated JAR file is only about 50 MB.

To run the JAR file, first stop the server if it’s running. Then, navigate to the target directory and run the java -jar command:

`java -jar guide-getting-started.jar`

When the server starts, go to the http://localhost:9080/system/properties URL to access your application that is now running out of the minimal runnable JAR file.

You can stop the server by pressing `CTRL+C` in the shell session that the server runs in.
