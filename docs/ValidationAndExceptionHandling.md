# Validation & Exception Handling — Q&A

---

**Q1. What is Bean Validation, and how do `@Valid`, `@NotNull`, `@Size`, `@Pattern` work?**

Bean Validation (JSR-380, implemented by Hibernate Validator) lets you declare validation rules directly on a class's fields using annotations, instead of writing manual `if` checks. Spring integrates with it via `@Valid`/`@Validated` on controller method parameters.

```java
public class UserDto {

    @NotNull(message = "Name is required")
    @Size(min = 2, max = 50, message = "Name must be 2-50 characters")
    private String name;

    @NotNull
    @Pattern(regexp = "^[A-Za-z0-9+_.-]+@(.+)$", message = "Invalid email format")
    private String email;

    @Min(18) @Max(100)
    private int age;
}
```

```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody UserDto dto) {
    return ResponseEntity.status(HttpStatus.CREATED).body(userService.create(dto));
}
```

- **`@Valid`** — triggers validation of the annotated object; if any constraint fails, Spring throws `MethodArgumentNotValidException` (for `@RequestBody`) before the method body even runs
- **`@NotNull`** — value must not be null
- **`@Size(min, max)`** — string/collection length must fall in range
- **`@Pattern(regexp)`** — string must match a regex

Other common ones: `@NotBlank` (not null and not just whitespace), `@NotEmpty`, `@Min`/`@Max`, `@Email`, `@Positive`, `@Past`/`@Future` (for dates).

**Important distinction:** `@Valid` is the standard Java/Jakarta annotation; `@Validated` is Spring's own variant that additionally supports validation groups and works on class-level for method parameter validation (e.g., `@PathVariable`, `@RequestParam`).

---

**Q2. How do you write a Custom Validator?**

When built-in constraints aren't enough (e.g., checking a field against a database, or validating a combination of fields), you create a custom annotation + validator.

```java
// 1. Define the annotation
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already in use";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// 2. Implement the validation logic
@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    @Autowired
    private UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        return email != null && !userRepository.existsByEmail(email);
    }
}

// 3. Use it
public class UserDto {
    @UniqueEmail
    private String email;
}
```

This pattern is a common interview exercise — you're expected to know all three pieces: the annotation definition, the `ConstraintValidator` implementation, and how it's applied.

---

**Q3. What are `@ControllerAdvice` and `@RestControllerAdvice`?**

They enable **global, cross-cutting handling** of exceptions (and other concerns like data binding or model attributes) across all `@Controller`/`@RestController` classes, instead of duplicating `try/catch` or `@ExceptionHandler` methods in every controller.

- **`@ControllerAdvice`** — global advice for traditional MVC controllers; methods can return view names
- **`@RestControllerAdvice`** — `@ControllerAdvice` + `@ResponseBody` applied globally, so returned objects are serialized directly (JSON), matching how `@RestController` works for regular endpoints

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(HttpStatus.NOT_FOUND.value(), ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

You can scope a `@ControllerAdvice` to specific packages or controller classes using `basePackages` or `assignableTypes`, rather than applying it globally.

---

**Q4. What is `@ExceptionHandler`?**

Marks a method as the handler for a specific exception type (or list of types). It can be placed:
- **Inside a single controller** — handles that exception only within that controller
- **Inside a `@ControllerAdvice`/`@RestControllerAdvice`** — handles it globally, across all controllers

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(err -> errors.put(err.getField(), err.getDefaultMessage()));
        return ResponseEntity.badRequest().body(errors);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse(500, "Unexpected error occurred"));
    }
}
```

Spring matches the **most specific** exception handler available — if both `ResourceNotFoundException` and a generic `Exception` handler exist, the more specific one wins for that exception type.

---

**Q5. How do you define Custom Exception Classes, and why do it?**

Custom exceptions express domain-specific failure conditions clearly (e.g., "user not found" vs. a generic `RuntimeException`), and let you attach exception-specific handling and HTTP status mapping.

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) {
        super(message);
    }
}
```

Usage:
```java
public User findById(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User not found with id: " + id));
}
```

**Best practice:** extend `RuntimeException` (unchecked) rather than `Exception` (checked) for most application-level exceptions — it avoids forcing every calling method up the stack to declare `throws`, and pairs cleanly with `@ExceptionHandler`, which works well with unchecked exceptions bubbling up naturally to the advice layer.

You can also skip a `@ExceptionHandler` entirely and annotate the exception directly:
```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException { ... }
```
This is simpler but less flexible — you can't customize the response body structure this way, only the status code.

---

**Q6. What should a Global Error Response Structure look like?**

A consistent, predictable error format across all endpoints — so API consumers can parse errors the same way regardless of which endpoint failed.

