# TypeScript TDD (테스트 주도 개발) 실천 가이드

TypeScript 프로젝트에서 TDD를 효과적으로 실천하기 위한 가이드입니다.

> 📚 **코딩 스타일**: 이 가이드는 ESLint + Prettier 코딩 컨벤션을 따릅니다.

## 📋 TDD란?

Test-Driven Development는 다음 사이클을 반복하는 개발 방법론입니다:

1. **Red** 🔴: 실패하는 테스트 작성
2. **Green** 🟢: 테스트를 통과하는 최소한의 코드 작성
3. **Refactor** 🔵: 코드 개선

## 🚀 빠른 시작

### 1. 테스트 파일 생성
```bash
# 구현 파일: calculator.ts
# 테스트 파일: calculator.test.ts (반드시 .test.ts 또는 .spec.ts로 끝나야 함)
```

### 2. 첫 번째 테스트 작성 (Red)
```typescript
// calculator.test.ts
import { add } from './calculator';

describe('Calculator', () => {
  test('should add two numbers', () => {
    const result = add(2, 3);
    const expected = 5;
    
    expect(result).toBe(expected);
  });
});
```

### 3. 최소 구현 (Green)
```typescript
// calculator.ts
export function add(a: number, b: number): number {
  return a + b;
}
```

### 4. 리팩토링 (Refactor)
필요한 경우 코드를 개선합니다.

## 📝 TypeScript 테스트 작성 패턴

### 1. 테스트 케이스 배열 패턴 (권장)

```typescript
// calculator.test.ts
import { calculate } from './calculator';

describe('Calculator', () => {
  describe('calculate', () => {
    const testCases = [
      {
        name: 'addition',
        a: 2,
        b: 3,
        op: '+',
        expected: 5,
        shouldThrow: false,
      },
      {
        name: 'subtraction',
        a: 5,
        b: 3,
        op: '-',
        expected: 2,
        shouldThrow: false,
      },
      {
        name: 'division by zero',
        a: 5,
        b: 0,
        op: '/',
        expected: 0,
        shouldThrow: true,
      },
    ];
    
    testCases.forEach(({ name, a, b, op, expected, shouldThrow }) => {
      test(name, () => {
        if (shouldThrow) {
          expect(() => calculate(a, b, op)).toThrow();
        } else {
          const result = calculate(a, b, op);
          expect(result).toBe(expected);
        }
      });
    });
  });
});
```

### 2. 에러 테스트

```typescript
// calculator.test.ts
import { divide } from './calculator';

describe('divide', () => {
  test('should divide two numbers correctly', () => {
    const result = divide(10, 2);
    expect(result).toBe(5);
  });
  
  test('should throw error for division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });
  
  test('should handle negative numbers', () => {
    const result = divide(-10, 2);
    expect(result).toBe(-5);
  });
});
```

### 3. 인터페이스 테스트

```typescript
// types/storage.ts
export interface Storage {
  save(key: string, value: Buffer): Promise<void>;
  load(key: string): Promise<Buffer | null>;
}

// __mocks__/storage.ts
export class MockStorage implements Storage {
  private data = new Map<string, Buffer>();

  async save(key: string, value: Buffer): Promise<void> {
    this.data.set(key, value);
  }

  async load(key: string): Promise<Buffer | null> {
    return this.data.get(key) || null;
  }

  // 테스트용 헬퍼 메서드
  clear(): void {
    this.data.clear();
  }

  has(key: string): boolean {
    return this.data.has(key);
  }
}

// service.test.ts
import { Service } from './service';
import { MockStorage } from '../__mocks__/storage';

describe('Service', () => {
  let storage: MockStorage;
  let service: Service;

  beforeEach(() => {
    storage = new MockStorage();
    service = new Service(storage);
  });

  afterEach(() => {
    storage.clear();
  });

  test('should process and save data', async () => {
    const testData = Buffer.from('test-data');
    
    await service.process('test-key', testData);
    
    // 검증
    const saved = await storage.load('test-key');
    expect(saved).toEqual(testData);
    expect(storage.has('test-key')).toBe(true);
  });

  test('should handle missing data', async () => {
    const result = await storage.load('non-existent-key');
    expect(result).toBeNull();
  });
});
```

## 🎯 TDD 실천 단계별 가이드

