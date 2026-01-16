---
name: react-architecture
description: "React 프론트엔드 아키텍처 가이드. Feature-based 구조로 컴포넌트, 상태 관리, API 호출을 체계적으로 구성합니다. React + TypeScript 프로젝트 생성 시 사용하세요."
---

# React Frontend Architecture Guide

## Overview

React 프론트엔드 프로젝트를 위한 아키텍처 가이드입니다.
Feature-based 구조로 관심사를 분리하고, 확장 가능한 코드베이스를 구성합니다.

## 아키텍처 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                    React Architecture                            │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                      App Layer                           │   │
│   │         (Router, Store, Global Providers)               │   │
│   └─────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│                             ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Features Layer                         │   │
│   │    (Feature Modules: components, hooks, api, types)     │   │
│   └─────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│                             ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Shared Layer                          │   │
│   │      (Common Components, Hooks, Utils, Types)           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   의존성 방향: App → Features → Shared                           │
└─────────────────────────────────────────────────────────────────┘
```

## 프로젝트 디렉토리 구조

### Feature-based 구조 (권장)

```
src/
├── app/                         # 앱 설정
│   ├── App.tsx                  # 루트 컴포넌트
│   ├── router/                  # 라우팅 설정
│   │   └── index.tsx
│   ├── store/                   # 전역 상태 관리 (선택)
│   │   └── index.ts
│   └── providers/               # 전역 Provider
│       └── AppProviders.tsx
│
├── features/                    # 기능별 모듈
│   ├── auth/                    # 인증 기능
│   │   ├── components/          # 기능 전용 컴포넌트
│   │   │   ├── LoginForm.tsx
│   │   │   └── SignupForm.tsx
│   │   ├── hooks/               # 커스텀 훅
│   │   │   └── useAuth.ts
│   │   ├── api/                 # API 호출
│   │   │   └── authApi.ts
│   │   ├── types/               # 타입 정의
│   │   │   └── index.ts
│   │   └── index.ts             # 기능 진입점 (export)
│   │
│   └── tasks/                   # 할일 기능
│       ├── components/
│       │   ├── TaskList.tsx
│       │   ├── TaskItem.tsx
│       │   └── TaskForm.tsx
│       ├── hooks/
│       │   ├── useTaskList.ts
│       │   └── useCreateTask.ts
│       ├── api/
│       │   └── taskApi.ts
│       ├── types/
│       │   └── index.ts
│       └── index.ts
│
├── shared/                      # 공유 리소스
│   ├── components/              # 공통 UI 컴포넌트
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Modal.tsx
│   │   └── Spinner.tsx
│   ├── hooks/                   # 공통 훅
│   │   ├── useLocalStorage.ts
│   │   └── useDebounce.ts
│   ├── utils/                   # 유틸리티 함수
│   │   ├── formatDate.ts
│   │   └── validators.ts
│   ├── types/                   # 공통 타입
│   │   └── index.ts
│   └── constants/               # 상수
│       └── index.ts
│
├── assets/                      # 정적 자원
│   ├── images/
│   └── styles/
│
└── main.tsx                     # 앱 진입점
```

### 테스트 구조

```
src/
├── features/
│   └── tasks/
│       ├── components/
│       │   ├── TaskList.tsx
│       │   └── __tests__/
│       │       └── TaskList.test.tsx
│       └── hooks/
│           ├── useTaskList.ts
│           └── __tests__/
│               └── useTaskList.test.ts
└── shared/
    └── components/
        ├── Button.tsx
        └── __tests__/
            └── Button.test.tsx
```

## 레이어별 구현 가이드

### 1. Feature Components

```typescript
// features/tasks/components/TaskList.tsx
import { useTaskList } from '../hooks/useTaskList';
import { TaskItem } from './TaskItem';
import { Spinner, ErrorMessage } from '@/shared/components';

export const TaskList = () => {
  const { data: tasks, isLoading, error } = useTaskList();

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage message={error.message} />;

  if (!tasks?.length) {
    return <p className="text-gray-500">할 일이 없습니다.</p>;
  }

  return (
    <ul className="space-y-2">
      {tasks.map((task) => (
        <TaskItem key={task.id} task={task} />
      ))}
    </ul>
  );
};
```

```typescript
// features/tasks/components/TaskItem.tsx
import type { Task } from '../types';

interface TaskItemProps {
  task: Task;
}

export const TaskItem = ({ task }: TaskItemProps) => {
  return (
    <li className="flex items-center gap-2 p-3 bg-white rounded shadow">
      <input
        type="checkbox"
        checked={task.status === 'COMPLETED'}
        readOnly
      />
      <span className={task.status === 'COMPLETED' ? 'line-through' : ''}>
        {task.title}
      </span>
    </li>
  );
};
```

### 2. Custom Hooks (API 연동)

```typescript
// features/tasks/hooks/useTaskList.ts
import { useQuery } from '@tanstack/react-query';
import { taskApi } from '../api/taskApi';

export const useTaskList = () => {
  return useQuery({
    queryKey: ['tasks'],
    queryFn: taskApi.getAll,
  });
};
```

```typescript
// features/tasks/hooks/useCreateTask.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { taskApi } from '../api/taskApi';

export const useCreateTask = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: taskApi.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['tasks'] });
    },
  });
};
```

### 3. API Layer

```typescript
// features/tasks/api/taskApi.ts
import type { Task, CreateTaskInput } from '../types';

const API_BASE = '/api/tasks';

