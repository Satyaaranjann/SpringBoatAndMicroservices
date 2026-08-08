# REST API Development — Q&A

---

**Q1. `@RestController` vs `@Controller` — what's the difference?**

Both are Spring MVC stereotypes for handling web requests, but they differ in how the return value is treated:

- **`@Controller`** — return value is typically resolved as a **view name** (e.g., a Thymeleaf/JSP template). To return raw data (JSON/XML) from a specific method, you must add `@ResponseBody` on that method.
- **`@RestController`** — equivalent to `@Controller` + `@ResponseBody` applied to **every** method automatically. Every handler method's return value is serialized directly into the HTTP response body (usually as JSON via Jackson).

```java
@Controller
public class WebController {
    @GetMapping("/home")
    public String home() { return "home"; }   // resolves to home.html/jsp view

    @GetMapping("/api/data")
    @ResponseBody
    public Data getData() { return new Data(); }  // returned as JSON
}

@RestController
public class ApiController {
    @GetMapping("/api/data")
    public Data getData() { return new Data(); }  // returned as JSON, no @ResponseBody needed
}
```

**Rule of thumb:** use `@Controller` for traditional server-rendered web apps, `@RestController` for REST APIs.

---

**Q2. What is `@RequestMapping`, and what are the HTTP method-specific mapping annotations?**

`@RequestMapping` maps HTTP requests to handler methods/classes, and can be configured with a path, HTTP method, headers, and more.

```java
@RequestMapping(value = "/users", method = RequestMethod.GET)
public List<User> getUsers() { ... }
```

Spring Boot provides shorthand, method-specific annotations built on top of `@RequestMapping` — these are what's actually used in practice:

| Annotation | Equivalent to | HTTP Method |
|---|---|---|
| `@GetMapping` | `@RequestMapping(method = GET)` | Read |
| `@PostMapping` | `@RequestMapping(method = POST)` | Create |
| `@PutMapping` | `@RequestMapping(method = PUT)` | Full update |
| `@PatchMapping` | `@RequestMapping(method = PATCH)` | Partial update |
| `@DeleteMapping` | `@RequestMapping(method = DELETE)` | Delete |

`@RequestMapping` at the class level is still commonly used to define a shared base path:
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping           // → GET /api/users
    @GetMapping("/{id}")  // → GET /api/users/{id}
}
```

---

**Q3. What is `@PathVariable`?**

Extracts a value directly from the **URI path** and binds it to a method parameter.

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}
```

If the variable name matches the parameter name, you can omit the explicit name; otherwise specify it: `@PathVariable("userId") Long id`. Used when the value is part of the **resource identity** (e.g., `/users/42` — 42 identifies a specific resource).

---

**Q4. What is `@RequestParam`?**

Extracts a value from the **query string** and binds it to a method parameter.

```java
// GET /users/search?name=John&page=0
@GetMapping("/users/search")
public List<User> search(
    @RequestParam String name,
    @RequestParam(defaultValue = "0") int page) {
    ...
}
```

Supports `required` (default `true`) and `defaultValue` attributes. Used for optional filters, pagination, sorting — parameters that qualify a request rather than identify a resource.

**Key distinction interviewers probe:** `@PathVariable` → identifies *which* resource; `@RequestParam` → filters/modifies *how* you query it.

---

**Q5. What are `@RequestBody` and `@ResponseBody`?**

- **`@RequestBody`** — deserializes the incoming HTTP request body (typically JSON) into a Java object, using an `HttpMessageConverter` (Jackson by default)
- **`@ResponseBody`** — serializes the returned Java object into the HTTP response body (JSON by default). Implicit on every method in a `@RestController`.

```java
@PostMapping("/users")
public User createUser(@RequestBody UserDto userDto) {
    return userService.create(userDto);   // implicitly @ResponseBody in a @RestController
}
```

Both rely on Spring's `HttpMessageConverter` mechanism to handle the conversion between JSON and Java objects.

---

**Q6. What is `ResponseEntity`, and why use it instead of returning a plain object?**

`ResponseEntity<T>` represents the **entire HTTP response** — status code, headers, and body — giving you full control, unlike returning a plain object (which always defaults to `200 OK`).

```java
@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    Optional<User> user = userService.findById(id);
    return user.map(ResponseEntity::ok)
               .orElseGet(() -> ResponseEntity.notFound().build());
}

@PostMapping("/users")
public ResponseEntity<User> createUser(@RequestBody UserDto dto) {
    User created = userService.create(dto);
    return ResponseEntity
        .status(HttpStatus.CREATED)
        .header("Location", "/users/" + created.getId())
        .body(created);
}
```

Use it whenever the status code or headers need to vary based on logic (not found → 404, created → 201, etc.) — which is most real-world API scenarios.

---

**Q7. What are the important HTTP Status Codes to know for REST APIs?**

| Code | Meaning | Typical use |
|---|---|---|
| **200 OK** | Success | Successful GET/PUT/PATCH |
| **201 Created** | Resource created | Successful POST |
| **204 No Content** | Success, no body | Successful DELETE |
| **400 Bad Request** | Invalid client input | Failed validation |
| **401 Unauthorized** | Not authenticated | Missing/invalid credentials |
| **403 Forbidden** | Authenticated but not allowed | Insufficient permissions |
| **404 Not Found** | Resource doesn't exist | Invalid ID in URL |
| **409 Conflict** | State conflict | Duplicate resource, version conflict |
| **422 Unprocessable Entity** | Semantically invalid | Valid syntax, invalid business rule |
| **500 Internal Server Error** | Unhandled server exception | Bug/unexpected failure |

