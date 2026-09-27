````md
# Spring Boot API Development Standards

## 1. Objective

Use a layered REST API that keeps the persistence model separate from the public API:

- Request DTOs accept client input and contain validation rules.
- Response DTOs define what clients can see.
- Services contain business rules; repositories contain database access.
- A global handler turns exceptions into one consistent error shape.

Never expose JPA entities, passwords, tokens, or internal audit fields directly.

---

## 2. DTOs and Validation

Use a request DTO for input and a response DTO for output. Apply Jakarta Bean Validation to the request DTO and use `@Valid` in the controller.

```java
@Getter
@Builder
public class UserRequest {
    @NotBlank(message = "Name is required")
    @Size(max = 100, message = "Name must not exceed 100 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;
}

@Getter
@Builder
public class UserResponse {
    private Long id;
    private String name;
    private String email;
    private LocalDateTime createdAt;
}
```

```java
@PostMapping
public ResponseEntity<ApiResponse<UserResponse>> createUser(
        @Valid @RequestBody UserRequest request) {
    UserResponse user = userService.createUser(request);

    return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success("User created successfully", user));
}
```

Common validation annotations: `@NotNull`, `@NotBlank`, `@NotEmpty`, `@Size`, `@Email`, `@Min`, `@Max`, `@Positive`, and `@Pattern`.

---

## 3. Entity and Mapper

Entities model database tables only. Use a mapper for request-to-entity and entity-to-response conversion.

```java
@Entity
@Table(name = "users")
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String email;
    private String password;
    private LocalDateTime createdAt;
}

@Component
public class UserMapper {
    public User toEntity(UserRequest request) {
        return User.builder()
                .name(request.getName())
                .email(request.getEmail())
                .build();
    }

    public UserResponse toResponse(User user) {
        return UserResponse.builder()
                .id(user.getId())
                .name(user.getName())
                .email(user.getEmail())
                .createdAt(user.getCreatedAt())
                .build();
    }
}
```

For larger projects, consider MapStruct.

---

## 4. ApiResponse and ApiError

An API envelope is a design convention, not a Spring requirement. If adopted, use it consistently.

```java
@Getter
@Builder
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private List<ApiError> errors;
    private Instant timestamp;

    public static <T> ApiResponse<T> success(String message, T data) {
        return ApiResponse.<T>builder()
                .success(true)
                .message(message)
                .data(data)
                .timestamp(Instant.now())
                .build();
    }

    public static <T> ApiResponse<T> error(
            String message,
            List<ApiError> errors) {
        return ApiResponse.<T>builder()
                .success(false)
                .message(message)
                .errors(errors)
                .timestamp(Instant.now())
                .build();
    }
}

@Getter
@Builder
public class ApiError {
    private String field; // null for non-field errors
    private String message;
}
```

```json
{
  "success": false,
  "message": "Validation failed",
  "data": null,
  "errors": [
    {
      "field": "email",
      "message": "Invalid email format"
    }
  ]
}
```

---

## 5. Custom Exceptions and Global Handling

Throw custom exceptions for expected business failures. Do not return `null` for missing resources.

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

User user = userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException(
                "User not found with id: " + id));
```

Handle validation, known business errors, and unexpected errors centrally. Never expose stack traces or internal details.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleNotFound(
            ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(ApiResponse.error(ex.getMessage(), null));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(
            MethodArgumentNotValidException ex) {
        List<ApiError> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(error -> ApiError.builder()
                        .field(error.getField())
                        .message(error.getDefaultMessage())
                        .build())
                .toList();

        return ResponseEntity.badRequest()
                .body(ApiResponse.error("Validation failed", errors));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleUnexpected(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResponse.error("An unexpected error occurred", null));
    }
}
```

---

## 6. Service and Controller Layers

Controllers handle HTTP, validation, and response status. Services enforce business rules and return DTOs. Repositories perform database access.

