# Benchmark Jeka-Maven-Gradle for building Springboot Images

Benchmark build speed on a diifreent scenario of a given Spring-Boot app.

The application is a TODO list accessible via REST and GraphQL APIs.

Version used:
```
Java:       21-temurin
GraalVM:    23
Maven:      3.9.9
Jeka:       0.11.12
Gradle:     8.12
SpringBoot: 3.4.1
```


Command lines:

|                                             | Maven                                    | Jeka                                        | Gradle              |
|---------------------------------------------|------------------------------------------|---------------------------------------------|---------------------|
| Compile                                     | `clean compile`                          | `--clean project: compile`                  | `clean compileJava` |
| Create Jar from scratch (including tests)   | `clean package`                          | `--clean project: pack`                     | `clean build`       |
| Create native executable from compiled jars | `-Pnative native:compile`                | `native: compile`                           | Donnée 9            |
| Create Docker JVM-based image from scratch  | `clean spring-boot:build-image`          | `--clean project: test docker: build`       | Donnée 12           |
| Create Docker native image from scratch     | `clean -Pnative spring-boot:build-image` | `--clean project: test docker: buildNative` | Donnée 15           |


Result running on Apple M4 pro, RAM 48 GB:

|                                             | Maven   | Jeka   | Gradle    | Gradle (no daemon) |
|---------------------------------------------|---------|--------|-----------|--------------------|
| Compile                                     | 1.43s   | 0.49s  | 0.87s     | 6.69s              |
| Create Jar from scratch (including tests)   | 2.97s   | 1.93s  | 1.19s     | 7.20s              |
| Create native executable from compiled jars | 1m 12s  | 1m 10s | Donnée 9  |                    |
| Create Docker JVM-based image from scratch  | 9.71s   | 2.23s  | Donnée 12 |                    |
| Create Docker native image from scratch     | 59.818s | 59.31s | Donnée 15 |                    |