---
name: clean-architecture
description: "클린 아키텍처(헥사고날 아키텍처) 적용 가이드. 프로젝트 코드 생성 시 도메인 중심의 의존성 규칙을 따르는 디렉토리 구조와 패턴을 제공합니다. 사용자가 클린 아키텍처를 선택했거나 새 프로젝트 구조를 설계할 때 사용하세요."
---

# Clean Architecture Guide

## Overview

클린 아키텍처(헥사고날 아키텍처/포트와 어댑터)를 적용하여 비즈니스 로직이 외부 의존성(DB, 프레임워크, UI)에서 분리된 구조를 설계합니다.

## 핵심 원칙

```
┌─────────────────────────────────────────────────────────────────┐
│                     의존성 규칙                                  │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Adapter Layer                         │   │
│   │   (Controller, Repository구현체, External Services)     │   │
│   │                         │                                │   │
│   │                         ▼                                │   │
│   │   ┌─────────────────────────────────────────────────┐   │   │
│   │   │              Application Layer                   │   │   │
│   │   │         (UseCase, Port Interface)               │   │   │
│   │   │                     │                            │   │   │
│   │   │                     ▼                            │   │   │
│   │   │   ┌─────────────────────────────────────────┐   │   │   │
│   │   │   │           Domain Layer                   │   │   │   │
│   │   │   │    (Entity, Value Object, Domain         │   │   │   │
│   │   │   │     Service, Domain Event)               │   │   │   │
│   │   │   └─────────────────────────────────────────┘   │   │   │
│   │   └─────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   의존성 방향: Adapter → Application → Domain                    │
│   도메인은 외부를 모른다!                                        │
└─────────────────────────────────────────────────────────────────┘
```

### 의존성 규칙
1. **Domain Layer**: 어떤 외부 의존성도 없음 (순수 비즈니스 로직)
2. **Application Layer**: Domain에만 의존, Port 인터페이스 정의
3. **Adapter Layer**: Application과 Domain에 의존, Port 구현

## Spring Boot 디렉토리 구조

### 패키지 구조 (권장)

```
src/main/java/com/example/myapp/
├── domain/                          # 도메인 레이어
│   ├── model/                       # 엔티티 & 값 객체
│   │   ├── User.java               # 도메인 엔티티
│   │   ├── UserId.java             # 값 객체 (ID)
│   │   └── Email.java              # 값 객체
│   ├── service/                     # 도메인 서비스
│   │   └── UserDomainService.java
│   ├── event/                       # 도메인 이벤트
│   │   └── UserCreatedEvent.java
│   └── exception/                   # 도메인 예외
│       └── InvalidEmailException.java
│
├── application/                     # 애플리케이션 레이어
│   ├── port/
│   │   ├── in/                      # 인바운드 포트 (UseCase)
│   │   │   ├── CreateUserUseCase.java
│   │   │   └── GetUserUseCase.java
│   │   └── out/                     # 아웃바운드 포트 (Repository 인터페이스)
│   │       ├── UserRepository.java
│   │       └── EmailSender.java
│   ├── service/                     # 유스케이스 구현
│   │   └── UserService.java
│   └── dto/                         # Command & Query 객체
│       ├── CreateUserCommand.java
│       └── UserResponse.java
│
└── adapter/                         # 어댑터 레이어
    ├── in/
    │   └── web/                     # 웹 어댑터 (Controller)
    │       ├── UserController.java
    │       └── dto/
    │           └── CreateUserRequest.java
    └── out/
        ├── persistence/             # 영속성 어댑터
        │   ├── UserJpaRepository.java    # Spring Data JPA
        │   ├── UserPersistenceAdapter.java
        │   └── entity/
        │       └── UserJpaEntity.java    # JPA 엔티티 (도메인과 분리)
        └── external/                # 외부 서비스 어댑터
            └── SmtpEmailSender.java
```

### 테스트 구조

```
src/test/java/com/example/myapp/
├── domain/
│   └── model/
│       ├── UserTest.java           # 단위 테스트 (순수)
│       └── EmailTest.java
├── application/
│   └── service/
│       └── UserServiceTest.java    # 단위 테스트 (Mock 사용)
└── adapter/
    ├── in/web/
    │   └── UserControllerTest.java # @WebMvcTest
    └── out/persistence/
        └── UserPersistenceAdapterTest.java  # @DataJpaTest
```

## 레이어별 상세 가이드

### 1. Domain Layer

도메인 레이어는 **순수한 비즈니스 로직**만 포함합니다.

```java
// 도메인 엔티티 - JPA 어노테이션 없음!
public class User {
    private final UserId id;
    private Email email;
    private boolean active;

    public User(UserId id, Email email) {
        this.id = id;
        this.email = email;
        this.active = false;
    }

    public void activate() {
        if (this.active) {
            throw new IllegalStateException("User is already active");
        }
        this.active = true;
    }

    // getter만, setter 없음 (불변성)
}

// 값 객체 - 자체 검증 로직 포함
public record Email(String value) {
    public Email {
        if (value == null || !value.matches("^[\\w-\\.]+@[\\w-]+\\.[a-z]{2,}$")) {
            throw new InvalidEmailException(value);
        }
    }
}
```

**규칙:**
- 프레임워크 어노테이션 금지 (@Entity, @Component 등)
- 외부 라이브러리 의존 최소화
- 불변성 유지 (final, record 사용)
- 자체 검증 로직 포함

