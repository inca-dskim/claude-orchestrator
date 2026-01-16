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

**Spring Boot 3.x:**
```java
// Controller 테스트 - Spring Boot 3.x
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean  // Spring Boot 3.x
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

**Spring Boot 4.x:**
```java
// Controller 테스트 - Spring Boot 4
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;

@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockitoBean  // Spring Boot 4: @MockBean 대신 사용
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

> **주의**: Spring Boot 4에서 `@MockBean`과 `@SpyBean`은 deprecated되어 제거되었습니다.
> `@MockitoBean`과 `@MockitoSpyBean`을 사용하세요.

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

## TypeScript/Node.js TDD 예시

### 단위 테스트 (Jest)

```typescript
// __tests__/domain/Task.test.ts
import { Task, TaskStatus } from '../../src/domain/Task';

describe('Task', () => {
  // RED: 실패하는 테스트 먼저 작성
  describe('create', () => {
    it('should create a task with PENDING status', () => {
      const task = Task.create('Test Task');

      expect(task.title).toBe('Test Task');
      expect(task.status).toBe(TaskStatus.PENDING);
    });

    it('should throw error when title is empty', () => {
      expect(() => Task.create('')).toThrow('Title cannot be empty');
    });
  });

  describe('complete', () => {
    it('should mark task as completed', () => {
      const task = Task.create('Test');
      task.complete();

      expect(task.status).toBe(TaskStatus.COMPLETED);
    });

    it('should throw error when already completed', () => {
      const task = Task.create('Test');
      task.complete();

      expect(() => task.complete()).toThrow('already completed');
    });
  });
});
```

### Mock을 사용한 서비스 테스트

```typescript
// __tests__/application/CreateTaskService.test.ts
import { CreateTaskService } from '../../src/application/CreateTaskService';
import { TaskRepository } from '../../src/application/ports/TaskRepository';

describe('CreateTaskService', () => {
  let mockRepo: jest.Mocked<TaskRepository>;
  let service: CreateTaskService;

  beforeEach(() => {
    // Mock 설정
    mockRepo = {
      save: jest.fn(),
      findById: jest.fn(),
      findAll: jest.fn(),
    };
    service = new CreateTaskService(mockRepo);
  });

  it('should create and save a task', async () => {
    // Given
    mockRepo.save.mockImplementation(async (task) => task);

    // When
    const result = await service.execute({ title: 'New Task' });

    // Then
    expect(result.title).toBe('New Task');
    expect(mockRepo.save).toHaveBeenCalledTimes(1);
  });

  it('should throw error for invalid input', async () => {
    await expect(service.execute({ title: '' }))
      .rejects.toThrow('Title cannot be empty');

    expect(mockRepo.save).not.toHaveBeenCalled();
  });
});
```

### API 통합 테스트 (Supertest)

```typescript
// __tests__/infrastructure/TaskController.test.ts
import request from 'supertest';
import express from 'express';
import { TaskController } from '../../src/infrastructure/TaskController';
import { InMemoryTaskRepository } from '../../src/infrastructure/InMemoryTaskRepository';

describe('TaskController', () => {
  let app: express.Application;

  beforeEach(() => {
    app = express();
    app.use(express.json());

    const repo = new InMemoryTaskRepository();
    const controller = new TaskController(repo);
    app.use('/api/tasks', controller.router);
  });

  describe('POST /api/tasks', () => {
    it('should create a task and return 201', async () => {
      const response = await request(app)
        .post('/api/tasks')
        .send({ title: 'New Task' })
        .expect(201);

      expect(response.body.title).toBe('New Task');
      expect(response.body.status).toBe('PENDING');
    });

    it('should return 400 for invalid request', async () => {
      await request(app)
        .post('/api/tasks')
        .send({ title: '' })
        .expect(400);
    });
  });
});
```

### TypeScript TDD 워크플로우

1. **RED**: 테스트 파일 먼저 생성
   ```bash
   # 테스트 파일 생성
   touch __tests__/domain/Task.test.ts

   # 테스트 실행 (실패해야 함)
   npm test -- --watch
   ```

2. **GREEN**: 최소 구현 작성
   ```bash
   # 구현 파일 생성
   touch src/domain/Task.ts

   # 테스트 통과 확인
   ```

3. **REFACTOR**: 코드 개선
   ```bash
   # 테스트가 계속 통과하는지 확인하면서 리팩토링
   npm test
   ```

### package.json 테스트 설정

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  },
  "jest": {
    "preset": "ts-jest",
    "testEnvironment": "node",
    "roots": ["<rootDir>/__tests__"],
    "coverageThreshold": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80
      }
    }
  }
}
