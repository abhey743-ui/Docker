# Building Docker Images with Google Jib

## Overview

This is the third method of generating a Docker image for a Spring Boot microservice — **Jib**, a plugin maintained by Google Cloud Tools. Like Buildpacks, Jib requires **no Dockerfile**. Unlike Buildpacks, Jib does not rely on a separate builder image or even require a running Docker daemon in every mode — it builds the container image directly out of the Java build process itself.

---

## What Jib Is and How It Works

**Jib** is a Maven/Gradle plugin, built by Google, that constructs an OCI/Docker-compatible container image directly from your compiled Java application — without invoking `docker build`, and without needing a Dockerfile at all.

### The build process

1. Jib inspects your project's compiled output (classes, resources, dependencies) rather than a single fat JAR.
2. It separates these into **distinct layers**: dependencies (which rarely change), resources, and your application's own classes (which change most often).
3. Because dependencies are cached in their own layer, Jib only needs to rebuild and push the layer containing your actual class files on subsequent builds — dependency layers are reused as-is.
4. Depending on which Jib goal is invoked, the final image is either:
   - Pushed directly to a remote container registry (no local Docker required at all), or
   - Loaded into the local Docker daemon (this requires Docker to be running locally).

### Jib's Maven Goals

- **`jib:build`** — builds the image and pushes it straight to a remote registry. This mode does not require Docker to be installed on the machine at all, since Jib talks to registries over their HTTP API directly.
- **`jib:dockerBuild`** — builds the image and loads it into the **local Docker daemon**, exactly like the command used here. Docker must be running locally for this goal.
- **`jib:buildTar`** — builds the image as a `.tar` file on disk, without pushing or loading it anywhere.

### Why Use Jib

- **No Docker daemon required for registry pushes** — `jib:build` can run in CI/CD environments that don't have Docker installed at all, which is a meaningful operational simplification.
- **Fast, incremental builds** — because dependencies, resources, and application classes are split into separate layers, a code-only change only touches the smallest layer, making rebuilds significantly faster than rebuilding a full JAR-based image.
- **No Dockerfile or fat JAR needed** — Jib builds directly from your compiled classes, so there's no need to first package everything into a single executable JAR.
- **Reproducible builds** — given the same inputs, Jib produces bit-for-bit reproducible image layers, which is useful for build verification and caching in CI pipelines.

### Is the Resulting Image Production-Ready?

Yes. Jib is actively maintained by Google and is widely used in production Java deployments, particularly in CI/CD pipelines where avoiding a local Docker dependency is valuable.

---

## The pom.xml Configuration

```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>3.3.2</version>
  <configuration>
    <to>
      <image>eazybytes/${project.artifactId}:s4</image>
    </to>
  </configuration>
</plugin>
```

- **`groupId` / `artifactId` / `version`** — identifies the Jib plugin itself (`com.google.cloud.tools:jib-maven-plugin`, version `3.3.2`), which must be explicitly declared in `pom.xml` since, unlike `spring-boot-maven-plugin`, it isn't included by default.
- **`<to><image>`** — defines the destination image name and tag, following the same pattern as the earlier Buildpacks configuration:
  - `eazybytes/` is the namespace/organization prefix.
  - `${project.artifactId}` dynamically inserts the current module's name (`cards`, `accounts`, `loans`, etc.), letting the same configuration block be reused across every microservice.
  - `:s4` is the custom tag chosen for this build.

---

## The Command

```
mvn compile jib:dockerBuild
```

- **`compile`** — this Maven phase compiles the application's source code into `.class` files. Jib builds its image from these compiled classes rather than from a packaged JAR, so compilation (not packaging) is the only prerequisite step.
- **`jib:dockerBuild`** — this Jib goal builds the image and loads it directly into the local Docker daemon, tagged exactly as configured in `pom.xml` (`eazybytes/cards:s4` in this case).
- This must be run from inside the specific module's directory (e.g., `cd cards`), and requires Docker to be running locally, since the final image is loaded into the local Docker image store.

Once complete, the image can be run like any other Docker image:
```
docker run -p 8080:8080 eazybytes/cards:s4
```

---

## Summary

Jib takes the "no Dockerfile" philosophy a step further than Buildpacks by removing the dependency on a builder image and, in its registry-push mode, on Docker itself. It builds directly from compiled classes into layered images, optimizing for fast incremental rebuilds and reproducibility — making it a strong fit for CI/CD pipelines that push images to a registry without a local Docker daemon available.
