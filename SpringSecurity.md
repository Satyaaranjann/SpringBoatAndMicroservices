# Spring Security — Q&A

---

**Q1. Authentication vs Authorization — what's the difference?**

- **Authentication** — verifying **who** the user is (identity). "Is this really John, and does he have valid credentials?"
- **Authorization** — determining **what** an authenticated user is allowed to do (permissions). "Is John allowed to delete this resource?"

Authentication always happens first; authorization decisions are made based on the identity established during authentication. A request can fail at either stage:
- Fails authentication → **401 Unauthorized** (who are you? you haven't proven it)
- Passes authentication but fails authorization → **403 Forbidden** (I know who you are, but you can't do this)

This distinction — and correctly mapping it to 401 vs 403 — is one of the most commonly asked security fundamentals questions.

---

**Q2. What is the Spring Security Filter Chain?**

Spring Security is built on the **Servlet Filter** mechanism — every incoming HTTP request passes through a chain of filters **before** it ever reaches your controller. Each filter handles one specific security concern, and the request only proceeds to the next filter (and eventually the controller) if it passes.

Key filters in a typical chain (order matters):
1. **`SecurityContextPersistenceFilter`** — restores/stores the `SecurityContext` (who's authenticated) for the request
2. **`UsernamePasswordAuthenticationFilter`** — handles form-login authentication
3. **`BasicAuthenticationFilter`** — handles HTTP Basic auth
4. **Custom JWT filter** (if configured) — validates JWT tokens and sets authentication
5. **`ExceptionTranslationFilter`** — converts security exceptions into appropriate HTTP responses (401/403)
6. **`FilterSecurityInterceptor` / `AuthorizationFilter`** — makes the final authorization decision (allow/deny) based on configured rules

If any filter rejects the request (e.g., no valid token, bad credentials), the chain short-circuits and returns an error response — the controller is never reached.

---

**Q3. How do you configure `SecurityFilterChain`?**

In modern Spring Security (5.7+), configuration is done via a `SecurityFilterChain` bean instead of extending `WebSecurityConfigurerAdapter` (which is deprecated/removed).

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

Key parts:
- **`authorizeHttpRequests`** — defines which URL patterns require what level of access
- **`sessionCreationPolicy(STATELESS)`** — tells Spring not to create/use HTTP sessions, standard for JWT-based/stateless REST APIs
- **`addFilterBefore`** — inserts a custom filter (e.g., JWT validation) at a specific point in the chain

---

**Q4. In-Memory vs Database Authentication — what's the difference?**

**In-Memory Authentication** — user credentials are hardcoded/defined directly in configuration, held in memory. Useful only for quick prototyping, demos, or testing — never production.

```java
@Bean
public UserDetailsService userDetailsService() {
    UserDetails user = User.withUsername("admin")
        .password(passwordEncoder().encode("password"))
        .roles("ADMIN")
        .build();
    return new InMemoryUserDetailsManager(user);
}
```

**Database Authentication** — user credentials are stored in a real database and looked up via a custom `UserDetailsService` implementation, backed by a repository. This is the standard approach for real applications.

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found"));

        return org.springframework.security.core.userdetails.User
            .withUsername(user.getUsername())
            .password(user.getPassword())   // must already be hashed in the DB
            .roles(user.getRole())
            .build();
    }
}
```

Spring Security calls `loadUserByUsername()` automatically during the authentication process, then compares the provided password (hashed) against the stored hash using the configured `PasswordEncoder`.

---

**Q5. What is Password Encoding, and how does `BCryptPasswordEncoder` work?**

Passwords must **never** be stored in plain text. `PasswordEncoder` is Spring Security's abstraction for hashing passwords before storage and verifying them during login without ever decrypting the stored hash (since hashing is one-way).

**`BCryptPasswordEncoder`** is the standard, recommended implementation:
- Uses the **bcrypt** hashing algorithm, which is intentionally slow (configurable "strength"/work factor) to resist brute-force attacks
- Automatically generates and embeds a random **salt** into each hash, so identical passwords produce different hash outputs — preventing rainbow-table attacks
- Verification doesn't decrypt the hash — it re-hashes the input password with the same embedded salt and compares the result

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();  // default strength: 10
}

// Encoding (on registration)
String hashed = passwordEncoder.encode("myPassword123");

// Verifying (on login) — Spring Security does this internally
boolean matches = passwordEncoder.matches("myPassword123", hashed);
```

**Why not MD5/SHA-256 directly?** Those are fast general-purpose hashing algorithms — great for file integrity checks, terrible for passwords, because their speed makes brute-forcing millions of guesses per second feasible. Bcrypt (and similarly Argon2, scrypt) is deliberately slow and tunable to stay resistant as hardware gets faster.

---

**Q6. What is JWT Authentication, and how does it work in a Spring Boot app?**

**JWT (JSON Web Token)** is a compact, self-contained, digitally signed token used for **stateless** authentication — the server doesn't need to store session state; all the necessary claims (user identity, roles, expiry) live inside the token itself, and the signature guarantees it hasn't been tampered with.

**Structure:** `header.payload.signature` (Base64Url-encoded, dot-separated)

**Flow:**
1. Client sends credentials to `/login`
2. Server validates them, generates a signed JWT containing claims (username, roles, expiration), returns it to the client
3. Client stores the token (e.g., in memory or secure storage) and sends it on every subsequent request via the `Authorization: Bearer <token>` header
4. A custom filter (inserted before `UsernamePasswordAuthenticationFilter`) intercepts each request, validates the token's signature and expiry, extracts the identity, and sets the `SecurityContext` — no database/session lookup needed per request

```java
public class JwtAuthFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        String token = extractTokenFromHeader(request);
        if (token != null && jwtUtil.validateToken(token)) {
            String username = jwtUtil.extractUsername(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            UsernamePasswordAuthenticationToken authToken =
                new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(authToken);
        }
        chain.doFilter(request, response);
    }
}
```

**Why it fits REST APIs well:** it's stateless (no server-side session storage), scales horizontally without sticky sessions, and works naturally across services/domains — unlike traditional cookie-based sessions.

---

**Q7. What are OAuth2 and OpenID Connect (OIDC)?**

- **OAuth2** — an **authorization** framework/protocol that lets a user grant a third-party application limited access to their resources on another service, **without sharing their password** (e.g., "Sign in with Google" giving an app access to your calendar, but not your Google password). It's fundamentally about **delegated access**, not identity.
- **OpenID Connect (OIDC)** — a thin **identity layer built on top of OAuth2**. It adds the piece OAuth2 doesn't inherently provide: verified **authentication**/identity, via an additional **ID Token** (a JWT containing identity claims like name, email) alongside OAuth2's access token.

**Key OAuth2 roles:**
- **Resource Owner** — the user
- **Client** — the application requesting access
- **Authorization Server** — issues tokens (e.g., Google, Okta, Keycloak)
- **Resource Server** — hosts the protected resource, validates the token

**In Spring Boot**, `spring-boot-starter-oauth2-client` handles the "login with X" flow, and `spring-boot-starter-oauth2-resource-server` is used when your API itself needs to validate incoming OAuth2/JWT tokens issued by an external provider.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: your-client-id
            client-secret: your-client-secret
            scope: openid, profile, email
```

**Interview-level distinction to nail:** OAuth2 = authorization ("can this app access my data?"), OIDC = authentication ("who is this user, really?"). Confusing the two is a very common mistake interviewers listen for.

---

**Q8. What is Role-Based Access Control (RBAC)?**

A model where permissions are granted based on **roles** assigned to a user, rather than to individual users directly — simplifying permission management at scale (assign/revoke a role instead of managing dozens of individual permissions per user).

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
    .anyRequest().authenticated()
)
```

Roles are typically stored as `GrantedAuthority` objects (conventionally prefixed `ROLE_`, e.g., `ROLE_ADMIN`) attached to the authenticated `UserDetails`. `hasRole("ADMIN")` automatically checks for `ROLE_ADMIN` — the prefix is added implicitly, a detail that trips up many candidates when writing custom `GrantedAuthority` logic manually.

---

**Q9. `@PreAuthorize` vs `@Secured` — what's the difference?**

Both enable **method-level security** — restricting access to specific methods based on the caller's authorities — but differ in expressiveness:

| | `@Secured` | `@PreAuthorize` |
|---|---|---|
| Expression support | No — only simple role names | Yes — full **SpEL** (Spring Expression Language) |
| Can reference method arguments | No | Yes |
| Complex logic (AND/OR/custom checks) | No | Yes |
| Requires enabling | `@EnableGlobalMethodSecurity(securedEnabled = true)` (legacy) / `@EnableMethodSecurity` | `@EnableMethodSecurity` (modern default) |

```java
@Secured("ROLE_ADMIN")
public void deleteUser(Long id) { ... }

@PreAuthorize("hasRole('ADMIN') and #id != authentication.principal.id")
public void deleteUser(Long id) { ... }

@PreAuthorize("hasRole('ADMIN') or #username == authentication.name")
public User getUserProfile(String username) { ... }
```

**In practice:** `@PreAuthorize` is preferred almost universally today because of its flexibility (referencing method parameters, combining multiple conditions) — `@Secured` is largely legacy at this point.

---

**Q10. What is CORS, and how do you configure it in Spring Boot?**

**CORS (Cross-Origin Resource Sharing)** is a browser security mechanism that blocks a web page from making requests to a **different origin** (different domain, protocol, or port) than the one it was served from — unless the server explicitly allows it via response headers. This matters constantly in real-world Spring Boot APIs because the frontend (e.g., `localhost:3000` React app) and backend (`localhost:8080` API) are almost always on different origins during development, and often different subdomains in production.

```java
@Configuration
public class CorsConfig {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                    .allowedOrigins("https://myfrontend.com")
                    .allowedMethods("GET", "POST", "PUT", "DELETE")
                    .allowedHeaders("*")
                    .allowCredentials(true);
            }
        };
    }
}
```

Or within `SecurityFilterChain`:
```java
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

**Important:** CORS is a **browser-enforced** restriction, not a server-side security boundary — it doesn't stop non-browser clients (curl, Postman, server-to-server calls) from hitting your API. It only prevents a browser from letting *JavaScript on another origin* read the response. This distinction is a favorite "trick" interview question — CORS misconfiguration is not the same category of vulnerability as missing authentication.

---

**Q11. What is CSRF Protection, and when should it be enabled or disabled?**

**CSRF (Cross-Site Request Forgery)** is an attack where a malicious site tricks an authenticated user's browser into submitting a request to your application **using the user's existing session cookie**, without their knowledge (e.g., an auto-submitting form on an attacker's page hitting your `/transfer-funds` endpoint while the user is logged in).

**Spring Security's default:** CSRF protection is **enabled by default**, requiring a CSRF token to be included with state-changing requests (POST/PUT/DELETE) — the token is tied to the user's session and validated server-side, so an attacker's cross-site request (which can't know the token) is rejected.

```java
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
);
```

**When to disable it:**
- **Stateless REST APIs using JWT/token-based auth** (not cookies) — CSRF attacks specifically exploit the browser's automatic inclusion of cookies; if you're not using session cookies for auth, CSRF risk doesn't apply the same way, so it's commonly disabled:
  ```java
  http.csrf(csrf -> csrf.disable());
  ```

**When to keep it enabled:**
- Any application using **cookie/session-based authentication**, including traditional server-rendered web apps or SPAs that rely on session cookies

**Interview-level nuance:** disabling CSRF is safe *specifically because* JWT auth doesn't rely on cookies the browser sends automatically — it's not a blanket "REST APIs don't need CSRF protection" rule. If a REST API uses cookie-based session auth instead of JWT, CSRF protection is still very much needed.

---

## Extra / Important Interview Questions

**Q12. If both authentication and authorization fail for a request, which error code is returned?**

**401 Unauthorized** — authentication is checked before authorization; if the user isn't even authenticated, Spring Security never reaches the authorization decision, so 401 takes precedence over 403.

---

**Q13. Why is `SessionCreationPolicy.STATELESS` typically paired with JWT authentication?**

Because JWTs carry all necessary identity/claims within the token itself, there's no need for the server to maintain session state between requests. Setting `STATELESS` tells Spring Security not to create or use `HttpSession` at all — every request is authenticated independently via the token, which is essential for horizontal scaling (any server instance can validate any request without shared session storage).

---

**Q14. What's the risk of storing a JWT in `localStorage` versus an HttpOnly cookie?**

`localStorage` is accessible via JavaScript, making it vulnerable to **XSS (Cross-Site Scripting)** attacks — if an attacker injects malicious JS, they can read and exfiltrate the token directly. An **HttpOnly cookie** can't be accessed by JavaScript at all, mitigating XSS token theft, but reintroduces **CSRF** risk (since cookies are sent automatically by the browser) — meaning CSRF protection needs to be re-enabled if you switch to this storage approach. This trade-off is a strong signal of deeper security understanding in interviews.

---

**Q15. How would you handle JWT token expiration and renewal without forcing the user to log in again constantly?**

Implement a **refresh token** pattern: issue a short-lived access token (e.g., 15 minutes) alongside a longer-lived refresh token (e.g., 7 days, stored more securely). When the access token expires, the client uses the refresh token to obtain a new access token via a dedicated endpoint, without re-entering credentials. The refresh token itself should be revocable server-side (e.g., stored and checked against a database/blacklist) since, unlike a pure JWT, you want the ability to invalidate it if compromised.

---

**Q16. Can you have method-level security (`@PreAuthorize`) AND URL-level security (`authorizeHttpRequests`) in the same application? Which takes precedence?**

Yes, both can coexist and are commonly used together — URL-level rules act as a coarse-grained first line of defense (filter chain level, before the controller is even reached), while method-level annotations provide fine-grained control (e.g., checking that a user can only edit *their own* resource, not just that they hold a role). Both must pass — the request has to satisfy the filter chain's authorization rule (or there's no rule for that path) **and** the annotated method's expression; there's no override relationship, they're independent checks applied at different layers.

---

*Interview tip: Security questions are graded heavily on nuance — "CORS isn't a server-side security boundary," "401 vs 403 ordering," "why disabling CSRF is conditional, not universal." Reciting definitions gets you partway; explaining the *why* behind each trade-off is what separates strong answers.*