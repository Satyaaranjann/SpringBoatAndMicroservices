# Annotations, Security, JWT, Authentication & Networking — Q&A

---

## Section 1: Annotations (Deep Dive / Mixed Recall)

**Q1. Explain the difference between `@Component`, `@Bean`, `@Autowired`, and `@Qualifier` in one integrated example.**

```java
@Component                          // Spring auto-detects and registers this class as a bean
public class EmailNotifier implements Notifier { ... }

@Component
public class SmsNotifier implements Notifier { ... }

@Configuration
public class NotifierConfig {
    @Bean                             // manually registers a bean (used for classes you don't own, or need custom construction)
    @Primary                          // default choice when multiple Notifier beans exist
    public Notifier defaultNotifier(EmailNotifier emailNotifier) {
        return emailNotifier;
    }
}

@Service
public class AlertService {
    private final Notifier notifier;

    @Autowired                        // tells Spring to inject a matching bean here
    public AlertService(@Qualifier("smsNotifier") Notifier notifier) {   // overrides @Primary, picks explicitly
        this.notifier = notifier;
    }
}
```

This single example ties together auto-detection (`@Component`), manual registration (`@Bean`), injection (`@Autowired`), default resolution (`@Primary`), and explicit disambiguation (`@Qualifier`) — a common "explain these together" interview format.

---

**Q2. What's the difference between `@RequestMapping`, `@GetMapping`, and `@RestController`, and how do they compose on a single controller?**

```java
@RestController                      // = @Controller + @ResponseBody on every method
@RequestMapping("/api/v1/users")     // class-level base path
public class UserController {

    @GetMapping("/{id}")             // full path: GET /api/v1/users/{id}
    public User getUser(@PathVariable Long id) { ... }

    @PostMapping
    public User createUser(@Valid @RequestBody UserDto dto) { ... }
}
```

`@GetMapping`/`@PostMapping`/etc. are shorthand for `@RequestMapping(method = ...)`; class-level `@RequestMapping` sets a shared prefix that combines with each method's own mapping.

---

**Q3. What is the difference between annotations that are "marker" annotations (no logic) versus ones that trigger real behavior via AOP/reflection?**

- **Marker/semantic annotations** — carry no runtime behavior by themselves, just metadata read by something else (e.g., `@Service` — functionally identical to `@Component`, purely documentation/semantic value; a custom `@Auditable` annotation with no aspect watching it does nothing)
- **Behavior-triggering annotations** — actively intercepted by Spring's infrastructure to alter runtime behavior: `@Transactional` (AOP proxy wraps a transaction), `@Cacheable` (AOP proxy intercepts and checks a cache), `@Async` (AOP proxy dispatches to a thread pool), `@Valid` (triggers actual validation logic)

**Why this distinction matters in interviews:** it explains *why* self-invocation breaks `@Transactional`/`@Cacheable`/`@Async` but not `@Service`/`@Component` — the former rely on a **proxy** intercepting external calls; annotations that are purely metadata have no such mechanism to bypass in the first place.

---

**Q4. What's the difference between `@RequestBody`, `@ResponseBody`, `@ModelAttribute`, and `@RequestParam`, and when would you use `@ModelAttribute` specifically?**

- **`@RequestBody`** — binds the entire HTTP body (JSON) to an object
- **`@RequestParam`** — binds individual query string parameters
- **`@ModelAttribute`** — binds **multiple** request parameters (query string or form fields) onto the properties of a single object, commonly used for HTML form submissions in traditional MVC apps (`application/x-www-form-urlencoded`), less common in pure REST APIs which typically use `@RequestBody` with JSON instead

```java
// GET/POST with form fields: name=John&age=30
@PostMapping("/register")
public String register(@ModelAttribute UserForm form) { ... }  // binds name and age onto UserForm's fields
```

---

**Q5. What does `@ConditionalOnProperty`, `@ConditionalOnMissingBean`, and `@Profile` have in common, and how do they differ?**

All three are **conditional bean registration** mechanisms — a bean is only created if a certain condition holds — but the condition source differs:

- **`@Profile("dev")`** — condition is which Spring **profile** is active
- **`@ConditionalOnProperty(name = "feature.x.enabled", havingValue = "true")`** — condition is a specific **property value**
- **`@ConditionalOnMissingBean`** — condition is whether **another bean of that type already exists** in the context (commonly used in auto-configuration classes to "back off" if the user has defined their own bean)

```java
@Bean
@ConditionalOnProperty(name = "feature.newCheckout.enabled", havingValue = "true", matchIfMissing = false)
public CheckoutService newCheckoutService() { ... }

@Bean
@ConditionalOnMissingBean(CheckoutService.class)
public CheckoutService defaultCheckoutService() { ... }
```

This pattern (property-driven feature toggling of entire bean implementations) is a common real-world design question.

---

