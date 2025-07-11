# 시스템 아키텍처 설계 가이드

TypeScript 프로젝트의 아키텍처 설계를 위한 의사결정 가이드입니다.

## 📋 아키텍처 결정 템플릿 (ADR - Architecture Decision Record)

### ADR-001: [결정 제목]

**상황 (Context)**
- 현재 상황과 문제점 설명
- 해결해야 할 과제

**결정 (Decision)**
- 선택한 아키텍처/기술/패턴
- 구체적인 구현 방법

**근거 (Rationale)**
- 이 결정을 내린 이유
- 고려했던 대안들
- 장단점 분석

**결과 (Consequences)**
- 이 결정의 영향
- 향후 고려사항

---

## 🏗 아키텍처 패턴 가이드

### 1. 레이어드 아키텍처 (Layered Architecture)

```
┌─────────────────────────────────────┐
│           Presentation Layer        │ ← Controllers, Routes, GraphQL Resolvers
├─────────────────────────────────────┤
│           Application Layer         │ ← Services, Use Cases
├─────────────────────────────────────┤
│             Domain Layer            │ ← Business Logic, Entities, DTOs
├─────────────────────────────────────┤
│          Infrastructure Layer       │ ← Database, External APIs, Repositories
└─────────────────────────────────────┘
```

**언제 사용하나?**
- 중소 규모 웹 애플리케이션
- 명확한 계층 구조가 필요한 경우
- 팀이 전통적인 MVC 패턴에 익숙한 경우

**TypeScript 구현 예시:**
```typescript
// src/main.ts - Presentation Entry Point
import { AppModule } from './app.module';
import { NestFactory } from '@nestjs/core';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();

// src/controllers/user.controller.ts - Presentation Layer
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get(':id')
  async getUser(@Param('id') id: string): Promise<UserDto> {
    return this.userService.getUser(id);
  }
}

// src/services/user.service.ts - Application Layer
@Injectable()
export class UserService {
  constructor(private readonly userRepository: UserRepository) {}

  async getUser(id: string): Promise<UserDto> {
    const user = await this.userRepository.findById(id);
    return UserMapper.toDto(user);
  }
}

// src/domain/user.entity.ts - Domain Layer
export class User {
  constructor(
    public readonly id: string,
    public readonly email: string,
    public readonly name: string
  ) {}
}

// src/repositories/user.repository.ts - Infrastructure Layer
@Injectable()
export class UserRepository {
  constructor(private readonly db: Database) {}

  async findById(id: string): Promise<User | null> {
    // Database implementation
  }
}
```

### 2. 헥사고날 아키텍처 (Hexagonal Architecture)

```
                    ┌─────────────────┐
                    │   외부 시스템      │
                    │  (Web, CLI,     │
                    │   Database)     │
                    └─────────┬───────┘
                              │
                    ┌─────────▼───────┐
                    │      Port       │ ← 인터페이스 정의
                    ├─────────────────┤
                    │   Application   │ ← 비즈니스 로직
                    │      Core       │
                    ├─────────────────┤
                    │    Adapter      │ ← 구현체
                    └─────────────────┘
```

**언제 사용하나?**
- 복잡한 비즈니스 로직
- 다양한 외부 시스템 연동
- 테스트 용이성이 중요한 경우

**TypeScript 구현 예시:**
```typescript
// src/domain/ports/user.port.ts - Port (인터페이스)
export interface UserRepository {
  save(user: User): Promise<void>;
  findByEmail(email: string): Promise<User | null>;
}

export interface UserService {
  createUser(req: CreateUserRequest): Promise<User>;
}

// src/domain/user.ts - Core Domain
export class User {
  constructor(
    private readonly id: UserId,
    private email: Email,
    private readonly name: Name
  ) {}

  updateEmail(newEmail: Email): void {
    // 비즈니스 규칙 검증
    if (!this.validateEmailChange(newEmail)) {
      throw new Error('Invalid email change');
    }
    this.email = newEmail;
  }

  private validateEmailChange(newEmail: Email): boolean {
    // 검증 로직
    return true;
  }
}

// src/adapters/repositories/postgres-user.repository.ts - Adapter
@Injectable()
export class PostgresUserRepository implements UserRepository {
  constructor(private readonly db: Database) {}

  async save(user: User): Promise<void> {
    // PostgreSQL 구현
  }

  async findByEmail(email: string): Promise<User | null> {
    // PostgreSQL 구현
  }
}

// src/adapters/services/user.service.ts - Application Service
@Injectable()
export class UserServiceImpl implements UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly emailSvc: EmailService
  ) {}

  async createUser(req: CreateUserRequest): Promise<User> {
    // 구현 로직
  }
}
```