export const taskApi = {
  getAll: async (): Promise<Task[]> => {
    const response = await fetch(API_BASE);
    if (!response.ok) throw new Error('Failed to fetch tasks');
    return response.json();
  },

  create: async (input: CreateTaskInput): Promise<Task> => {
    const response = await fetch(API_BASE, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(input),
    });
    if (!response.ok) throw new Error('Failed to create task');
    return response.json();
  },

  complete: async (id: string): Promise<Task> => {
    const response = await fetch(`${API_BASE}/${id}/complete`, {
      method: 'PATCH',
    });
    if (!response.ok) throw new Error('Failed to complete task');
    return response.json();
  },
};
```

### 4. Types

```typescript
// features/tasks/types/index.ts
export type TaskStatus = 'PENDING' | 'IN_PROGRESS' | 'COMPLETED';

export interface Task {
  id: string;
  title: string;
  status: TaskStatus;
  createdAt: string;
}

export interface CreateTaskInput {
  title: string;
}
```

### 5. Feature Entry Point

```typescript
// features/tasks/index.ts
// Public API for this feature
export { TaskList } from './components/TaskList';
export { TaskForm } from './components/TaskForm';
export { useTaskList } from './hooks/useTaskList';
export { useCreateTask } from './hooks/useCreateTask';
export type { Task, CreateTaskInput } from './types';
```

### 6. Shared Components

```typescript
// shared/components/Button.tsx
import { ButtonHTMLAttributes, ReactNode } from 'react';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger';
  isLoading?: boolean;
  children: ReactNode;
}

export const Button = ({
  variant = 'primary',
  isLoading = false,
  children,
  disabled,
  ...props
}: ButtonProps) => {
  const baseStyles = 'px-4 py-2 rounded font-medium transition-colors';
  const variantStyles = {
    primary: 'bg-blue-500 text-white hover:bg-blue-600',
    secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
    danger: 'bg-red-500 text-white hover:bg-red-600',
  };

  return (
    <button
      className={`${baseStyles} ${variantStyles[variant]} ${
        disabled || isLoading ? 'opacity-50 cursor-not-allowed' : ''
      }`}
      disabled={disabled || isLoading}
      {...props}
    >
      {isLoading ? 'Loading...' : children}
    </button>
  );
};
```

### 7. App Setup

```typescript
// app/App.tsx
import { AppProviders } from './providers/AppProviders';
import { AppRouter } from './router';

export const App = () => {
  return (
    <AppProviders>
      <AppRouter />
    </AppProviders>
  );
};
```

```typescript
// app/providers/AppProviders.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactNode } from 'react';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60,
      retry: 1,
    },
  },
});

export const AppProviders = ({ children }: { children: ReactNode }) => {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};
```

```typescript
// app/router/index.tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { TaskList } from '@/features/tasks';

export const AppRouter = () => {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<TaskList />} />
        {/* Add more routes */}
      </Routes>
    </BrowserRouter>
  );
};
```

## 테스트 가이드

### 컴포넌트 테스트 (React Testing Library)

```typescript
// features/tasks/components/__tests__/TaskList.test.tsx
import { render, screen } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { TaskList } from '../TaskList';

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('TaskList', () => {
  it('should render loading state initially', () => {
    render(<TaskList />, { wrapper: createWrapper() });

    expect(screen.getByRole('status')).toBeInTheDocument();
  });

  it('should render empty state when no tasks', async () => {
    // Mock API to return empty array
    render(<TaskList />, { wrapper: createWrapper() });

    expect(await screen.findByText(/할 일이 없습니다/)).toBeInTheDocument();
  });
});
```

### Hook 테스트

```typescript
// features/tasks/hooks/__tests__/useTaskList.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useTaskList } from '../useTaskList';

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('useTaskList', () => {
  it('should fetch tasks successfully', async () => {
    const { result } = renderHook(() => useTaskList(), {
      wrapper: createWrapper(),
    });

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });

    expect(result.current.data).toBeDefined();
  });
});
```

## 상태 관리 선택 가이드

| 상태 유형 | 권장 도구 |
|-----------|-----------|
| 서버 상태 (API 데이터) | TanStack Query (React Query) |
| 전역 클라이언트 상태 | Zustand 또는 Context API |
| 로컬 컴포넌트 상태 | useState |
| 폼 상태 | React Hook Form |
| URL 상태 | React Router |

## 체크리스트

프로젝트 구조 검증 시 확인할 사항:

- [ ] Feature별로 폴더가 분리되어 있는가?
- [ ] 각 Feature는 components, hooks, api, types를 포함하는가?
- [ ] Feature의 index.ts에서 Public API를 export하는가?
- [ ] Shared 컴포넌트는 재사용 가능하게 설계되었는가?
- [ ] Custom Hook으로 비즈니스 로직이 분리되었는가?
- [ ] API 호출이 별도 파일로 분리되었는가?
- [ ] TypeScript 타입이 정의되어 있는가?
- [ ] 테스트가 컴포넌트/훅 옆에 위치하는가?

## 주의사항

1. **Feature 간 의존성 최소화**: Feature는 서로를 직접 import하지 않음
2. **Shared는 범용적으로**: Shared 컴포넌트는 특정 Feature에 의존하지 않음
3. **API 추상화**: API 호출을 직접 컴포넌트에서 하지 않고 hook을 통해 호출
4. **타입 안전성**: any 사용 지양, 명확한 타입 정의
