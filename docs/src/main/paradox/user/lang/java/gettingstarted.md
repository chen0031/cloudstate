# Getting started with stateful services in Java

## Prerequisites

Java version
: Cloudstate Java support requires at least Java 8, though we recommend using Java 11, which has better support for running in containers. While it is possible to build a GraalVM native image for Cloudstate Java user functions, at this stage Cloudstate offers no specific assistance or configuration to aid in doing that.

Build tool
: Cloudstate does not require any particular build tool, you can select your own.

protoc
: Since Cloudstate is based on gRPC, you need a protoc compiler to compile gRPC protobuf descriptors. While this can be done by downloading, installing and running protoc manually, most popular build tools have a protoc plugin which will automatically compile protobuf descriptors during your build for you.

docker
: Cloudstate runs in Kubernetes with [Docker], hence you will need Docker to build a container that you can deploy to Kubernetes. Most popular build tools have plugins that assist in building Docker images.

In addition to the above, you will need to install the Cloudstate java support library, which can be done as follows:
    
Maven
: @@@vars
```xml
<depependency>
  <groupId>io.cloudstate</groupId>
  <artifactId>cloudstate-java-support</artifactId>
  <version>$cloudstate.java-support.version$</version>
</depependency>
```
@@@
  
sbt
: @@@vars
```scala
libraryDependencies += "io.cloudstate" % "cloudstate-java-support" % "$cloudstate.java-support.version$"
```
@@@

gradle
: @@@vars
```groovy
compile group: 'io.cloudstate', name: 'cloudstate-java-support', version: '$cloudstate.java-support.version$'
```
@@@

## Maven example

A minimal Maven example pom file, which uses the [Xolstice Maven Protocol Buffers Plugin](https://www.xolstice.org/protobuf-maven-plugin/) and the [Fabric8 Docker Maven Plugin](https://dmp.fabric8.io/), for a shopping cart service, is shown below:

@@@vars
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>shopping-cart</artifactId>
  <version>1.0-SNAPSHOT</version>

  <packaging>jar</packaging>

  <build>
    <extensions>
      <extension>
        <groupId>kr.motd.maven</groupId>
        <artifactId>os-maven-plugin</artifactId>
        <version>1.6.0</version>
      </extension>
    </extensions>
    <plugins>
      
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>
      
      <plugin>
        <groupId>org.xolstice.maven.plugins</groupId>
        <artifactId>protobuf-maven-plugin</artifactId>
        <version>0.6.1</version>
        <configuration>
          <protocArtifact>com.google.protobuf:protoc:3.9.1:exe:${os.detected.classifier}</protocArtifact>
        </configuration>
        <executions>
          <execution>
            <goals>
              <goal>compile</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
      
      <plugin>
        <groupId>io.fabric8</groupId>
        <artifactId>docker-maven-plugin</artifactId>
        <version>0.26.1</version>
        <configuration>
          <images>
            <image>
              <build>
                <name>my-dockerhub-username/my-stateful-service:%l</name>
                <from>adoptopenjdk/11-jre-hotspot</from>
                <tags>
                  <tag>latest</tag>
                </tags>
                <assembly>
                  <descriptorRef>artifact-with-dependencies</descriptorRef>
                </assembly>
                <entryPoint>
                  <arg>java</arg>
                  <arg>-cp</arg>
                  <arg>/maven/*</arg>
                  <arg>com.example.ShoppingCartMain</arg>
                </entryPoint>
              </build>
            </image>
          </images>
        </configuration>
        <executions>
          <execution>
            <id>build-docker-image</id>
            <phase>package</phase>
            <goals>
              <goal>build</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
      
    </plugins>
  </build>

  <dependencies>
    <dependency>
      <groupId>io.cloudstate</groupId>
      <artifactId>cloudstate-java-support</artifactId>
      <version>$cloudstate.java-support.version$</version>
    </dependency>
  </dependencies>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
</project>
```
@@@

Subsequent source locations and build commands will assume the above Maven project, and may need to be adapted to your particular build tool and setup.

## Protobuf files

The Xolstice Maven plugin assumes a location of `src/main/proto` for your protobuf files. In addition, it includes any protobuf files from your Java dependencies in the protoc include path, so there's nothing you need to do to pull in either the Cloudstate protobuf types, or any of the Google standard protobuf types, they are all automatically available for import.

So, if you were to build the example shopping cart application shown earlier in @ref:[gRPC descriptors](../../features/grpc.md), you could simply paste that protobuf into `src/main/proto/shoppingcart.proto`. You may wish to also define the Java package, to ensure the package name used conforms to Java package naming conventions:

```proto
option java_package = "com.example";
```

Now if you run `mvn compile`, you'll find your generated protobuf files in `target/generated-sources/protobuf/java`.

## Creating a main class

Your main class will be responsible for creating the Cloudstate gRPC server, registering the entities for it to serve, and starting it. To do this, you can use the @javadoc[`CloudState`](io.cloudstate.javasupport.CloudState) server builder, for example:

@@snip [ShoppingCartMain.java](/docs/src/test/java/docs/user/gettingstarted/ShoppingCartMain.java) { #shopping-cart-main }

We will see more details on registering entities in the coming pages.

## Parameter injection

Cloudstate entities work by annotating classes and methods to be instantiated and invoked by the Cloudstate server. The methods and constructors invoked by the server can be injected with parameters of various types from the context of the invocation. For example, an `@CommandHandler` annotated method may take an argument for the message type for that gRPC service call, in addition it may take a `CommandContext` parameter.

Exactly which context parameters are available depend on the type of entity and the type of handler, in subsequent pages we'll detail which parameters are available in which circumstances. The order of the parameters in the method signature can be anything, parameters are matched by type and sometimes by annotation. The following context parameters are available in every context:

| Type                                                   | Annotation            | Description           |
|--------------------------------------------------------|-----------------------|-----------------------|
| @javadoc[`Context`](io.cloudstate.javasupport.Context) |                       | The super type of all Cloudstate contexts. Every invoker makes a subtype of this available for injection, and method or constructor may accept that sub type, or any super type of that subtype that is a subtype of `Context`. |
| `java.lang.String` | @javadoc[`@EntityId`](io.cloudstate.javasupport.EntityId) | The ID of the entity. |