### 3. 마이크로서비스 아키텍처

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  User Service │    │ Order Service│    │ Payment Svc  │
│              │    │              │    │              │
│ ┌──────────┐ │    │ ┌──────────┐ │    │ ┌──────────┐ │
│ │    DB    │ │    │ │    DB    │ │    │ │    DB    │ │
│ └──────────┘ │    │ └──────────┘ │    │ └──────────┘ │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼───────┐
                    │  API Gateway │
                    └──────────────┘
```

**언제 사용하나?**
- 대규모 시스템
- 독립적인 배포가 필요한 경우
- 팀이 분리되어 있는 경우

**TypeScript 구현 고려사항:**
```typescript
// 서비스 간 통신
export interface UserServiceClient {
  getUser(userID: string): Promise<User>;
}

// gRPC 클라이언트
@Injectable()
export class GrpcUserClient implements UserServiceClient {
  private client: UserServiceClient;

  constructor() {
    this.client = new UserServiceClient('localhost:50051', grpc.credentials.createInsecure());
  }

  async getUser(userID: string): Promise<User> {
    return new Promise((resolve, reject) => {
      this.client.getUser({ id: userID }, (err, response) => {
        if (err) reject(err);
        else resolve(response.user);
      });
    });
  }
}

// HTTP 클라이언트
@Injectable()
export class HttpUserClient implements UserServiceClient {
  constructor(private readonly httpService: HttpService) {}

  async getUser(userID: string): Promise<User> {
    const response = await this.httpService.get(`/users/${userID}`).toPromise();
    return response.data;
  }
}

// 서비스 디스커버리
export interface ServiceDiscovery {
  discover(serviceName: string): Promise<string[]>;
}
```

## 🔧 기술 스택 결정 가이드

### 데이터베이스 선택

| 요구사항 | 추천 | 이유 |
|---------|------|------|
| ACID 트랜잭션 필요 | PostgreSQL | 강력한 ACID 지원, JSON 지원 |
| 높은 성능, 단순 스키마 | MySQL | 성능 최적화, 단순함 |
| 스키마 유연성 필요 | MongoDB | NoSQL, 스키마 유연성 |
| 캐싱 | Redis | 메모리 기반, 다양한 데이터 구조 |
| 시계열 데이터 | InfluxDB/TimescaleDB | 시계열 최적화 |

**TypeScript 데이터베이스 접근 패턴:**
```typescript
// database/connection.ts
import { Pool, PoolClient } from 'pg';

export class Database {
    private primary: Pool;
    private replica: Pool;

    constructor() {
        this.primary = new Pool({
            host: process.env.DB_PRIMARY_HOST,
            port: Number(process.env.DB_PRIMARY_PORT),
            database: process.env.DB_NAME,
            user: process.env.DB_USER,
            password: process.env.DB_PASSWORD,
        });

        this.replica = new Pool({
            host: process.env.DB_REPLICA_HOST,
            port: Number(process.env.DB_REPLICA_PORT),
            database: process.env.DB_NAME,
            user: process.env.DB_USER,
            password: process.env.DB_PASSWORD,
        });
    }

    async query(text: string, params?: any[]): Promise<any> {
        return this.replica.query(text, params);
    }

    async execute(text: string, params?: any[]): Promise<any> {
        return this.primary.query(text, params);
    }
}

// repository 패턴
export interface Repository<T> {
    create(entity: Omit<T, 'id'>): Promise<T>;
    getById(id: string): Promise<T | null>;
    update(id: string, entity: Partial<T>): Promise<T>;
    delete(id: string): Promise<void>;
}
```

### 웹 프레임워크 선택

| 프레임워크 | 적합한 경우 | 특징 |
|-----------|------------|------|
| Express.js | 대부분의 웹 애플리케이션 | 성숙한 생태계, 많은 미들웨어 |
| Fastify | 고성능 API | 매우 빠름, 스키마 기반 검증 |
| NestJS | 엔터프라이즈 애플리케이션 | Angular 스타일, 데코레이터, DI |
| Koa.js | 모던 비동기 애플리케이션 | async/await 중심, 경량 |
| Hapi.js | 설정 중심 API | 매우 유연, 내장 기능 많음 |

**Express.js 사용 예시:**
```typescript
// src/server/server.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import { userRoutes } from './routes/user.routes';

