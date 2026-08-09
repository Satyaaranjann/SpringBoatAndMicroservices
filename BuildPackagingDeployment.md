# Build, Packaging & Deployment — Q&A

---

**Q1. Maven vs Gradle for Spring Boot — what's the difference?**

Both are build tools that manage dependencies, compilation, testing, and packaging — the choice is largely a team/project preference, but they differ meaningfully:

| | Maven | Gradle |
|---|---|---|
| Configuration format | XML (`pom.xml`) | Groovy or Kotlin DSL (`build.gradle` / `build.gradle.kts`) |
| Build performance | Slower — rebuilds more, less caching by default | Faster — incremental builds, build caching, parallel execution |
| Flexibility | Convention-driven, rigid lifecycle phases | Highly flexible/scriptable, can express custom logic naturally |
| Learning curve | Simpler to read for newcomers (declarative XML) | Steeper — DSL feels more like real code, more power but more to learn |
| Dependency management | `<dependencyManagement>`, BOM imports | Similar concept via platform/BOM imports |
| Spring Boot support | `spring-boot-maven-plugin` | `org.springframework.boot` Gradle plugin |
| Industry adoption | Still very widely used, especially in enterprise/legacy | Increasingly popular, especially in newer/greenfield and Android-adjacent projects |

**Maven example:**
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

**Gradle example:**
```groovy
plugins {
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.4'
}
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

**Practical takeaway for interviews:** neither is objectively "correct" — Maven's rigidity (fixed lifecycle: validate → compile → test → package → verify → install → deploy) makes builds predictable and easy to reason about across teams; Gradle's flexibility and performance (via incremental builds/caching) pay off more in larger, more complex, or monorepo-style projects. Most Spring Boot tutorials and enterprise codebases still default to Maven, but Gradle adoption is steadily growing.

---

**Q2. Executable JAR vs WAR — what's the difference?**

**WAR (Web Application Archive)** — the traditional Java web deployment format; packaged to be deployed **into an external servlet container** (a standalone Tomcat, JBoss, WebSphere installation) that's installed and managed separately from the application.

**Executable JAR (the Spring Boot default)** — a **self-contained** JAR that bundles the application code **plus an embedded server** (Tomcat/Jetty/Undertow) inside it, runnable directly with `java -jar app.jar` — no external server installation needed at all.

```xml
<!-- Maven: produces an executable JAR with an embedded server by default -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

```bash
java -jar order-service-1.0.0.jar
# app starts, embedded Tomcat binds to port 8080 — no separate server needed
```

**Why Spring Boot favors executable JARs:**
- Simplifies deployment — the artifact is the entire runtime unit, consistent across dev/staging/prod
- Enables containerization naturally — a JAR + JVM in a Docker image is far simpler than installing/configuring an external Tomcat inside a container
- Removes "works on my server config but not yours" class of problems, since the server config travels with the app

**When WAR is still used:** deploying into an **existing, shared** application server infrastructure that an organization already runs and wants to keep managing centrally (common in older enterprise environments) — Spring Boot still supports building a WAR (by extending `SpringBootServletInitializer` and marking the embedded container dependency as `provided`) for exactly this legacy scenario, but it's the exception, not the default, in modern Spring Boot development.

---

**Q3. What are Layered JARs?**

By default, an executable JAR bundles **everything** — your application code and all its dependencies — into a single flat layer. When building a **Docker image** from this, any tiny code change forces Docker to re-download/re-copy the **entire** JAR (including all unchanged third-party dependencies) into a new image layer, since Docker layer caching works at the granularity of whatever you `COPY` — wasteful and slow for CI/CD.

**Layered JARs** split the JAR's contents into logical layers based on how frequently they change:

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers>
            <enabled>true</enabled>
        </layers>
    </configuration>
</plugin>
```

**Default layers (ordered least-to-most likely to change):**
1. **`dependencies`** — third-party libraries (rarely change between builds)
2. **`spring-boot-loader`** — Spring Boot's own bootstrap classes (almost never change)
3. **`snapshot-dependencies`** — SNAPSHOT dependencies (change more often than release deps)
4. **`application`** — your actual compiled application code (changes on every build)

```dockerfile
FROM eclipse-temurin:21-jre AS builder
WORKDIR application
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract

