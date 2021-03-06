# Basics

You're here to learn how to use Earthly. Welcome! Earthly is a build automation tool based upon modern development technologies available today, like Docker.

First, a few concepts:

* Earthly - the build automation system as a whole
* `earth` - the CLI tool used to interact with Earthly
* Earthfile - a file (named literally `Earthfile`) which contains a series of targets and their respective recipes
* buildkitd - a [daemon built by the Docker team](https://github.com/moby/buildkit) and used by Earthly to execute builds. It executes LLB, the same low-level primitives used when building Dockerfiles. The buildkitd daemon is started automatically in a docker container, by `earth`, when executing builds.
* recipe - a specific series of build steps
* target - the label used to identify a recipe. 'Target' is also used to refer to a build of a specific target.
* build context - the main directory made available to the build for copying files from
* artifact - a file resulting from executing a target (not all targets have artifacts)
* image - a docker image resulting from executing a target (not all targets have images)

Now that we got that out of the way, we can now be more specific:

Earthly is a build automation tool where all recipes are executed in docker containers, with layer caching principles similar to Dockerfiles, and where the only thing shared between recipes are declared artifacts and images.

A key difference from a Dockerfile build is that Earthly can be used to build not just images, but also artifacts - regular files or directories that can be written back onto the host filesystem.

One of the key principles of Earthly is that the best build tooling of a specific language is built by the community of that language itself. Earthly does not intend to replace that tooling, but rather to leverage and augment it.

## Installation

Before going any further, it is advisable that you install `earth` on your computer, so you can follow along and try out the examples. See the [installation page](../installation/installation.md).

## A simple Earthfile

Earthfiles are always named `Earthfile`, regardless of their location in the codebase.

{% method %}
{% sample lang="Go" %}
Here is a sample earthfile of a Go app

```Dockerfile
# Earthfile

FROM golang:1.13-alpine3.11
WORKDIR /go-example

build:
    COPY main.go .
    RUN go build -o build/go-example main.go
    SAVE ARTIFACT build/go-example /go-example AS LOCAL build/go-example

docker:
    COPY +build/go-example .
    ENTRYPOINT ["/go-example/go-example"]
    SAVE IMAGE go-example:latest
```

The code of the app might look like this

```go
// main.go

package main

import "fmt"

func main() {
	fmt.Println("hello world")
}
```

{% sample lang="JavaScript" %}
Here is a sample earthfile of a JS app

```Dockerfile
# Earthfile

FROM node:13.10.1-alpine3.11
WORKDIR /js-example

build:
    # In JS, there's nothing to build in this simple form.
    # The source is also the artifact used in production.
    COPY index.js .
    SAVE ARTIFACT index.js /dist/index.js AS LOCAL ./dist/index.js

docker:
    COPY +build/dist dist
    ENTRYPOINT ["node", "./dist/index.js"]
    SAVE IMAGE js-example:latest
```

The code of the app might look like this

```js
// index.js

console.log("hello world");
```

{% sample lang="Java" %}
Here is a sample earthfile of a Java app

```Dockerfile
# Earthfile

FROM openjdk:8-jdk-alpine
RUN apk add --update --no-cache gradle
WORKDIR /java-example

build:
    COPY build.gradle ./
    COPY src src
    RUN gradle build
    RUN gradle install
    SAVE ARTIFACT build/install/java-example/bin /bin AS LOCAL build/bin
    SAVE ARTIFACT build/install/java-example/lib /lib AS LOCAL build/lib

docker:
    COPY +build/bin bin
    COPY +build/lib lib
    ENTRYPOINT ["/java-example/bin/java-example"]
    SAVE IMAGE java-example:latest
```

The code of the app might look like this

```java
// src/main/java/hello/HelloWorld.java

package hello;

public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```

{% endmethod %}

You will notice that the recipes look very much like Dockerfiles. This is an intentional design decision. Existing Dockerfiles can be ported to earthfiles by copy-pasting them over and then tweaking them slightly. Compared to Dockerfile syntax, some commands are new (like `SAVE ARTIFACT`), others have additional semantics (like `COPY +target/some-artifact`) and other semantics are removed (like `FROM ... AS ...` and `COPY --from`).

All earthfiles start with a base recipe. This is the only recipe which does not have an explicit target name - the name is always implied to be `base`. All other target implicitly inherit from `base`. You can imagine that all recipes start with an implicit `FROM +base`.

## Executing a build

In this particular example, we can see two explicit targets: `build` and `docker`. In order to execute the build, we can run, for example:

```bash
earth +build
```

or

```bash
earth +docker
```

The output might look like this:

![Earth build output](img/go-example.png)

Notice how to the left of `|`, within the output, we can see some targets like `+base`, `+build` and `+docker` . Notice how the output is interleaved between `+docker` and `+build`. This is because the system executes independent build steps in parallel. The reason this is possible effortlessly is because only very few things are shared between the builds of the recipes and those things are declared and obvious. The rest is completely isolated.

In addition, notice how even though the base is used as part of both `build` and `docker`, it is only executed once. This is because the system deduplicates execution, where possible.

Furthermore, the fact that the `docker` target depends on the `build` target is visible within the command `COPY +build/...`. Through this command, the system knows that it also needs to build the target `+build`, in order to satisfy the dependency on the artifact.

Finally, notice how the output of the build: the docker image `go-example:latest` and the file `build/go-example` is only written after the build is declared a success. This is due to another isolation principle of Earthly: a build either succeeds completely or it fails altogether.

Once the build has executed, we can run the resulting docker image to try it out:

{% method %}
{% sample lang="Go" %}
```
$ docker run --rm go-example:latest
hello world
```
{% sample lang="JavaScript" %}
```
$ docker run --rm js-example:latest
hello world
```
{% sample lang="Java" %}
```
$ docker run --rm java-example:latest
hello world
```
{% endmethod %}

{% hint style='info' %}
##### Note

Targets have a particular referencing convention which helps Earthly to identify which recipe to execute. In the simplest form, targets are referenced by `+<target-name>` - for example, `+build`. For more details see the [target referencing page](./target-ref.md).
{% endhint %}

## Detailed explanation

Going back to the example earthfile definition, here is what each command does:

{% method %}
{% sample lang="Go" %}
```Dockerfile
# Earthfile

# The build starts from a docker image: golang:1.13-alpine3.11
FROM golang:1.13-alpine3.11
# We change the current working directory.
WORKDIR /go-example

# The above commands are inherited implicitly by all targets below
# (as if they started with FROM +base).

# Declare a target, build.
build:
    # Define the recipe of the target build as follows:

    # Copy main.go from the build context to the build environment, as a layer.
    COPY main.go .
    # Run a go build command as a layer.
    # This uses the previously copied main.go file.
    RUN go build -o build/go-example main.go
    # Save the output of the build command as an artifact. Call this
    # artifact /go-example (it can be later referenced as +build/go-example).
    # In addition, store the artifact as a local file (on the host) named
    # build/go-example. This local file is only written if the entire build
    # succeeds.
    SAVE ARTIFACT build/go-example /go-example AS LOCAL build/go-example

# Declare a target, docker.
docker:
    # Define the recipe of the target docker as follows:

    # Copy the artifact /go-example produced by another target, +build, to the
    # current directory within the build container.
    COPY +build/go-example .
    # Set the entrypoint for the resulting docker image.
    ENTRYPOINT ["/go-example/go-example"]
    # Save the current state as a docker image, which will have the docker tag
    # go-example:latest. This image is only made available to the host's docker
    # if the entire build succeeds.
    SAVE IMAGE go-example:latest
```
{% sample lang="JavaScript" %}
```Dockerfile
# Earthfile

# The build starts from a docker image: node:13.10.1-alpine3.11
FROM node:13.10.1-alpine3.11
# We change the current working directory.
WORKDIR /js-example

# The above commands are inherited implicitly by all targets below
# (as if they started with FROM +base).

# Declare a target, build.
build:
    # Define the recipe of the target build as follows:

    # Copy index.js from the build context to the build environment, as a
    # layer.
    COPY index.js .
    # Save the index.js in an artifact dir called dist (it can be later
    # referenced as +build/dist). In addition, store the artifact as a
    # local file (on the host) named dist/index.js. This local file is only
    # written if the entire build succeeds.
    SAVE ARTIFACT index.js /dist/index.js AS LOCAL ./dist/index.js

# Declare a target, docker.
docker:
    # Define the recipe of the target docker as follows:

    # Copy the artifact /dist produced by another target, +build, to the
    # current directory within the build container.
    COPY +build/dist dist
    # Set the entrypoint for the resulting docker image.
    ENTRYPOINT ["node", "./dist/index.js"]
    # Save the current state as a docker image, which will have the docker tag
    # js-example:latest. This image is only made available to the host's docker
    # if the entire build succeeds.
    SAVE IMAGE js-example:latest
```
{% sample lang="Java" %}
```Dockerfile
# Earthfile

# The build starts from a docker image: openjdk:8-jdk-alpine
FROM openjdk:8-jdk-alpine
# We install gradle using alpine's apk command.
RUN apk add --update --no-cache gradle
# We change the current working directory.
WORKDIR /java-example

# The above commands are inherited implicitly by all targets below
# (as if they started with FROM +base).

# Declare a target, build.
build:
    # Define the recipe of the target build as follows:

    # Copy build.gradle and src from the build context to the build
    # environment, as layers.
    COPY build.gradle ./
    COPY src src
    # Run the gradle build and gradle install commands as layers.
    # These use the previously copied src dir.
    RUN gradle build
    RUN gradle install
    # Save the output of the build command as artifacts. Call these
    # artifacts bin and lib (they can be later referenced as +build/bin and
    # +build/lib respectively).
    # In addition, store the artifacts as local dirs (on the host) named
    # build/bin and build/lib. This local dirs are only written if the entire
    # build succeeds.
    SAVE ARTIFACT build/install/java-example/bin /bin AS LOCAL build/bin
    SAVE ARTIFACT build/install/java-example/lib /lib AS LOCAL build/lib

# Declare a target, docker.
docker:
    # Define the recipe of the target docker as follows:

    # Copy the artifacts /bin and /lib produced by another target, +build, to
    # the current directory within the build container.
    COPY +build/bin bin
    COPY +build/lib lib
    # Set the entrypoint for the resulting docker image.
    ENTRYPOINT ["/java-example/bin/java-example"]
    # Save the current state as a docker image, which will have the docker tag
    # java-example:latest. This image is only made available to the host's
    # docker if the entire build succeeds.
    SAVE IMAGE java-example:latest
```
{% endmethod %}

## Adding dependecies in the mix

Let's imagine now that in our simple app, we now want to add a programming language dependency. Here's how our build might look like as a result

{% method %}
{% sample lang="Go" %}
```go.mod
// go.mod

module github.com/earthly/earthly/examples/go

go 1.13

require github.com/sirupsen/logrus v1.5.0
```

The code of the app might look like this

```go
// main.go

package main

import "github.com/sirupsen/logrus"

func main() {
	logrus.Info("hello world")
}
```

The build then might become

```Dockerfile
# Earthfile

FROM golang:1.13-alpine3.11
WORKDIR /go-example

build:
    COPY go.mod go.sum .
    COPY main.go .
    RUN go build -o build/go-example main.go
    SAVE ARTIFACT build/go-example /go-example AS LOCAL build/go-example

docker:
    COPY +build/go-example .
    ENTRYPOINT ["/go-example/go-example"]
    SAVE IMAGE go-example:latest
```
{% sample lang="JavaScript" %}
```json
// package.json

{
  "name": "example-js",
  "version": "0.0.1",
  "description": "Hello world",
  "private": true,
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "MPL-2.0",
  "devDependencies": {
    "webpack": "^4.42.1",
    "webpack-cli": "^3.3.11"
  },
  "dependencies": {
    "http-server": "^0.12.1"
  }
}
```

The code of the app might look like this

```js
// src/index.js

function component() {
    const element = document.createElement('div');
    element.innerHTML = "hello world"
    return element;
}

document.body.appendChild(component());
```

```html
<!-- dist/index.html -->

<!doctype html>
<html>

<head>
    <title>Getting Started</title>
</head>

<body>
    <script src="./main.js"></script>
</body>

</html>
```

The build then might become

```Dockerfile
# Earthfile

FROM node:13.10.1-alpine3.11
WORKDIR /js-example

build:
    COPY package.json package-lock.json ./
    COPY src src
    COPY dist dist
    RUN npm install
    RUN npx webpack
    SAVE ARTIFACT dist /dist AS LOCAL ./dist

docker:
    COPY package.json package-lock.json ./
    RUN npm install
    COPY +build/dist dist
    ENTRYPOINT ["node", "./dist/index.js"]
    SAVE IMAGE js-example:latest
```
{% sample lang="Java" %}
```groovy
// build.gradle

apply plugin: 'java'
apply plugin: 'application'

mainClassName = 'hello.HelloWorld'

repositories {
    mavenCentral()
}

jar {
    baseName = 'hello-world'
    version = '0.0.1'
}

sourceCompatibility = 1.8
targetCompatibility = 1.8

dependencies {
    compile "joda-time:joda-time:2.2"
    testCompile "junit:junit:4.12"
}
```

The code of the app might look like this

```java
// src/main/java/hello/HelloWorld.java

package hello;

import org.joda.time.LocalTime;

public class HelloWorld {
    public static void main(String[] args) {
        LocalTime currentTime = new LocalTime();
        System.out.println(currentTime + " - hello world");
    }
}
```

The Earthfile file would not change

```Dockerfile
# Earthfile

FROM openjdk:8-jdk-alpine
RUN apk add --update --no-cache gradle
WORKDIR /java-example

build:
    COPY build.gradle ./
    COPY src src
    RUN gradle build
    RUN gradle install
    SAVE ARTIFACT build/install/java-example/bin /bin AS LOCAL build/bin
    SAVE ARTIFACT build/install/java-example/lib /lib AS LOCAL build/lib

docker:
    COPY +build/bin bin
    COPY +build/lib lib
    ENTRYPOINT ["/java-example/bin/java-example"]
    SAVE IMAGE java-example:latest
```
{% endmethod %}

However, as we build this new setup and make changes to the main source code, we notice that the dependencies are downloaded every single time we change the source code. While the build is not necessarily incorrect, it is inefficient for proper development speed.

## Efficient caching of dependencies

The reason the build is inefficient is because we have not made proper use of layer caching. When a file changes, the corresponding `COPY` command is re-executed without cache, causing all commands after it to also re-execute without cache.

If, however, we could first download the dependencies and only afterwards copy and build the code, then the cache would be reused every time we changed the code.

{% method %}
{% sample lang="Go" %}
```Dockerfile
# Earthfile

FROM golang:1.13-alpine3.11
WORKDIR /go-example

build:
    # Download deps before copying code.
    COPY go.mod go.sum .
    RUN go mod download
    # Also save these back to host, in case go.sum changes.
    SAVE ARTIFACT go.mod AS LOCAL go.mod
	SAVE ARTIFACT go.sum AS LOCAL go.sum
    # Copy and build code.
    COPY main.go .
    RUN go build -o build/go-example main.go
    SAVE ARTIFACT build/go-example /go-example AS LOCAL build/go-example

docker:
    COPY +build/go-example .
    ENTRYPOINT ["/go-example/go-example"]
    SAVE IMAGE go-example:latest
```
{% sample lang="JavaScript" %}
```Dockerfile
# Earthfile

FROM node:13.10.1-alpine3.11
WORKDIR /js-example

build:
    # Download deps before copying code.
    COPY package.json package-lock.json ./
    RUN npm install
    # Also save these back to host, in case package-lock.json changes.
    SAVE ARTIFACT package.json AS LOCAL ./package.json
    SAVE ARTIFACT package-lock.json AS LOCAL ./package-lock.json
    # Copy and build code.
    COPY src src
    COPY dist dist
    RUN npx webpack
    SAVE ARTIFACT dist /dist AS LOCAL ./dist

docker:
    COPY package.json package-lock.json ./
    RUN npm install
    COPY +build/dist dist
    ENTRYPOINT ["node", "./dist/index.js"]
    SAVE IMAGE js-example:latest
```
{% sample lang="Java" %}
```Dockerfile
# Earthfile

FROM openjdk:8-jdk-alpine
RUN apk add --update --no-cache gradle
WORKDIR /java-example

build:
    # Download deps before copying code.
    COPY build.gradle ./
    RUN gradle build
    # Copy and build code.
    COPY src src
    RUN gradle build
    RUN gradle install
    SAVE ARTIFACT build/install/java-example/bin /bin AS LOCAL build/bin
    SAVE ARTIFACT build/install/java-example/lib /lib AS LOCAL build/lib

docker:
    COPY +build/bin bin
    COPY +build/lib lib
    ENTRYPOINT ["/java-example/bin/java-example"]
    SAVE IMAGE java-example:latest
```
{% endmethod %}

For a primer into Dockerfile layer caching see [this article](https://pythonspeed.com/articles/docker-caching-model/). The same principles apply to Earthfiles.

## Reduce code duplication

In some cases, the dependencies might be used in more than one build target. For this usecase, we might want to separate dependency downloading and reuse it. For this reason, let's consider breaking this out into a separate build target, called `deps`. We can then inherit from `deps` by using the command `FROM +deps`.

Note that in our case, only the JavaScript version has an example where `FROM +deps` is used in more than one place: both in `build` and in `docker`. Nevertheless, all versions show how dependencies may be separated.

{% method %}
{% sample lang="Go" %}
```Dockerfile
# Earthfile

FROM golang:1.13-alpine3.11
WORKDIR /go-example

deps:
    COPY go.mod go.sum ./
	RUN go mod download
	SAVE ARTIFACT go.mod AS LOCAL go.mod
	SAVE ARTIFACT go.sum AS LOCAL go.sum
	SAVE IMAGE

build:
    FROM +deps
    COPY main.go .
    RUN go build -o build/go-example main.go
    SAVE ARTIFACT build/go-example /go-example AS LOCAL build/go-example

docker:
    COPY +build/go-example .
    ENTRYPOINT ["/go-example/go-example"]
    SAVE IMAGE go-example:latest
```
{% sample lang="JavaScript" %}
```Dockerfile
# Earthfile

FROM node:13.10.1-alpine3.11
WORKDIR /js-example

deps:
    COPY package.json ./
    COPY package-lock.json ./
    RUN npm install
    SAVE ARTIFACT package.json AS LOCAL ./package.json
    SAVE ARTIFACT package-lock.json AS LOCAL ./package-lock.json
    SAVE IMAGE

build:
    FROM +deps
    COPY src src
    COPY dist dist
    RUN npx webpack
    SAVE ARTIFACT dist /dist AS LOCAL dist

docker:
    FROM +deps
    COPY +build/dist ./dist
    EXPOSE 8080
    ENTRYPOINT ["/js-example/node_modules/http-server/bin/http-server", "./dist"]
    SAVE IMAGE js-example:latest
```
{% sample lang="Java" %}
```Dockerfile
# Earthfile

FROM openjdk:8-jdk-alpine
RUN apk add --update --no-cache gradle
WORKDIR /java-example

deps:
    COPY build.gradle ./
    RUN gradle build
    SAVE IMAGE

build:
    FROM +deps
    COPY src src
    RUN gradle build
    RUN gradle install
    SAVE ARTIFACT build/install/java-example/bin AS LOCAL build/bin
    SAVE ARTIFACT build/install/java-example/lib AS LOCAL build/lib

docker:
    COPY +build/bin bin
    COPY +build/lib lib
    ENTRYPOINT ["/java-example/bin/java-example"]
    SAVE IMAGE java-example:latest
```
{% endmethod %}

Notice how at the end of the `deps` recipe, we issued a `SAVE IMAGE` command. In this case, it is not for the purpose of saving as an image that would be used outside of the build: the command has no docker tag associated with it. Instead, it is for the purpose of reusing the image within the build, from another target (via `FROM +deps`).

## Next steps

Congratulations, you made it this far!

To learn more about Earthly, take a look at the [examples directory on GitHub](https://github.com/earthly/earthly/tree/master/examples), where you will find the complete code used in this guide:

* [Go](https://github.com/earthly/earthly/tree/master/examples/go)
* [JavaScript](https://github.com/earthly/earthly/tree/master/examples/js)
* [Java](https://github.com/earthly/earthly/tree/master/examples/java)

## See also

* The [Earthfile reference](../earthfile/earthfile.md)
* The [earth command reference](../earth-command/earth-command.md)
* More [examples](../examples/examples.md)
