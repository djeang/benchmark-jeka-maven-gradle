-------- 
Skeleton
--------
Benchmarking Build Tools for Spring Boot: Maven, Gradle, and Jeka
1. Introduction
   Objective of the article: Compare the performance of Maven, Gradle, and Jeka in building a Spring Boot application.
   Importance of build times in modern development, especially for CI/CD pipelines.
2. Description of the Test Application
   Overview of the Spring Boot application used for the benchmark.
   Key dependencies and features.
   Rationale for choosing this application to represent typical projects.
3. Benchmark Setup
   Details of the testing environment:
   Hardware specifications (CPU, RAM, disk, etc.).
   Operating system and tool versions (Maven, Gradle, Jeka).
   Methodology:
   Metrics measured (build time, Docker image size, etc.).
   Build scenarios (JVM-based, native, Docker image).
4. Build Process for Each Tool
   Maven
   Plugins used.
   Specific configuration.
   Gradle
   Gradle scripts and plugins.
   Any optimizations applied.
   Jeka
   Jeka build script configuration.
   Unique aspects compared to Maven and Gradle.
5. Results and Analysis
   Presenting data in tables or charts:
   Build times for each scenario.
   Docker image sizes.
   Comparative performance analysis:
   Speed.
   Ease of configuration.
   Additional observations (e.g., native builds handling).
6. Strengths and Weaknesses of Each Tool
   A table or summary comparing pros and cons.
   Considerations for choosing a tool based on the context (project size, team preferences, etc.).
7. Conclusion
   Summary of the results.
   Recommendations based on specific use cases.
   Highlighting Jeka's potential as a lightweight and fast alternative.
8. Appendices
   Configuration scripts for Maven, Gradle, and Jeka.
   Commands used to reproduce the benchmark.
   Link to the test application's source code (if open source).


---------------

# Benchmark Jeka-Maven-Gradle for building Springboot Images

## Objective

Compare the performance of *Maven*, *Gradle*, and *Jeka* in building a Cloud-Native Spring Boot application, including 
creation of Docker and native images.

This compares in following situations:

  - Powerfull Develop workstation where all caches ar available
  - Medium windows workstation where all caches ar available
  - Mimic CI-CD pipeline where all cache disabled (no docker cache, no wrapped tool cached, no java deps cached) 

We want to compare vanilla tools performance from basic configuration without more complex optimizing settings 
(build caches, ...). 
THis should be the settings for short to medium sizes codebase, popular in micro-services architectures, 
that springboot-native generally target at.

## Application Test