FROM eclipse-temurin:21-jre
WORKDIR application
COPY --from=builder application/dependencies/ ./
COPY --from=builder application/spring-boot-loader/ ./
COPY --from=builder application/snapshot-dependencies/ ./
COPY --from=builder application/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

**Why this matters:** Docker caches each `COPY` instruction as a separate image layer — since `dependencies` rarely changes, that layer stays **cached** across builds, and only the small `application` layer (your actual code) gets rebuilt/re-pushed on each code change. This dramatically speeds up both build times and image push/pull times in CI/CD, especially as the dependency set grows large relative to your own code.

---

**Q4. How do you Dockerize a Spring Boot Application?**

A basic (non-optimized) Dockerfile for a Spring Boot executable JAR:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/order-service-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
docker build -t order-service:1.0.0 .
docker run -p 8080:8080 order-service:1.0.0
```

**Key considerations for a production-quality Docker image:**
- **Use a JRE base image, not a full JDK** — you only need to *run* the compiled application, not compile it; a JRE image is smaller (fewer attack surface/vulnerabilities, faster pulls)
- **Use a specific, pinned base image tag** (e.g., `eclipse-temurin:21-jre`, not `latest`) — for reproducible builds and to avoid surprise breaking changes
- **Run as a non-root user** — running containers as root is a security anti-pattern; create and switch to a dedicated user
- **Externalize configuration** via environment variables rather than baking environment-specific config into the image (see Q7)
- **Set explicit JVM memory limits** appropriate for the container's resource limits (`-Xmx`, or rely on the JVM's container-aware defaults in modern JDKs, which read cgroup limits automatically)

```dockerfile
FROM eclipse-temurin:21-jre
RUN addgroup --system spring && adduser --system spring --ingroup spring
USER spring:spring
WORKDIR /app
COPY target/order-service-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

**Q5. What are Multi-Stage Docker Builds, and why use them?**

A **multi-stage build** uses multiple `FROM` statements in a single Dockerfile — an early stage **builds** the application (needs the full JDK, Maven/Gradle, source code), and a later, separate stage copies **only the final artifact** into a clean, minimal runtime image — discarding the build tools, source code, and intermediate files entirely from the final image.

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 ./mvnw clean package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Why this matters:**
- **Smaller final image** — the JDK, Maven/Gradle cache, and full source tree (potentially hundreds of MB) never end up in the deployed image; only the compiled JAR does
- **Better security posture** — fewer tools and less surface area in the production image means fewer things that can be exploited (no compiler, no build tool, no source code sitting in a running container)
- **Self-contained, reproducible builds** — the build itself happens inside Docker using a pinned JDK version, so you don't depend on whatever's installed on the CI runner or developer's machine matching production exactly

This is now considered a baseline best practice for containerizing any compiled application, not just Spring Boot specifically — combined with layered JARs (Q3), it's the standard modern approach.

---

**Q6. What are the Kubernetes Basics relevant to running Spring Boot — Deployments, Services, ConfigMaps?**

**Deployment** — declares the **desired state** for running your application's pods: which container image to use, how many replicas (instances) to run, resource limits, and how to roll out updates.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests: { memory: "512Mi", cpu: "250m" }
            limits: { memory: "1Gi", cpu: "500m" }
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
```

Kubernetes continuously ensures the actual state matches this — if a pod crashes, it's automatically restarted; if you scale `replicas` to 5, Kubernetes schedules two more pods.

**Service** — provides a **stable network identity** (a fixed DNS name/virtual IP) in front of a dynamic, changing set of pods, and load-balances traffic across them, so other services/clients don't need to track individual pod IPs (which change constantly as pods are recreated).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

**ConfigMap** — stores **non-sensitive configuration** externally from the container image, injected into pods as environment variables or mounted files — enabling the same image to run with different config per environment without rebuilding it.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  LOG_LEVEL: "INFO"
```

```yaml
# referenced in the Deployment
envFrom:
  - configMapRef:
      name: order-service-config
```