### 2. Application Layer

애플리케이션 레이어는 **유스케이스를 조율**합니다.

```java
// 인바운드 포트 (UseCase 인터페이스)
public interface CreateUserUseCase {
    UserResponse createUser(CreateUserCommand command);
}

// 아웃바운드 포트 (Repository 인터페이스)
public interface UserRepository {
    User save(User user);
    Optional<User> findById(UserId id);
    Optional<User> findByEmail(Email email);
}

// 유스케이스 구현
@Service
@RequiredArgsConstructor
public class UserService implements CreateUserUseCase, GetUserUseCase {
    private final UserRepository userRepository;  // 포트에 의존

    @Override
    @Transactional
    public UserResponse createUser(CreateUserCommand command) {
        Email email = new Email(command.email());

        // 중복 검사 (비즈니스 규칙)
        if (userRepository.findByEmail(email).isPresent()) {
            throw new DuplicateEmailException(email);
        }

        User user = new User(UserId.generate(), email);
        user.activate();

        User savedUser = userRepository.save(user);
        return UserResponse.from(savedUser);
    }
}
```

**규칙:**
- 인바운드 포트: 외부에서 애플리케이션을 호출하는 인터페이스
- 아웃바운드 포트: 애플리케이션이 외부를 호출하는 인터페이스
- 트랜잭션은 이 레이어에서 관리

### 3. Adapter Layer

어댑터 레이어는 **외부와의 통신**을 담당합니다.

```java
// 웹 어댑터 (Controller)
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {
    private final CreateUserUseCase createUserUseCase;  // 포트에 의존

    @PostMapping
    public ResponseEntity<UserResponse> createUser(@RequestBody CreateUserRequest request) {
        CreateUserCommand command = request.toCommand();
        UserResponse response = createUserUseCase.createUser(command);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}

// 영속성 어댑터
@Component
@RequiredArgsConstructor
public class UserPersistenceAdapter implements UserRepository {
    private final UserJpaRepository jpaRepository;

    @Override
    public User save(User user) {
        UserJpaEntity entity = UserJpaEntity.from(user);
        UserJpaEntity saved = jpaRepository.save(entity);
        return saved.toDomain();
    }

    @Override
    public Optional<User> findById(UserId id) {
        return jpaRepository.findById(id.value())
                .map(UserJpaEntity::toDomain);
    }
}

// JPA 엔티티 (도메인 엔티티와 분리)
@Entity
@Table(name = "users")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class UserJpaEntity {
    @Id
    private String id;
    private String email;
    private boolean active;

    public static UserJpaEntity from(User user) {
        UserJpaEntity entity = new UserJpaEntity();
        entity.id = user.getId().value();
        entity.email = user.getEmail().value();
        entity.active = user.isActive();
        return entity;
    }

    public User toDomain() {
        User user = new User(new UserId(id), new Email(email));
        if (active) user.activate();
        return user;
    }
}
```

**규칙:**
- 도메인 엔티티 ↔ JPA 엔티티 매핑 로직 포함
- 컨트롤러는 UseCase 포트에만 의존
- 영속성 어댑터는 Repository 포트를 구현

## 멀티 모듈 구조 (대규모 프로젝트)

```
my-app/
├── domain/                          # 모듈: 도메인
│   ├── build.gradle
│   └── src/main/java/
│       └── com/example/domain/
│
├── application/                     # 모듈: 애플리케이션
│   ├── build.gradle                 # domain 모듈 의존
│   └── src/main/java/
│       └── com/example/application/
│
├── adapter-web/                     # 모듈: 웹 어댑터
│   ├── build.gradle                 # application 모듈 의존
│   └── src/main/java/
│       └── com/example/adapter/web/
│
├── adapter-persistence/             # 모듈: 영속성 어댑터
│   ├── build.gradle                 # application 모듈 의존
│   └── src/main/java/
│       └── com/example/adapter/persistence/
│
└── bootstrap/                       # 모듈: 애플리케이션 부트스트랩
    ├── build.gradle                 # 모든 모듈 의존
    └── src/main/java/
        └── com/example/MyApplication.java
```

## 체크리스트

프로젝트 구조 검증 시 확인할 사항:

- [ ] Domain 레이어에 프레임워크 어노테이션이 없는가?
- [ ] Domain 엔티티가 JPA 엔티티와 분리되어 있는가?
- [ ] UseCase 인터페이스가 정의되어 있는가?
- [ ] Repository가 인터페이스(포트)로 정의되어 있는가?
- [ ] Controller가 UseCase 포트에 의존하는가?
- [ ] 의존성 방향이 Adapter → Application → Domain인가?
- [ ] 테스트가 레이어별로 분리되어 있는가?

## 주의사항

1. **과도한 추상화 지양**: 소규모 프로젝트에서는 레이어드 아키텍처가 더 적합할 수 있음
2. **점진적 적용**: 처음부터 완벽한 구조보다 핵심 도메인부터 적용
3. **팀 합의 필수**: 아키텍처 패턴은 팀 전체가 이해하고 따라야 효과적
4. **CRUD 위주 기능**: 단순 CRUD는 클린 아키텍처가 과할 수 있음 (비즈니스 로직이 복잡한 경우에 적합)
