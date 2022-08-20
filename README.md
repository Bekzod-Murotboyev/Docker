# _[Java & Docker](https://docs.docker.com/language/java/)_

## _What is a container?_

> Now that you’ve run a container, what is a container? Simply put, a container is a sandboxed process on your machine
> that is isolated from all other processes on the host machine. That isolation leverages kernel namespaces and cgroups,
> features that have been in Linux for a long time. Docker has worked to make these capabilities approachable and easy
> to
> use. To summarize, a container:
> - is a runnable instance of an image. You can create, start, stop, move, or delete a container using the DockerAPI or
    CLI.
> - can be run on local machines, virtual machines or deployed to the cloud.
> - is portable (can be run on any OS).
> - is isolated from other containers and runs its own software, binaries, and configurations.

## _What is a container image?_

> When running a container, it uses an isolated filesystem. This custom filesystem is provided by a container image.
> Since the image contains the container’s filesystem, it must contain everything needed to run an application - all
> dependencies, configurations, scripts, binaries, etc. The image also contains other configuration for the container,
> such as environment variables, a default command to run, and other metadata.

## _What is a Dockerfile?_

> A Dockerfile is a text document that contains the instructions to assemble a Docker image. When we tell Docker to
> build our image by executing the docker build command, Docker reads these instructions, executes them, and creates a
> Docker image as a result.

* ***Docker file for spring boot application***

```
# syntax=docker/dockerfile:1

FROM eclipse-temurin:17-jdk-jammy

WORKDIR /app

COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:resolve

COPY src ./src

CMD ["./mvnw", "spring-boot:run"]
```

- The first line to add to a Dockerfile is a # syntax parser directive. While optional, this directive instructs the
  Docker builder what syntax to use when parsing the Dockerfile, and allows older Docker versions with BuildKit enabled
  to upgrade the parser before starting the build. Parser directives must appear before any other comment, whitespace,
  or Dockerfile instruction in your Dockerfile, and should be the first line in Dockerfiles.We recommend using
  docker/dockerfile:1, which always points to the latest release of the version 1 syntax. BuildKit automatically checks
  for updates of the syntax before building, making sure you are using the most current version.

```
#syntax=docker/dockerfile:1
```

- Next, we need to add a line in our Dockerfile that tells Docker what base image we would like to use for our
  application.Docker images can be inherited from other images. For this guide, we use Eclipse Termurin, one of the most
  popular official images with a build-worthy JDK.

```
FROM eclipse-temurin:17-jdk-jammy
```

- To make things easier when running the rest of our commands, let’s set the image’s working directory. This instructs
  Docker to use this path as the default location for all subsequent commands. By doing this, we do not have to type out
  full file paths but can use relative paths based on the working directory.

```
WORKDIR /app
```

- Before we can run mvnw dependency, we need to get the Maven wrapper and our pom.xml file into our image. We’ll use the
  COPY command to do this. The COPY command takes two parameters. The first parameter tells Docker what file(s) you
  would like to copy into the image. The second parameter tells Docker where you want that file(s) to be copied to.
  We’ll copy all those files and directories into our working directory - /app.

```
  COPY .mvn/ .mvn
  COPY mvnw pom.xml ./ 
  ```

- Once we have our pom.xml file inside the image, we can use the RUN command to execute the command mvnw dependency:
  resolve. This works exactly the same way as if we were running mvnw (or mvn) dependency locally on our machine, but
  this time the dependencies will be installed into the image.

```
RUN ./mvnw dependency:resolve
```

- At this point, we have an Eclipse Termurin image that is based on OpenJDK version 17, and we have also installed our
  dependencies. The next thing we need to do is to add our source code into the image. We’ll use the COPY command just
  like we did with our pom.xml file above.

```
COPY src ./src
```

- This COPY command takes all the files located in the current directory and copies them into the image. Now, all we
  have to do is to tell Docker what command we want to run when our image is executed inside a container. We do this
  using the CMD command.

```
CMD ["./mvnw", "spring-boot:run"]
```

## _.dockerignore file_

- To increase the performance of the build, and as a general best practice, we recommend that you create a .dockerignore
  file in the same directory as the Dockerfile. For this tutorial, your .dockerignore file should contain just one line:

``` 
target
 ```

- This line excludes the target directory, which contains output from Maven, from the Docker build context. There are
  many good reasons to carefully structure a .dockerignore file, but this one-line file is good enough for now.

## _Build an image_

- _Now that we’ve created our Dockerfile, let’s build our image. To do this, we use the docker build command. The docker
  build command builds Docker images from a Dockerfile and a “context”. A build’s context is the set of files located in
  the specified PATH or URL. The Docker build process can access any of the files located in this context._
- _The build command optionally takes a `--tag` flag. The tag is used to set the name of the image and an optional tag
  in the format `name:tag`. We’ll leave off the optional tag for now to help simplify things. If we do not pass a tag,
  Docker uses `latest` as its default tag. You can see this in the last line of the build output._

```
 docker build --tag java-docker .
```

- To list images, simply run the `docker images` command.
- To create a new tag for the image we’ve built above, run the following command:

```
 docker tag java-docker:latest java-docker:v1.0.0
```

- To remove image, we’ll use the `rmi` command. The `rmi` command stands for `remove image`.

```
 docker rmi <IMAGE_NAME>
 ```

```
 docker rmi <IMAGE_NAME>:<TAG>
```

```
 docker rmi <IMAGE_ID>
```

- To run container, just run the following command:

```
 docker run --rm -d -p 8080:8080 --name springboot-server java-docker
```

- `--rm `   `remove`  Automatically remove the container when it exits
- `-d`   `detach`  Run container in background and print container ID
- `-p`   `publish`   `<PORT_SERVER>:<PORT_CONTAINER>`  Publish a container's port(s) to the host
- `--name`  Assign a name to the container

## _Run a database in a container_

- Let’s create our volumes now. We’ll create one for the data and one for configuration of Postgresql.

```
docker volume create postgres_data
docker volume create postgres_config
```

- Now we’ll create a network that our application and database will use to talk to each other. The network is called a
  user-defined bridge network and gives us a nice DNS lookup service which we can use when creating our connection
  string.

```
docker network create postgres_net
```

- Now, let’s run Postgresql in a container and attach to the volumes and network we created above. Docker pulls the
  image from Hub and runs it locally.

```
docker run -itd --rm  -v postgres_data:/var/lib/postgres \
-v postgres_config:/etc/postgres/conf.d \
--network postgres_net \
--name postgres_server \
-e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=<YOUR_PASSWORD> \
-e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=<YOUR_DATABASE> \
-p 5432:5432 postgres
```
- Okay, now that we have a running Postgresql, let’s update our Dockerfile to activate the Postgresql Spring profile defined in the application and switch from an in-memory H2 database to the Postgresql server we just created.

We only need to add the Postgresql profile as an argument to the CMD definition.
```
CMD ["./mvnw", "spring-boot:run", "-Dspring-boot.run.profiles=postgres"]
```
- Let’s build our image.
```
 docker build -t java-docker .
```
- Now, let’s run our container. This time, we need to set the POSTGRESQL_URL environment variable so that our application knows what connection string to use to access the database. We’ll do this using the docker run command.

```
 docker run --rm -d \
--name springboot-server \
--network postgres_net \
-e POSTGRESQL_URL=jdbc:postgresql://postgres_server/<YOUR_DATABASE> \
-p 8080:8080 java-docker
```
