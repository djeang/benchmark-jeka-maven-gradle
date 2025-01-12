# Cloud-Native Spring Boot Build Benchmark: Maven vs JeKa vs Gradle

## Objective

Compare the performance of *Maven*, *JeKa*, and *Gradle* in building a Spring Boot cloud-native application, including creating both JVM and native Docker images.

This comparison is performed in the following scenarios:
- **Development on workstations**: Tested on a high-performance MacBook and a medium-performance Windows machine (with all caches available).
- **CI/CD pipeline simulation**: All caches disabled (no Docker cache, no wrapper tool cache, no Java dependencies cache).

The goal is to evaluate the performance of the tools using their vanilla configurations, without peripheral tooling 
optimizations (build caches, ).

This setup is designed to reflect, at reasonable effort, typical conditions for small to medium-sized codebases,
which are common in microservices architectures and generally targeted by Spring Boot cloud-native applications.

## Application Test
The benchmarks are based on an example from the [Baeldung Spring Boot Native tutorial](https://www.baeldung.com/spring-native-intro), which resolves over 50 JARs in the classpath.

The small codebase is used intentionally since Java compilation is relatively fast, as discussed [here](https://mill-build.org/blog/1-java-compile.html).

## Setup
The project is configured to be built with the three mentioned build tools.

### Versions Used
```
Java:       21-temurin
GraalVM:    23
Maven:      3.9.9
Jeka:       0.11.12
Gradle:     8.12
SpringBoot: 3.4.1
```

### Methodology
The project is configured to build using *Maven*, *Jeka*, and *Gradle*.
- The listed actions are executed with each build tool. Execution times are provided in the tables below.
- Each action for each build tool is performed twice to eliminate the impact of tool or image fetching. Only the second (final) result is recorded.
- Local scripts such as `./mvnw`, `./jeka`, and `./gradle` are used to execute build tool commands.
- Elapsed times are measured with the `time` utility on *macOS* and `Measure-Command { .\my command | Out-Default }` on *Windows PowerShell*.
- All commands are executed from the IntelliJ terminal.

#### Development Time Scenario
- Measurements are taken using high-performance *macOS* hardware.
- Measurements are also taken on medium-tier *Windows* hardware.
- Each command is executed at least twice to ensure proper caching, with only the best results being considered.

#### Pipeline in a Clean Environment
This involves a "naked" setup with only Java installed:
- No build tools pre-installed (Maven, Gradle, and Jeka wrappers are used).
- No dependencies fetched (new repository setup).
- No Docker caches.
  Between each execution, the `clean-caches.sh` script is run to ensure a clean environment.

### Build Process
For Maven, we use the `native-maven-plugin` and `spring-boot-maven-plugin` plugins.  
For Gradle, we use the `native-gradle-plugin` and `org.springframework.boot` plugins.  
For Jeka, we use only the `springboot-plugin`, as native support is directly integrated into Jeka.

### Command Lines
The following commands are executed for each build tool:

|                                             | Maven                                    | Jeka                               | Gradle                                 |
|---------------------------------------------|------------------------------------------|------------------------------------|----------------------------------------|
| Clean                                       | `clean`                                  | `--clean`                          | `clean`                                |
| Compile                                     | `clean compile`                          | `--clean compile`                  | `clean compileJava`                    |
| Create Jar from scratch (including tests)   | `clean package`                          | `--clean pack`                     | `clean build -PdisableNative`          |
| Create native executable from compiled jars | `-Pnative native:compile`                | `native:compile`                   | `nativeCompile`                        |
| Create Docker JVM-based image from scratch  | `clean spring-boot:build-image`          | `--clean test docker:build`        | `clean bootBuildImage -PdisableNative` |
| Create Docker native image from scratch     | `clean -Pnative spring-boot:build-image` | `--clean test docker:buildNative`  | `clean bootBuildImage`                 |

---

## Results and Analysis

### Measurements on High-End macOS Hardware
**Specs:** Apple M4 Pro, 48 GB RAM

|                                             | Maven   | Jeka   | Gradle | Gradle (no daemon) |
|---------------------------------------------|---------|--------|--------|--------------------|
| Clean                                       | 0.85s   | 0.27s  | 0.36s  | 2.41s              |
| Compile                                     | 1.43s   | 0.49s  | 0.87s  | 6.69s              |
| Create Jar from scratch (including tests)   | 2.97s   | 1.93s  | 2.39s  | 8.99s              |
| Create native executable from compiled jars | 1m 12s  | 1m 10s | 1m 11s | 1m 17s             |
| Create Docker JVM-based image from scratch  | 9.71s   | 2.23s  | 7.04s  | 13.60s             |
| Create Docker native image from scratch     | 59.81s  | 59.31s | 58.89s | 1m 6s              |

**Observations:**
- There is no significant performance gap between build tools except when Gradle operates without the daemon, which is noticeably slower.
- Jeka is consistently faster than both Maven and Gradle for most tasks.
- Jeka significantly outperforms others for creating JVM-based images.
- While creating native executables is time-consuming, it remains below one minute.
- Generating native executables takes longer on macOS compared to creating Docker native images.

**Analysis:**  
Jeka uses pure Dockerfile technology to generate builds (Docker context is dynamically generated), requiring less processing compared to *Paketo* builds.

---

### Measurements on Medium Windows Hardware
**Specs:** 11th Gen Intel Core i7-1165G7, 16 GB RAM

|                                             | Maven  | Jeka   | Gradle | Gradle (no daemon) |
|---------------------------------------------|--------|--------|--------|--------------------|
| Clean                                       | 2.63s  | 0.39s  | 1.10s  | 8.35s              |
| Compile                                     | 4.16s  | 1.13s  | 2.15s  | 10.86s             |
| Create Jar from scratch (including tests)   | 10.53s | 5.70s  | 7.13s  | 16.61s             |
| Create native executable from compiled jars | 5m 39s | 4m 32s | 4m 41s | 4m 38s             |
| Create Docker JVM-based image from scratch  | 14.49s | 5.73s  | 8.81s  | 27.22s             |
| Create Docker native image from scratch     | 2m 53s | 2m 45s | 2m 53s | 3m 36s             |

**Observations:**
- Jeka demonstrates even greater speed advantages on lower-performance hardware.
- Tasks generally require twice as much time on medium hardware compared to macOS, except native compilation, which can take five times longer.

**Analysis:**  
Native executables are created using *GraalVM native-image* tools, which seem less optimized on Windows. For testing, creating a native Docker image is more efficient than building native executables directly on the developer's workstation.

---

### Measurements with Zero Cache
**Specs:** Apple M4 Pro, 48 GB RAM

|                                             | Maven  | Jeka   | Gradle | Gradle (no daemon) |
|---------------------------------------------|--------|--------|--------|--------------------|
| Clean                                       | 7.86s  | 1.32s  | 25.68s | 22.74s             |
| Compile                                     | 19.40s | 24.58s | 33.82s | 34.18s             |
| Create Jar from scratch (including tests)   | 27.15s | 38.87s | fail   | 42.59s             |
| Create native executable from compiled jars | 1m 38s | 1m 31s | fail   | 1m 53s             |
| Create Docker JVM-based image from scratch  | 1m 27s | 31.40s | fail   | 2m 4s              |
| Create Docker native image from scratch     | 2m 26s | 2m 28s | fail   | 2m 40s             |

**Observations:**
- Gradle struggles with wrapper installation time.
- Complex actions fail on Gradle without a cache (possibly due to missing prerequisites).
- Jeka starts quickly but loses time when dependencies are not pre-cached.
- For longer operations, Gradle’s slow startup has less impact.
- Jeka remains superior for creating JVM-based images in fresh environments.

**Analysis:**  
Smaller build tools like Jeka start faster. However, Jeka’s dependency management relies on *Ivy*, which is slower when repos are empty. In environments without a Docker build cache, Jeka has an advantage as it downloads fewer images.

---

### Image Size
The table lists the produced image sizes. Note that Jeka optimizes Docker caching by properly layering JVM images.

|               | Size  | Base Image                       |
|---------------|-------|----------------------------------|
| Maven JVM     | 268MB | paketobuildpacks/run-jammy-tiny  |
| Maven Native  | 148MB | paketobuildpacks/run-jammy-tiny  |
| Jeka JVM      | 245MB | eclipse-temurin:23-jre-alpine    |
| Jeka Native   | 133MB | paketobuildpacks/run-jammy-tiny  |
| Gradle JVM    | 268MB | paketobuildpacks/run-jammy-tiny  |
| Gradle Native | 148MB | paketobuildpacks/run-jammy-tiny  |

---

### Ease of Configuration

Maven pom.xml
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

Jeka jeka.properties
```properties
jeka.java.version=21

jeka.default.kbean=project
jeka.inject.classpath=dev.jeka:springboot-plugin
@springboot=
```

Gradle build.gradle
```groovy
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'org.graalvm.buildtools:native-gradle-plugin:0.10.3'  // Replace with the correct plugin version and plugin ID
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
        languageVersion = JavaLanguageVersion.of(21) // Specify the Java version (e.g., 11, 17)
    }
}

test {
    useJUnitPlatform()
}

repositories {
    mavenCentral()
}
```

For both *Maven* and *Gradle* we need to install *GraalVM*. This is not needed for *JeKa* as it automatically download it.
We setup the host machine so that all build tool use the same *GraalVM* installation.

```shell
export GRAALVM_HOME=~/.jeka/cache/jdks/graalvm-23
export PATH=$PATH:$GRAALVM_HOME/bin
```

## Analysis:

JeKa excels in configuration simplicity. Not only is the configuration much more concise, but there is no need to manually install GraalVM or JDK 21, as JeKa automatically installs them if they are missing.

## Conclusion:

All three tools are suitable for creating Spring Boot Docker images. 
Choosing a build tool depends on personal preference: some developers stick with traditional, well-established tools, while others opt for simplicity, performance, or innovation.

## Resources:
- Customize Docker Images with JeKa: https://jeka-dev.github.io/jeka/reference/kbeans-docker/)