export class Server {
  private app: express.Application;
  private server?: http.Server;

  constructor() {
    this.app = express();
    this.setupMiddleware();
    this.setupRoutes();
  }

  private setupMiddleware(): void {
    this.app.use(helmet());
    this.app.use(cors());
    this.app.use(express.json());
    this.app.use(express.urlencoded({ extended: true }));
  }

  private setupRoutes(): void {
    this.app.use('/api/users', userRoutes);
    this.app.get('/health', (req, res) => {
      res.status(200).json({ status: 'OK' });
    });
  }

  start(port: number): void {
    this.server = this.app.listen(port, () => {
      console.log(`Server running on port ${port}`);
    });
  }
}
```

## 📊 성능 고려사항

### 1. 동시성 패턴

```typescript
// Worker Pool 패턴
import { Worker } from 'worker_threads';

export class WorkerPool {
    private workers: Worker[] = [];
    private taskQueue: Array<{ task: any; resolve: Function; reject: Function }> = [];
    private availableWorkers: Worker[] = [];

    constructor(private workerCount: number) {}

    async start(): Promise<void> {
        for (let i = 0; i < this.workerCount; i++) {
            const worker = new Worker('./worker.js');
            this.workers.push(worker);
            this.availableWorkers.push(worker);
            this.setupWorker(worker);
        }
    }

    private setupWorker(worker: Worker): void {
        worker.on('message', (result) => {
            // 작업 완료 처리
            this.handleWorkerResult(worker, result);
        });
        
        worker.on('error', (error) => {
            console.error('Worker error:', error);
        });
    }

    private handleWorkerResult(worker: Worker, result: any): void {
        // 워커를 사용 가능 목록에 다시 추가
        this.availableWorkers.push(worker);
        
        // 대기 중인 작업이 있다면 처리
        this.processNextTask();
    }

    private processNextTask(): void {
        if (this.taskQueue.length > 0 && this.availableWorkers.length > 0) {
            const task = this.taskQueue.shift()!;
            const worker = this.availableWorkers.shift()!;
            
            worker.postMessage(task.task);
        }
    }

    async addTask(task: any): Promise<any> {
        return new Promise((resolve, reject) => {
            this.taskQueue.push({ task, resolve, reject });
            this.processNextTask();
        });
    }
}

// Promise 기반 동시성 패턴
class ConcurrencyManager {
  static async parallelMap<T, R>(
    items: T[],
    mapper: (item: T) => Promise<R>,
    concurrency: number = 5
  ): Promise<R[]> {
    const results: R[] = [];
    const semaphore = new Semaphore(concurrency);

    const promises = items.map(async (item, index) => {
      await semaphore.acquire();
      try {
        const result = await mapper(item);
        results[index] = result;
      } finally {
        semaphore.release();
      }
    });

    await Promise.all(promises);
    return results;
  }

  static async pipelineProcess<T>(
    items: T[],
    stages: Array<(item: T) => Promise<T>>
  ): Promise<T[]> {
    let results = [...items];
    
    for (const stage of stages) {
      results = await Promise.all(results.map(stage));
    }
    
    return results;
  }
}

class Semaphore {
  private permits: number;
  private waitQueue: Array<() => void> = [];

  constructor(permits: number) {
    this.permits = permits;
  }

  async acquire(): Promise<void> {
    if (this.permits > 0) {
      this.permits--;
      return;
    }

    return new Promise((resolve) => {
      this.waitQueue.push(resolve);
    });
  }

  release(): void {
    this.permits++;
    const resolve = this.waitQueue.shift();
    if (resolve) {
      this.permits--;
      resolve();
    }
  }
}
```

### 2. 캐싱 전략

```typescript
// 메모리 캐시
interface CacheItem<V> {
  value: V;
  expiry: number;
}

class MemoryCache<K extends string | number, V> {
  private items = new Map<K, CacheItem<V>>();
  private ttl: number;

  constructor(ttlMs: number) {
    this.ttl = ttlMs;
  }

  get(key: K): V | null {
    const item = this.items.get(key);
    if (!item || Date.now() > item.expiry) {
      this.items.delete(key);
      return null;
    }
    return item.value;
  }

