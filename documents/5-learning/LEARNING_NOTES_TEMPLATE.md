# TypeScript 패키지 및 기능 학습 노트

이 문서는 프로젝트 진행 중 사용한 TypeScript/Node.js 패키지와 기능들에 대한 학습 내용을 기록합니다.

## 📚 Node.js 표준 모듈

### `http` / `https`
HTTP 클라이언트와 서버 구현을 위한 표준 모듈

#### 사용한 기능
- **`createServer()`**: HTTP 서버 생성
- **`request()`**: HTTP 요청 생성
- **`IncomingMessage`**: HTTP 요청 객체
- **`ServerResponse`**: HTTP 응답 객체

#### 예제
```typescript
import { createServer, IncomingMessage, ServerResponse } from 'http';

const server = createServer((req: IncomingMessage, res: ServerResponse) => {
  res.setHeader('Content-Type', 'application/json');
  res.statusCode = 200;
  res.end(JSON.stringify({ message: 'Hello, World!' }));
});

server.listen(8080, () => {
  console.log('Server running on port 8080');
});

// Express.js 사용 예
import express from 'express';

const app = express();

app.get('/', (req, res) => {
  res.json({ message: 'Hello, World!' });
});

app.listen(8080, () => {
  console.log('Server running on port 8080');
});
```

#### 학습 포인트
- Express.js가 더 편리한 API 제공
- 미들웨어 패턴: `(req, res, next) => void` 형태
- 비동기 처리 시 Promise/async-await 사용