```java
public class ErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private List<String> details;   // e.g., field-level validation errors
    // constructors, getters/setters
}
```

Example JSON response:
```json
{
  "timestamp": "2026-08-09T10:15:30",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "path": "/api/v1/users",
  "details": [
    "name: Name must be 2-50 characters",
    "email: Invalid email format"
  ]
}
```

Wired up centrally:
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
        List<String> details = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.toList());

        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(), 400, "Bad Request",
            "Validation failed", request.getRequestURI(), details);

        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(), 404, "Not Found",
            ex.getMessage(), request.getRequestURI(), null);
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

**Why this matters in interviews:** it shows you think about API consumers, not just "does it work" — a consistent error contract is a strong signal of production-quality API design.

---

## Extra / Important Interview Questions

**Q7. What's the difference between `@Valid` and `@Validated`?**

- `@Valid` — standard Jakarta Bean Validation annotation; triggers validation but has no support for validation groups
- `@Validated` — Spring's own annotation; supports **validation groups** (validating different subsets of constraints in different contexts, e.g., "create" vs "update"), and is required (instead of `@Valid`) when validating `@PathVariable`/`@RequestParam` at the method level, or applying class-level validation via `@Validated` on the controller itself

```java
public interface OnCreate {}
public interface OnUpdate {}

public class UserDto {
    @Null(groups = OnCreate.class)
    @NotNull(groups = OnUpdate.class)
    private Long id;
}

@PostMapping
public ResponseEntity<User> create(@Validated(OnCreate.class) @RequestBody UserDto dto) { ... }
```

---

**Q8. What happens if validation fails on a `@RequestBody` vs a `@RequestParam`?**

- **`@RequestBody` + `@Valid`** → throws `MethodArgumentNotValidException`
- **`@RequestParam`/`@PathVariable` + `@Validated` on the class** → throws `ConstraintViolationException`

These are **different exception types**, so a global handler needs to catch both separately if you're validating both kinds of input:
```java
@ExceptionHandler(MethodArgumentNotValidException.class)   // body validation
@ExceptionHandler(ConstraintViolationException.class)       // param/path validation
```
This is a classic "gotcha" interview question — many candidates only handle one and miss the other.

---

**Q9. If you don't define a global exception handler, what does Spring Boot return by default?**

Spring Boot's default `/error` endpoint (via `BasicErrorController`) returns a generic Whitelabel Error Page (HTML) or a generic JSON error body like:
```json
{
  "timestamp": "...",
  "status": 500,
  "error": "Internal Server Error",
  "path": "/api/v1/users"
}
```
It lacks any domain-specific detail (no field-level messages, generic status codes), which is exactly why teams implement a `@RestControllerAdvice` — to replace this default with a structured, informative response.

---

**Q10. Should you ever expose the raw exception message/stack trace to API clients?**

No — not for unexpected/internal exceptions. Leaking stack traces or internal exception messages (e.g., raw SQL exceptions) is a **security risk** (exposes implementation details, table names, library versions) and unhelpful to API consumers. Best practice: log the full exception server-side (with stack trace), but return a generic, safe message to the client for unexpected errors — while still returning specific, helpful messages for expected/validated business exceptions (like "user not found").

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
    log.error("Unexpected error", ex);   // full detail logged internally
    return ResponseEntity.status(500)
        .body(new ErrorResponse(500, "An unexpected error occurred")); // generic to client
}
```

---

**Q11. How would you handle exceptions thrown inside an `@Async` method? Does `@ExceptionHandler` catch them?**

No — `@ExceptionHandler`/`@ControllerAdvice` only catches exceptions thrown within the normal Spring MVC request-handling thread. Since `@Async` methods run on a **separate thread**, exceptions thrown there don't propagate back to the caller and are invisible to the global exception handler. To handle them, you implement `AsyncUncaughtExceptionHandler` and register it via `AsyncConfigurer`, or handle the `CompletableFuture`'s exception explicitly with `.exceptionally()`.

---

**Q12. Why is exception handling considered a "cross-cutting concern," and how does `@ControllerAdvice` relate to AOP?**

A cross-cutting concern is logic needed across many unrelated parts of an application (logging, security, transactions, exception handling) that doesn't belong to any single module's core responsibility. `@ControllerAdvice` is conceptually similar to **Aspect-Oriented Programming (AOP)** — it intercepts behavior (exceptions) across many controllers without each controller needing to know about it, keeping business logic in controllers/services clean and centralizing the concern in one place.

---

*Interview tip: A strong answer to "how do you handle exceptions in Spring Boot" walks through the layers — custom exceptions → `@ExceptionHandler` → `@RestControllerAdvice` → consistent error response structure — rather than describing each piece in isolation. That end-to-end narrative is what distinguishes a senior-level answer.*