**How these three fit together conceptually:** the **Deployment** defines and manages the pods running your app (referencing the Docker image built in Q4/Q5); the **Service** gives other things a stable way to reach those pods regardless of how many exist or where they're scheduled; the **ConfigMap** feeds environment-specific, non-sensitive configuration into those pods — mirroring exactly the externalized-configuration principle covered earlier for `application.yml`/profiles, just at the infrastructure layer instead of the application layer.

---

**Q7. What are CI/CD Pipelines, and how do GitHub Actions and Jenkins fit in?**

**CI/CD (Continuous Integration / Continuous Delivery-Deployment)** automates the path from a code commit to a running, deployed application — building, testing, packaging, and (for CD) deploying automatically, removing manual, error-prone steps and enabling fast, reliable, frequent releases.

**GitHub Actions** — CI/CD defined as YAML workflows living directly in the repository (`.github/workflows/`), triggered by events (push, pull request, tag), and run on GitHub-hosted (or self-hosted) runners.

```yaml
name: CI/CD
on:
  push:
    branches: [main]

jobs:
  build-test-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin' }

      - name: Build and test
        run: ./mvnw clean verify

      - name: Build Docker image
        run: docker build -t myregistry/order-service:${{ github.sha }} .

      - name: Push to registry
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u "${{ secrets.REGISTRY_USER }}" --password-stdin
          docker push myregistry/order-service:${{ github.sha }}

      - name: Deploy to Kubernetes
        run: kubectl set image deployment/order-service order-service=myregistry/order-service:${{ github.sha }}
```

**Jenkins** — a self-hosted, highly extensible automation server, configured via a `Jenkinsfile` (Groovy-based pipeline-as-code), typically used in organizations wanting full control over their CI/CD infrastructure (on-premise, custom plugin ecosystem, complex multi-team pipeline orchestration).

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh './mvnw clean package' }
        }
        stage('Test') {
            steps { sh './mvnw test' }
        }
        stage('Docker Build & Push') {
            steps {
                sh 'docker build -t myregistry/order-service:${BUILD_NUMBER} .'
                sh 'docker push myregistry/order-service:${BUILD_NUMBER}'
            }
        }
        stage('Deploy') {
            steps { sh 'kubectl apply -f k8s/deployment.yaml' }
        }
    }
}
```

**GitHub Actions vs Jenkins — the practical distinction:** GitHub Actions is **hosted/managed**, tightly integrated with GitHub, and simpler to get started with (no infrastructure to maintain) — a strong default for projects already on GitHub. Jenkins is **self-hosted**, more configurable/extensible via its large plugin ecosystem, and often preferred by larger enterprises needing full control over build infrastructure, complex approval gates, or on-premise constraints that a hosted SaaS CI can't satisfy.

---

**Q8. How should Environment Variables and Secrets Management be handled in deployment?**

**Environment variables** are the standard mechanism for injecting environment-specific, non-sensitive configuration into a containerized Spring Boot app — matching Spring Boot's property precedence (env vars override packaged `application.yml` values, as covered earlier) and Docker/Kubernetes' native support for injecting them per-container.

```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    value: "prod"
  - name: SPRING_DATASOURCE_URL
    value: "jdbc:postgresql://db-host:5432/orders"
```

**Secrets** (passwords, API keys, tokens, certificates) require a **different handling path entirely** — they should never live in a ConfigMap, a Dockerfile, source control, or a plain environment variable definition checked into a YAML manifest that's committed to Git.

**Kubernetes Secrets** — a dedicated Kubernetes object (base64-encoded, not encrypted by default at rest unless additionally configured) for sensitive values, injected similarly to ConfigMaps but with tighter access controls:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
type: Opaque
data:
  db-password: cGFzc3dvcmQxMjM=   # base64-encoded
```
```yaml
env:
  - name: SPRING_DATASOURCE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-service-secrets
        key: db-password
```

**Dedicated secrets managers** (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) — the more robust production pattern, offering **encryption at rest**, fine-grained access policies, automatic secret **rotation**, and a full audit trail of who/what accessed a given secret — capabilities plain Kubernetes Secrets don't provide out of the box. Spring Cloud Vault integrates directly with HashiCorp Vault so secrets can be fetched securely at startup without ever touching a config file or environment variable definition in plain text.

