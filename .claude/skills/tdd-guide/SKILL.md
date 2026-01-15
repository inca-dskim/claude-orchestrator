---
name: tdd-guide
description: "TDD(테스트 주도 개발) 적용 가이드. 프로젝트 코드 생성 시 Red-Green-Refactor 사이클을 따라 테스트를 먼저 작성하고 구현하는 워크플로우를 제공합니다. 사용자가 TDD를 선택했거나 테스트 코드 작성이 필요할 때 사용하세요."
---

# TDD Guide

## Overview

프로젝트 코드 생성 시 TDD(Test-Driven Development) 원칙을 적용하는 가이드입니다.
테스트를 먼저 작성하고, 테스트를 통과하는 최소한의 코드를 구현한 뒤, 리팩토링하는 사이클을 따릅니다.

## TDD 사이클

```
┌─────────────────────────────────────────────────────────┐
│                    TDD Cycle                            │
│                                                         │
│    ┌─────────┐    ┌─────────┐    ┌─────────────┐       │
│    │  RED    │───▶│  GREEN  │───▶│  REFACTOR   │       │
│    │(실패)   │    │(통과)   │    │(개선)       │       │
│    └─────────┘    └─────────┘    └─────────────┘       │
│         ▲                              │               │
│         └──────────────────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

### 1. RED - 실패하는 테스트 작성

구현하려는 기능에 대한 테스트를 먼저 작성합니다.
이 시점에서 테스트는 **반드시 실패**해야 합니다.

```java
// 예시: 사용자 등록 기능 테스트
@Test
void shouldCreateUserWithValidEmail() {
    // Given
    CreateUserCommand command = new CreateUserCommand("user@example.com", "password123");

    // When
    User user = userService.createUser(command);

    // Then
    assertThat(user.getEmail()).isEqualTo("user@example.com");
    assertThat(user.isActive()).isTrue();
}
```

### 2. GREEN - 테스트 통과하는 최소 코드 작성

테스트를 통과하는 **가장 간단한** 코드를 작성합니다.
완벽한 코드가 아니어도 됩니다. 테스트만 통과하면 됩니다.

```java
// 최소 구현
public User createUser(CreateUserCommand command) {
    User user = new User(command.email(), command.password());
    user.activate();
    return userRepository.save(user);
}
```

### 3. REFACTOR - 코드 개선

테스트가 통과하는 상태를 유지하면서 코드를 개선합니다.
- 중복 제거
- 가독성 향상
- 설계 개선

## 적용 순서

프로젝트 코드 생성 시 다음 순서를 따릅니다:

### Step 1: 도메인 모델 정의

핵심 비즈니스 개념을 먼저 정의합니다.

```
1. 엔티티(Entity) 식별
2. 값 객체(Value Object) 정의
3. 도메인 이벤트 정의
4. 비즈니스 규칙 명세
```

### Step 2: 테스트 작성 (기능별)

각 기능에 대해 테스트를 먼저 작성합니다.

**테스트 작성 순서:**
1. 도메인 로직 단위 테스트 (가장 먼저)
2. 유스케이스/서비스 테스트
3. 어댑터(Controller, Repository) 통합 테스트
4. E2E 테스트 (필요시)

### Step 3: 구현

테스트를 통과하는 코드를 작성합니다.

**구현 순서:**
1. 도메인 모델 (Entity, Value Object)
2. 도메인 서비스
3. 유스케이스/애플리케이션 서비스
4. 어댑터 (Controller, Repository 구현체)

### Step 4: 리팩토링 및 반복

코드를 개선하고 다음 기능으로 넘어갑니다.

## 테스트 종류별 가이드

### 단위 테스트 (Unit Test)

```java
// 도메인 로직 테스트 - 외부 의존성 없음
@Test
void emailShouldBeValid() {
    assertThatThrownBy(() -> new Email("invalid-email"))
        .isInstanceOf(InvalidEmailException.class);
}
```

### 통합 테스트 (Integration Test)

```java
// Repository 통합 테스트
@DataJpaTest
class UserRepositoryTest {
    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldSaveAndFindUser() {
        User user = new User("test@example.com");
        userRepository.save(user);

        Optional<User> found = userRepository.findByEmail("test@example.com");
        assertThat(found).isPresent();
    }
}
```

### API 테스트

```java
// Controller 테스트
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private CreateUserUseCase createUserUseCase;

    @Test
    void shouldCreateUser() throws Exception {
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"email": "user@example.com", "password": "pass123"}
                    """))
            .andExpect(status().isCreated());
    }
}
```

## 테스트 커버리지

### 권장 커버리지

| 영역 | 목표 커버리지 |
|------|--------------|
| 도메인 로직 | 90% 이상 |
| 유스케이스 | 80% 이상 |
| 어댑터 | 70% 이상 |
| 전체 | 80% 이상 |

### 커버리지보다 중요한 것

- 핵심 비즈니스 로직의 엣지 케이스 커버
- 실패 케이스 테스트
- 경계값 테스트

## 주의사항

1. **테스트 없이 코드 작성 금지**: 반드시 테스트 먼저
2. **한 번에 하나씩**: 한 번에 하나의 테스트만 실패하도록
3. **작은 단계로**: 큰 기능도 작은 단계로 나누어 TDD 적용
4. **리팩토링 시 테스트 유지**: 테스트가 깨지면 안 됨

## 프레임워크별 테스트 도구

### Java/Spring Boot
- JUnit 5
- AssertJ
- Mockito
- Spring Test

### TypeScript/Node.js
- Jest
- Supertest

### Python
- pytest
- pytest-mock
