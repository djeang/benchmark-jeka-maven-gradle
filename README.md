# Benchmark Jeka-Maven-Gradle for building Springboot Images

Benchmark build speed on a diifreent scenario of a given Spring-Boot app.

The application is a TODO list accessible via REST and GraphQL APIs.

## Methodology

The project is configured to build with both *Maven*, *Jeka* and *Gradle*.

- Some listed actions are executed using each build tool. Execution times are mentioned in tables below.
- Each action, for each build tool, is executed subsequently twice, in order that tool/image fetching won't impact the result. We keep only the last measure.
- We use local scripts  as `./mvnw`, `./jeka` or `./gradle` for running build tool commands.
- Elapsed times are measured using `time` utility on *MacOS* and `Measure-Command { .\my command | Out-Default}` on *Windows Powershell*
- Commands are executed in IntelliJ terminal.

## Version Used
```
Java:       21-temurin
GraalVM:    23
Maven:      3.9.9
Jeka:       0.11.12
Gradle:     8.12
SpringBoot: 3.4.1
```

## Command lines:

|                                             | Maven                                    | Jeka                               | Gradle                                 |
|---------------------------------------------|------------------------------------------|------------------------------------|----------------------------------------|
| Clean                                       | `clean`                                  | `--clean`                          | `clean`                                |
| Compile                                     | `clean compile`                          | `--clean compile`                  | `clean compileJava`                    |
| Create Jar from scratch (including tests)   | `clean package`                          | `--clean pack`                     | `clean build -PdisableNative`          |
| Create native executable from compiled jars | `-Pnative native:compile`                | `native: compile`                  | `nativeCompile`                        |
| Create Docker JVM-based image from scratch  | `clean spring-boot:build-image`          | `--clean test docker: build`       | `clean bootBuildImage -PdisableNative` |
| Create Docker native image from scratch     | `clean -Pnative spring-boot:build-image` | `--clean test docker: buildNative` | `clean bootBuildImage`                 |


## Measures running on Apple M4 pro, RAM 48 GB:

|                                             | Maven   | Jeka   | Gradle | Gradle (no daemon) |
|---------------------------------------------|---------|--------|--------|--------------------|
| Clean                                       | 0.85s   | 0.27s  | 0.36s  | 2.41s              |
| Compile                                     | 1.43s   | 0.49s  | 0.87s  | 6.69s              |
| Create Jar from scratch (including tests)   | 2.97s   | 1.93s  | 1.19s  | 7.20s              |
| Create native executable from compiled jars | 1m 12s  | 1m 10s | 1m 11s | 1m 17s             |
| Create Docker JVM-based image from scratch  | 9.71s   | 2.23s  | 7.04s  | 13.60s             |
| Create Docker native image from scratch     | 59.818s | 59.31s | 58.89s | 1m 06s             |

## Measures running on Windows 11th Gen Intel(R) Core(TM) i7-1165G7 @2.80GHz, RAM 16 GB:

|                                             | Maven  | Jeka   | Gradle | Gradle (no daemon) |
|---------------------------------------------|--------|--------|--------|--------------------|
| Clean                                       | 2.63s  | 0.39s  | 1.10s  | 8.35s              |
| Compile                                     | 4.16s  | 1.13s  | 2.15s  | 10.86s             |
| Create Jar from scratch (including tests)   | 10.53s | 5.70s  | 3.15s  | 13.31s             |
| Create native executable from compiled jars | 5m 39s | 4m 32s | 4m 41s | 4m 38s             |
| Create Docker JVM-based image from scratch  | 14.49s | 5.73s  | 5.40s  | 27.22s             |
| Create Docker native image from scratch     | 2m 53s | 2m 45s | 2m 53s | 3m 36s             |


## Image Size


|               | Size  | Base-Image                       |        | 
|---------------|-------|----------------------------------|--------|
| Maven JVM     | 268MB | paketobuildpacks/run-jammy-tiny  |        | 
| Maven Native  | 148MB | paketobuildpacks/run-jammy-tiny  |        | 
| JeKa JVM      | 245MB | eclipse-temurin:23-jre-alpine    |        | 
| JeKa Native   | 133MB | paketobuildpacks/run-jammy-tiny  |        |
| Gradle JVM    | 268MB | paketobuildpacks/run-jammy-tiny  |        | 
| Gradle Native | 148MB | paketobuildpacks/run-jammy-tiny  |        | 