### Step 1: 요구사항을 테스트로 변환

**요구사항**: "사용자는 두 숫자를 더할 수 있어야 한다"

```typescript
// calculator.test.ts
import { add } from './calculator';

describe('Calculator', () => {
  test('should add two numbers', () => {
    // Given
    const a = 5;
    const b = 3;
    
    // When
    const result = add(a, b);
    
    // Then
    expect(result).toBe(8);
  });
});
```

### Step 2: 컴파일 에러 해결

```typescript
// calculator.ts
// 최소한의 구현으로 컴파일 에러만 해결
export function add(a: number, b: number): number {
  return 0; // 일단 컴파일만 되도록
}
```

### Step 3: 테스트 통과시키기

```typescript
// calculator.ts
export function add(a: number, b: number): number {
  return a + b; // 실제 구현
}
```

### Step 4: 추가 테스트 케이스

```typescript
// calculator.test.ts
import { add } from './calculator';

describe('Calculator', () => {
  test('should add negative numbers', () => {
    const result = add(-5, 3);
    expect(result).toBe(-2);
  });

  test('should add with zero', () => {
    const result = add(0, 5);
    expect(result).toBe(5);
  });

  test('should add two negative numbers', () => {
    const result = add(-3, -7);
    expect(result).toBe(-10);
  });
});
```

## 🧪 고급 테스트 패턴

### 1. 중첩된 describe 블록 활용

```typescript
// user.service.test.ts
import { UserService } from './user.service';
import { User } from '../types/user';

describe('UserService', () => {
  let service: UserService;

  beforeEach(() => {
    service = new UserService();
  });

  describe('create', () => {
    test('should create user with valid data', async () => {
      const user: User = {
        name: 'John',
        email: 'john@example.com'
      };
      
      const result = await service.create(user);
      
      expect(result).toBeDefined();
      expect(result.id).toBeDefined();
      expect(result.name).toBe(user.name);
    });
    
    test('should throw error for invalid email', async () => {
      const user: User = {
        name: 'John',
        email: 'invalid'
      };
      
      await expect(service.create(user)).rejects.toThrow('Invalid email');
    });
  });
  
  describe('update', () => {
    test('should update existing user', async () => {
      // Update 관련 테스트
      const user = await service.create({
        name: 'John',
        email: 'john@example.com'
      });
      
      const updatedUser = await service.update(user.id, {
        name: 'Jane'
      });
      
      expect(updatedUser.name).toBe('Jane');
    });
  });
});
```

### 2. 테스트 헬퍼 함수

```typescript
// test/helpers.ts
export function createMockUser(overrides: Partial<User> = {}): User {
  return {
    id: 'test-id',
    name: 'Test User',
    email: 'test@example.com',
    ...overrides
  };
}

export function expectToHaveBeenCalledOnceWith<T extends any[]>(
  mockFn: jest.MockedFunction<any>,
  ...args: T
): void {
  expect(mockFn).toHaveBeenCalledTimes(1);
  expect(mockFn).toHaveBeenCalledWith(...args);
}

export async function expectToReject(
  promise: Promise<any>,
  errorMessage?: string
): Promise<void> {
  try {
    await promise;
    throw new Error('Expected promise to reject');
  } catch (error) {
    if (errorMessage) {
      expect(error.message).toBe(errorMessage);
    }
  }
}

// 사용 예
// user.service.test.ts
import { createMockUser, expectToHaveBeenCalledOnceWith } from '../test/helpers';

test('should create user', async () => {
  const mockUser = createMockUser({ name: 'John Doe' });
  const mockRepository = {
    save: jest.fn().mockResolvedValue(mockUser)
  };
  
  const service = new UserService(mockRepository);
  const result = await service.create(mockUser);
  
  expectToHaveBeenCalledOnceWith(mockRepository.save, mockUser);
  expect(result).toEqual(mockUser);
});
```

### 3. 성능 테스트