## Section 2: Security, JWT, and Authentication

**Q6. Walk through the complete JWT authentication flow in a Spring Boot microservice, end to end.**

1. **Login request** — client sends credentials to `POST /auth/login`
2. **Credential verification** — an `AuthenticationManager` (backed by a `UserDetailsService` + `PasswordEncoder`) verifies the username/password against stored (hashed) credentials
3. **Token generation** — on success, the server builds a JWT containing claims (subject/username, roles, issued-at, expiration), signs it with a secret key (HMAC) or private key (RSA/EC), and returns it to the client
4. **Client stores the token** — typically in memory or secure storage (not `localStorage`, ideally, due to XSS risk — see earlier security section)
5. **Subsequent requests** — client sends `Authorization: Bearer <token>` on every request
6. **Token validation filter** — a custom filter (`OncePerRequestFilter`), placed before `UsernamePasswordAuthenticationFilter` in the chain, extracts the token, verifies its **signature** (proves it wasn't tampered with) and **expiration**, extracts the username/roles, and populates the `SecurityContext`
7. **Authorization** — downstream, `authorizeHttpRequests` rules or `@PreAuthorize` checks the now-populated `SecurityContext` to allow/deny the request

```java
public class JwtAuthFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String token = extractBearerToken(req);
        if (token != null && jwtUtil.isValid(token)) {
            String username = jwtUtil.extractUsername(token);
            List<GrantedAuthority> authorities = jwtUtil.extractAuthorities(token);
            var authToken = new UsernamePasswordAuthenticationToken(username, null, authorities);
            SecurityContextHolder.getContext().setAuthentication(authToken);
        }
        chain.doFilter(req, res);
    }
}
```

**A strong answer explicitly separates "authentication" (steps 1-3) from "authorization" (step 7)** — a very common thing weaker answers blur together.

---

**Q7. How is a JWT actually verified as "not tampered with"? Explain the role of the signature.**

A JWT has three parts: `header.payload.signature`. The signature is computed by the issuing server as `HMACSHA256(base64(header) + "." + base64(payload), secretKey)` (for symmetric signing) or via an RSA/EC private key (for asymmetric signing).

**Verification process:** the receiving service recomputes the signature from the received header+payload using the **same secret key** (HMAC) or the corresponding **public key** (RSA/EC), and compares it to the signature included in the token. If they match, the payload hasn't been altered since signing — because any change to the header or payload would produce a completely different signature when recomputed, and the attacker doesn't have the key needed to forge a matching one.

**Important clarification often missed:** the JWT payload is only **signed**, not **encrypted** (unless you specifically use JWE, which is rare) — anyone can **decode** (base64-decode, not decrypt) and read the payload's contents without the key; the signature only guarantees **integrity** (it wasn't modified), not confidentiality. This is why sensitive data (passwords, SSNs) should never be put directly into JWT claims.

---

**Q8. Symmetric (HMAC) vs Asymmetric (RSA/EC) JWT signing — when would you choose each in a microservices architecture?**

- **HMAC (symmetric)** — the **same secret key** is used to both sign and verify. Simple and fast, but **every service that needs to verify tokens must have the same secret key**, meaning the secret has to be securely distributed to every service — increasing the blast radius if any one service is compromised (a leaked key from any service lets an attacker forge tokens for the entire system)
- **RSA/EC (asymmetric)** — the **issuing** service (e.g., a central Auth Server) holds the **private key** and signs tokens; every other service only needs the corresponding **public key** to verify signatures — public keys can be freely distributed/published without security risk, since possessing a public key doesn't let you forge new valid tokens

**In a microservices architecture with many services needing to verify tokens issued by one central auth service, asymmetric signing is the stronger, more scalable choice** — it's also what OAuth2/OIDC providers (Google, Okta, Keycloak) use in practice, publishing their public keys via a JWKS (JSON Web Key Set) endpoint that resource servers fetch to verify tokens without ever needing the private key.

---

**Q9. How do you handle JWT logout, given that JWTs are stateless and the server doesn't track "active sessions"?**

This is a well-known tricky point precisely **because** JWTs are stateless by design — there's no server-side session to simply delete on logout, unlike traditional session-cookie authentication.