**The core principle to state clearly in an interview:** configuration and secrets are handled through **fundamentally different pipelines** with different security guarantees — treating a database password the same way you treat `LOG_LEVEL=INFO` (both as "just an env var") is a common, serious mistake; secrets need encryption, restricted access, rotation, and audit trails that ordinary configuration doesn't.

---

## Extra / Important Interview Questions

**Q9. If you `docker build` a Spring Boot app without multi-stage builds or layering, what specifically goes wrong in a CI/CD pipeline over time?**

Every commit — even a one-line code change — invalidates Docker's cache for the entire application layer (since the whole JAR, dependencies included, is copied as one blob), forcing a full re-push of the entire image (often hundreds of MB) to the registry on every single build, and a full re-pull on every deployment target. This significantly slows down CI/CD pipeline duration and registry bandwidth/storage costs at scale, compared to a layered + multi-stage approach where only the small, frequently-changing `application` layer needs to move on most builds.

---

**Q10. Why is running a container as root considered risky, even though the container is "isolated"?**

Container isolation (via Linux namespaces/cgroups) is not an absolute security boundary — container escape vulnerabilities have occurred in real-world CVEs, and a process running as root **inside** a compromised container has a meaningfully larger blast radius if such an escape occurs, compared to a non-root, least-privilege process. Running as a dedicated non-root user is a defense-in-depth practice: it doesn't prevent every attack, but it significantly limits what an attacker can do even if they gain code execution inside the container.

---

**Q11. What's the difference between `resources.requests` and `resources.limits` in a Kubernetes Deployment, and why do both matter for a Spring Boot app specifically?**

- **`requests`** — the amount of CPU/memory Kubernetes **guarantees** is reserved for the pod, and uses for scheduling decisions (which node has room)
- **`limits`** — the **maximum** the pod is allowed to consume; exceeding the memory limit gets the pod **OOMKilled** (forcibly terminated), while exceeding the CPU limit results in throttling (not termination)

For a Spring Boot app specifically, this matters because the JVM's memory behavior needs to be tuned to actually **respect** these container limits — modern JDKs (10+) are container-aware and size the default heap based on the container's memory limit automatically, but under-provisioning `limits` relative to what the JVM actually needs (heap + metaspace + thread stacks + native memory) is a very common cause of mysterious `OOMKilled` pod restarts in production that don't show up as a Java `OutOfMemoryError` in the application logs at all — because the container itself was killed from outside the JVM.

---

**Q12. In a rolling deployment, why does a correctly-configured readiness probe matter more than people initially assume?**

During a rolling update, Kubernetes replaces old pods with new ones gradually. If the readiness probe isn't configured (or is too permissive, e.g., checking only that the port is open rather than `/actuator/health/readiness`), Kubernetes may start routing live traffic to a **new pod before the Spring application context has fully finished starting up** (still initializing beans, warming caches, establishing DB connection pools) — resulting in a burst of failed requests/errors during every deployment, even though nothing is actually "broken." A correctly wired readiness probe ensures traffic only reaches a pod once it's genuinely ready to serve requests, making rolling deployments actually zero-downtime rather than just nominally so.

---

**Q13. Why might a team choose to bake `SPRING_PROFILES_ACTIVE` into the Kubernetes Deployment manifest rather than into the Docker image itself?**

Baking it into the Docker image ties that specific image build to one environment — you'd need to build a separate image per environment (dev/staging/prod), defeating the "build once, deploy everywhere" principle and multiplying build/registry overhead. Injecting it via the Deployment manifest (or ConfigMap) means the **exact same image artifact**, already tested in staging, is promoted unchanged to production — only the externally-injected environment variable differs, which is both faster (no rebuild between environments) and safer (you're not testing one artifact and shipping a different one).

---

*Interview tip: Deployment questions increasingly reward practical incident-response instincts — "why did this pod get OOMKilled," "why are deployments causing brief errors," "why is the CI pipeline slow" — over reciting Dockerfile syntax. Framing answers around what breaks in production (and why) demonstrates real operational experience.*