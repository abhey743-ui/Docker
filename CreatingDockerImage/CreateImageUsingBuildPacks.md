# Building Docker Images with Cloud Native Buildpacks (Spring Boot)

## Overview

This is the second method of generating a Docker image for a Spring Boot microservice — **Cloud Native Buildpacks (CNB)**. Unlike the manual Dockerfile approach, this method requires **no Dockerfile at all**. Instead, the `spring-boot-maven-plugin` (already part of a standard Spring Boot project) takes your compiled application and automatically produces a runnable, OCI-compliant container image.

---

## What Buildpacks Are and How They Work

**Cloud Native Buildpacks** is a specification — originally pioneered by Heroku, later standardized by Pivotal/VMware and the CNCF — for turning application source code directly into a container image without a hand-written Dockerfile.

Spring Boot integrates this natively through `spring-boot-maven-plugin` (and the Gradle equivalent), so no separate tool installation is required.

### The build process, step by step

1. Running the build-image goal causes Maven to pull down a **builder image**. Spring Boot defaults to **Paketo Buildpacks** as its builder.
2. The builder runs a **detection phase**, inspecting the project to determine what kind of application it is (in this case, a Java application built with Maven, producing a JAR).
3. It then runs the **build phase**, selecting an appropriate JRE, adding any required OS-level dependencies, and assembling the application.
4. Rather than producing one large, undivided image layer, the builder splits the app into **multiple layers**: dependencies, the Spring Boot loader, snapshot dependencies, and the application's own classes are each kept separate.
5. The finished image is loaded directly into the **local Docker daemon** — this is why Docker still needs to be running locally, even though no Dockerfile is involved.

### Why Use Buildpacks Instead of a Manual Dockerfile

- **No Dockerfile maintenance** — there is no JAR name, path, or version to keep manually synchronized, which was a direct pain point of the manual Dockerfile approach.
- **Smarter layer caching** — because application classes are isolated from dependencies, a code-only change only requires rebuilding and pushing that one small layer, rather than the entire JAR. This meaningfully speeds up incremental builds.
- **Security and patching by default** — Paketo images use hardened base OS layers, run as a non-root user by default, and can have their base OS "rebased" with security patches independently of a full application rebuild.
- **Consistency across microservices** — every service in a multi-module project is built the same standardized way, regardless of who configured it, removing a common source of drift between services.

### Is the Resulting Image Production-Ready?

Yes. Paketo Buildpacks are explicitly designed for production use, and this lineage traces directly back to Heroku's and Cloud Foundry's production-grade buildpack systems. This is not a development-only shortcut — it is a legitimate production deployment path.

---

## The pom.xml Configuration

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <configuration>
        <image>
          <name>eazybytes/${project.artifactId}:s4</name>
        </image>
        <excludes>
          <exclude>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
          </exclude>
        </excludes>
      </configuration>
    </plugin>
  </plugins>
</build>
```

### `spring-boot-maven-plugin`
This plugin is what exposes the `build-image` goal in the first place. It is typically already present in a standard Spring Boot project; the additions here are entirely inside its `<configuration>` block.

### `<image><name>`
This controls the tag applied to the final image:
- `eazybytes/` acts as a namespace or organization prefix — useful when pushing the image to a registry such as Docker Hub under an organization account.
- `${project.artifactId}` is a Maven property that automatically inserts whatever module is currently being built (`accounts`, `loans`, `cards`, etc.). This means the identical configuration block can be copied into every microservice's `pom.xml` without any manual edits.
- `:s4` is a custom tag chosen for this build (a version or iteration label).

### `<excludes>` for Lombok
Lombok is a **compile-time-only** annotation processor — it generates boilerplate code such as getters, setters, and constructors during compilation, and serves no purpose at runtime. Since Buildpacks package the application's actual runtime dependencies into image layers, explicitly excluding Lombok keeps it out of the final image. This reduces image size and avoids bundling a dependency the running application will never use.

---

## The Command

```
mvn spring-boot:build-image
```

- This Maven goal, provided by `spring-boot-maven-plugin`, triggers the full Buildpacks-based image build.
- It must be run from inside the specific module's directory (e.g., `cd loans`), so Maven can locate that module's `pom.xml`.
- No separate `docker build` command is needed — Maven communicates with the local Docker daemon directly, runs the entire detect → build → export pipeline, and places the finished image straight into the local `docker images` list, tagged exactly as configured (`eazybytes/loans:s4` in this case).

Once complete, the image can be run exactly like any other Docker image:
```
docker run -p 8080:8080 eazybytes/loans:s4
```

---

## Summary

The Buildpacks approach shifts the responsibility of image construction away from the developer and onto a standardized, actively maintained builder. Instead of manually tracking base images, JAR paths, and versions in a Dockerfile, the developer only needs to configure a name/tag and any dependency exclusions in `pom.xml`, then run a single Maven goal. The trade-off for this convenience is less granular, line-by-line control over the image compared to writing a Dockerfile by hand.
