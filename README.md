# 📊 Cloud-Native Spring Boot Build Benchmark: Maven vs. Jeka vs. Gradle

## 🎯 Objective
This benchmark compares the build performance of Maven, Jeka, and Gradle for Spring Boot applications. It focuses on creating JVM and native Docker images in cloud-native environments.

Two scenarios are tested:
- Development on workstations: Tested on a high-performance MacBook and a medium-performance Windows machine, with all caches enabled.
- Simulated CI/CD pipeline: Run with all caches disabled, including no Docker cache, wrapper cache, or Java dependencies cache.

The performance is measured using default configurations, without optimizations such as build caches. This represents conditions for small to medium-sized codebases in microservices architectures.

## 🧪 Application Test
The benchmark is based on an example from the [Baeldung Spring Boot Native tutorial](https://www.baeldung.com/spring-native-intro), with a project that has over 50 JARs in the classpath.

A small codebase is used, as Java compilation is fast, as explained [here](https://mill-build.org/blog/1-java-compile.html).

## 🛠️ Setup
The project is configured for Maven, Jeka, and Gradle.

Tests use GraalVM for consistency, which provides tools for native compilation. These tools are not needed for Jeka native builds or native Docker images.

### Versions Used
```
Java       : 21-GraalVM
Maven      : 3.9.9
Jeka       : 0.11.12
Gradle     : 8.12
Spring Boot: 3.4.1
```

### Methodology
The project builds with Maven, Jeka, and Gradle.

- Each build action is performed with every tool, with execution times recorded in the tables below.
- Every action runs twice per tool to account for initial tool or dependency fetching—only the second result is logged.
- Commands use the respective wrapper scripts: `./mvnw`, `./jeka`, and `./gradlew`.
- Times are measured via the `time` utility on macOS and `Measure-Command { .\my-command | Out-Default }` on Windows PowerShell.
- All runs occur from the IntelliJ IDEA terminal.

#### Local Development Workflow
- Benchmarks collected on high-performance macOS hardware.
- Supplementary tests on mid-range Windows hardware.
- Each command executes at least twice to ensure caching stability, recording only the fastest result.

#### Pipeline Workflow
This simulates a [GitHub Actions pipeline](https://github.com/djeang/benchmark-jeka-maven-gradle/blob/master/.github/workflows/global-caches.yml) with minimal caching (just dependencies and wrappers).

### Build Process
- **Maven**: Uses `native-maven-plugin` and `spring-boot-maven-plugin`.
- **Jeka**: Uses the `springboot-plugin`, with built-in native build support.
- **Gradle**: Uses `native-gradle-plugin` and the `org.springframework.boot` plugin.

### Command Lines
The commands executed for each tool:

| Task                                      | Maven                                      | Jeka                               | Gradle                                   |
|-------------------------------------------|--------------------------------------------|------------------------------------|------------------------------------------|
| Clean                                     | `clean`                                    | `--clean`                          | `clean`                                  |
| Compile                                | `clean compile`                            | `--clean compile`                  | `clean compileJava`                      |
| Create JAR from scratch (incl. tests)     | `clean package`                            | `--clean pack`                     | `clean build -PdisableNative`            |
| Create native executable from compiled JARs | `-Pnative native:compile`                  | `native: compile`                  | `nativeCompile`                          |
| Create Docker JVM-based image from scratch| `clean spring-boot:build-image`            | `--clean test docker: build`       | `clean bootBuildImage -PdisableNative`   |
| Create Docker native image from scratch   | `clean -Pnative spring-boot:build-image`   | `--clean test docker: buildNative` | `clean bootBuildImage`                   |

---

## 📈 Results and Analysis

### Measurements on High-End macOS Hardware
**Specs:** Apple M4 Pro, 48 GB RAM

| Task                                      | Maven    | Jeka    | Gradle  | Gradle (no daemon) |
|-------------------------------------------|----------|---------|---------|--------------------|
| Clean                                     | 0.85s    | 0.27s   | 0.36s   | 2.41s              |
| Compile                                   | 1.43s    | 0.49s   | 0.87s   | 6.69s              |
| Create JAR from scratch (incl. tests)     | 2.97s    | 1.93s   | 2.29s   | 8.99s              |
| Create native executable from compiled JARs | 1m 12s  | 1m 10s  | 1m 11s  | 1m 17s             |
| Create Docker JVM-based image from scratch| 9.71s    | 2.23s   | 7.04s   | 13.60s             |
| Create Docker native image from scratch   | 59.81s   | 59.31s  | 58.89s  | 1m 6s              |

**Observations:**
- There is little difference between the tools, except Gradle without the daemon, which is slower.
- Jeka is faster than Maven and Gradle in most tasks.
- Jeka is faster for creating JVM-based images.
- Native executables take less than one minute.
- On macOS, creating native executables is slightly faster than creating native Docker images.

**Analysis:**  
Jeka uses Dockerfile generation (dynamically building Docker contexts), which requires less processing than Paketo-based builds used by the others.

---

### Measurements on Medium Windows Hardware
**Specs:** 11th Gen Intel Core i7-1165G7, 16 GB RAM

| Task                                      | Maven   | Jeka   | Gradle | Gradle (no daemon) |
|-------------------------------------------|---------|--------|--------|--------------------|
| Clean                                     | 2.63s   | 0.29s  | 1.10s  | 8.35s              |
| Compile                                   | 4.16s   | 1.13s  | 2.15s  | 10.86s             |
| Create JAR from scratch (incl. tests)     | 10.53s  | 5.70s  | 7.13s  | 16.61s             |
| Create native executable from compiled JARs | 5m 39s | 4m 32s | 4m 41s | 4m 38s             |
| Create Docker JVM-based image from scratch| 14.49s  | 5.73s  | 8.81s  | 27.22s             |
| Create Docker native image from scratch   | 2m 53s  | 2m 45s | 2m 53s | 3m 36s             |

**Observations:**
- Jeka shows larger performance differences on lower-performance hardware.
- Most tasks take about twice as long as on macOS, but native compilation can take up to five times longer.

**Analysis:**  
GraalVM's native-image tools are less optimized on Windows, which increases times. For testing, creating native Docker images is more efficient than building native executables on the workstation.

---

### Measurements in GitHub Pipeline

| Task                                      | Maven  | Jeka  | Gradle |
|-------------------------------------------|--------|-------|--------|
| Create JAR from scratch (incl. tests)     | 7s     | 7s    | 8s     |
| Create native executable from compiled JARs | 5m 47s| 5m 30s| 5m 29s |
| Create Docker JVM-based image from scratch| 30s    | 8s    | 16s    |
| Create Docker native image from scratch   | 3m 54s | 3m 41s| 3m 35s |

**Observations:**
- Native images take at least 3 minutes more than JVM images.
- The tools perform similarly overall, but Jeka is faster for JVM Docker images compared to Gradle.

**Analysis:**  
Tools like Jeka start faster. Jeka's Ivy-based dependency management is slightly slower with empty repositories, but in environments without Docker build cache, Jeka downloads fewer images.

---

### Image Sizes
The table lists the produced image sizes. Jeka optimizes Docker layering for JVM images.

| Image             | Size  | Base Image                          |
|-------------------|-------|-------------------------------------|
| Maven JVM         | 268MB | paketobuildpacks/run-jammy-tiny     |
| Maven Native      | 148MB | paketobuildpacks/run-jammy-tiny     |
| Jeka JVM          | 245MB | eclipse-temurin:23-jre-alpine       |
| Jeka Native       | 133MB | paketobuildpacks/run-jammy-tiny     |
| Gradle JVM        | 268MB | paketobuildpacks/run-jammy-tiny     |
| Gradle Native     | 148MB | paketobuildpacks/run-jammy-tiny     |

---

## ⚙️ Configuration

The following compares the configuration required to perform the same tasks for each tool. Note the conciseness of Jeka.

### Maven `pom.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.1</version>
    </parent>

    <groupId>org.github.djeang</groupId>
    <artifactId>benchmark-maven</artifactId>
    <version>0.0.1-SNAPSHOT</version>
   
    <build>
        <plugins>
            <plugin>
                <groupId>org.graalvm.buildtools</groupId>
                <artifactId>native-maven-plugin</artifactId>
                <configuration>
                    <skipNativeTests>true</skipNativeTests>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

    <properties>
        <java.version>21</java.version>
    </properties>

</project>
```

### Jeka `jeka.properties`
```properties
jeka.java.version=21

jeka.kbean.default=project
jeka.classpath=dev.jeka:springboot-plugin
@springboot=on
```

### Gradle `build.gradle`
```groovy
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'org.graalvm.buildtools:native-gradle-plugin:0.10.3'
    }
}

plugins {
    id 'java'
    id 'org.springframework.boot' version '3.4.1'
}

apply plugin: 'io.spring.dependency-management'
if (!project.hasProperty('disableNative')) {
    apply plugin: 'org.graalvm.buildtools.native'
}

group = 'org.github.djeang'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

test {
    useJUnitPlatform()
}

repositories {
    mavenCentral()
}
```

For Maven and Gradle, GraalVM must be installed manually. Jeka handles this automatically. To ensure consistency, configure the host with a shared GraalVM:

```
export GRAALVM_HOME=~/.jeka/cache/jdks/graalvm-23
export PATH=$PATH:$GRAALVM_HOME/bin
```

## 📚 Resources
- Customize Docker Images with Jeka: [Jeka Documentation](https://jeka-dev.github.io/jeka/reference/kbeans-docker/)