  set(key: K, value: V): void {
    this.items.set(key, {
      value,
      expiry: Date.now() + this.ttl
    });
  }

  delete(key: K): boolean {
    return this.items.delete(key);
  }

  clear(): void {
    this.items.clear();
  }

  size(): number {
    return this.items.size;
  }
}

// Redis 캐시 래퍼
import Redis from 'ioredis';

class RedisCache {
  private client: Redis;

  constructor(options?: Redis.RedisOptions) {
    this.client = new Redis(options);
  }

  async get<T>(key: string): Promise<T | null> {
    const value = await this.client.get(key);
    if (!value) return null;
    
    try {
      return JSON.parse(value);
    } catch (error) {
      return value as T;
    }
  }

  async set(key: string, value: any, ttlSeconds?: number): Promise<void> {
    const serialized = typeof value === 'string' ? value : JSON.stringify(value);
    
    if (ttlSeconds) {
      await this.client.setex(key, ttlSeconds, serialized);
    } else {
      await this.client.set(key, serialized);
    }
  }

  async delete(key: string): Promise<boolean> {
    const result = await this.client.del(key);
    return result > 0;
  }

  async exists(key: string): Promise<boolean> {
    const result = await this.client.exists(key);
    return result > 0;
  }
}
```

### 3. 데이터베이스 최적화

```typescript
// 연결 풀 설정
import { Pool, PoolConfig } from 'pg';

interface DatabaseConfig {
    host: string;
    port: number;
    database: string;
    user: string;
    password: string;
    maxOpenConns: number;
    maxIdleConns: number;
    connMaxLifetime: number;
}

function setupDatabase(cfg: DatabaseConfig): Pool {
    const poolConfig: PoolConfig = {
        host: cfg.host,
        port: cfg.port,
        database: cfg.database,
        user: cfg.user,
        password: cfg.password,
        // 최대 열린 연결 수
        max: cfg.maxOpenConns,
        // 최대 유휴 연결 수
        min: cfg.maxIdleConns,
        // 연결 최대 수명
        idleTimeoutMillis: cfg.connMaxLifetime,
    };
    
    return new Pool(poolConfig);
}

// 배치 처리
export class UserRepository {
    constructor(private pool: Pool) {}

    async createBatch(users: User[]): Promise<void> {
        const client = await this.pool.connect();
        
        try {
            await client.query('BEGIN');
            
            const query = `
                INSERT INTO users (id, email, name) VALUES ($1, $2, $3)
            `;
            
            for (const user of users) {
                await client.query(query, [user.id, user.email, user.name]);
            }
            
            await client.query('COMMIT');
        } catch (error) {
            await client.query('ROLLBACK');
            throw error;
        } finally {
            client.release();
        }
    }
}
```

## 🔐 보안 아키텍처

### 1. 인증/인가

```typescript
// JWT 토큰 관리
import jwt from 'jsonwebtoken';

export class TokenManager {
    private secretKey: string;
    private accessTTL: number;
    private refreshTTL: number;

    constructor(secretKey: string, accessTTL: number = 3600, refreshTTL: number = 86400) {
        this.secretKey = secretKey;
        this.accessTTL = accessTTL;
        this.refreshTTL = refreshTTL;
    }

    generateTokens(userId: string): { accessToken: string; refreshToken: string } {
        const accessToken = this.generateAccessToken(userId);
        const refreshToken = this.generateRefreshToken(userId);
        
        return {
            accessToken,
            refreshToken
        };
    }

    private generateAccessToken(userId: string): string {
        return jwt.sign(
            { userId, type: 'access' },
            this.secretKey,
            { expiresIn: this.accessTTL }
        );
    }

    private generateRefreshToken(userId: string): string {
        return jwt.sign(
            { userId, type: 'refresh' },
            this.secretKey,
            { expiresIn: this.refreshTTL }
        );
    }

    verifyToken(token: string): any {
        try {
            return jwt.verify(token, this.secretKey);
        } catch (error) {
            throw new Error('Invalid token');
        }
    }
}

// 미들웨어 체인
// Express 미들웨어
export function authMiddleware(tokenManager: TokenManager) {
    return (req: any, res: any, next: any) => {
        const token = extractToken(req);
        
        if (!token) {
            return res.status(401).json({ error: 'Unauthorized' });
        }
        
        try {
            const claims = tokenManager.verifyToken(token);
            req.user = claims;
            next();
        } catch (error) {
            return res.status(401).json({ error: 'Invalid token' });
        }
    };
}