```java
@Service
@RequiredArgsConstructor
public class UserServiceImpl implements UserService {
    private final UserRepository userRepository;
    private final UserMapper userMapper;

    @Override
    public UserResponse createUser(UserRequest request) {
        User saved = userRepository.save(userMapper.toEntity(request));
        return userMapper.toResponse(saved);
    }
}

@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;
}
```

---

## 7. Pagination, Sorting, and Filtering

Paginate at the database level with Spring Data `Pageable`; do not load every row and paginate in Java. Use a bounded default page size and stable default sorting.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findByDepartment(String department, Pageable pageable);
}

public Page<UserResponse> getUsers(
        String department,
        Pageable pageable) {

    Page<User> users = (department == null || department.isBlank())
            ? userRepository.findAll(pageable)
            : userRepository.findByDepartment(department, pageable);

    return users.map(userMapper::toResponse);
}
```

```java
@GetMapping
public ResponseEntity<ApiResponse<Page<UserResponse>>> getUsers(
        @RequestParam(required = false) String department,

        @PageableDefault(
                size = 10,
                sort = "id",
                direction = Sort.Direction.ASC
        )
        Pageable pageable) {

    return ResponseEntity.ok(
            ApiResponse.success(
                    "Users retrieved successfully",
                    userService.getUsers(department, pageable)
            )
    );
}
```

```http
GET /api/v1/users
GET /api/v1/users?page=0&size=10
GET /api/v1/users?page=0&size=10&sort=name,asc
GET /api/v1/users?department=IT&page=0&size=10&sort=name,asc
```

Spring Data pages are zero-based: `page=0` is the first page. `Page<T>` includes `content`, `totalElements`, `totalPages`, `size`, and `number`. Later, map `Page<T>` to a custom `PageResponse<T>` if you do not want Spring types in your public API contract.

---

## 8. HTTP Status Codes

| Scenario | Status |
| --- | --- |
| Successful GET | `200 OK` |
| Successful POST | `201 CREATED` |
| Successful PUT/PATCH | `200 OK` |
| Successful DELETE | `204 NO CONTENT` or `200 OK` with a body |
| Invalid request | `400 BAD REQUEST` |
| Authentication required | `401 UNAUTHORIZED` |
| Access denied | `403 FORBIDDEN` |
| Resource not found | `404 NOT FOUND` |
| Duplicate or state conflict | `409 CONFLICT` |
| Unexpected error | `500 INTERNAL SERVER ERROR` |

---

## 9. Recommended Project Structure

```text
src/main/java/com/example/application
├── controller/  UserController.java
├── service/     UserService.java, impl/UserServiceImpl.java
├── repository/  UserRepository.java
├── entity/      User.java
├── dto/         request/UserRequest.java, response/UserResponse.java
├── mapper/      UserMapper.java
├── response/    ApiResponse.java, ApiError.java
└── exception/   GlobalExceptionHandler.java, ResourceNotFoundException.java
```

---

## 10. Development Rules

1. Keep entities inside the persistence layer; use DTOs at the API boundary.
2. Put validation on request DTOs and use `@Valid`.
3. Keep controllers thin, services business-focused, and repositories data-focused.
4. Use a mapper for Entity <-> DTO conversion.
5. Use `@RestControllerAdvice` and custom exceptions for predictable failures.
6. Keep success and error response shapes consistent if using `ApiResponse<T>`.
7. Paginate and filter in the repository query; configure safe defaults.
8. Use accurate HTTP status codes and never expose sensitive data or internal errors.

---

## 11. Final Architecture

```text
Request:
Client -> Controller -> Request DTO + @Valid -> Service
       -> Mapper -> Entity -> Repository -> Database

Response:
Database -> Entity -> Mapper -> Response DTO -> ApiResponse<T> -> Client

Errors:
Exception -> GlobalExceptionHandler -> ApiResponse<Void> -> Client
```
````