**Common approaches:**
1. **Short expiration + refresh tokens** — keep access tokens short-lived (e.g., 15 minutes); "logout" just means the client discards both tokens locally and stops using them; even if the access token were somehow reused, it expires quickly regardless
2. **Token blacklist/denylist** — maintain a store (Redis, with TTL matching the token's remaining expiry) of explicitly revoked token IDs (`jti` claim); the validation filter checks this blacklist on every request — this reintroduces some server-side state, partially trading away pure statelessness for the ability to truly revoke a token immediately
3. **Refresh token revocation** — since refresh tokens are typically already tracked server-side (in a database, to allow issuing new access tokens), revoking the **refresh token** on logout is straightforward; the still-valid access token remains usable until it naturally expires (an accepted trade-off given its short lifespan)

**The honest interview answer:** pure JWT stateless auth **cannot** provide instant, guaranteed revocation without reintroducing some server-side state — you're always trading off between statelessness (scalability) and immediate revocability (security), and the right balance depends on the application's risk tolerance.

---

**Q10. What is the difference between Authentication and Authorization at the code level in Spring Security — which objects represent each?**

- **Authentication** is represented by the `Authentication` object (populated in the `SecurityContext`) — it answers "who is this?" and holds the principal (typically `UserDetails`), credentials, and granted authorities, established by an `AuthenticationManager`/`AuthenticationProvider`
- **Authorization** is the **decision process** that consults the already-established `Authentication` object to determine access — via `AuthorizationManager` (modern Spring Security) evaluating rules like `hasRole()`, `hasAuthority()`, or a `@PreAuthorize` SpEL expression

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
boolean isAdmin = auth.getAuthorities().stream()
    .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
```

Authentication happens **once** per request (establishing who's calling), while authorization decisions can happen **multiple times** within the same request (URL-level filter chain check, then again at method-level via `@PreAuthorize`), all consulting the same established `Authentication`.

---

**Q11. Explain the difference between `hasRole()` and `hasAuthority()` in Spring Security expressions.**

`hasRole("ADMIN")` is essentially syntactic sugar for `hasAuthority("ROLE_ADMIN")` — Spring Security **automatically prepends** the `ROLE_` prefix when you use `hasRole()`. This is a very common source of bugs: if your `GrantedAuthority` objects are created **without** the `ROLE_` prefix (e.g., just `"ADMIN"`), then `hasRole("ADMIN")` will **fail to match**, because it's actually checking for `"ROLE_ADMIN"` — you'd need `hasAuthority("ADMIN")` instead, which checks the authority string exactly as given, with no prefix manipulation.

```java
// If your custom UserDetailsService grants authorities WITHOUT "ROLE_" prefix:
new SimpleGrantedAuthority("ADMIN")   // no prefix

// Then this FAILS silently (denies access) because it checks for "ROLE_ADMIN":
.hasRole("ADMIN")

// This works correctly:
.hasAuthority("ADMIN")
```

This exact prefix mismatch is a favorite "why isn't my role check working" debugging question.

---

**Q12. How would you secure service-to-service (machine-to-machine) communication in a microservices architecture — is user JWT propagation the right approach?**

Propagating the **original user's JWT** to downstream services works for simple cases (Service A calls Service B on behalf of the user, and B needs to know who the user is / what they're allowed to do) — but it's not always sufficient or appropriate:

- **OAuth2 Client Credentials flow** — for genuine machine-to-machine calls with no user context at all (a scheduled job, an internal service calling another purely for its own purposes), the calling service authenticates as **itself** (using its own client ID/secret) against the Authorization Server, obtaining a token representing the *service's* identity, not a user's
- **Token exchange / delegation** — some architectures use a distinct mechanism where Service A exchanges the user's token for a new, scoped-down token specifically for calling Service B, rather than blindly forwarding the original — limiting what B can do with it (principle of least privilege) and allowing tracking of the actual call chain
- **mTLS (mutual TLS)** — for pure service identity verification at the network level (regardless of user context), services present certificates to each other, commonly managed via a service mesh (Istio, Linkerd) rather than application code

**What a strong answer signals:** recognizing that "just forward the JWT everywhere" is a reasonable simple default but not automatically correct for every service-to-service scenario, especially where the calling service's own identity — not just the end user's — matters.

---

## Section 3: Networking

**Q13. Explain what happens at the network level between a browser and a Spring Boot REST API for a single HTTPS request — from DNS to response.**

1. **DNS resolution** — the client resolves the API's hostname to an IP address
2. **TCP handshake** — client and server perform a 3-way handshake (SYN, SYN-ACK, ACK) to establish a TCP connection
3. **TLS handshake** (for HTTPS) — client and server negotiate a TLS version/cipher suite, the server presents its certificate (verified against a trusted CA), and a symmetric session key is established for encrypting the actual data (this is why HTTPS has extra round-trip latency compared to plain HTTP, especially on the first connection — though TLS 1.3 and connection reuse/keep-alive reduce this)
4. **HTTP request sent** — the actual HTTP request (method, headers, body) is sent over the now-encrypted connection
5. **Server processing** — the embedded server (Tomcat/Netty) accepts the connection, hands it off through the servlet/reactive pipeline — filters (including Spring Security's chain), then the `DispatcherServlet`, then the matched controller method
6. **Response sent back** — serialized (JSON via Jackson), sent back over the same TCP connection
7. **Connection reuse (keep-alive)** — modern HTTP/1.1+ and HTTP/2 typically **reuse** the same TCP/TLS connection for subsequent requests rather than repeating the full handshake each time, which is why connection pooling (on both client and server sides) matters for performance

**Why this matters for interview credibility:** many candidates can describe Spring's internal request handling but can't explain what's happening below the application layer — being able to speak to the TCP/TLS layer signals broader systems understanding, valued at senior levels.

---

**Q14. What's the difference between HTTP/1.1 and HTTP/2, and does it matter for a Spring Boot API?**

| | HTTP/1.1 | HTTP/2 |
|---|---|---|
| Connections | Typically one request in-flight per TCP connection at a time (though pipelining exists, it's rarely used) — browsers open multiple parallel connections to compensate | **Multiplexing** — many requests/responses interleaved over a **single** TCP connection concurrently |
| Headers | Sent as plain text, repeated on every request | **Header compression** (HPACK), reducing overhead for repeated headers |
| Head-of-line blocking | Yes, at the connection level (browsers work around it with multiple connections) | Reduced at the HTTP layer (though can still occur at the underlying TCP layer) |

**Does it matter for Spring Boot specifically:** embedded servers (Tomcat, Netty) support HTTP/2, and enabling it can meaningfully reduce latency for clients making many concurrent requests to the same API (fewer connection setups, better multiplexing) — particularly relevant for APIs serving rich frontends making many parallel calls. It requires TLS in practice (browsers only support HTTP/2 over HTTPS), and is enabled via:
```yaml
server:
  http2:
    enabled: true
  ssl:
    enabled: true
```

---

**Q15. What is the difference between a Load Balancer operating at Layer 4 vs Layer 7, and which would you use in front of a Spring Boot microservices cluster?**

- **Layer 4 (Transport layer)** — routes traffic based on IP address and TCP/UDP port alone, with **no visibility into HTTP content** (can't see URLs, headers, cookies). Faster (less processing per packet), but can't make routing decisions based on the actual request content.
- **Layer 7 (Application layer)** — understands HTTP itself — can route based on URL path, headers, cookies, hostname (e.g., route `/api/orders/**` to the Order service, `/api/users/**` to the User service) — enabling exactly the kind of routing an **API Gateway** (Spring Cloud Gateway, Nginx, an AWS ALB) needs to do.

**For a microservices cluster:** an **L7 load balancer/API Gateway** is typically needed at the edge (to route by path to the right service, apply auth, rate limiting, etc.), while **L4 load balancing** is often used internally/at a lower level (e.g., distributing raw TCP traffic across pod replicas within Kubernetes, where Layer 7-aware routing decisions have already been made further up the chain, or aren't needed because it's routing to identical replicas of the same service).

---

**Q16. Your Spring Boot service calls a downstream service, and the call occasionally hangs indefinitely, eventually exhausting your thread pool. What networking-level configuration is missing, and how do you fix it?**

Missing **timeouts** on the HTTP client. Without explicit timeouts, a client will wait indefinitely (or until the OS's own very long default TCP timeout) for a response from a slow/hung downstream service — and under load, many such hung calls can exhaust the calling service's own thread pool, causing a cascading failure even though the *calling* service's own code has no bug.

```java
// WebClient with explicit timeouts
WebClient client = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(
        HttpClient.create()
            .responseTimeout(Duration.ofSeconds(3))
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)
    ))
    .build();
```

**Three distinct timeout types worth naming explicitly (a strong signal of real production experience):**
- **Connection timeout** — how long to wait to **establish** the TCP connection itself
- **Read/response timeout** — how long to wait for the response **after** the connection is established and the request is sent
- **Write timeout** — how long to wait while **sending** the request body (relevant for large payloads / slow networks)

Beyond timeouts, this is also exactly the scenario a **Circuit Breaker** (Resilience4j) should wrap — timeouts prevent indefinite hangs on a single call, while a circuit breaker prevents *repeatedly* attempting calls to a service that's clearly already failing, protecting the thread pool at a higher level.

---

**Q17. What is DNS caching, and how can it cause subtle bugs in a Spring Boot app running in Kubernetes or a cloud environment with dynamic IPs?**

The **JVM itself caches DNS lookups** — by default, successful DNS resolutions are cached **indefinitely** (`networkaddress.cache.ttl`), and this can cause real problems in dynamic cloud environments: if a downstream service's IP address changes (a pod restarts and gets a new IP, a load balancer's backend changes), a long-running JVM process might keep sending requests to the **old, now-invalid IP** because it never re-resolves the hostname, since its cached answer hasn't expired.

**Fix:** explicitly set a reasonable DNS cache TTL (rather than the JVM's default of effectively "forever" for successful lookups) so the JVM periodically re-resolves hostnames:
```java
// via a system property, or programmatically at startup
java.security.Security.setProperty("networkaddress.cache.ttl", "60");   // re-resolve every 60 seconds
```

This is a genuinely subtle, real-world bug — services appearing to work fine after deployment, then mysteriously failing to reach a dependency hours later after that dependency's pod was rescheduled — that's a strong signal of hands-on cloud-native production experience when brought up unprompted.

---

**Q18. What's the difference between `ClusterIP`, `NodePort`, and `LoadBalancer` Kubernetes Service types, from a networking perspective, for exposing a Spring Boot app?**

- **`ClusterIP`** (default) — exposes the service **only within the cluster**, via an internal, stable virtual IP — used for internal service-to-service communication (e.g., Order service calling Inventory service), never directly reachable from outside the cluster
- **`NodePort`** — exposes the service on a **static port on every node's IP** in the cluster, making it reachable from outside — but typically only used for development/testing, since it ties external access to specific node IPs/ports directly, which isn't a robust production pattern
- **`LoadBalancer`** — provisions an actual **external cloud load balancer** (via the cloud provider's integration — AWS ELB, GCP Load Balancer) that routes external traffic into the cluster — the standard way to expose a service to the public internet in a cloud-managed Kubernetes setup

**Practical mapping for a typical microservices deployment:** internal services (Order, Inventory, Payment) use `ClusterIP` (only reachable from within the cluster, e.g., by the API Gateway); the **API Gateway itself** is exposed via a `LoadBalancer` (or often an `Ingress` resource, which is a more flexible, HTTP-aware layer built on top of these Service types, supporting host/path-based routing similar to an L7 load balancer).

---

## Rapid-Fire Cross-Topic Questions

**Q19. If a `@PreAuthorize("hasRole('ADMIN')")` check silently always fails despite the user clearly having an ADMIN role in the database, what are the first two things you'd check?**

1. Whether `@EnableMethodSecurity` (or the legacy `@EnableGlobalMethodSecurity`) is actually present on a configuration class — without it, `@PreAuthorize` is silently ignored entirely
2. Whether the `ROLE_` prefix mismatch (Q11) is the culprit — check exactly what string the `GrantedAuthority` actually contains versus what `hasRole()` is implicitly checking for

---

**Q20. Why might a JWT that worked fine yesterday suddenly fail signature verification today, with no code changes deployed?**

Most likely a **key rotation** occurred (the signing key changed) without the verifying service(s) being updated to use the new key/public key — common when using symmetric HMAC keys distributed via configuration, if only some services received the updated secret. This is exactly the kind of operational risk asymmetric signing + a JWKS endpoint (Q8) is designed to reduce, since public keys can be fetched dynamically/refreshed rather than manually redistributed as a shared secret.

---

**Q21. A mobile client complains that API calls work on WiFi but frequently time out on cellular networks. Is this a Spring Boot server-side problem?**

Not necessarily — cellular networks have meaningfully higher latency and less stable connectivity than WiFi, so a **client-side timeout configured too aggressively** (assuming WiFi-like latency) is a very common root cause, not a server issue at all. That said, it's worth checking server-side timeout/keep-alive settings too, and whether the server correctly supports **connection retries/resumption** — but the first, most likely culprit is client-side timeout configuration not accounting for realistic mobile network conditions.

---

*Interview tip: This combined topic area (annotations + security + JWT + networking) is often used specifically to test whether your Spring Boot knowledge is grounded in **real request-handling mechanics** — proxies, filter chains, TCP/TLS, DNS — rather than annotation memorization alone. Being able to trace a request from DNS resolution through the security filter chain to the controller, and explain what breaks at each layer, is the signal top MNC interviewers are calibrating for at experienced levels.*

# Deploying Java Projects on a Server — Spring Boot vs Plain Java

Both follow the same overall philosophy (build → package → transfer → run → keep alive → expose), but differ in **what gets packaged** and **how it's run**. This guide covers both side by side.

---

## The Core Difference Up Front

| | Plain Java Project | Spring Boot Project |
|---|---|---|
| Package format | Plain `.jar` (just your compiled classes) or `.war` | Executable "fat/uber" `.jar` — includes **all dependencies + embedded server** |
| Needs external server? | Yes, if it's a web app (needs Tomcat/JBoss installed separately) — no, if it's a standalone CLI/batch program | No — the server (Tomcat/Netty) is bundled inside the JAR itself |
| How you run it | `java -cp classes:libs/* com.example.Main` or deploy `.war` into Tomcat's `webapps/` folder | `java -jar app.jar` — that's it |
| Dependency management at runtime | You must supply the classpath (all JARs) manually or via a WAR's `WEB-INF/lib` | Bundled inside the single JAR — nothing else needed |

**The one-line summary interviewers want to hear:** a plain Java web app needs you to install and manage a separate application server; Spring Boot **is** the server — you just run the JAR directly.

---

## Part 1: Deploying a Plain Java Project

There are really **two very different cases** under "plain Java project" — a standalone program (no web server involved) and a traditional Java web app (Servlet/JSP, deployed as a WAR into Tomcat). Both are covered.

### Case A: Plain Java Standalone Application (no web framework)

**Step 1 — Compile the code**
```bash
javac -d out src/com/example/*.java
```

**Step 2 — Package into a JAR**
```bash
jar cfe app.jar com.example.Main -C out .
```
`cfe` = create a jar, with a manifest specifying the entry point (`Main` class), from the compiled `out` directory.

**Step 3 — Handle dependencies (if any)**
Plain `javac`/`jar` has no built-in dependency management like Maven/Gradle. Either:
- Bundle all dependency JARs alongside your app and reference them via `-cp`, or
- Build a "fat JAR" using a build tool (Maven Shade Plugin / Gradle Shadow Plugin) that merges everything into one JAR — this is effectively what Spring Boot does automatically for you

**Step 4 — Transfer to the server**
```bash
scp app.jar deploy-user@server-ip:/opt/app/
```

**Step 5 — Run it on the server**
```bash
java -jar app.jar
# or, with a separate classpath of dependency jars:
java -cp "app.jar:libs/*" com.example.Main
```

**Step 6 — Keep it running persistently**
Same as any JVM process — wrap it in a `systemd` service (see the shared section below) so it survives SSH disconnects and auto-restarts on crash/reboot.

---

### Case B: Traditional Java Web App (Servlet/JSP → WAR → Tomcat)

This is the classic pre-Spring-Boot enterprise Java deployment model.

**Step 1 — Build the WAR file**
Using Maven with the `war` packaging type:
```xml
<packaging>war</packaging>
```
```bash
mvn clean package
# produces target/myapp.war
```
The WAR bundles your compiled classes, JSPs/static resources, and dependency JARs (in `WEB-INF/lib`) — but **not** a server; it's built to be deployed **into** one.

**Step 2 — Install and configure a servlet container on the server**
Unlike Spring Boot, you must set this up yourself — download and install Apache Tomcat (or WildFly, JBoss, WebSphere) on the target server:
```bash
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.x/bin/apache-tomcat-10.1.x.tar.gz
tar -xzf apache-tomcat-10.1.x.tar.gz -C /opt/
```

**Step 3 — Deploy the WAR into Tomcat**
```bash
cp target/myapp.war /opt/apache-tomcat-10.1.x/webapps/
```
Tomcat **auto-detects and deploys** any WAR dropped into its `webapps/` folder — it unpacks it and starts serving it, typically at `http://server-ip:8080/myapp/`.

**Step 4 — Start/manage Tomcat**
```bash
/opt/apache-tomcat-10.1.x/bin/startup.sh
# or, more robustly, run Tomcat itself as a systemd service
```

**Step 5 — Check deployment**
```bash
tail -f /opt/apache-tomcat-10.1.x/logs/catalina.out
curl http://localhost:8080/myapp/
```

**Key limitation this highlights (a common interview talking point):** the server (Tomcat) and the application are **separately versioned and managed** — upgrading Tomcat, or running two apps that need different Tomcat versions on the same machine, becomes genuinely painful. This exact pain point is a major reason Spring Boot's embedded-server model became the industry standard.

---

## Part 2: Deploying a Spring Boot Project

**Step 1 — Build the executable JAR**
```bash
./mvnw clean package -DskipTests
# produces target/order-service-1.0.0.jar — already includes the embedded Tomcat + all dependencies
```

**Step 2 — Transfer it to the server**
```bash
scp target/order-service-1.0.0.jar deploy-user@server-ip:/opt/app/
```

**Step 3 — Set environment-specific configuration**
```bash
export SPRING_PROFILES_ACTIVE=prod
export SPRING_DATASOURCE_URL=jdbc:postgresql://db-host:5432/orders
export SPRING_DATASOURCE_PASSWORD=$(cat /etc/secrets/db-password)
```

**Step 4 — Run it directly — no external server needed**
```bash
java -jar order-service-1.0.0.jar
```
That's genuinely it — the embedded Tomcat starts inside the JVM process itself, binds to the configured port (default 8080), and starts serving requests immediately.

**Step 5 — Keep it running persistently (systemd)**
```ini
# /etc/systemd/system/order-service.service
[Unit]
Description=Order Service
After=network.target

[Service]
User=deploy-user
EnvironmentFile=/opt/app/order-service.env
ExecStart=/usr/bin/java -jar /opt/app/order-service-1.0.0.jar
SuccessExitStatus=143
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable order-service
sudo systemctl start order-service
sudo systemctl status order-service
```

**Step 6 — Put a reverse proxy in front (Nginx)** — for TLS termination and to avoid exposing the JVM's port directly:
```nginx
server {
    listen 443 ssl;
    server_name api.example.com;
    ssl_certificate     /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Step 7 — Verify**
```bash
curl https://api.example.com/actuator/health
sudo journalctl -u order-service -f
```

---

## Side-by-Side: The Same Deployment Journey, Compared

| Step | Plain Java Web App (WAR) | Spring Boot |
|---|---|---|
| 1. Build | `mvn package` → `.war` | `mvn package` → executable `.jar` |
| 2. Server setup | Install Tomcat separately on the server | Not needed — server is inside the JAR |
| 3. Deploy | Copy WAR into Tomcat's `webapps/` folder | Copy JAR anywhere, run it directly |
| 4. Run | Start Tomcat (which then runs your app) | `java -jar app.jar` — runs your app directly |
| 5. Process management | Manage Tomcat as a service | Manage your JAR directly as a service |
| 6. Reverse proxy | Nginx in front of Tomcat's port | Nginx in front of the embedded server's port |
| 7. Multiple apps on one server | Multiple WARs deployed into the same Tomcat instance (shared server, shared JVM — a fault in one can affect others) | Multiple independent JARs, each with its own embedded server/JVM process (isolated) |

**The isolation point (Step 7) is a strong interview talking point:** in the WAR/shared-Tomcat model, a memory leak or classloader issue in one deployed app can degrade or crash the shared Tomcat instance, affecting every other app deployed alongside it. Spring Boot's one-JAR-per-JVM-process model gives each application true process-level isolation — closer to how microservices are meant to be deployed.

---

## The Modern Answer for Both: Containerize Instead

In practice today, **both** plain Java apps and Spring Boot apps are increasingly deployed via **Docker**, which sidesteps most of the manual server-setup differences above:

```dockerfile
# Spring Boot
FROM eclipse-temurin:21-jre
COPY target/order-service-1.0.0.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```dockerfile
# Plain Java (standalone)
FROM eclipse-temurin:21-jre
COPY app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```dockerfile
# Traditional WAR + Tomcat
FROM tomcat:10-jre21
COPY target/myapp.war /usr/local/tomcat/webapps/
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

Once containerized, deployment to a server or Kubernetes cluster looks **identical regardless of which of the three you started with** — `docker run`, or `kubectl apply` a Deployment manifest — because the container itself now bundles whatever runtime each app needs. This is a major reason Docker became the deployment standard: it normalizes away exactly these historical differences between deployment styles.

---

*Interview tip: If asked "how do you deploy a Java app," the strongest answers explicitly name which kind of Java app it is first (standalone, traditional WAR, or Spring Boot) — since the process genuinely differs — and then explain why Spring Boot's embedded-server model simplified this compared to the traditional WAR/Tomcat approach, before mentioning that containerization has since normalized the difference further.*


# Dependency Injection (DI) — Detailed Explanation

---

## 1. What is Dependency Injection?

**Dependency Injection** is a design pattern where an object's dependencies (the other objects/services it needs to function) are **provided to it from the outside**, rather than the object creating them itself.

It's a specific form of **Inversion of Control (IoC)** — instead of your class controlling how its dependencies are created, that control is inverted and handed to a container/framework (in Spring Boot, the **Spring IoC Container**).

---

## 2. The Problem DI Solves

**Without DI (tight coupling):**

```java
public class UserService {
    private UserRepository userRepository = new UserRepositoryImpl(); // hardcoded dependency

    public User getUser(Long id) {
        return userRepository.findById(id);
    }
}
```

Problems with this approach:
- `UserService` is **tightly coupled** to `UserRepositoryImpl` — can't swap implementations easily
- Hard to **unit test** (can't mock `UserRepository` without modifying the class)
- If `UserRepositoryImpl`'s constructor changes, every class that does `new UserRepositoryImpl()` breaks
- No central place to manage object lifecycles

**With DI (loose coupling):**

```java
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) { // dependency injected via constructor
        this.userRepository = userRepository;
    }

    public User getUser(Long id) {
        return userRepository.findById(id);
    }
}
```

Now `UserService` doesn't know or care *how* `UserRepository` is created — it just receives a ready-to-use instance.

---

## 3. How DI Works — Step by Step

1. **Define beans** — Spring scans classes annotated with `@Component`, `@Service`, `@Repository`, `@Controller`, or beans declared via `@Bean` methods in `@Configuration` classes.
2. **Instantiate beans** — At application startup, Spring's IoC container creates instances of these classes and stores them in the **ApplicationContext** (a container/registry of all managed beans).
3. **Resolve dependencies** — When Spring creates a bean that needs another bean (e.g. `UserService` needs `UserRepository`), it looks in the ApplicationContext for a matching bean and injects it.
4. **Wire everything together** — This happens recursively; Spring builds a dependency graph and resolves it in the correct order.
5. **Manage lifecycle** — Spring also manages when beans are created, initialized, and destroyed (especially relevant for singleton-scoped beans that live for the app's lifetime).

---

## 4. Types of Dependency Injection

### A. Constructor Injection (recommended)

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;

    // With one constructor, @Autowired is optional (Spring 4.3+)
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}
```

**Why recommended:**
- Allows fields to be `final` (immutable, thread-safe)
- Dependencies are guaranteed to be non-null once the object is constructed
- Makes testing easy — just pass mocks into the constructor directly
- Fails fast at startup if a dependency is missing (rather than a NullPointerException later)

### B. Setter Injection

```java
@Service
public class UserService {
    private UserRepository userRepository;

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

Useful for **optional dependencies** — the bean can function (perhaps in a limited way) even if this setter is never called.

### C. Field Injection (discouraged)

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
}
```

Easiest to write, but:
- Can't make the field `final`
- Hides dependencies (not visible in constructor signature — harder to see what a class needs at a glance)
- Harder to unit test without Spring or reflection-based mocking
- Circular dependencies fail silently until runtime

---

## 5. How Spring Resolves *Which* Bean to Inject

Spring primarily injects **by type**. If there's ambiguity (multiple beans of the same type), it needs help:

```java
public interface Sender { void send(String msg); }

@Component("emailSender")
public class EmailSender implements Sender { }

@Component("smsSender")
public class SmsSender implements Sender { }

@Service
public class NotificationService {
    private final Sender sender;

    public NotificationService(@Qualifier("emailSender") Sender sender) {
        this.sender = sender; // explicitly picks EmailSender
    }
}
```

Alternatively, mark one as the default:

```java
@Component
@Primary
public class EmailSender implements Sender { }
```

---

## 6. Bean Scopes (affects how DI instances behave)

| Scope | Behavior |
| ----- | -------- |
| `singleton` (default) | One shared instance per Spring container |
| `prototype`           | New instance every time the bean is requested |
| `request`             | One instance per HTTP request (web apps) |
| `session`             | One instance per HTTP session |

```java
@Component
@Scope("prototype")
public class ReportGenerator { }
```

---

## 7. DI and Testing — Where It Really Pays Off

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService; // Mockito injects the mock via constructor

    @Test
    void testGetUser() {
        when(userRepository.findById(1L)).thenReturn(new User("Alice"));
        User result = userService.getUser(1L);
        assertEquals("Alice", result.getName());
    }
}
```

Because `UserService` receives `UserRepository` via its constructor, you can substitute a **mock** in tests instead of hitting a real database — this is the single biggest practical benefit of DI.

---

## 8. Circular Dependency Problem

```java
@Service
public class A {
    public A(B b) { }
}

@Service
public class B {
    public B(A a) { }  // circular!
}
```

With **constructor injection**, this throws `BeanCurrentlyInCreationException` at startup — Spring can't decide which one to build first.

**Fixes:**
- Refactor to remove the circular reference (usually a sign of poor separation of concerns)
- Use setter/field injection for one side (breaks the immediate cycle at construction time, though it's a workaround, not a real fix)
- Use `@Lazy` on one dependency to defer its initialization

---

## 9. DI vs IoC — Clarifying the Relationship

- **IoC (Inversion of Control)** is the broader principle: the framework controls object creation/flow, not your code.
- **DI (Dependency Injection)** is *one implementation* of IoC — specifically about how dependencies get supplied to objects.
- Other IoC mechanisms include the Service Locator pattern (though DI is generally preferred since dependencies are explicit, not hidden behind a lookup call).

---

## 10. Common Interview Questions on DI

**Q1: Why is Constructor Injection preferred over Field Injection?**

It allows immutable (`final`) fields, makes dependencies explicit and testable without a Spring context, and fails fast at startup if a required bean is missing — field injection can hide missing dependencies until runtime.

**Q2: What is `@Autowired` doing internally?**

It tells Spring's `AutowiredAnnotationBeanPostProcessor` to inject a matching bean from the ApplicationContext, resolved by type (and `@Qualifier`/bean name if there's ambiguity).

**Q3: What happens if Spring can't find a bean to inject?**

It throws `NoSuchBeanDefinitionException` at startup (with constructor injection) — this is one reason constructor injection is favored, since the failure surfaces immediately rather than at first use.

**Q4: What if there are multiple beans of the same type?**

Spring throws `NoUniqueBeanDefinitionException` unless you disambiguate with `@Qualifier`, `@Primary`, or by injecting a `List<Sender>`/`Map<String, Sender>` to get all implementations.

**Q5: Can you inject a `List` of all beans implementing an interface?**

```java
@Autowired
private List<Sender> allSenders; // Spring injects every Sender bean automatically
```

Yes — useful for strategy patterns where you want to iterate over all implementations.

**Q6: Is DI specific to Spring?**

No — DI is a general OOP design pattern, also used in .NET (via built-in DI container), Angular, Google Guice, Dagger (Android), etc. Spring just provides one popular implementation via its IoC container.