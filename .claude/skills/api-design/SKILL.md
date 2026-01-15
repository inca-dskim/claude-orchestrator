---
name: api-design
description: "RESTful API 설계 가이드. URL 네이밍, HTTP 메서드, 상태 코드, 응답 형식 등 API 설계 원칙을 제공합니다. REST API를 설계하거나 구현할 때 사용하세요."
---

# RESTful API Design Guide

## Overview

일관되고 직관적인 RESTful API를 설계하기 위한 가이드입니다.
REST 원칙을 따르면서 실용적인 API를 구현하는 방법을 제공합니다.

## URL 설계 원칙

### 기본 규칙

```
# 리소스는 명사, 복수형 사용
GET    /api/users              # 사용자 목록 조회
GET    /api/users/{id}         # 특정 사용자 조회
POST   /api/users              # 사용자 생성
PUT    /api/users/{id}         # 사용자 전체 수정
PATCH  /api/users/{id}         # 사용자 부분 수정
DELETE /api/users/{id}         # 사용자 삭제

# 케밥 케이스 사용 (하이픈)
/api/user-profiles             # O
/api/userProfiles              # X
/api/user_profiles             # X

# 소문자 사용
/api/users                     # O
/api/Users                     # X

# 동사 사용 금지
/api/users                     # O
/api/getUsers                  # X
/api/createUser                # X
```

### 계층적 리소스

```
# 1:N 관계 표현
GET    /api/users/{userId}/orders              # 사용자의 주문 목록
GET    /api/users/{userId}/orders/{orderId}    # 사용자의 특정 주문
POST   /api/users/{userId}/orders              # 사용자의 주문 생성

# 깊이는 2단계까지 권장
/api/users/{userId}/orders/{orderId}/items     # 허용하지만 권장하지 않음
/api/orders/{orderId}/items                    # 더 나은 방식
```

### 액션 엔드포인트

```
# 리소스가 아닌 액션이 필요한 경우
POST   /api/users/{id}/activate                # 사용자 활성화
POST   /api/users/{id}/deactivate              # 사용자 비활성화
POST   /api/orders/{id}/cancel                 # 주문 취소
POST   /api/auth/login                         # 로그인
POST   /api/auth/logout                        # 로그아웃
POST   /api/emails/send                        # 이메일 발송

# 검색 엔드포인트
GET    /api/users/search?q=keyword             # 검색
POST   /api/users/search                       # 복잡한 검색 (body 사용)
```

## HTTP 메서드

| 메서드 | 용도 | 멱등성 | 안전 | 요청 Body |
|--------|------|--------|------|-----------|
| GET | 조회 | O | O | X |
| POST | 생성, 액션 | X | X | O |
| PUT | 전체 교체 | O | X | O |
| PATCH | 부분 수정 | X | X | O |
| DELETE | 삭제 | O | X | X |

### PUT vs PATCH

```java
// PUT - 전체 리소스 교체 (모든 필드 필요)
PUT /api/users/1
{
    "email": "new@example.com",
    "name": "New Name",
    "phone": "010-1234-5678",
    "address": "Seoul"
}

// PATCH - 부분 수정 (변경할 필드만)
PATCH /api/users/1
{
    "name": "New Name"
}
```

## HTTP 상태 코드

### 성공 응답 (2xx)

| 코드 | 의미 | 사용 상황 |
|------|------|----------|
| 200 OK | 성공 | GET, PUT, PATCH 성공 |
| 201 Created | 생성됨 | POST로 리소스 생성 성공 |
| 204 No Content | 내용 없음 | DELETE 성공, 응답 body 없음 |

### 클라이언트 오류 (4xx)

| 코드 | 의미 | 사용 상황 |
|------|------|----------|
| 400 Bad Request | 잘못된 요청 | 유효성 검사 실패, 잘못된 형식 |
| 401 Unauthorized | 인증 필요 | 인증 토큰 없음/만료 |
| 403 Forbidden | 권한 없음 | 인증됨, 권한 부족 |
| 404 Not Found | 찾을 수 없음 | 리소스 없음 |
| 409 Conflict | 충돌 | 중복 데이터, 상태 충돌 |
| 422 Unprocessable Entity | 처리 불가 | 비즈니스 로직 위반 |
| 429 Too Many Requests | 요청 과다 | Rate Limit 초과 |

### 서버 오류 (5xx)

| 코드 | 의미 | 사용 상황 |
|------|------|----------|
| 500 Internal Server Error | 서버 오류 | 예상치 못한 서버 오류 |
| 502 Bad Gateway | 게이트웨이 오류 | 외부 서비스 응답 오류 |
| 503 Service Unavailable | 서비스 불가 | 서버 과부하, 점검 중 |

## 응답 형식

### 단일 리소스 응답

```json
// GET /api/users/1
{
    "id": 1,
    "email": "user@example.com",
    "name": "홍길동",
    "status": "ACTIVE",
    "createdAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2024-01-15T10:30:00Z"
}
```

### 컬렉션 응답 (페이징)

```json
// GET /api/users?page=0&size=20
{
    "content": [
        {
            "id": 1,
            "email": "user1@example.com",
            "name": "사용자1"
        },
        {
            "id": 2,
            "email": "user2@example.com",
            "name": "사용자2"
        }
    ],
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 100,
        "totalPages": 5
    }
}
```

### 에러 응답

