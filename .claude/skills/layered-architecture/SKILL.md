---
name: layered-architecture
description: "레이어드 아키텍처 적용 가이드. Controller-Service-Repository 패턴으로 단순하고 익숙한 구조를 제공합니다. 클린 아키텍처가 과도하거나 빠른 개발이 필요할 때 사용하세요."
---

# Layered Architecture Guide

## Overview

레이어드 아키텍처는 가장 전통적이고 널리 사용되는 아키텍처 패턴입니다.
계층 간 명확한 책임 분리로 이해하기 쉽고, 빠른 개발이 가능합니다.

## 아키텍처 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                    Layered Architecture                         │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │              Presentation Layer                          │  │
│   │         (Controller, DTO, Request/Response)             │  │
│   └─────────────────────────┬───────────────────────────────┘  │
│                             │                                   │
│                             ▼                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │               Business Layer                             │  │
│   │              (Service, Business Logic)                   │  │
│   └─────────────────────────┬───────────────────────────────┘  │
│                             │                                   │
│                             ▼                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │              Persistence Layer                           │  │
│   │           (Repository, Entity, DAO)                      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   의존성 방향: Presentation → Business → Persistence            │
└─────────────────────────────────────────────────────────────────┘
```

### 레이어별 책임

| 레이어 | 책임 | 주요 컴포넌트 |
|--------|------|---------------|
| Presentation | HTTP 요청/응답 처리 | Controller, DTO |
| Business | 비즈니스 로직 수행 | Service |
| Persistence | 데이터 영속화 | Repository, Entity |

## Spring Boot 디렉토리 구조

### 기능별 패키지 (권장)

```
src/main/java/com/example/myapp/
├── user/                            # 사용자 도메인
│   ├── controller/
│   │   └── UserController.java
│   ├── service/
│   │   ├── UserService.java         # 인터페이스
│   │   └── UserServiceImpl.java     # 구현체
│   ├── repository/
│   │   └── UserRepository.java
│   ├── entity/
│   │   └── User.java
│   └── dto/
│       ├── CreateUserRequest.java
│       ├── UpdateUserRequest.java
│       └── UserResponse.java
│
├── order/                           # 주문 도메인
│   ├── controller/
│   ├── service/
│   ├── repository/
│   ├── entity/
│   └── dto/
│
├── common/                          # 공통 모듈
│   ├── config/                      # 설정
│   │   └── WebConfig.java
│   ├── exception/                   # 예외 처리
│   │   ├── GlobalExceptionHandler.java
│   │   └── BusinessException.java
│   └── util/                        # 유틸리티
│       └── DateUtils.java
│
└── MyAppApplication.java
```

### 계층별 패키지 (대안)

```
src/main/java/com/example/myapp/
├── controller/
│   ├── UserController.java
│   └── OrderController.java
├── service/
│   ├── UserService.java
│   └── OrderService.java
├── repository/
│   ├── UserRepository.java
│   └── OrderRepository.java
├── entity/
│   ├── User.java
│   └── Order.java
├── dto/
│   ├── user/
│   └── order/
└── config/
```

## 레이어별 구현 가이드

### 1. Presentation Layer (Controller)

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        UserResponse response = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        UserResponse response = userService.getUser(id);
        return ResponseEntity.ok(response);
    }

    @GetMapping
    public ResponseEntity<Page<UserResponse>> getUsers(
            @PageableDefault(size = 20) Pageable pageable) {
        Page<UserResponse> response = userService.getUsers(pageable);
        return ResponseEntity.ok(response);
    }

    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {
        UserResponse response = userService.updateUser(id, request);
        return ResponseEntity.ok(response);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
}
```

**규칙:**
- 비즈니스 로직 금지 (Service에 위임)
- 요청 검증은 `@Valid`로 처리
- 응답 형식 통일 (ResponseEntity 사용)

### 2. Business Layer (Service)

```java
// 인터페이스 (선택사항)
public interface UserService {
    UserResponse createUser(CreateUserRequest request);
    UserResponse getUser(Long id);
    Page<UserResponse> getUsers(Pageable pageable);
    UserResponse updateUser(Long id, UpdateUserRequest request);
    void deleteUser(Long id);
}

// 구현체
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @Override
    @Transactional
    public UserResponse createUser(CreateUserRequest request) {
        // 중복 검사
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException(request.getEmail());
        }

        // 엔티티 생성
        User user = User.builder()
                .email(request.getEmail())
                .password(passwordEncoder.encode(request.getPassword()))
                .name(request.getName())
                .build();

        User savedUser = userRepository.save(user);
        return UserResponse.from(savedUser);
    }

    @Override
    public UserResponse getUser(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id));
        return UserResponse.from(user);
    }

    @Override
    public Page<UserResponse> getUsers(Pageable pageable) {
        return userRepository.findAll(pageable)
                .map(UserResponse::from);
    }

    @Override
    @Transactional
    public UserResponse updateUser(Long id, UpdateUserRequest request) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id));

        user.update(request.getName(), request.getEmail());
        return UserResponse.from(user);
    }

    @Override
    @Transactional
    public void deleteUser(Long id) {
        if (!userRepository.existsById(id)) {
            throw new UserNotFoundException(id);
        }
        userRepository.deleteById(id);
    }
}
```