Example from [Baeldung Springboot native tutorial](https://www.baeldung.com/spring-native-intro) where deps reslve in - > 50 jars in classpath.

Small code base -> ok (java compilation is quite fast: https://mill-build.org/blog/1-java-compile.html).

## Setup

We configure the project to be build with the 3 build tools.

### Version Used
```
Java:       21-temurin
GraalVM:    23
Maven:      3.9.9
Jeka:       0.11.12
Gradle:     8.12
SpringBoot: 3.4.1
```

### Methodology

The project is configured to build with both *Maven*, *Jeka* and *Gradle*.

- Some listed actions are executed using each build tool. Execution times are mentioned in tables below.
- Each action, for each build tool, is executed subsequently twice, in order that tool/image fetching won't impact the result. We keep only the last measure.
- We use local scripts  as `./mvnw`, `./jeka` or `./gradle` for running build tool commands.
- Elapsed times are measured using `time` utility on *MacOS* and `Measure-Command { .\my command | Out-Default}` on *Windows Powershell*
- Commands are executed in IntelliJ terminal.
- 
####  Development time scenario

- Measure on powerfull MacOS 
- Measure in medium Windows hardware

- Run each command at twice or more in order to have everything cached. Take best result.

#### Pipeline in naked environment

Only Java installed:
  - No Build tools (use Maven, Gradle or Jeka wrappers)
  - No Dependency fetched (repo)
  - No Docker caches

We run the `clean-caches.sh` script between each execution.

## Build Process

For maven, we use the `native-maven-plugin` and `spring-boot-maven-plugin` plugins.

For Gradle, we use the `native-gradle-plugin` and `org.springframework.boot`plugins.

For Jeka, we use only the `springboot-plugin` as native support is bundled in Jeka.

### Command lines

The following methods are executed:

|                                             | Maven                                    | Jeka                               | Gradle                                 |
|---------------------------------------------|------------------------------------------|------------------------------------|----------------------------------------|
| Clean                                       | `clean`                                  | `--clean`                          | `clean`                                |
| Compile                                     | `clean compile`                          | `--clean compile`                  | `clean compileJava`                    |
| Create Jar from scratch (including tests)   | `clean package`                          | `--clean pack`                     | `clean build -PdisableNative`          |
| Create native executable from compiled jars | `-Pnative native:compile`                | `native: compile`                  | `nativeCompile`                        |
| Create Docker JVM-based image from scratch  | `clean spring-boot:build-image`          | `--clean test docker: build`       | `clean bootBuildImage -PdisableNative` |
| Create Docker native image from scratch     | `clean -Pnative spring-boot:build-image` | `--clean test docker: buildNative` | `clean bootBuildImage`                 |


## Result and analysis

### Measures on MacOS high-end hardware

Specs: Apple M4 pro, RAM 48 GB.

|                                             | Maven   | Jeka   | Gradle | Gradle (no daemon) |
|---------------------------------------------|---------|--------|--------|--------------------|
| Clean                                       | 0.85s   | 0.27s  | 0.36s  | 2.41s              |
| Compile                                     | 1.43s   | 0.49s  | 0.87s  | 6.69s              |
| Create Jar from scratch (including tests)   | 2.97s   | 1.93s  | 2.39s  | 8.99s              |
| Create native executable from compiled jars | 1m 12s  | 1m 10s | 1m 11s | 1m 17s             |
| Create Docker JVM-based image from scratch  | 9.71s   | 2.23s  | 7.04s  | 13.60s             |
| Create Docker native image from scratch     | 59.818s | 59.31s | 58.89s | 1m 06s             |

Observations:

  - Generally there is no big gap performance from a build tool from another, except concerning Gradle with daemon which is significantly slower.
  - Jeka is sensibly faster than Gradle and Maven for most of the tasks.
  - Jeka is significantly faster for creating JVM-base images.
  - Creating native executable images is quite long but still under 1 minute.
  - Creating a native executable takes longer on MacOS than generating a Docker native image.

Analysis:
  
Jeka uses pure Dockerfile technology to generate the build (the Docker context build is generated dynamically),
which requires less processing than the *Paketo* builds.


### Measures on Windows Medium-Hardware

Specs: 11th Gen Intel(R) Core(TM) i7-1165G7 @2.80GHz, RAM 16 GB

|                                             | Maven  | Jeka   | Gradle | Gradle (no daemon) |
|---------------------------------------------|--------|--------|--------|--------------------|
| Clean                                       | 2.63s  | 0.39s  | 1.10s  | 8.35s              |
| Compile                                     | 4.16s  | 1.13s  | 2.15s  | 10.86s             |
| Create Jar from scratch (including tests)   | 10.53s | 5.70s  | 7.13s  | 16.61s             |
| Create native executable from compiled jars | 5m 39s | 4m 32s | 4m 41s | 4m 38s             |
| Create Docker JVM-based image from scratch  | 14.49s | 5.73s  | 8.81s  | 27.22s             |
| Create Docker native image from scratch     | 2m 53s | 2m 45s | 2m 53s | 3m 36s             |

Observations:

- JeKa speed difference reveals even greater when using lower hardware
- There is an average x2 extra time compared with High-end hardware, except for creating native which is x5

Analysis:

The native executable has been created using *GraalVM native image* tools, which seems much less optimized on Windows.

Creating native executable on Developer worstation might not be a good idea for testing strategy, 
creating native Docker image is much more efficient.


### Measure with Zero Cache

Specs: Apple M4 pro, RAM 48 GB.

|                                             | Maven  | Jeka   | Gradle | Gradle (no daemon) |
|---------------------------------------------|--------|--------|--------|--------------------|
| Clean                                       | 7.86s  | 1.32s  | 25.68s | 22.74s             |
| Compile                                     | 19.40s | 24.58s | 33.82s | 34.18s             |
| Create Jar from scratch (including tests)   | 27.15s | 38.87s | fail   | 42.59s             |
| Create native executable from compiled jars | 1m 38s | 1m 31s | fail   | 1m 53s             |
| Create Docker JVM-based image from scratch  | 1m 27s | 31.40s | fail   | 2m 04s             |
| Create Docker native image from scratch     | 2m 26s | 2m 28s | fail   | 2m 40s             |

Observations:

- Gradle suffers from time for installing form wrapper
- Non basic actions fail with Gradle daemon (missing some caches?)
- Jeka start much quicker but loose time when downloading dependencies is needed
- The more the operation takes, the less Gradle start-up slowliness is impacting
- Jeka remains significantly faster to build JVM-bases images

Analysis:

- The smaller the build tool is, the faster it starts
- Jeka dependency management is based on *IVY* which downloads dependencies slower, impacting when local repo is empty.
- When Docker build cache is absent, Jeka advantages get bigger as it requires less image downloading.

### Image Size

This are the produced image size. Note that for JeKa, the JVM image is properly layered for optimizing Docker caching.

|               | Size  | Base-Image                       |        
|---------------|-------|----------------------------------|
| Maven JVM     | 268MB | paketobuildpacks/run-jammy-tiny  | 
| Maven Native  | 148MB | paketobuildpacks/run-jammy-tiny  | 
| JeKa JVM      | 245MB | eclipse-temurin:23-jre-alpine    | 
| JeKa Native   | 133MB | paketobuildpacks/run-jammy-tiny  | 
| Gradle JVM    | 268MB | paketobuildpacks/run-jammy-tiny  | 
| Gradle Native | 148MB | paketobuildpacks/run-jammy-tiny  | 


### Ease of Configuration

Here are the build configuration used by both tools. For bievty and equité we don't mention dependencies,
as Jeka declares it in a [specific *dependencies.tct* file](dependencies.txt).

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

Analysis:
 
As it promises, JeKa wins at configuration simplicity, none the configuration much more concise, but also 
we don't need to install GraalVM or even the JDK-21 as it installs it automatically when missing.

## Conclusion

Booth 3 tools are workable for making Springboot Docker image,
Choosing a build tool is a matter of choice: some prefer to stick with traditional well-etablished tools, 
other seeks for simplicy, performance or innovation.


Resources:
  - Customize Docker Images with Jeka: https://jeka-dev.github.io/jeka/reference/kbeans/#docker