**Interview tip:** be ready to justify why a specific code fits a specific scenario — e.g., "why 409 and not 400 for a duplicate email during registration?" (400 = malformed request; 409 = the request is well-formed but conflicts with existing state).

---

**Q8. What is Content Negotiation?**

The process by which a REST API decides **what format** to return a response in (JSON, XML, etc.) based on what the client requests — typically via the `Accept` header.

```
Accept: application/json   → response serialized as JSON
Accept: application/xml    → response serialized as XML (if configured)
```

Spring uses `HttpMessageConverter`s registered for each supported media type, selecting the appropriate one based on the `Accept` header (or a URL suffix/parameter if configured). By default, Spring Boot returns JSON via Jackson unless XML support (Jackson XML or JAXB) is explicitly added.

You can also restrict which media types an endpoint produces/consumes:
```java
@GetMapping(value = "/users/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
@PostMapping(value = "/users", consumes = MediaType.APPLICATION_JSON_VALUE)
```

---

**Q9. What are common API Versioning Strategies?**

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| **URI versioning** | `/api/v1/users` | Simple, visible, easy to route/cache | "Pollutes" the URI, resource identity technically changes per version |
| **Query parameter** | `/api/users?version=1` | Easy to add | Easy to forget/omit, less RESTful |
| **Custom header** | `X-API-Version: 1` | Keeps URI clean | Less discoverable, harder to test via browser |
| **Media type / Accept header** | `Accept: application/vnd.company.v1+json` | Most "RESTful" per HATEOAS purists | Complex to implement and document, poor tooling support |

**Most common in practice:** URI versioning (`/api/v1/...`) — it's the simplest to implement, document, and route at the gateway/load-balancer level, even though it's debated as the "purest" REST approach.

---

**Q10. How do you build a CRUD REST API in Spring Boot? (Full example)**

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public ResponseEntity<List<User>> getAllUsers() {
        return ResponseEntity.ok(userService.findAll());
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return userService.findById(id)
                .map(ResponseEntity::ok)
                .orElseGet(() -> ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody UserDto dto) {
        User created = userService.create(dto);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(@PathVariable Long id, @Valid @RequestBody UserDto dto) {
        return ResponseEntity.ok(userService.update(id, dto));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

This maps the five core operations to the correct HTTP methods and status codes: **GET (200)**, **POST (201)**, **PUT (200)**, **DELETE (204)** — the pattern interviewers expect you to reproduce from memory.

---

## Extra / Important Interview Questions

**Q11. Why is `PUT` supposed to be idempotent but `POST` is not?**

`PUT` replaces a resource entirely at a known URI — calling it multiple times with the same payload results in the same end state (idempotent). `POST` typically creates a *new* resource each time it's called (e.g., submitting the same POST twice creates two separate records) — so it's not idempotent by definition. This distinction is a classic interview question testing REST semantics understanding, not just annotation syntax.

---

**Q12. What's the difference between `PUT` and `PATCH`?**

`PUT` replaces the **entire** resource — any fields omitted from the request body are typically treated as null/reset. `PATCH` applies a **partial update** — only the fields included in the request are modified, the rest remain untouched.

---

**Q13. How does Spring know how to convert a Java object to JSON and back?**

Via `HttpMessageConverter`s — specifically `MappingJackson2HttpMessageConverter`, auto-configured when Jackson is on the classpath (bundled in `spring-boot-starter-web`). It inspects the `Content-Type`/`Accept` headers and the declared return type, then uses Jackson's `ObjectMapper` to serialize/deserialize.

---

**Q14. What happens if `@RequestBody` receives malformed JSON?**

Spring throws an `HttpMessageNotReadableException`, which — if unhandled — results in a `400 Bad Request` response by default. This is commonly caught in a global `@ExceptionHandler` to return a consistent, custom error response format instead of Spring's default error page.

---

**Q15. How would you design pagination for a "get all users" endpoint that could return millions of records?**

Never return the full list. Use `@RequestParam` for `page` and `size` (or cursor-based pagination for very large datasets), combined with Spring Data's `Pageable`:
```java
@GetMapping
public ResponseEntity<Page<User>> getUsers(Pageable pageable) {
    return ResponseEntity.ok(userService.findAll(pageable));
}
// GET /api/v1/users?page=0&size=20&sort=name,asc
```
This is a very common follow-up to CRUD questions — interviewers want to see you think beyond the "happy path" toy example.

---

**Q16. If a client sends `Accept: application/xml` but your API only supports JSON, what happens?**

Spring returns a **`406 Not Acceptable`** response, because none of the registered `HttpMessageConverter`s can satisfy the requested media type. This is a good example question to show you understand what's happening under the hood of content negotiation, not just that JSON "just works."

---

**Q17. Why might you avoid exposing your JPA `@Entity` classes directly as REST API request/response bodies?**

- **Tight coupling** — any DB schema change directly breaks the API contract
- **Over-exposure** — entities may carry internal fields (audit columns, relationships) that shouldn't be public
- **Lazy-loading issues** — serializing an entity with `LAZY` relationships outside a transaction can throw `LazyInitializationException`
- **Validation mismatch** — DB constraints and API input validation rules often differ

The standard fix is a **DTO (Data Transfer Object)** layer — map entities to DTOs (manually, or via MapStruct/ModelMapper) at the controller boundary, keeping the API contract independent of the persistence model.

---

*Interview tip: For REST questions, interviewers often care less about annotation syntax and more about REST principles — idempotency, correct status codes, and why you wouldn't expose entities directly. Anchor your answers in "why," not just "how."*