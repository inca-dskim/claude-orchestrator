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

## TypeScript/Node.js 디렉토리 구조

### 패키지 구조 (권장)

```
src/
├── domain/                          # 도메인 레이어 (순수 TypeScript)
│   ├── entities/                    # 도메인 엔티티
│   │   ├── Task.ts
│   │   └── TaskStatus.ts
│   ├── value-objects/               # 값 객체
│   │   └── TaskId.ts
│   └── exceptions/                  # 도메인 예외
│       └── InvalidTaskException.ts
│
├── application/                     # 애플리케이션 레이어
│   ├── ports/
│   │   ├── in/                      # 인바운드 포트 (UseCase)
│   │   │   └── CreateTaskUseCase.ts
│   │   └── out/                     # 아웃바운드 포트 (Repository 인터페이스)
│   │       └── TaskRepository.ts
│   ├── usecases/                    # 유스케이스 구현
│   │   └── CreateTaskService.ts
│   └── dto/                         # Command & Query 객체
│       ├── CreateTaskCommand.ts
│       └── TaskResponse.ts
│
├── infrastructure/                  # 인프라스트럭처 레이어 (어댑터)
│   ├── web/                         # 웹 어댑터 (Express/Fastify Controller)
│   │   ├── TaskController.ts
│   │   └── dto/
│   │       └── CreateTaskRequest.ts
│   └── persistence/                 # 영속성 어댑터
│       ├── InMemoryTaskRepository.ts
│       └── PrismaTaskRepository.ts
│
└── index.ts                         # 앱 진입점 (의존성 주입)
```

### 테스트 구조

```
__tests__/
├── domain/
│   └── entities/
│       └── Task.test.ts            # 단위 테스트 (순수)
├── application/
│   └── usecases/
│       └── CreateTaskService.test.ts   # 단위 테스트 (Mock 사용)
└── infrastructure/
    └── web/
        └── TaskController.test.ts  # 통합 테스트 (Supertest)
```

## TypeScript 레이어별 상세 가이드

### 1. Domain Layer (TypeScript)

```typescript
// domain/entities/Task.ts - 순수 TypeScript, 프레임워크 무관
import { v4 as uuid } from 'uuid';

export enum TaskStatus {
  PENDING = 'PENDING',
  IN_PROGRESS = 'IN_PROGRESS',
  COMPLETED = 'COMPLETED',
}

export class Task {
  private constructor(
    private readonly _id: string,
    private _title: string,
    private _status: TaskStatus,
    private readonly _createdAt: Date
  ) {}

  static create(title: string): Task {
    if (!title || title.trim().length === 0) {
      throw new Error('Task title cannot be empty');
    }
    return new Task(uuid(), title.trim(), TaskStatus.PENDING, new Date());
  }

  static reconstruct(id: string, title: string, status: TaskStatus, createdAt: Date): Task {
    return new Task(id, title, status, createdAt);
  }

  complete(): void {
    if (this._status === TaskStatus.COMPLETED) {
      throw new Error('Task is already completed');
    }
    this._status = TaskStatus.COMPLETED;
  }

  get id(): string { return this._id; }
  get title(): string { return this._title; }
  get status(): TaskStatus { return this._status; }
  get createdAt(): Date { return this._createdAt; }
}
```

### 2. Application Layer (TypeScript)

```typescript
// application/ports/out/TaskRepository.ts - 아웃바운드 포트
import { Task } from '../../../domain/entities/Task';

export interface TaskRepository {
  save(task: Task): Promise<Task>;
  findById(id: string): Promise<Task | null>;
  findAll(): Promise<Task[]>;
}

// application/usecases/CreateTaskService.ts - 유스케이스 구현
import { Task } from '../../domain/entities/Task';
import { TaskRepository } from '../ports/out/TaskRepository';

export interface CreateTaskInput {
  title: string;
}

export interface CreateTaskOutput {
  id: string;
  title: string;
  status: string;
  createdAt: Date;
}

export class CreateTaskService {
  constructor(private readonly taskRepository: TaskRepository) {}

  async execute(input: CreateTaskInput): Promise<CreateTaskOutput> {
    const task = Task.create(input.title);
    const savedTask = await this.taskRepository.save(task);

    return {
      id: savedTask.id,
      title: savedTask.title,
      status: savedTask.status,
      createdAt: savedTask.createdAt,
    };
  }
}
```

### 3. Infrastructure Layer (TypeScript)

```typescript
// infrastructure/persistence/InMemoryTaskRepository.ts
import { Task, TaskStatus } from '../../domain/entities/Task';
import { TaskRepository } from '../../application/ports/out/TaskRepository';

export class InMemoryTaskRepository implements TaskRepository {
  private tasks: Map<string, Task> = new Map();

  async save(task: Task): Promise<Task> {
    this.tasks.set(task.id, task);
    return task;
  }

  async findById(id: string): Promise<Task | null> {
    return this.tasks.get(id) || null;
  }

  async findAll(): Promise<Task[]> {
    return Array.from(this.tasks.values());
  }
}

// infrastructure/web/TaskController.ts - Express Controller
import { Router, Request, Response } from 'express';
import { CreateTaskService } from '../../application/usecases/CreateTaskService';
import { TaskRepository } from '../../application/ports/out/TaskRepository';

export class TaskController {
  public router: Router;

  constructor(private readonly taskRepository: TaskRepository) {
    this.router = Router();
    this.initializeRoutes();
  }

  private initializeRoutes(): void {
    this.router.post('/', this.createTask.bind(this));
    this.router.get('/', this.getAllTasks.bind(this));
  }

  private async createTask(req: Request, res: Response): Promise<void> {
    try {
      const service = new CreateTaskService(this.taskRepository);
      const result = await service.execute({ title: req.body.title });
      res.status(201).json(result);
    } catch (error) {
      res.status(400).json({ error: (error as Error).message });
    }
  }

  private async getAllTasks(_req: Request, res: Response): Promise<void> {
    const tasks = await this.taskRepository.findAll();
    res.json(tasks.map(t => ({
      id: t.id,
      title: t.title,
      status: t.status,
      createdAt: t.createdAt,
    })));
  }
}
```