function extractToken(req: any): string | null {
    const authHeader = req.headers.authorization;
    if (authHeader && authHeader.startsWith('Bearer ')) {
        return authHeader.slice(7);
    }
    return null;
}
```

### 2. Rate Limiting

```typescript
// Token Bucket 구현
interface Bucket {
    tokens: number;
    lastRefill: number;
}

export class RateLimiter {
    private buckets: Map<string, Bucket> = new Map();
    private capacity: number;
    private refillRate: number;
    private refillInterval: number;

    constructor(capacity: number, refillRate: number, refillInterval: number = 1000) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.refillInterval = refillInterval;
    }

    allow(key: string): boolean {
        const bucket = this.getBucket(key);
        const now = Date.now();
        
        // 토큰 보충
        const timePassed = now - bucket.lastRefill;
        const tokensToAdd = Math.floor(timePassed / this.refillInterval) * this.refillRate;
        
        bucket.tokens = Math.min(this.capacity, bucket.tokens + tokensToAdd);
        bucket.lastRefill = now;
        
        if (bucket.tokens > 0) {
            bucket.tokens--;
            return true;
        }
        
        return false;
    }

    private getBucket(key: string): Bucket {
        let bucket = this.buckets.get(key);
        if (!bucket) {
            bucket = {
                tokens: this.capacity - 1,
                lastRefill: Date.now()
            };
            this.buckets.set(key, bucket);
        }
        return bucket;
    }
}
```

## 📝 아키텍처 문서화

### 1. C4 모델 다이어그램

```
Level 1: System Context
┌─────────────────────────────────────────────────┐
│                   User                          │
│                     │                           │
│                     ▼                           │
│  ┌─────────────────────────────────────────┐    │
│  │           Our System                    │    │
│  │                                         │    │
│  │  ┌─────────┐    ┌─────────┐           │    │
│  │  │   API   │    │Database │           │    │
│  │  └─────────┘    └─────────┘           │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘

Level 2: Container Diagram
┌─────────────────────────────────────────────────┐
│                Web Browser                      │
│                     │                           │
│                     ▼                           │
│  ┌─────────────────────────────────────────┐    │
│  │           Load Balancer                 │    │
│  │                  │                      │    │
│  │                  ▼                      │    │
│  │  ┌─────────┐    ┌─────────┐           │    │
│  │  │  API    │───▶│Database │           │    │
│  │  │ Server  │    │         │           │    │
│  │  └─────────┘    └─────────┘           │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

### 2. 의사결정 로그

```yaml
# docs/decisions/001-database-choice.md
# ADR-001: PostgreSQL을 주 데이터베이스로 선택

## Status
Accepted

## Context
사용자 데이터와 트랜잭션 데이터를 저장할 데이터베이스가 필요합니다.
- ACID 트랜잭션 지원 필요
- JSON 데이터 타입 지원 필요
- 높은 동시성 처리 필요

## Decision
PostgreSQL을 주 데이터베이스로 사용합니다.

## Consequences
### Positive
- 강력한 ACID 트랜잭션 지원
- 우수한 JSON 지원 (JSONB)
- 활발한 커뮤니티와 생태계

### Negative
- MySQL 대비 설정이 복잡
- 일부 클라우드 서비스에서 비용이 높을 수 있음
```

## 🎯 아키텍처 검증 체크리스트

### 확장성
- [ ] 수평 확장이 가능한가?
- [ ] 상태가 없는(stateless) 설계인가?
- [ ] 병목점이 명확히 식별되는가?

### 가용성
- [ ] 단일 장애점(SPOF)이 없는가?
- [ ] 장애 격리가 가능한가?
- [ ] 복구 절차가 명확한가?

### 보안
- [ ] 최소 권한 원칙을 따르는가?
- [ ] 인증/인가가 적절히 구현되었는가?
- [ ] 민감한 데이터가 암호화되는가?

### 유지보수성
- [ ] 코드가 모듈화되어 있는가?
- [ ] 의존성 방향이 올바른가?
- [ ] 테스트하기 쉬운 구조인가?

이 가이드를 참고하여 프로젝트에 적합한 아키텍처를 설계하고 문서화하세요!