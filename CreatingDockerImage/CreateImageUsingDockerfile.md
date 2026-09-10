**# Understanding the Dockerfile for the Accounts Service

## Overview

This Dockerfile represents the **manual (traditional) approach** to containerizing a Java/Spring Boot application. In this method, the developer writes every instruction by hand — the base image, the labeling, the copy step, and the entrypoint — and takes full responsibility for keeping the file correct and up to date. It works well and is widely used, but it does mean the team owns the maintenance overhead: any change in Java version, JAR naming convention, or build output path has to be manually reflected here.

Below is a breakdown of the file, followed by prerequisites and the exact CLI steps to build the image.

---

## Line-by-Line Breakdown

### 1. `FROM eclipse-temurin:21-jdk`
- This is the **base image** instruction — every Docker image is built on top of another image, and this is the starting layer.
- `eclipse-temurin:21-jdk` is an OpenJDK-based distribution (Eclipse Temurin, formerly AdoptOpenJDK) that ships **Java 21 with the full JDK** (not just the JRE).
- Using the full JDK (instead of a slimmer JRE-only image) means the container has compiler tools and other development utilities available, which increases image size but ensures compatibility if anything at runtime needs JDK-level tooling.
- This line effectively guarantees that the Java 21 runtime environment exists inside the container before anything else happens.

### 2. `LABEL "org.opencontainers.image.authors"="eazybytes.com"`
- `LABEL` adds **metadata** to the image in a key-value format.
- Here, it follows the **OpenContainers Image Spec** standard (`org.opencontainers.image.authors`), which is the modern, tool-recognized way of tagging image ownership/authorship.
- This replaces the older, now-deprecated `MAINTAINER` instruction (correctly commented out in your file) — `MAINTAINER` was deprecated because `LABEL` is more flexible and can hold arbitrary metadata (version, source repo, licenses, etc.), not just a single maintainer field.
- This line has **no effect on runtime behavior** — it's purely informational and useful for image scanning, auditing, and registry catalogs.

### 3. `COPY target/accounts-0.0.1-SNAPSHOT.jar accounts-0.0.1-SNAPSHOT.jar`
- `COPY` moves a file from your **local build context** (your machine, where `docker build` is run) into the image's filesystem.
- `target/accounts-0.0.1-SNAPSHOT.jar` is the source path — this assumes your project has **already been compiled and packaged** into a JAR by Maven (this is the standard output folder for Maven builds).
- The destination `accounts-0.0.1-SNAPSHOT.jar` places the JAR at the image's working directory (root, by default, since no `WORKDIR` was set).
- This is the step that actually gets your compiled application bytecode **into** the container.

### 4. `ENTRYPOINT ["java", "-jar", "accounts-0.0.1-SNAPSHOT.jar"]`
- `ENTRYPOINT` defines the **default command that runs when the container starts**.
- Written in **exec form** (JSON array syntax) rather than shell form — this is the recommended practice because exec form runs the process directly as PID 1, allowing it to properly receive OS signals (like `SIGTERM` for graceful shutdown), instead of wrapping it in a shell.
- Effectively, this tells Docker: "every time a container is started from this image, run `java -jar accounts-0.0.1-SNAPSHOT.jar`" — which boots the Spring Boot application.

---

## Prerequisites Before Building the Image

Before running `docker build`, the following must be in place:

1. **The JAR must already exist.** This Dockerfile does *not* compile your source code — it only copies a pre-built artifact. So you must first run your Maven build:
   ```
   mvn clean package
   ```
   (or `mvn clean install` if you need it in your local repo too). This should be run from the project root, and it must succeed without errors, producing `target/accounts-0.0.1-SNAPSHOT.jar`.

2. **Verify the JAR name and path match exactly.** The filename in the `COPY` instruction must match the actual generated JAR name (including version number) in the `target/` folder. If your `pom.xml` version changes, this Dockerfile line needs a manual update too — this is part of the maintenance overhead mentioned earlier.

3. **The Dockerfile must sit in (or reference relative to) the project root**, so that `target/` is visible in the Docker build context.

4. **Docker Engine/Docker Desktop must be installed and running** on the machine executing the build.

5. **Confirm you're in the correct directory** in your terminal — the one containing both the `Dockerfile` and the `target/` folder — before running any Docker CLI commands.

---

## Step-by-Step CLI Instructions to Build and Run the Image

### Step 1: Navigate to the project directory
```
cd /path/to/accounts-service
```
This should be the folder containing the `Dockerfile` and the `target/` directory.

### Step 2: Build the JAR (if not already built)
```
mvn clean package
```
Confirm that `accounts-0.0.1-SNAPSHOT.jar` now exists inside `target/`.

### Step 3: Build the Docker image
```
docker build -t accounts:0.0.1-SNAPSHOT .
```
- `-t accounts:0.0.1-SNAPSHOT` tags the image with a readable name and version.
- The trailing `.` tells Docker to use the current directory as the **build context** (this is where it looks for the `Dockerfile` and the files referenced in `COPY`).

### Step 4: Confirm the image was created
```
docker images
```
You should see `accounts` listed with the tag `0.0.1-SNAPSHOT`.

### Step 5: Run a container from the image
```
docker run -p 8080:8080 accounts:0.0.1-SNAPSHOT
```
- `-p 8080:8080` maps port 8080 on your host machine to port 8080 inside the container (adjust if your Spring Boot app uses a different `server.port`).

### Step 6: Verify the container is running
```
docker ps
```
This lists all currently running containers, confirming your accounts service is up.

### Step 7 (Optional): View application logs
```
docker logs -f <container_id_or_name>
```
Useful for confirming the Spring Boot application started correctly without errors.

### Step 8 (Optional): Stop the container
```
docker stop <container_id_or_name>
```

---

## Summary

This Dockerfile follows the classic single-stage, manually-written approach: pick a JDK base image, label it, copy in a pre-built JAR, and define the run command. It's simple and transparent, but it places the responsibility of keeping the JAR name, version, and build steps in sync entirely on the developer.**
