# Benchmark Jeka-Maven-Gradle for building Springboot Images

Benchmark build speed on a diifreent scenario of a given Spring-Boot app.

The application is a TODO list accessible via REST and GraphQL APIs.

## Methodology

The project is configured to build with both *Maven*, *Jeka* and *Gradle*.

- Some listed actions are executed using each build tool. Execution times are mentioned in tables below.
- Each action, for each build tool, is executed subsequently twice, in order that tool/image fetching won't impact the result. We keep only the last measure.
- We use local scripts  as `./mvnw`, `./jeka` or `./gradle` for running build tool commands.

## Version Used
```
Java:       21-temurin
GraalVM:    23
Maven:      3.9.9
Jeka:       0.11.12
Gradle:     8.12
SpringBoot: 3.4.1
```

Command lines:

|                                             | Maven                                    | Jeka                                        | Gradle                                 |
|---------------------------------------------|------------------------------------------|---------------------------------------------|----------------------------------------|
| Clean                                       | `clean`                                  | `--clean`                                   | `clean`                                |
| Compile                                     | `clean compile`                          | `--clean project: compile`                  | `clean compileJava`                    |
| Create Jar from scratch (including tests)   | `clean package`                          | `--clean project: pack`                     | `clean build`                          |
| Create native executable from compiled jars | `-Pnative native:compile`                | `native: compile`                           | `nativeCompile`                        |
| Create Docker JVM-based image from scratch  | `clean spring-boot:build-image`          | `--clean project: test docker: build`       | `clean bootBuildImage -PdisableNative` |
| Create Docker native image from scratch     | `clean -Pnative spring-boot:build-image` | `--clean project: test docker: buildNative` | `clean bootBuildImage`                 |


Result running on Apple M4 pro, RAM 48 GB:

|                                             | Maven   | Jeka   | Gradle | Gradle (no daemon) |
|---------------------------------------------|---------|--------|--------|--------------------|
| Clean                                       | 0.85s   | 0.27s  | 0.36s  | 2.41s              |
| Compile                                     | 1.43s   | 0.49s  | 0.87s  | 6.69s              |
| Create Jar from scratch (including tests)   | 2.97s   | 1.93s  | 1.19s  | 7.20s              |
| Create native executable from compiled jars | 1m 12s  | 1m 10s | 1m 11s | 1m 17s             |
| Create Docker JVM-based image from scratch  | 9.71s   | 2.23s  | 7.04s  | 13.60s             |
| Create Docker native image from scratch     | 59.818s | 59.31s | 58.89s | 1m 06s             |