#### 참고 자료
- [Node.js HTTP 공식 문서](https://nodejs.org/api/http.html)
- [Express.js 공식 문서](https://expressjs.com/)

---

### `AbortController` / `AbortSignal`
비동기 작업 취소를 위한 웹 표준 API

#### 사용한 기능
- **`AbortController`**: 취소 신호 생성
- **`AbortSignal`**: 취소 신호 전달
- **`setTimeout`**: 타임아웃 설정
- **`fetch`**: 취소 가능한 HTTP 요청

#### 예제
```typescript
// 타임아웃 설정
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch('/api/users', {
    signal: controller.signal
  });
  
  clearTimeout(timeoutId);
  const data = await response.json();
  return data;
} catch (error) {
  if (error.name === 'AbortError') {
    console.log('Request timeout');
  }
  throw error;
}

// 사용자 정의 비동기 함수에서 취소 처리
async function processData(signal: AbortSignal): Promise<void> {
  for (let i = 0; i < 1000; i++) {
    if (signal.aborted) {
      throw new Error('Operation aborted');
    }
    await processItem(i);
  }
}
```

#### 학습 포인트
- 모든 비동기 작업에 AbortSignal 전달 권장
- 타임아웃 설정 시 `clearTimeout` 호출 필요
- AbortSignal은 요청 범위 데이터 전달용으로만 사용

#### 참고 자료
- [MDN AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)

---

### `pg` / `mysql2`
SQL 데이터베이스 연동을 위한 패키지

#### 사용한 기능
- **`Pool`**: 연결 풀 관리
- **`query()`**: SQL 쿼리 실행
- **`Client`**: 단일 연결 관리
- **`transaction()`**: 트랜잭션 처리

#### 예제
```typescript
import { Pool, PoolClient } from 'pg';

// 연결 풀 설정
const pool = new Pool({
  user: 'username',
  host: 'localhost',
  database: 'mydb',
  password: 'password',
  port: 5432,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// 쿼리 실행
interface User {
  id: string;
  email: string;
  name: string;
}

async function getUserById(id: string): Promise<User | null> {
  const query = 'SELECT id, email, name FROM users WHERE id = $1';
  
  try {
    const result = await pool.query<User>(query, [id]);
    return result.rows[0] || null;
  } catch (error) {
    console.error('Database query failed:', error);
    throw error;
  }
}

// 트랜잭션
async function createUserWithProfile(user: User): Promise<void> {
  const client: PoolClient = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // 사용자 생성
    await client.query(
      'INSERT INTO users (id, email) VALUES ($1, $2)',
      [user.id, user.email]
    );
    
    // 프로필 생성
    await client.query(
      'INSERT INTO profiles (user_id, name) VALUES ($1, $2)',
      [user.id, user.name]
    );
    
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

#### 학습 포인트
- 연결 풀 설정이 성능에 큰 영향
- 타입 안전성을 위해 제네릭 사용
- SQL injection 방지를 위해 파라미터 바인딩 필수
- 트랜잭션 사용 시 try-catch-finally 패턴

---

### `JSON`
JSON 처리를 위한 내장 객체

#### 사용한 기능
- **`JSON.stringify()`**: 객체를 JSON 문자열로 변환
- **`JSON.parse()`**: JSON 문자열을 객체로 변환
- **`replacer`**: 직렬화 커스터마이징
- **`reviver`**: 역직렬화 커스터마이징

#### 예제
```typescript
interface User {
  id: string;
  email: string;
  name?: string;
  age: number;
  createdAt: Date;
}

// 직렬화
const user: User = {
  id: '1',
  email: 'user@example.com',
  age: 30,
  createdAt: new Date()
};

// 기본 직렬화
const jsonString = JSON.stringify(user);

// 커스텀 직렬화 (Date 처리)
const jsonWithDates = JSON.stringify(user, (key, value) => {
  if (value instanceof Date) {
    return value.toISOString();
  }
  return value;
});

// 역직렬화
const parsed = JSON.parse(jsonString) as User;

// 커스텀 역직렬화 (Date 복원)
const parsedWithDates = JSON.parse(jsonWithDates, (key, value) => {
  if (key === 'createdAt' && typeof value === 'string') {
    return new Date(value);
  }
  return value;
}) as User;

// Express.js에서 사용
app.post('/users', (req, res) => {
  const user: User = req.body;
  
  // 유효성 검사
  if (!user.email || !user.id) {
    return res.status(400).json({ error: 'Invalid user data' });
  }
  
  // 저장 로직...
  
  res.json(user);
});
```

#### 학습 포인트
- TypeScript에서 타입 안전성을 위해 타입 어설션 사용
- Date 객체는 자동 직렬화되지 않음 (replacer/reviver 필요)
- JSON 파싱 시 예외 처리 필수
- Express.js는 자동으로 JSON 파싱 처리

---

### `Date`
시간 처리를 위한 내장 객체

#### 사용한 기능
- **`new Date()`**: 현재 시간 또는 특정 시간 생성
- **`toISOString()`**: ISO 8601 형식으로 변환
- **`getTime()`**: 타임스탬프 반환
- **`setInterval`/`setTimeout`**: 타이머 설정

#### 예제
```typescript
// 현재 시간
const now = new Date();
console.log(now.toISOString()); // 2024-01-12T10:30:00.000Z

// 시간 간격 계산
const fiveMinutesLater = new Date(now.getTime() + 5 * 60 * 1000);

// 시간 비교
if (new Date() > fiveMinutesLater) {
  console.log('Deadline exceeded');
}

// 타이머 (Promise 기반)
function delay(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// 사용 예
async function processWithDelay() {
  console.log('Starting...');
  await delay(5000);
  console.log('5 seconds later');
}

// 주기적 실행
const intervalId = setInterval(() => {
  console.log('Tick:', new Date().toISOString());
}, 1000);

// 정리
setTimeout(() => {
  clearInterval(intervalId);
}, 10000);

// 날짜 포맷팅 (date-fns 사용)
import { format, parseISO } from 'date-fns';

const formatted = format(new Date(), 'yyyy-MM-dd HH:mm:ss');
const parsed = parseISO('2024-01-12T10:30:00.000Z');
```

#### 학습 포인트
- JavaScript Date는 내장 포맷팅 옵션이 제한적
- 복잡한 날짜 처리는 date-fns 또는 dayjs 사용 권장
- 타이머 사용 후 반드시 정리 (메모리 누수 방지)

---

## 🔧 외부 패키지

### `pg`
PostgreSQL 드라이버

#### 설치
```bash
npm install pg
npm install --save-dev @types/pg
```

#### 사용법
```typescript
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: 'postgresql://user:password@localhost:5432/dbname',
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false
});

// 연결 테스트
pool.on('connect', () => {
  console.log('Connected to PostgreSQL');
});

pool.on('error', (err) => {
  console.error('PostgreSQL connection error:', err);
});
```

#### 학습 포인트
- 타입 정의 패키지 별도 설치 필요
- 환경변수로 연결 문자열 관리
- SSL 설정은 프로덕션 환경에서 필수

---

### `express`
HTTP 웹 프레임워크

#### 설치
```bash
npm install express
npm install --save-dev @types/express
```

#### 사용한 기능
- 라우팅
- 미들웨어
- 에러 핸들링
- 정적 파일 서빙

#### 예제
```typescript
import express, { Request, Response, NextFunction } from 'express';
import cors from 'cors';
import helmet from 'helmet';

const app = express();

// 미들웨어 설정
app.use(helmet());
app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// 로깅 미들웨어
app.use((req: Request, res: Response, next: NextFunction) => {
  console.log(`${req.method} ${req.path}`);
  next();
});

// 라우팅
app.get('/users/:id', async (req: Request, res: Response) => {
  try {
    const userId = req.params.id;
    const user = await getUserById(userId);
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    console.error('Error fetching user:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// 에러 처리 미들웨어
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong!' });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

#### 학습 포인트
- 미들웨어 순서가 중요
- 에러 핸들링 미들웨어는 4개 매개변수 필요
- 라우트 핸들러에서 async/await 사용 시 try-catch 필수

#### 대안
- `fastify`: 더 빠른 성능
- `koa`: 더 모던한 async/await 중심
- `nestjs`: 엔터프라이즈급 프레임워크

---

### `jest`
테스트 프레임워크

#### 설치
```bash
npm install --save-dev jest @types/jest ts-jest
```

#### 사용한 기능
- **테스트 케이스**: `test()`, `describe()`
- **어설션**: `expect()`, `toBe()`, `toEqual()`
- **Mock**: `jest.fn()`, `jest.spyOn()`
- **비동기 테스트**: `async/await`, `resolves/rejects`

#### 예제
```typescript
// user.service.test.ts
import { UserService } from './user.service';
import { UserRepository } from './user.repository';

// Mock 설정
jest.mock('./user.repository');
const mockUserRepository = UserRepository as jest.MockedClass<typeof UserRepository>;

describe('UserService', () => {
  let service: UserService;
  let mockRepo: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockRepo = new mockUserRepository() as jest.Mocked<UserRepository>;
    service = new UserService(mockRepo);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  test('should create user successfully', async () => {
    // Arrange
    const userData = { email: 'test@example.com', name: 'Test User' };
    const expectedUser = { id: '1', ...userData };
    
    mockRepo.create.mockResolvedValue(expectedUser);
    
    // Act
    const result = await service.createUser(userData);
    
    // Assert
    expect(result).toEqual(expectedUser);
    expect(mockRepo.create).toHaveBeenCalledWith(userData);
    expect(mockRepo.create).toHaveBeenCalledTimes(1);
  });
  
  test('should throw error when email is invalid', async () => {
    // Arrange
    const invalidData = { email: 'invalid-email', name: 'Test' };
    
    // Act & Assert
    await expect(service.createUser(invalidData))
      .rejects.toThrow('Invalid email format');
  });
  
  test('should handle repository error', async () => {
    // Arrange
    const userData = { email: 'test@example.com', name: 'Test' };
    mockRepo.create.mockRejectedValue(new Error('Database error'));
    
    // Act & Assert
    await expect(service.createUser(userData))
      .rejects.toThrow('Database error');
  });
});
```

#### 학습 포인트
- Mock 설정 시 타입 안전성 유지
- `beforeEach`/`afterEach`로 테스트 격리
- 비동기 테스트는 `resolves`/`rejects` 사용
- `toHaveBeenCalledWith`로 호출 인자 검증

---

### `jsonwebtoken`
JWT 토큰 처리 패키지

#### 설치
```bash
npm install jsonwebtoken
npm install --save-dev @types/jsonwebtoken
```

#### 사용한 기능
- JWT 토큰 생성
- JWT 토큰 검증
- Claims 추출

#### 예제
```typescript
import jwt from 'jsonwebtoken';

interface JwtPayload {
  userId: string;
  role: string;
  iat?: number;
  exp?: number;
}

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';

function generateToken(userId: string, role: string): string {
  const payload: JwtPayload = {
    userId,
    role
  };
  
  return jwt.sign(payload, JWT_SECRET, {
    expiresIn: '24h',
    issuer: 'myapp'
  });
}

function validateToken(token: string): JwtPayload {
  try {
    const decoded = jwt.verify(token, JWT_SECRET) as JwtPayload;
    return decoded;
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      throw new Error('Token expired');
    }
    if (error instanceof jwt.JsonWebTokenError) {
      throw new Error('Invalid token');
    }
    throw error;
  }
}

// Express 미들웨어
function authMiddleware(req: Request, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'No token provided' });
  }
  
  const token = authHeader.substring(7);
  
  try {
    const payload = validateToken(token);
    req.user = payload; // 타입 확장 필요
    next();
  } catch (error) {
    res.status(401).json({ error: error.message });
  }
}
```

#### 학습 포인트
- 환경변수로 시크릿 키 관리
- 토큰 만료 시간 설정 필수
- 에러 타입별 처리 필요

---

## 🎯 TypeScript 관용구 및 패턴

### 에러 처리 패턴

#### 커스텀 에러 클래스
```typescript
class AppError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public isOperational: boolean = true
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 400);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404);
  }
}

// 사용 예
async function processUser(id: string): Promise<User> {
  try {
    const user = await getUserFromDB(id);
    if (!user) {
      throw new NotFoundError('User');
    }
    
    if (!validateUser(user)) {
      throw new ValidationError('Invalid user data');
    }
    
    return user;
  } catch (error) {
    if (error instanceof AppError) {
      throw error; // 재던지기
    }
    throw new AppError('Internal server error', 500);
  }
}
```

#### Result 패턴
```typescript
type Result<T, E = Error> = {
  success: true;
  data: T;
} | {
  success: false;
  error: E;
};

function createResult<T>(data: T): Result<T, never> {
  return { success: true, data };
}

function createError<E>(error: E): Result<never, E> {
  return { success: false, error };
}

async function safeGetUser(id: string): Promise<Result<User, string>> {
  try {
    const user = await getUserById(id);
    if (!user) {
      return createError('User not found');
    }
    return createResult(user);
  } catch (error) {
    return createError(error.message);
  }
}

// 사용 예
const result = await safeGetUser('123');
if (result.success) {
  console.log(result.data.name); // 타입 안전
} else {
  console.error(result.error);
}
```

#### 사용한 패턴
- 커스텀 에러 클래스로 에러 분류
- Result 타입으로 타입 안전한 에러 처리
- 에러 체이닝과 컨텍스트 보존

### 인터페이스 패턴

#### 작은 인터페이스
```typescript
// ✅ 좋은 예 - 작은 인터페이스
interface Reader<T> {
  read(id: string): Promise<T | null>;
}

interface Writer<T> {
  write(entity: T): Promise<void>;
}

// 구성으로 큰 인터페이스 만들기
interface ReadWriter<T> extends Reader<T>, Writer<T> {}
```

#### 사용하는 곳에서 인터페이스 정의
```typescript
// ❌ 나쁜 예 - repository 패키지에서 정의
// repository/user.repository.ts
export interface UserRepository { ... }

// ✅ 좋은 예 - service 패키지에서 정의
// service/user.service.ts
interface UserRepository { ... }  // service가 필요한 것만 정의
```

### 비동기 패턴

#### Worker Pool 패턴
```typescript
class WorkerPool<T, R> {
  private workers: Worker[] = [];
  private jobQueue: T[] = [];
  private resultQueue: R[] = [];
  private processing = false;

  constructor(private workerCount: number, private processor: (job: T) => Promise<R>) {}

  async start(): Promise<void> {
    const promises = Array.from({ length: this.workerCount }, (_, i) => 
      this.createWorker(i)
    );
    
    await Promise.all(promises);
  }

  private async createWorker(id: number): Promise<void> {
    while (this.processing) {
      const job = this.jobQueue.shift();
      if (!job) {
        await new Promise(resolve => setTimeout(resolve, 10));
        continue;
      }
      
      try {
        const result = await this.processor(job);
        this.resultQueue.push(result);
      } catch (error) {
        console.error(`Worker ${id} error:`, error);
      }
    }
  }

  addJob(job: T): void {
    this.jobQueue.push(job);
  }

  getResults(): R[] {
    return [...this.resultQueue];
  }

  stop(): void {
    this.processing = false;
  }
}
```

#### AbortController를 이용한 취소
```typescript
async function doWork(signal: AbortSignal): Promise<void> {
  for (let i = 0; i < 1000; i++) {
    // 취소 신호 확인
    if (signal.aborted) {
      throw new Error('Operation aborted');
    }
    
    // 작업 수행
    await performTask(i);
  }
}

// 사용 예
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);

try {
  await doWork(controller.signal);
  clearTimeout(timeoutId);
} catch (error) {
  if (error.message === 'Operation aborted') {
    console.log('Work was cancelled');
  }
}
```

---

## 📝 코딩 팁

### 변수 명명 규칙
```typescript
// ✅ 좋은 예
const userCount: number = 0;
const maxRetries = 3;
const DEFAULT_TIMEOUT = 30 * 1000; // 30초

// ❌ 나쁜 예  
const uc = 0;           // 축약어 사용
const max_retries = 3;  // 언더스코어 사용
```

### 제로값 활용
```typescript
// ✅ 제로값 활용
const users: User[] = []; // 빈 배열로 초기화
const userMap = new Map<string, User>(); // 빈 Map

// ✅ 선택적 초기화
let users: User[] | null = null;
if (users === null) {
    users = [];
}
```

### 인터페이스 구현 확인
```typescript
// 컴파일 타임에 인터페이스 구현 확인
const _: Repository<User> = new UserRepository();
const _handler: RequestHandler = myHandler;
```

### 유틸리티 타입
```typescript
// 유틸리티 타입 정의
type Optional<T> = T | null | undefined;
type Nullable<T> = T | null;
type Omit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;
type Partial<T> = { [P in keyof T]?: T[P] };

// 사용 예
interface User {
  id: string;
  email: string;
  name: string;
  age: number;
}

type CreateUserRequest = Omit<User, 'id'>;
type UpdateUserRequest = Partial<Omit<User, 'id'>>;
```

---

## 🔗 유용한 링크

### 공식 문서
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)
- [Node.js Documentation](https://nodejs.org/docs/)
- [npm Documentation](https://docs.npmjs.com/)

### 추천 도서
- "Effective TypeScript" by Dan Vanderkam
- "Programming TypeScript" by Boris Cherny
- "Node.js Design Patterns" by Mario Casciaro

### 유용한 도구
- [ESLint](https://eslint.org/): 자바스크립트 린터
- [Prettier](https://prettier.io/): 코드 포매터
- [tsx](https://github.com/esbuild-kit/tsx): TypeScript 실행 도구
- [nodemon](https://nodemon.io/): 개발 서버 자동 재시작
- [ts-node](https://typestrong.org/ts-node/): TypeScript 실행 도구

### 유용한 라이브러리
- [Lodash](https://lodash.com/): 유틸리티 함수
- [Zod](https://zod.dev/): 스키마 유효성 검사
- [date-fns](https://date-fns.org/): 날짜 유틸리티
- [Axios](https://axios-http.com/): HTTP 클라이언트

---

**마지막 업데이트**: 2024-01-12  
**다음 업데이트 예정**: 새로운 패키지 사용 시

> 💡 **참고**: 이 문서는 프로젝트 진행 중 지속적으로 업데이트됩니다. 새로운 패키지나 기능을 사용할 때마다 해당 내용을 추가해주세요.