## Maven philosophy

> **In short**: Maven is a declarative build tool: the `pom.xml` describes what the project is (coordinates, dependencies, plugins), not how to build it. It relies on convention over configuration (a standard directory layout and a standard lifecycle), so every Maven project builds the same way with `mvn verify`. Dependencies are downloaded from repositories, with their transitive dependencies.

- **Convention over configuration**: sources in `src/main/java`, resources in `src/main/resources`, tests in `src/test/java`, output in `target/`. Following the conventions means almost nothing to configure.
- **Declarative**: the POM (Project Object Model) describes the project; the build logic lives in plugins. Unlike Ant (2000), where each project wrote its own build script step by step.
- **Standard lifecycle**: the same phases for every project (`compile`, `test`, `package`, `install`...), so any developer or CI knows how to build any project.
- **Dependency management**: a dependency is declared by its coordinates; Maven downloads it and its own dependencies from a repository (Maven Central, or a company Nexus/Artifactory), and caches them in `~/.m2/repository`.

Compared with Gradle: Gradle keeps the conventions and dependency management, but the build is a script (Kotlin or Groovy), more flexible, and faster thanks to its incremental build and cache. Maven is more rigid, but every project looks the same.

## The POM and coordinates

> **In short**: A project is identified by its coordinates `groupId:artifactId:version` (GAV). A `-SNAPSHOT` version is a work in progress that can change, a release version is immutable. A POM can inherit from a parent POM (configuration, dependency versions), like every Spring Boot project inheriting from `spring-boot-starter-parent`.

```xml
<project>
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.0</version>
  </parent>

  <groupId>com.shop</groupId>
  <artifactId>order-service</artifactId>
  <version>1.2.0-SNAPSHOT</version>
  <packaging>jar</packaging>

  <properties>
    <java.version>25</java.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId> <!-- no version: managed by the parent -->
    </dependency>
  </dependencies>
</project>
```

- **`SNAPSHOT` vs release**: `1.2.0-SNAPSHOT` can be republished and Maven re-downloads it; `1.2.0` is published once and never changes. Releases must not depend on snapshots.
- **Effective POM**: the POM merged with its parents and Maven's defaults, shown by `mvn help:effective-pom`.
- **Profiles**: configuration activated on demand (`-Pprod`) or by a condition (an environment variable, the JDK version).

## Lifecycle, phases and plugins

> **In short**: The default lifecycle is a sequence of phases (`validate`, `compile`, `test`, `package`, `verify`, `install`, `deploy`); running a phase runs all the previous ones. Phases do nothing by themselves: plugin goals are bound to them (Surefire runs the unit tests in `test`, the jar plugin builds the jar in `package`). `mvn verify` is the usual full build; `install` also copies the artifact to the local repository.

| Phase     | Does (with the default plugins)                                                       |
| --------- | ------------------------------------------------------------------------------------- |
| `compile` | compiles `src/main/java` (`maven-compiler-plugin`)                                    |
| `test`    | runs the unit tests, `*Test` classes (`maven-surefire-plugin`)                        |
| `package` | builds the jar / war (`maven-jar-plugin`, or Spring Boot's repackage)                 |
| `verify`  | runs the integration tests, `*IT` classes (`maven-failsafe-plugin`), and other checks |
| `install` | copies the artifact into `~/.m2/repository`, for the other local projects             |
| `deploy`  | publishes the artifact to the remote repository (Nexus)                               |

- Three lifecycles: `clean` (deletes `target/`), `default` (the build), `site` (documentation). `mvn clean verify` chains two of them.
- A goal can also be run directly, outside of any phase: `mvn dependency:tree`, `mvn spring-boot:run`.
- `install` is only needed when another local project depends on this artifact; in a multi-module build, `verify` is enough (see below).
- Failsafe runs the integration tests in `integration-test` and only fails the build in `verify`, so that `post-integration-test` (stopping a server, a container) still runs.
- `-DskipTests` skips running the tests; `-Dmaven.test.skip=true` doesn't even compile them.

## Dependencies: scopes, transitivity, versions

> **In short**: The scope says on which classpath a dependency goes: `compile` (default, everywhere), `provided` (compile but not packaged: the servlet API), `runtime` (not for compiling: a JDBC driver), `test`. Dependencies are transitive; on a version conflict, the nearest declaration in the tree wins, not the highest version. `dependencyManagement` and BOMs centralize versions without adding dependencies.

| Scope      | Compile             | Tests | Packaged / runtime              | Example                             |
| ---------- | ------------------- | ----- | ------------------------------- | ----------------------------------- |
| `compile`  | yes                 | yes   | yes                             | Spring, Jackson                     |
| `provided` | yes                 | yes   | no, supplied by the environment | servlet API in a Tomcat war, Lombok |
| `runtime`  | no                  | yes   | yes                             | PostgreSQL JDBC driver              |
| `test`     | no (only test code) | yes   | no                              | JUnit, Mockito, Testcontainers      |

- **Transitive dependencies**: depending on `spring-boot-starter-web` brings Spring MVC, Jackson, Tomcat... `mvn dependency:tree` shows the tree.
- **Version conflicts**: two paths bring two versions of the same library; Maven keeps the **nearest** one in the tree (at equal depth, the first declared), even if it's older. A new version of a library can thus be silently replaced by an old one: declaring the version explicitly, or in `dependencyManagement`, fixes it.
- **Exclusions**: `<exclusions>` removes an unwanted transitive dependency (e.g. Tomcat to use Jetty).
- **`dependencyManagement` vs `dependencies`**: `dependencies` adds the dependency; `dependencyManagement` only fixes its version (and scope) for when it's declared, in this project or its children.
- **BOM** (Bill of Materials): a POM that only contains a `dependencyManagement`, imported with `<scope>import</scope>` and `<type>pom</type>`. It gives a set of versions known to work together (`spring-boot-dependencies`, `testcontainers-bom`), without inheriting from a parent.

## Multi-module projects

> **In short**: A parent POM with `<packaging>pom</packaging>` lists modules; Maven's reactor builds them in dependency order in one command. The parent shares configuration and versions by inheritance. `-pl` / `-am` build only one module and what it needs, and the Maven Wrapper (`mvnw`) pins the Maven version of the project.

```text
shop/
├── pom.xml          # <packaging>pom</packaging>, <modules>: domain, application, web
├── domain/pom.xml
├── application/pom.xml   # depends on domain
└── web/pom.xml           # depends on application, builds the Spring Boot jar
```

- **Aggregation** (`<modules>` in the parent: build them together) and **inheritance** (`<parent>` in the child: share configuration) are two separate mechanisms, usually combined in the same parent POM.
- **Reactor**: computes the build order from the dependencies between modules (`domain` → `application` → `web`); a module uses the freshly built version of its siblings, no `install` needed.
- `mvn verify -pl web -am`: builds only `web` and the modules it depends on (`-am`: also make).
- **Maven Wrapper**: `./mvnw` downloads and uses the Maven version declared in the project, so every developer and the CI use the same one.
- Modules are also a way to enforce an architecture: a `domain` module without the Spring dependency can't use it (see [Hexagonal architecture](../../cross-cutting/HEXAGONAL_ARCHITECTURE.md#rules-to-respect)).