### 4. 진입점 (의존성 주입)

```typescript
// index.ts
import express from 'express';
import { TaskController } from './infrastructure/web/TaskController';
import { InMemoryTaskRepository } from './infrastructure/persistence/InMemoryTaskRepository';

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

// 의존성 주입
const taskRepository = new InMemoryTaskRepository();
const taskController = new TaskController(taskRepository);

app.use('/api/tasks', taskController.router);

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## TypeScript 테스트 예시

### 도메인 단위 테스트 (Jest)

```typescript
// __tests__/domain/entities/Task.test.ts
import { Task, TaskStatus } from '../../../src/domain/entities/Task';

describe('Task', () => {
  describe('create', () => {
    it('should create a task with PENDING status', () => {
      const task = Task.create('Test Task');

      expect(task.title).toBe('Test Task');
      expect(task.status).toBe(TaskStatus.PENDING);
      expect(task.id).toBeDefined();
    });

    it('should throw error when title is empty', () => {
      expect(() => Task.create('')).toThrow('Task title cannot be empty');
    });

    it('should trim whitespace from title', () => {
      const task = Task.create('  Test Task  ');
      expect(task.title).toBe('Test Task');
    });
  });

  describe('complete', () => {
    it('should change status to COMPLETED', () => {
      const task = Task.create('Test Task');
      task.complete();
      expect(task.status).toBe(TaskStatus.COMPLETED);
    });

    it('should throw error when already completed', () => {
      const task = Task.create('Test Task');
      task.complete();
      expect(() => task.complete()).toThrow('Task is already completed');
    });
  });
});
```

### 유스케이스 단위 테스트 (Mock 사용)

```typescript
// __tests__/application/usecases/CreateTaskService.test.ts
import { CreateTaskService } from '../../../src/application/usecases/CreateTaskService';
import { TaskRepository } from '../../../src/application/ports/out/TaskRepository';
import { Task } from '../../../src/domain/entities/Task';

describe('CreateTaskService', () => {
  let mockRepository: jest.Mocked<TaskRepository>;
  let service: CreateTaskService;

  beforeEach(() => {
    mockRepository = {
      save: jest.fn(),
      findById: jest.fn(),
      findAll: jest.fn(),
    };
    service = new CreateTaskService(mockRepository);
  });

  it('should create and save a task', async () => {
    mockRepository.save.mockImplementation(async (task) => task);

    const result = await service.execute({ title: 'New Task' });

    expect(result.title).toBe('New Task');
    expect(result.status).toBe('PENDING');
    expect(mockRepository.save).toHaveBeenCalledTimes(1);
  });

  it('should throw error for empty title', async () => {
    await expect(service.execute({ title: '' }))
      .rejects.toThrow('Task title cannot be empty');
  });
});
```

### 컨트롤러 통합 테스트 (Supertest)

```typescript
// __tests__/infrastructure/web/TaskController.test.ts
import request from 'supertest';
import express from 'express';
import { TaskController } from '../../../src/infrastructure/web/TaskController';
import { InMemoryTaskRepository } from '../../../src/infrastructure/persistence/InMemoryTaskRepository';

describe('TaskController', () => {
  let app: express.Application;

  beforeEach(() => {
    app = express();
    app.use(express.json());

    const repository = new InMemoryTaskRepository();
    const controller = new TaskController(repository);
    app.use('/api/tasks', controller.router);
  });

  describe('POST /api/tasks', () => {
    it('should create a task', async () => {
      const response = await request(app)
        .post('/api/tasks')
        .send({ title: 'New Task' })
        .expect(201);

      expect(response.body.title).toBe('New Task');
      expect(response.body.status).toBe('PENDING');
    });

    it('should return 400 for empty title', async () => {
      await request(app)
        .post('/api/tasks')
        .send({ title: '' })
        .expect(400);
    });
  });

  describe('GET /api/tasks', () => {
    it('should return all tasks', async () => {
      await request(app)
        .post('/api/tasks')
        .send({ title: 'Task 1' });

      const response = await request(app)
        .get('/api/tasks')
        .expect(200);

      expect(response.body).toHaveLength(1);
    });
  });
});
```

## 주의사항

1. **과도한 추상화 지양**: 소규모 프로젝트에서는 레이어드 아키텍처가 더 적합할 수 있음
2. **점진적 적용**: 처음부터 완벽한 구조보다 핵심 도메인부터 적용
3. **팀 합의 필수**: 아키텍처 패턴은 팀 전체가 이해하고 따라야 효과적
4. **CRUD 위주 기능**: 단순 CRUD는 클린 아키텍처가 과할 수 있음 (비즈니스 로직이 복잡한 경우에 적합)