**규칙:**
- 트랜잭션 관리 (`@Transactional`)
- 비즈니스 로직 집중
- 예외는 비즈니스 예외로 변환

### 3. Persistence Layer (Repository & Entity)

```java
// Entity
@Entity
@Table(name = "users")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class User extends BaseTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String password;

    @Column(nullable = false)
    private String name;

    @Enumerated(EnumType.STRING)
    private UserStatus status = UserStatus.ACTIVE;

    @Builder
    public User(String email, String password, String name) {
        this.email = email;
        this.password = password;
        this.name = name;
    }

    public void update(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public void deactivate() {
        this.status = UserStatus.INACTIVE;
    }
}

// Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);

    @Query("SELECT u FROM User u WHERE u.status = :status")
    Page<User> findByStatus(@Param("status") UserStatus status, Pageable pageable);
}

// BaseTimeEntity
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter
public abstract class BaseTimeEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

**규칙:**
- Entity는 도메인 로직 최소화 (단순 상태 변경만)
- `@Builder`로 생성 제어
- 기본 생성자 `protected`로 제한

### 4. DTO

```java
// Request DTO
public record CreateUserRequest(
    @NotBlank(message = "이메일은 필수입니다")
    @Email(message = "올바른 이메일 형식이 아닙니다")
    String email,

    @NotBlank(message = "비밀번호는 필수입니다")
    @Size(min = 8, message = "비밀번호는 8자 이상이어야 합니다")
    String password,

    @NotBlank(message = "이름은 필수입니다")
    String name
) {}

// Response DTO
public record UserResponse(
    Long id,
    String email,
    String name,
    String status,
    LocalDateTime createdAt
) {
    public static UserResponse from(User user) {
        return new UserResponse(
            user.getId(),
            user.getEmail(),
            user.getName(),
            user.getStatus().name(),
            user.getCreatedAt()
        );
    }
}
```

## 테스트 구조

```
src/test/java/com/example/myapp/
├── user/
│   ├── controller/
│   │   └── UserControllerTest.java    # @WebMvcTest
│   ├── service/
│   │   └── UserServiceTest.java       # 단위 테스트 (Mockito)
│   └── repository/
│       └── UserRepositoryTest.java    # @DataJpaTest
└── integration/
    └── UserIntegrationTest.java       # @SpringBootTest
```

### 테스트 예시

```java
// Controller 테스트
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void createUser_ValidRequest_ReturnsCreated() throws Exception {
        // given
        CreateUserRequest request = new CreateUserRequest(
            "test@example.com", "password123", "Test User");
        UserResponse response = new UserResponse(
            1L, "test@example.com", "Test User", "ACTIVE", LocalDateTime.now());

        given(userService.createUser(any())).willReturn(response);

        // when & then
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.email").value("test@example.com"));
    }
}

// Service 테스트
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private PasswordEncoder passwordEncoder;

    @InjectMocks
    private UserServiceImpl userService;

    @Test
    void createUser_DuplicateEmail_ThrowsException() {
        // given
        CreateUserRequest request = new CreateUserRequest(
            "existing@example.com", "password123", "Test");
        given(userRepository.existsByEmail(request.email())).willReturn(true);

        // when & then
        assertThatThrownBy(() -> userService.createUser(request))
            .isInstanceOf(DuplicateEmailException.class);
    }
}
```

## 클린 아키텍처와 비교

| 항목 | 레이어드 아키텍처 | 클린 아키텍처 |
|------|------------------|--------------|
| 복잡도 | 낮음 | 높음 |
| 학습 곡선 | 낮음 | 높음 |
| 테스트 용이성 | 보통 | 높음 |
| DB 의존성 | Entity가 JPA에 의존 | 도메인이 독립적 |
| 적합한 프로젝트 | CRUD 위주, 소규모 | 복잡한 비즈니스 로직 |
| 개발 속도 | 빠름 | 상대적으로 느림 |

## 언제 레이어드 아키텍처를 선택하나?

**적합한 경우:**
- CRUD 위주의 단순한 애플리케이션
- 빠른 MVP 개발이 필요한 경우
- 팀이 클린 아키텍처에 익숙하지 않은 경우
- 비즈니스 로직이 단순한 경우
- 프로젝트 규모가 작은 경우

**부적합한 경우:**
- 복잡한 비즈니스 규칙이 많은 경우
- 외부 시스템 연동이 많은 경우
- 도메인 로직의 단위 테스트가 중요한 경우

## 체크리스트

- [ ] Controller에 비즈니스 로직이 없는가?
- [ ] Service에 트랜잭션이 적절히 설정되었는가?
- [ ] Entity의 기본 생성자가 protected인가?
- [ ] DTO와 Entity가 분리되어 있는가?
- [ ] 적절한 예외 처리가 되어 있는가?
- [ ] 입력 검증이 @Valid로 처리되는가?
