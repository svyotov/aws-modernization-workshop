# Modern Container Application

**Expected Outcome:**

- 100 level install of Docker Tooling.

- Build the Pet Store application within a Container.

- Deploy the Pet Store application to a Wildfly based Application
    Server.

**Lab Requirements:** Cloud9 IDE

**Average Lab Time:** 30-45 minutes

## Introduction

In this module, we will start exploring our **Modern Container
Application**, the **Pet Store** application.

The Pet Store application is a Java 7 based application that runs on top
of the Wildfly (JBoss) application server. The application is built
using Maven. We will walk through the steps to build the application
using a Maven container then deploy our WAR file to the Wildfly
container. We will be using multi-stage builds to facilitate creating a
minimal docker container.

## Building the Dockerfile

![Docker Image](../../images/docker-image.png)

### Step 1

To get started, make sure you are using a new `terminal` window and
switch to the `containerize-application` folder within this repository.

    cd ~/environment/aws-modernization-workshop/modules/containerize-application/

### Step 2

Double-click to open the `Dockerfile` in the Cloud9 editor, using the
**Environment** navigation pane, as shows below:

![Dockerfile](../../images/dockerfile-nav.png)

## Reviewing the Dockerfile

![Dockerfile layers](../../images/dockerfile-layers.png)

The first `stage` of our `Dockerfile` will be to build the application.
We will use the `maven:3.5-jdk-7` container from **Docker Hub** as our
base image. The first `stage` of our `Dockerfile` will need to copy the
code into the container, install the dependencies, then build the
application using Maven.

The second `stage` of our `Dockerfile` will be to build our Wildfly
application server. We will need to copy the WAR file we created in the
first stage to the second stage. The WAR file is built to
`/usr/src/app/target/applicationPetstore.war` in the first stage. We
will instruct `docker` to copy that WAR file to the
`/opt/jboss/wildfly/standalone/deployments/applicationPetstore.war` in
the new `stage`. We will also have `docker` copy the `standalone.xml`
file to the proper location
`/opt/jboss/wildfly/standalone/configuration/standalone.xml`. Wildfly
will deploy that application on boot.