```typescript
// performance.test.ts
import { performance } from 'perf_hooks';
import { add } from './calculator';

describe('Performance Tests', () => {
  test('add function should be fast', () => {
    const iterations = 1000000;
    const start = performance.now();
    
    for (let i = 0; i < iterations; i++) {
      add(100, 200);
    }
    
    const end = performance.now();
    const duration = end - start;
    
    // 1초 이내에 100만 번 실행되어야 함
    expect(duration).toBeLessThan(1000);
  });
  
  test('string concatenation performance', () => {
    const iterations = 10000;
    const start = performance.now();
    
    for (let i = 0; i < iterations; i++) {
      let s = '';
      for (let j = 0; j < 10; j++) {
        s += 'a';
      }
    }
    
    const end = performance.now();
    const duration = end - start;
    
    console.log(`String concatenation took ${duration}ms`);
  });
});

// jest.config.jsで 벤치마크 테스트를 별도로 실행
// npm test -- --testNamePattern="Performance Tests" --verbose
```

### 4. 비동기 테스트

```typescript
// async.test.ts
import { DatabaseService } from './database.service';
import { UserService } from './user.service';

describe('Async Operations', () => {
  let dbService: DatabaseService;
  let userService: UserService;

  beforeEach(async () => {
    dbService = new DatabaseService();
    userService = new UserService(dbService);
    await dbService.connect();
  });

  afterEach(async () => {
    await dbService.disconnect();
  });

  test('should handle concurrent user creation', async () => {
    const users = [
      { name: 'User1', email: 'user1@example.com' },
      { name: 'User2', email: 'user2@example.com' },
      { name: 'User3', email: 'user3@example.com' },
    ];
    
    // 병렬로 사용자 생성
    const promises = users.map(user => userService.create(user));
    const results = await Promise.all(promises);
    
    expect(results).toHaveLength(3);
    results.forEach((result, index) => {
      expect(result.name).toBe(users[index].name);
      expect(result.email).toBe(users[index].email);
    });
  });
  
  test('should handle promise rejection', async () => {
    const invalidUser = { name: '', email: 'invalid' };
    
    await expect(userService.create(invalidUser))
      .rejects.toThrow('Invalid user data');
  });
  
  test('should timeout long operations', async () => {
    const longOperation = () => new Promise(resolve => 
      setTimeout(resolve, 5000)
    );
    
    const timeoutPromise = new Promise((_, reject) => 
      setTimeout(() => reject(new Error('Timeout')), 1000)
    );
    
    await expect(Promise.race([longOperation(), timeoutPromise]))
      .rejects.toThrow('Timeout');
  }, 2000); // 2초 타임아웃
});
```

## 🔧 테스트 도구 및 명령어

### 기본 테스트 명령어

```bash
# 모든 테스트 실행
npm test

# Watch 모드
npm test -- --watch

# 특정 테스트 파일만 실행
npm test calculator.test.ts

# 테스트 커버리지
npm test -- --coverage

# 상세 모드
npm test -- --verbose

# 특정 테스트 이름으로 실행
npm test -- --testNamePattern="should add"

# 특정 테스트 스위트만 실행
npm test -- --testPathPattern="calculator"

# 업데이트된 테스트만 실행
npm test -- --onlyChanged

# 스냅샷 업데이트
npm test -- --updateSnapshot

# 병렬 실행
npm test -- --maxWorkers=4

# 자세한 에러 정보
npm test -- --verbose --no-coverage

# CI 환경에서 실행
npm test -- --ci --coverage --watchAll=false
```

### 테스트 구조화

```
project/
├── src/
│   ├── calculator.ts
│   └── calculator.test.ts    # 단위 테스트
├── tests/
│   ├── unit/               # 단위 테스트
│   ├── integration/        # 통합 테스트
│   ├── e2e/               # End-to-End 테스트
│   ├── fixtures/           # 테스트 데이터
│   │   └── test_cases.json
│   └── setup.ts            # 테스트 설정
├── __mocks__/             # Mock 파일
│   └── database.ts
└── jest.config.js         # Jest 설정
```

## 📊 테스트 커버리지 목표

### 커버리지 기준
- **최소**: 70%
- **권장**: 80%
- **이상적**: 90%+

### 커버리지 제외 대상
- 생성된 코드 (mockgen, protoc 등)
- main 함수
- 단순 getter/setter

### 커버리지 향상 팁
```typescript
// 경계값 테스트
describe('BoundaryValues', () => {
    const testCases = [
        { name: 'minimum', input: 0, expected: true },
        { name: 'maximum', input: 100, expected: true },
        { name: 'below minimum', input: -1, expected: false },
        { name: 'above maximum', input: 101, expected: false },
    ];

    testCases.forEach(({ name, input, expected }) => {
        it(`should handle ${name} value correctly`, () => {
            const result = validateRange(input);
            expect(result).toBe(expected);
        });
    });
});
```

