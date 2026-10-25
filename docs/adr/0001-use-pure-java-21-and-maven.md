# 0001. Use pure Java 21 and Maven, no frameworks

- **Status:** Accepted
- **Date:** 2026-10-25

## Context

This project exists to build deep understanding of server internals: sockets, wire protocols,
concurrency, non-blocking I/O, persistence and JVM behaviour. It isn't meant to ship a product.

Frameworks such as Spring Boot, Netty and Vert.x solve exactly the problems this project wants to
study. With one of them, the event loop, buffer management, thread model and wiring would all be
done before any learning happened. The project also needs to be easy to build in CI and easy to
explain in an interview.

## Decision

- **No frameworks or runtime libraries.** Only the JDK. JUnit 5 is the single dependency, test scope only.
- **Java 21 (LTS)**, set with `maven.compiler.release=21`.
- **Maven** as the build tool, with `mvn -B verify` as the single build-and-test command locally and in CI.

## Alternatives considered

- **Spring Boot / Netty / Vert.x**: production-grade, but they hide the parts being studied
  (Netty *is* the NIO event loop of Week 4). Rejected. Netty's design can still be read for reference.
- **Java 17 (LTS)**: widely deployed, but it lacks virtual threads (Week 2), record patterns and
  pattern matching for `switch` (Week 5). Rejected.
- **Latest non-LTS Java (22+)**: newer features, but a shorter support window, and the roadmap doesn't
  need anything past 21. Rejected; revisit when the next LTS lands.
- **Gradle**: faster and more flexible, but the Kotlin/Groovy build scripts add a second thing to learn
  and debug. Maven's declarative, convention-first POM is what most Java teams and CI templates assume.
  Rejected for this project.
- **No build tool (javac + scripts)**: shows exactly what a build does, but it means hand-rolling test
  running, dependency download and CI. Rejected.

## Consequences

- **Positive:** every behaviour in the server is code written in this repo and can be explained line
  by line. Java 21 gives virtual threads, records, sealed interfaces and pattern matching out of the box.
  The build is one command, and CI is a few lines.
- **Negative:** much more code to write, including things like argument parsing, config loading and
  dependency wiring. No battle-tested protocol handling, so bugs are expected (and are the point).
  Results won't match production Redis performance.
- **Follow-ups:** the Bonus week builds a mini DI container, which will need its own ADR on whether
  to keep it. Revisit the Java version at the next LTS.