The jPetstore application requires the PostrgeSQL driver. We install
that during image build by declaring a `RUN` statement in the
`Dockerfile` like the one below.

    # install postgresql support
    RUN mkdir -p $JBOSS_HOME/modules/system/layers/base/org/postgresql/main
    COPY ./postgresql $JBOSS_HOME/modules/system/layers/base/org/postgresql/main
    RUN /bin/sh -c '$JBOSS_HOME/bin/standalone.sh &' \
      && sleep 10 \
      && $JBOSS_HOME/bin/jboss-cli.sh --connect --command="/subsystem=datasources/jdbc-driver=postgresql:add(driver-name=postgresql,driver-module-name=org.postgresql, driver-class-name=org.postgresql.Driver)" \
      && $JBOSS_HOME/bin/jboss-cli.sh --connect --command=:shutdown \
      && rm -rf $JBOSS_HOME/standalone/configuration/standalone_xml_history/ \
      && rm -rf $JBOSS_HOME/standalone/log/*

Let’s now review the full `Dockerfile` step by step and walk through
the result. The first stage will be based off our `build tools`
container, in this case that is maven. This container could be a
container within your organization that houses all the tools your
development team needs. We alias this first stage as the `build` stage.

    FROM maven:3.5-jdk-7 AS build

Next we are going to install the dependencies. We do this ahead of time
so that we can leverage the Docker build cache to increase the speed of
build times. To accomplish this we copy the `pom.xml` and install the
dependencies. We then copy our application code and build the
application. This enables developers to iterate on their code without
having to install dependencies each time.

    # set the working directory
    WORKDIR /usr/src/app

    # copy just the pom.xml
    COPY ./app/pom.xml /usr/src/app/pom.xml

    # just install the dependencies for caching
    RUN mvn dependency:go-offline

Finally, we build the application using `maven`.

    # copy the application code
    COPY ./app /usr/src/app

    # package the application
    RUN mvn package -Dmaven.test.skip=true

Next, in the same `Dockerfile` we create a new stage called
`application.` Then from the `build` stage, we copy the WAR file we
compiled to the Wildfly deployments folder. Wildfly will deploy this WAR
on boot. We then expose port `8080` for our application and port `9990`
for Wildfly management. We then set up the prerequisites for the
`ENTRYPOINT` script to start the application.

    # create our Wildfly based application server
    FROM jboss/wildfly:11.0.0.Final AS application

    # install postgresql support
    RUN mkdir -p $JBOSS_HOME/modules/system/layers/base/org/postgresql/main
    COPY ./postgresql $JBOSS_HOME/modules/system/layers/base/org/postgresql/main
    RUN /bin/sh -c '$JBOSS_HOME/bin/standalone.sh &' \
      && sleep 10 \
      && $JBOSS_HOME/bin/jboss-cli.sh --connect --command="/subsystem=datasources/jdbc-driver=postgresql:add(driver-name=postgresql,driver-module-name=org.postgresql, driver-class-name=org.postgresql.Driver)" \
      && $JBOSS_HOME/bin/jboss-cli.sh --connect --command=:shutdown \
      && rm -rf $JBOSS_HOME/standalone/configuration/standalone_xml_history/ \
      && rm -rf $JBOSS_HOME/standalone/log/*

    # copy war file from build layer to application layer
    COPY --from=build /usr/src/app/target/applicationPetstore.war /opt/jboss/wildfly/standalone/deployments/applicationPetstore.war

    # install nc for entrypoint script and copy the entrypoint script
    USER root
    RUN yum install nc -y
    USER jboss
    COPY ./docker-entrypoint.sh /opt/jboss/docker-entrypoint.sh

    # copy our configuration
    COPY ./standalone.xml /opt/jboss/wildfly/standalone/configuration/standalone.xml

    # expose the application port and the management port
    EXPOSE 8080 9990

    # run the application
    ENTRYPOINT [ "/opt/jboss/wildfly/bin/standalone.sh" ]
    CMD [ "-b", "0.0.0.0", "-bmanagement", "0.0.0.0" ]

## Defining the Application

For this workshop we use the tool `docker-compose` to simulate a
full-stack development environment that consists of multiple containers
communicating with each other.

### Step 1

Download the docker-compose binary by using the `terminal`.

    sudo curl -kLo ~/bin/docker-compose https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m) && \
    sudo chmod +x ~/bin/docker-compose && \
    exec $SHELL

Next, we need to define how our application will run. We do this by
defining the structure of our application and it’s dependencies in a
`docker-compose.yml` file. This file contains the complete environment
required for our application.

### Step 2

Open the file called `docker-compose.yml`. Let’s review the contents. To
start your file should look like this:

    version: '3.4'

    services:

The next section defines the PostgreSQL service. The PostgreSQL service
will run our database. We will use the official PostgreSQL image
available from **Docker Hub**. Next, we will map the PostgreSQL port
`5432` to the machine port for easy access.

Finally, we will define a few environment variables to configure our
instance.

    version: '3.4'

    services:

      postgres:
        image: postgres:9.6
        ports:
          - 5432:5432
        environment:
          - 'POSTGRES_DB=petstore'
          - 'POSTGRES_USER=admin'
          - 'POSTGRES_PASSWORD=password'

### Step 3

Now we will define our Pet Store application. Docker Compose supports
building containers as well, so we will use a special syntax for
defining this container. In our `yaml` file we will create a new service
called `petstore` and configure our build configuration. Next, will add
a `depends_on` config so that the `petstore` container boots after our
`postgres` container. Similar to our `postgres` ports we will map port
`8080` to our machine for easy access. Now we will use some environment
variables to configure our database with the application.

      petstore:
        build: ./
        depends_on:
          - postgres
        ports:
          - 8080:8080
        environment:
          - 'DB_URL=jdbc:postgresql://postgres:5432/petstore?ApplicationName=applicationPetstore'
          - 'DB_HOST=postgres'
          - 'DB_PORT=5432'
          - 'DB_NAME=petstore'
          - 'DB_USER=admin'
          - 'DB_PASS=password'

## Running the Application

To run the application, we will execute the following Docker Compose
commands from the `terminal`.

> **Note**
>
> Make sure the ‘terminal\` sessions’ current working directory is
> `~/environment/aws-modernization-workshop/modules/containerize-application/`
> working directory.

### Step 1

Run the database container in the background (`-d` or daemon flag). We
don’t need the database logs to clog our application logs.

    docker-compose up -d postgres

Example output:

    Creating network "containerize-application_default" with the default driver
    Pulling postgres (postgres:9.6)...
    9.6: Pulling from library/postgres
    743f2d6c1f65: Pull complete
    5d307000f290: Pull complete
    29837b5e9b78: Pull complete
    3090df574038: Pull complete
    dc0b4463fa0e: Pull complete
    1fb834895f59: Pull complete
    59169bd605be: Pull complete
    a950d631bfe9: Pull complete
    de13bddd861e: Pull complete
    79d927ac55bb: Pull complete
    cd90504b6086: Pull complete
    1817e506cb08: Pull complete
    17ea2bd116a5: Pull complete
    d2d177a7b6ae: Pull complete
    Digest: sha256:97fcdcff5106e995661864bebf1fd6881553471b88e2afd6f98fbcb775bf66b7
    Status: Downloaded newer image for postgres:9.6
    Creating containerize-application_postgres_1 ... done

### Step 2

Build out petstore application.

    docker-compose build petstore

Example output (*redacted for brevity*):

    Building petstore
    Step 1/19 : FROM maven:3.5-jdk-7 AS build
    3.5-jdk-7: Pulling from library/maven
    61be48634cb9: Pull complete
    fa696905a590: Pull complete
    b6dd2322bbef: Pull complete
    29bf78e897aa: Pull complete
    bb3e0783f7ce: Pull complete
    d642aa9d6e20: Pull complete
    f276ed06c956: Pull complete
    453e99a1d4cd: Pull complete
    a611c8ef8d0a: Pull complete
    fb5daf008876: Pull complete
    Digest: sha256:566898e199b1a9038b74786ac6ad740f3e6006d276c81cce8f32fcfe7d84912f
    Status: Downloaded newer image for maven:3.5-jdk-7
     ---> 5f03adaf2bbf
    Step 2/19 : WORKDIR /usr/src/app
     ---> Running in 7e4ee451b8b6

    ...

    Step 18/19 : ENTRYPOINT [ "/opt/jboss/docker-entrypoint.sh" ]
     ---> Running in 8a21e4479ffc
    Removing intermediate container 8a21e4479ffc
     ---> 9771a00b8677
    Step 19/19 : CMD [ "-b", "0.0.0.0", "-bmanagement", "0.0.0.0" ]
     ---> Running in a6f2156d6e58
    Removing intermediate container a6f2156d6e58
     ---> ac38f026e2e0

    Successfully built ac38f026e2e0
    Successfully tagged containerize-application_petstore:latest

### Step 3

Run the application container in the foreground and live stream the logs
to `stdout`. If you get an error, press `[Ctrl + C]`, make the necessary
updates to the Dockerfile and re-build the container by re-running the
**step 2** command.

    docker-compose up petstore

Example output:

    containerize-application_postgres_1 is up-to-date
    Creating containerize-application_petstore_1 ... done
    Attaching to containerize-application_petstore_1
    petstore_1  | PostgreSQL server postgres is ready on 5432 - starting wildfly /opt/jboss/wildfly/bin/standalone.sh
    petstore_1  | =========================================================================
    petstore_1  |
    petstore_1  |   JBoss Bootstrap Environment
    petstore_1  |
    petstore_1  |   JBOSS_HOME: /opt/jboss/wildfly
    petstore_1  |
    petstore_1  |   JAVA: /usr/lib/jvm/java/bin/java
    petstore_1  |
    petstore_1  |   JAVA_OPTS:  -server -Xms64m -Xmx512m -XX:MetaspaceSize=96M -XX:MaxMetaspaceSize=256m -Djava.net.preferIPv4Stack=true -Djboss.modules.system.pkgs=org.jboss.byteman -Djava.awt.headless=true
    petstore_1  |
    petstore_1  | =========================================================================
    petstore_1  |
    petstore_1  | 18:53:35,766 INFO  [org.jboss.modules] (main) JBoss Modules version 1.6.1.Final
    petstore_1  | 18:53:36,268 INFO  [org.jboss.msc] (main) JBoss MSC version 1.2.7.SP1
    petstore_1  | 18:53:36,430 INFO  [org.jboss.as] (MSC service thread 1-2) WFLYSRV0049: WildFly Full 11.0.0.Final (WildFly Core 3.0.8.Final) starting
    petstore_1  | 18:53:36,510 INFO  [org.jboss.vfs] (MSC service thread 1-2) VFS000002: Failed to clean existing content for temp file provider of type temp. Enable DEBUG level log to find what caused this
    petstore_1  | 18:53:39,056 INFO  [org.jboss.as.controller.management-deprecated] (Controller Boot Thread) WFLYCTL0028: Attribute 'security-realm' in the resource at address '/core-service=management/management-interface=http-interface' is deprecated, and may be removed in future version. See the attribute description in the output of the read-resource-description operation to learn more about the deprecation.
    petstore_1  | 18:53:39,109 INFO  [org.wildfly.security] (ServerService Thread Pool -- 15) ELY00001: WildFly Elytron version 1.1.6.Final
    petstore_1  | 18:53:39,146 INFO  [org.jboss.as.controller.management-deprecated] (ServerService Thread Pool -- 27) WFLYCTL0028: Attribute 'security-realm' in the resource at address '/subsystem=undertow/server=default-server/https-listener=https' is deprecated, and may be removed in future version. See the attribute description in the output of the read-resource-description operation to learn more about the deprecation.
    petstore_1  | 18:53:39,483 INFO  [org.jboss.as.repository] (ServerService Thread Pool -- 14) WFLYDR0001: Content added at location /opt/jboss/wildfly/standalone/data/content/bb/fe95ab2a2cc839ea70e23db45f1015bfd87a43/content
    petstore_1  | 18:53:39,522 INFO  [org.jboss.as.server] (Controller Boot Thread) WFLYSRV0039: Creating http management service using socket-binding (management-http)
    petstore_1  | 18:53:39,563 INFO  [org.xnio] (MSC service thread 1-2) XNIO version 3.5.4.Final
    petstore_1  | 18:53:39,588 INFO  [org.xnio.nio] (MSC service thread 1-2) XNIO NIO Implementation Version 3.5.4.Final

### Step 4

To preview the application, you will need to click **Preview** from the
top menu of the Cloud9 environment, then **Preview Running
Application**. This will open a new window and pre-populate the full URL
to your preview domain.

![preview](../../images/preview.png)

Now that we have confirmed that the container is functioning, press
`[Ctrl + c]` in the `terminal` to stop the running container and close
the **Preview** tab.

In the next module, we will look at how to lay the ground work for
deploying the application into production, by creating an [Amazon
Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/).

**Note:** If you are running this tutorial locally instead of Cloud9 you can now use [http://localhost:8080](http://localhost:8080) to preview your application.

**Note:** If you would like you could bring everything together up using `docker-compose up` and then when done tear down the components using `docker-compose down`.

## Other languages

This tutorial showed how to build a Java based docker. It is possible to containerize other languages such as Python [[1]](https://blog.realkinetic.com/building-minimal-docker-containers-for-python-applications-37d0272c52f3), Golang [[1]](https://medium.com/@chemidy/create-the-smallest-and-secured-golang-docker-image-based-on-scratch-4752223b7324)[[2]](https://blog.codeship.com/building-minimal-docker-containers-for-go-applications/)[[3]](https://github.com/chemidy/smallest-secured-golang-docker-image)[[4]](https://flaviocopes.com/golang-docker/), and C# [[1]](https://codefresh.io/docker-tutorial/c-sharp-in-docker/).

## Best practices

It is useful to go though docker best practices [[1]](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) and [[2]](https://snyk.io/blog/10-docker-image-security-best-practices/).

Some of the major ones are

1. Prefer minimal base images (see examples bellow)
2. Clean yum, apt, apk cache after install or use --no-cache
3. Least privileged user (`USER <something>` instead of `USER root`)
4. Don't leak sensitive information to Docker images (certificates, passwords etc)
5. Beware of recursive copy `COPY . .`
6. Use metadata labels `LABEL maintainer="me@example.com"`
7. Use multi-stage build for small and secure images
8. Use a linter such as [hadolint](https://github.com/hadolint/hadolint)

Having multiple stages or using alpine as base reduces total image size drastically. Bellow are a couple of non working examples to explain docker image size reduction. For the full details please see the links above.

Non working example of using a single stage (largest)

```Dockerfile
FROM golang:1.8.3 as builder
WORKDIR /go/src/github.com/flaviocopes/findlinks
RUN go get -d -v golang.org/x/net/html
COPY findlinks.go  .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o findlinks .
RUN yum --no-cache add ca-certificates
WORKDIR /root/
CMD ["/go/src/github.com/flaviocopes/findlinks/findlinks"]
```

Non working example of docker build using alpine (smaller)

```Dockerfile
FROM golang:1.8.3 as builder
WORKDIR /go/src/github.com/flaviocopes/findlinks
RUN go get -d -v golang.org/x/net/html
COPY findlinks.go  .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o findlinks .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /go/src/github.com/flaviocopes/findlinks/findlinks .
CMD ["./findlinks"]
```

Non working example of docker build using scratch, which will result in ~ 21.2MB with everything required for the go application, compared to images of 500Mb when not using multi stages.

```Dockerfile
############################
# STEP 1 build executable binary
############################
FROM golang:alpine AS builder
# Install git.
# Git is required for fetching the dependencies.
RUN apk update && apk add --no-cache git
WORKDIR $GOPATH/src/mypackage/myapp/
# this recursive copy is happening in the builder
# and the files will not be in the final docker image
COPY . .
# Fetch dependencies.
# Using go get.
RUN go get -d -v
# Build the binary.
RUN go build -o /go/bin/hello
############################
# STEP 2 build a small image
############################
FROM scratch
# Copy our static executable.
COPY --from=builder /go/bin/hello /go/bin/hello
# Run the hello binary.
ENTRYPOINT ["/go/bin/hello"]
```

## Next

At this step we have completed the basic docker introduction. We can bring our running dockers down using `docker-compose down` and proceed to the next tutorial [Container registry](/modules/container-registry/readme.md "Container Registry").
