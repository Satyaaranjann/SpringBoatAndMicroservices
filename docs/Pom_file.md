# Spring Boot `pom.xml` — Detailed Guide

Covers a typical Maven `pom.xml` for a Spring Boot REST API / microservice project, section by section.

---

## Full Sample `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
        <relativePath/>
    </parent>

    <groupId>com.company</groupId>
    <artifactId>order-service</artifactId>
    <version>1.0.0</version>
    <name>order-service</name>
    <description>Order microservice for e-commerce platform</description>
    <packaging>jar</packaging>

    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2023.0.2</spring-cloud.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- Core Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Database -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>

        <!-- Microservices: Eureka + Feign + Config -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>

        <!-- Resilience4j -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
        </dependency>

        <!-- Actuator (health/metrics) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Security -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- MapStruct -->
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>1.5.5.Final</version>
        </dependency>

        <!-- Swagger / OpenAPI -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.5.0</version>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
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

</project>
```

---

## Section-by-Section Explanation

### 1. `<parent>` — Spring Boot Parent POM

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>
```

Inherits sensible defaults: Java compiler settings, UTF-8 encoding, dependency version management (so you don't specify versions for most Spring dependencies), and plugin configuration. This is the biggest reason Spring Boot POMs stay short — versions are managed centrally.

### 2. Project Coordinates (GAV)

```xml
<groupId>com.company</groupId>
<artifactId>order-service</artifactId>
<version>1.0.0</version>
```

- **groupId** — your organization/company namespace (reverse domain convention)
- **artifactId** — the project/module name, becomes the JAR filename
- **version** — your app's version (`1.0.0`, or `1.0.0-SNAPSHOT` during active development)

### 3. `<packaging>`

```xml
<packaging>jar</packaging>
```

`jar` for a standalone Spring Boot app (embedded server) — the modern default. `war` only if deploying to an external servlet container like a standalone Tomcat.

### 4. `<properties>`

```xml
<java.version>17</java.version>
<spring-cloud.version>2023.0.2</spring-cloud.version>
```

Custom variables reused throughout the file. `java.version` tells the parent POM which JDK to compile against. `spring-cloud.version` must be compatible with your Spring Boot version — mismatches are a common source of dependency conflicts in microservices.

### 5. `<dependencyManagement>` — Spring Cloud BOM

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

This imports Spring Cloud's Bill of Materials (BOM) — it centrally manages compatible versions for Eureka, Feign, Config Server, Gateway, etc. Only needed if you're building microservices; skip this for a plain REST API.

### 6. `<dependencies>` — Grouped by Purpose

| Dependency | Purpose |
|---|---|
| `spring-boot-starter-web` | REST controllers, embedded Tomcat, Jackson JSON |
| `spring-boot-starter-data-jpa` | Hibernate + Spring Data repositories |
| `mysql-connector-j` | JDBC driver (scope `runtime` — only needed at runtime, not compile time) |
| `spring-boot-starter-validation` | `@Valid`, `@NotNull`, `@Email` etc. on DTOs |
| `spring-kafka` | Kafka producer/consumer support |
| `spring-cloud-starter-netflix-eureka-client` | Service registration/discovery |
| `spring-cloud-starter-openfeign` | Declarative REST clients for calling other services |
| `spring-cloud-starter-config` | Pull config from a centralized Config Server |
| `spring-cloud-starter-circuitbreaker-resilience4j` | Circuit breaker/retry patterns |
| `spring-boot-starter-actuator` | `/health`, `/metrics` endpoints |
| `spring-boot-starter-oauth2-resource-server` | JWT validation |
| `lombok` | Generates getters/setters/constructors via annotations (`optional=true` since it's compile-time only) |
| `mapstruct` | Compile-time entity↔DTO mapping |
| `springdoc-openapi-starter-webmvc-ui` | Swagger UI at `/swagger-ui.html` |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, MockMvc |
| `spring-kafka-test` | Embedded Kafka for integration tests |

**Note**: most dependencies have **no `<version>` tag** — that's the parent POM's dependency management doing its job. Only dependencies *not* managed by Spring Boot's BOM (like MapStruct, Springdoc) need an explicit version.

### 7. `<scope>` Values Commonly Used

| Scope | Meaning |
|---|---|
| *(default)* `compile` | Needed at compile time AND runtime, packaged into the JAR |
| `runtime` | Only needed when running, not for compiling your code (e.g. JDBC drivers) |
| `test` | Only available during test compilation/execution, not packaged |
| `optional=true` | Not passed transitively to projects that depend on this one (used for Lombok) |

### 8. `<build><plugins>` — Spring Boot Maven Plugin

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

This is what makes `mvn package` produce an **executable "fat" JAR** — bundling all dependencies plus an embedded Tomcat into a single runnable `.jar`. Without it, you'd get a plain JAR that can't run standalone with `java -jar`. The `<excludes>` block here keeps Lombok out of the final JAR since it's compile-time-only tooling.

---

## REST API vs Microservice — `pom.xml` Differences

| Section | Plain REST API | Microservice |
|---|---|---|
| `spring-cloud-dependencies` BOM | Not needed | Required |
| `spring-cloud-starter-netflix-eureka-client` | Skip | Required |
| `spring-cloud-starter-openfeign` | Skip | Required |
| `spring-cloud-starter-config` | Skip | Add if using centralized config |
| `spring-cloud-starter-circuitbreaker-resilience4j` | Optional | Common |
| Everything else (web, JPA, validation, actuator, security) | Same | Same |

---

## Quick Command Reference

| Command | Purpose |
|---|---|
| `mvn clean install` | Clean, compile, test, and install to local repo |
| `mvn package` | Build the executable JAR (via spring-boot-maven-plugin) |
| `mvn spring-boot:run` | Run the app directly without building a JAR first |
| `java -jar target/order-service-1.0.0.jar` | Run the built JAR |
| `mvn dependency:tree` | View full dependency tree (useful for resolving version conflicts) |