## ✅ TDD 체크리스트

### 테스트 작성 전
- [ ] 요구사항을 명확히 이해했는가?
- [ ] 테스트 케이스를 도출했는가?
- [ ] 엣지 케이스를 고려했는가?

### 테스트 작성 중
- [ ] 테스트가 실패하는 것을 확인했는가? (Red)
- [ ] 하나의 테스트는 하나의 기능만 테스트하는가?
- [ ] 테스트 이름이 명확한가?

### 구현 중
- [ ] 테스트를 통과하는 최소한의 코드를 작성했는가? (Green)
- [ ] 모든 테스트가 통과하는가?
- [ ] 리팩토링이 필요한가? (Refactor)

### 테스트 작성 후
- [ ] 테스트 커버리지가 충분한가?
- [ ] 테스트가 독립적인가?
- [ ] 테스트 실행 시간이 적절한가?

## 💡 Best Practices

### 1. 테스트는 문서다
```typescript
// 좋은 예: 테스트 이름이 기능을 설명
describe('UserService', () => {
  describe('createUser', () => {
    test('should create user successfully with valid data', async () => {
      // ...
    });
    
    test('should throw ValidationError when email is invalid', async () => {
      // ...
    });
  });
});

// 나쁜 예: 모호한 테스트 이름
test('user test 1', () => {
  // ...
});
```

### 2. AAA 패턴 사용
```typescript
test('should process input correctly', async () => {
  // Arrange (준비)
  const service = new Service();
  const input = 'test data';
  
  // Act (실행)
  const result = await service.process(input);
  
  // Assert (검증)
  expect(result).toBe('expected');
});

// 또는 Given-When-Then 패턴
test('should calculate discount correctly', () => {
  // Given
  const priceCalculator = new PriceCalculator();
  const product = { price: 100, category: 'electronics' };
  
  // When
  const discountedPrice = priceCalculator.calculateDiscount(product);
  
  // Then
  expect(discountedPrice).toBe(90);
});
```

### 3. 테스트 격리
```typescript
// 각 테스트는 독립적이어야 함
describe('UserService', () => {
  let service: UserService;
  let mockDatabase: jest.Mocked<Database>;
  
  beforeEach(() => {
    // 각 테스트마다 새로운 인스턴스 생성
    mockDatabase = {
      save: jest.fn(),
      find: jest.fn(),
      delete: jest.fn(),
    };
    service = new UserService(mockDatabase);
  });
  
  afterEach(() => {
    // 테스트 후 정리
    jest.clearAllMocks();
  });
  
  test('should create user', async () => {
    // 이 테스트는 다른 테스트에 영향을 주지 않음
    const user = { name: 'John', email: 'john@example.com' };
    mockDatabase.save.mockResolvedValue(user);
    
    const result = await service.create(user);
    
    expect(result).toEqual(user);
  });
});
```

### 4. 명확한 실패 메시지
```typescript
// 좋은 예
test('should add two numbers', () => {
  const result = add(2, 3);
  expect(result).toBe(5); // Jest가 자동으로 명확한 메시지 제공
});

// 커스텀 메시지
test('should validate email format', () => {
  const isValid = validateEmail('invalid-email');
  expect(isValid).toBe(true); // 실패 시: Expected: true, Received: false
});

// 더 상세한 에러 메시지
test('should throw specific error for invalid input', () => {
  expect(() => divide(10, 0)).toThrow('Division by zero is not allowed');
});

// 나쁜 예
test('division test', () => {
  expect(true).toBe(false); // 실패 시: 무엇을 테스트하는지 알 수 없음
});
```

## 🎓 다음 단계

1. 요구사항을 테스트로 변환하는 연습
2. Mock 라이브러리 사용법 학습 (`jest.mock`, `@testing-library/jest-dom`)
3. 통합 테스트 작성
4. 테스트 자동화 (CI/CD)

TDD는 연습이 필요한 기술입니다. 작은 기능부터 시작하여 점진적으로 복잡한 기능에 적용해보세요!