```json
// 400 Bad Request - 유효성 검사 실패
{
    "status": 400,
    "code": "VALIDATION_ERROR",
    "message": "입력값이 올바르지 않습니다",
    "errors": [
        {
            "field": "email",
            "message": "올바른 이메일 형식이 아닙니다",
            "rejectedValue": "invalid-email"
        },
        {
            "field": "password",
            "message": "비밀번호는 8자 이상이어야 합니다",
            "rejectedValue": "1234"
        }
    ],
    "timestamp": "2024-01-15T10:30:00Z"
}

// 404 Not Found
{
    "status": 404,
    "code": "USER_NOT_FOUND",
    "message": "사용자를 찾을 수 없습니다",
    "timestamp": "2024-01-15T10:30:00Z"
}

// 409 Conflict - 중복
{
    "status": 409,
    "code": "DUPLICATE_EMAIL",
    "message": "이미 사용 중인 이메일입니다",
    "timestamp": "2024-01-15T10:30:00Z"
}
```

## Spring Boot 구현

### 공통 응답 DTO

```java
// 에러 응답
public record ErrorResponse(
    int status,
    String code,
    String message,
    List<FieldError> errors,
    LocalDateTime timestamp
) {
    public record FieldError(
        String field,
        String message,
        Object rejectedValue
    ) {}

    public static ErrorResponse of(int status, String code, String message) {
        return new ErrorResponse(status, code, message, null, LocalDateTime.now());
    }

    public static ErrorResponse of(int status, String code, String message,
                                   List<FieldError> errors) {
        return new ErrorResponse(status, code, message, errors, LocalDateTime.now());
    }
}

// 페이징 응답
public record PageResponse<T>(
    List<T> content,
    PageInfo page
) {
    public record PageInfo(
        int number,
        int size,
        long totalElements,
        int totalPages
    ) {}

    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(
            page.getContent(),
            new PageInfo(
                page.getNumber(),
                page.getSize(),
                page.getTotalElements(),
                page.getTotalPages()
            )
        );
    }
}
```

### Controller 구현

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping
    public ResponseEntity<PageResponse<UserResponse>> getUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        Page<UserResponse> users = userService.getUsers(PageRequest.of(page, size));
        return ResponseEntity.ok(PageResponse.from(users));
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        UserResponse user = userService.getUser(id);
        return ResponseEntity.ok(user);
    }

    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        UserResponse user = userService.createUser(request);
        URI location = URI.create("/api/users/" + user.id());
        return ResponseEntity.created(location).body(user);
    }

    @PatchMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        UserResponse user = userService.updateUser(id, request);
        return ResponseEntity.ok(user);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }

    @PostMapping("/{id}/activate")
    public ResponseEntity<UserResponse> activateUser(@PathVariable Long id) {
        UserResponse user = userService.activateUser(id);
        return ResponseEntity.ok(user);
    }
}
```

### 글로벌 예외 처리

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(
            MethodArgumentNotValidException e) {
        List<ErrorResponse.FieldError> fieldErrors = e.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> new ErrorResponse.FieldError(
                error.getField(),
                error.getDefaultMessage(),
                error.getRejectedValue()
            ))
            .toList();

        return ResponseEntity.badRequest()
            .body(ErrorResponse.of(400, "VALIDATION_ERROR",
                "입력값이 올바르지 않습니다", fieldErrors));
    }

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse.of(404, "USER_NOT_FOUND", e.getMessage()));
    }

    @ExceptionHandler(DuplicateEmailException.class)
    public ResponseEntity<ErrorResponse> handleDuplicateEmail(DuplicateEmailException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(ErrorResponse.of(409, "DUPLICATE_EMAIL", e.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.of(500, "INTERNAL_ERROR", "서버 오류가 발생했습니다"));
    }
}
```

## 쿼리 파라미터

### 필터링

```
GET /api/users?status=ACTIVE                    # 상태 필터
GET /api/users?status=ACTIVE&role=ADMIN         # 복수 필터
GET /api/orders?createdAfter=2024-01-01         # 날짜 필터
GET /api/products?minPrice=1000&maxPrice=5000   # 범위 필터
```

### 정렬

```
GET /api/users?sort=createdAt,desc              # 단일 정렬
GET /api/users?sort=name,asc&sort=createdAt,desc # 복수 정렬
```

### 페이징

```
GET /api/users?page=0&size=20                   # 페이지 기반
GET /api/users?offset=0&limit=20                # 오프셋 기반
GET /api/users?cursor=abc123&size=20            # 커서 기반
```

### 필드 선택 (Sparse Fieldsets)

```
GET /api/users?fields=id,name,email             # 필요한 필드만
GET /api/users/1?include=orders                 # 연관 데이터 포함
```

## API 버전 관리

```
# URL 경로 방식 (권장)
GET /api/v1/users
GET /api/v2/users

# 헤더 방식
GET /api/users
Accept: application/vnd.myapp.v1+json

# 쿼리 파라미터 방식
GET /api/users?version=1
```

## 보안 헤더

```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .headers(headers -> headers
                .contentTypeOptions(Customizer.withDefaults())
                .xssProtection(Customizer.withDefaults())
                .frameOptions(frame -> frame.deny())
            )
            .build();
    }
}
```

## 체크리스트

### URL 설계
- [ ] 리소스는 복수형 명사인가?
- [ ] URL은 소문자, 케밥 케이스인가?
- [ ] 계층 깊이가 2단계 이하인가?

### HTTP 메서드
- [ ] CRUD에 맞는 메서드를 사용하는가?
- [ ] POST는 생성에만 사용하는가?
- [ ] DELETE 후 204를 반환하는가?

### 응답
- [ ] 일관된 응답 형식을 사용하는가?
- [ ] 적절한 상태 코드를 반환하는가?
- [ ] 에러 응답에 충분한 정보가 있는가?

### 문서화
- [ ] API 문서가 최신 상태인가?
- [ ] 요청/응답 예시가 있는가?
