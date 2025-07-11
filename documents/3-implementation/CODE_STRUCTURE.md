# TypeScript 코드 구조화 가이드

TypeScript 프로젝트에서 깨끗하고 유지보수 가능한 코드 구조를 만들기 위한 가이드입니다.

## 📋 기본 원칙

### 1. 단일 책임 원칙 (Single Responsibility Principle)
- 하나의 클래스/함수는 하나의 책임만 가져야 함
- 변경의 이유가 하나여야 함

### 2. 의존성 역전 원칙 (Dependency Inversion Principle)
- 고수준 모듈이 저수준 모듈에 의존해서는 안 됨
- 구상 클래스가 아닌 추상화에 의존해야 함

### 3. 개방-폐쇄 원칙 (Open-Closed Principle)
- 확장에는 열려있고, 변경에는 닫혀있어야 함

## 🏗 프로젝트 구조

### 레이어드 아키텍처 구조

```
src/
├── controllers/           # 프레젠테이션 레이어
│   ├── user.controller.ts
│   └── base.controller.ts
├── services/             # 애플리케이션 레이어
│   ├── user.service.ts
│   └── interfaces/
│       └── user.service.interface.ts
├── repositories/         # 데이터 액세스 레이어
│   ├── user.repository.ts
│   └── interfaces/
│       └── user.repository.interface.ts
├── models/              # 도메인 모델
│   ├── user.model.ts
│   └── base.model.ts
├── dto/                 # 데이터 전송 객체
│   ├── user.dto.ts
│   └── common.dto.ts
├── middlewares/         # 미들웨어
│   ├── auth.middleware.ts
│   └── error.middleware.ts
├── utils/               # 유틸리티 함수
│   ├── logger.ts
│   ├── validator.ts
│   └── crypto.ts
├── config/              # 설정 관리
│   ├── database.ts
│   ├── server.ts
│   └── index.ts
├── types/               # 타입 정의
│   ├── user.types.ts
│   └── common.types.ts
├── constants/           # 상수 정의
│   └── index.ts
├── decorators/          # 데코레이터 (NestJS 사용시)
│   └── roles.decorator.ts
├── guards/              # 가드 (NestJS 사용시)
│   └── auth.guard.ts
├── interceptors/        # 인터셉터 (NestJS 사용시)
│   └── logging.interceptor.ts
├── pipes/               # 파이프 (NestJS 사용시)
│   └── validation.pipe.ts
├── filters/             # 필터 (NestJS 사용시)
│   └── http-exception.filter.ts
└── app.ts               # 애플리케이션 진입점
```

## 📁 디렉토리별 상세 가이드

### 1. Controllers (컨트롤러)

HTTP 요청을 처리하고 응답을 반환하는 역할

```typescript
// src/controllers/user.controller.ts
import { Request, Response, NextFunction } from 'express';
import { UserService } from '../services/user.service';
import { CreateUserDto, UpdateUserDto } from '../dto/user.dto';
import { ApiResponse } from '../types/common.types';

export class UserController {
  constructor(private readonly userService: UserService) {}

  async createUser(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const createUserDto: CreateUserDto = req.body;
      const user = await this.userService.createUser(createUserDto);
      
      const response: ApiResponse<typeof user> = {
        success: true,
        data: user,
        message: 'User created successfully'
      };
      
      res.status(201).json(response);
    } catch (error) {
      next(error);
    }
  }

  async getUser(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { id } = req.params;
      const user = await this.userService.getUserById(id);
      
      const response: ApiResponse<typeof user> = {
        success: true,
        data: user
      };
      
      res.json(response);
    } catch (error) {
      next(error);
    }
  }

  async updateUser(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { id } = req.params;
      const updateUserDto: UpdateUserDto = req.body;
      const user = await this.userService.updateUser(id, updateUserDto);
      
      const response: ApiResponse<typeof user> = {
        success: true,
        data: user,
        message: 'User updated successfully'
      };
      
      res.json(response);
    } catch (error) {
      next(error);
    }
  }

  async deleteUser(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { id } = req.params;
      await this.userService.deleteUser(id);
      
      const response: ApiResponse<null> = {
        success: true,
        data: null,
        message: 'User deleted successfully'
      };
      
      res.json(response);
    } catch (error) {
      next(error);
    }
  }
}
```

### 2. Services (서비스)

비즈니스 로직을 처리하는 역할

```typescript
// src/services/interfaces/user.service.interface.ts
import { CreateUserDto, UpdateUserDto, UserResponseDto } from '../../dto/user.dto';

export interface IUserService {
  createUser(createUserDto: CreateUserDto): Promise<UserResponseDto>;
  getUserById(id: string): Promise<UserResponseDto>;
  updateUser(id: string, updateUserDto: UpdateUserDto): Promise<UserResponseDto>;
  deleteUser(id: string): Promise<void>;
  getUserByEmail(email: string): Promise<UserResponseDto | null>;
}
```

```typescript
// src/services/user.service.ts
import { IUserService } from './interfaces/user.service.interface';
import { IUserRepository } from '../repositories/interfaces/user.repository.interface';
import { CreateUserDto, UpdateUserDto, UserResponseDto } from '../dto/user.dto';
import { User } from '../models/user.model';
import { NotFoundError, ConflictError } from '../utils/errors';
import { Logger } from '../utils/logger';

export class UserService implements IUserService {
  private logger = new Logger('UserService');

  constructor(private readonly userRepository: IUserRepository) {}

  async createUser(createUserDto: CreateUserDto): Promise<UserResponseDto> {
    this.logger.info('Creating user', { email: createUserDto.email });

    // 이메일 중복 확인
    const existingUser = await this.userRepository.findByEmail(createUserDto.email);
    if (existingUser) {
      throw new ConflictError('Email already exists');
    }

    // 유저 생성
    const user = User.create(createUserDto);
    const savedUser = await this.userRepository.save(user);

    this.logger.info('User created successfully', { userId: savedUser.id });
    return this.mapToDto(savedUser);
  }

  async getUserById(id: string): Promise<UserResponseDto> {
    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new NotFoundError('User not found');
    }

    return this.mapToDto(user);
  }

  async updateUser(id: string, updateUserDto: UpdateUserDto): Promise<UserResponseDto> {
    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new NotFoundError('User not found');
    }

    // 이메일 변경 시 중복 확인
    if (updateUserDto.email && updateUserDto.email !== user.email) {
      const existingUser = await this.userRepository.findByEmail(updateUserDto.email);
      if (existingUser) {
        throw new ConflictError('Email already exists');
      }
    }

    user.update(updateUserDto);
    const updatedUser = await this.userRepository.save(user);

    return this.mapToDto(updatedUser);
  }

  async deleteUser(id: string): Promise<void> {
    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new NotFoundError('User not found');
    }

    await this.userRepository.delete(id);
  }

  async getUserByEmail(email: string): Promise<UserResponseDto | null> {
    const user = await this.userRepository.findByEmail(email);
    return user ? this.mapToDto(user) : null;
  }

  private mapToDto(user: User): UserResponseDto {
    return {
      id: user.id,
      email: user.email,
      name: user.name,
      createdAt: user.createdAt.toISOString(),
      updatedAt: user.updatedAt.toISOString()
    };
  }
}
```

### 3. Repositories (리포지토리)

데이터 액세스를 담당하는 역할

```typescript
// src/repositories/interfaces/user.repository.interface.ts
import { User } from '../../models/user.model';

export interface IUserRepository {
  save(user: User): Promise<User>;
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findAll(limit?: number, offset?: number): Promise<User[]>;
  delete(id: string): Promise<void>;
  count(): Promise<number>;
}
```

```typescript
// src/repositories/user.repository.ts
import { Pool, PoolClient } from 'pg';
import { IUserRepository } from './interfaces/user.repository.interface';
import { User } from '../models/user.model';
import { DatabaseError } from '../utils/errors';

export class UserRepository implements IUserRepository {
  constructor(private readonly db: Pool) {}

  async save(user: User): Promise<User> {
    const client: PoolClient = await this.db.connect();
    
    try {
      const query = user.id 
        ? `UPDATE users SET email = $1, name = $2, updated_at = $3 WHERE id = $4 RETURNING *`
        : `INSERT INTO users (id, email, name, created_at, updated_at) VALUES ($1, $2, $3, $4, $5) RETURNING *`;
      
      const values = user.id
        ? [user.email, user.name, user.updatedAt, user.id]
        : [user.id, user.email, user.name, user.createdAt, user.updatedAt];

      const result = await client.query(query, values);
      return User.fromRow(result.rows[0]);
    } catch (error) {
      throw new DatabaseError('Failed to save user', error);
    } finally {
      client.release();
    }
  }

  async findById(id: string): Promise<User | null> {
    const client: PoolClient = await this.db.connect();
    
    try {
      const query = 'SELECT * FROM users WHERE id = $1';
      const result = await client.query(query, [id]);
      
      return result.rows[0] ? User.fromRow(result.rows[0]) : null;
    } catch (error) {
      throw new DatabaseError('Failed to find user by id', error);
    } finally {
      client.release();
    }
  }

  async findByEmail(email: string): Promise<User | null> {
    const client: PoolClient = await this.db.connect();
    
    try {
      const query = 'SELECT * FROM users WHERE email = $1';
      const result = await client.query(query, [email]);
      
      return result.rows[0] ? User.fromRow(result.rows[0]) : null;
    } catch (error) {
      throw new DatabaseError('Failed to find user by email', error);
    } finally {
      client.release();
    }
  }

  async findAll(limit = 10, offset = 0): Promise<User[]> {
    const client: PoolClient = await this.db.connect();
    
    try {
      const query = 'SELECT * FROM users ORDER BY created_at DESC LIMIT $1 OFFSET $2';
      const result = await client.query(query, [limit, offset]);
      
      return result.rows.map(row => User.fromRow(row));
    } catch (error) {
      throw new DatabaseError('Failed to find all users', error);
    } finally {
      client.release();
    }
  }

  async delete(id: string): Promise<void> {
    const client: PoolClient = await this.db.connect();
    
    try {
      const query = 'DELETE FROM users WHERE id = $1';
      await client.query(query, [id]);
    } catch (error) {
      throw new DatabaseError('Failed to delete user', error);
    } finally {
      client.release();
    }
  }

  async count(): Promise<number> {
    const client: PoolClient = await this.db.connect();
    
    try {
      const query = 'SELECT COUNT(*) FROM users';
      const result = await client.query(query);
      
      return parseInt(result.rows[0].count);
    } catch (error) {
      throw new DatabaseError('Failed to count users', error);
    } finally {
      client.release();
    }
  }
}
```

### 4. Models (모델)

도메인 엔티티를 정의하는 역할

```typescript
// src/models/user.model.ts
import { v4 as uuidv4 } from 'uuid';
import { CreateUserDto, UpdateUserDto } from '../dto/user.dto';
import { ValidationError } from '../utils/errors';

export class User {
  public readonly id: string;
  public email: string;
  public name: string;
  public readonly createdAt: Date;
  public updatedAt: Date;

  constructor(
    id: string,
    email: string,
    name: string,
    createdAt: Date,
    updatedAt: Date
  ) {
    this.id = id;
    this.email = email;
    this.name = name;
    this.createdAt = createdAt;
    this.updatedAt = updatedAt;
    
    this.validate();
  }

  static create(createUserDto: CreateUserDto): User {
    const now = new Date();
    return new User(
      uuidv4(),
      createUserDto.email,
      createUserDto.name,
      now,
      now
    );
  }

  static fromRow(row: any): User {
    return new User(
      row.id,
      row.email,
      row.name,
      new Date(row.created_at),
      new Date(row.updated_at)
    );
  }

  update(updateUserDto: UpdateUserDto): void {
    if (updateUserDto.email !== undefined) {
      this.email = updateUserDto.email;
    }
    if (updateUserDto.name !== undefined) {
      this.name = updateUserDto.name;
    }
    this.updatedAt = new Date();
    
    this.validate();
  }

  private validate(): void {
    if (!this.email || !this.isValidEmail(this.email)) {
      throw new ValidationError('Invalid email format');
    }
    
    if (!this.name || this.name.trim().length < 2) {
      throw new ValidationError('Name must be at least 2 characters long');
    }
  }

  private isValidEmail(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }
}
```

### 5. DTOs (Data Transfer Objects)

데이터 전송 객체 정의

```typescript
// src/dto/user.dto.ts
import { IsEmail, IsString, IsOptional, MinLength } from 'class-validator';

export class CreateUserDto {
  @IsEmail({}, { message: 'Invalid email format' })
  email: string;

  @IsString({ message: 'Name must be a string' })
  @MinLength(2, { message: 'Name must be at least 2 characters long' })
  name: string;
}

export class UpdateUserDto {
  @IsOptional()
  @IsEmail({}, { message: 'Invalid email format' })
  email?: string;

  @IsOptional()
  @IsString({ message: 'Name must be a string' })
  @MinLength(2, { message: 'Name must be at least 2 characters long' })
  name?: string;
}

export class UserResponseDto {
  id: string;
  email: string;
  name: string;
  createdAt: string;
  updatedAt: string;
}
```

### 3. `pkg/` - 공개 라이브러리

다른 프로젝트에서도 사용할 수 있는 라이브러리 코드입니다.

```typescript
// src/utils/errors.ts
export interface ErrorResponse {
    error: string;
    code: number;
    message: string;
}

export class AppError extends Error {
    constructor(
        public readonly code: number,
        public readonly message: string,
        public readonly isOperational: boolean = true
    ) {
        super(message);
        this.name = this.constructor.name;
        Error.captureStackTrace(this, this.constructor);
    }
}

export function createErrorResponse(code: number, message: string): ErrorResponse {
    return {
        error: 'Error',
        code,
        message
    };
}

// src/utils/validator.ts
import { z } from 'zod';

export const emailSchema = z.string().email();

export function validateEmail(email: string): boolean {
    try {
        emailSchema.parse(email);
        return true;
    } catch {
        return false;
    }
}

export function validatePassword(password: string): { isValid: boolean; error?: string } {
    if (password.length < 8) {
        return { isValid: false, error: 'Password must be at least 8 characters' };
    }
    
    const hasUpper = /[A-Z]/.test(password);
    const hasLower = /[a-z]/.test(password);
    const hasNumber = /\d/.test(password);
    
    if (!hasUpper || !hasLower || !hasNumber) {
        return { isValid: false, error: 'Password must contain uppercase, lowercase, and number' };
    }
    
    return { isValid: true };
}
```

## 🔄 의존성 방향

### 클린 아키텍처 원칙

```
외부 → 내부 방향으로만 의존성이 흘러야 함

src/app → src/controllers → src/services → src/models
        ↘ src/repositories → src/models
```

### 의존성 주입 패턴

```typescript
// src/app.ts
import { createServer } from 'http';
import { Database } from './database';
import { UserController } from './controllers/user.controller';
import { UserService } from './services/user.service';
import { UserRepository } from './repositories/user.repository';

export class App {
    private server: ReturnType<typeof createServer>;
    private db: Database;

    constructor() {
        this.db = new Database();
        this.setupDependencies();
    }

    private setupDependencies(): void {
        // 의존성 주입
        const userRepo = new UserRepository(this.db);
        const userService = new UserService(userRepo);
        const userController = new UserController(userService);
        
        // 라우터 설정
        this.setupRoutes(userController);
    }

    private setupRoutes(userController: UserController): void {
        // Express 또는 Fastify 등을 사용한 라우터 설정
        this.server.get('/users', userController.getUsers.bind(userController));
        this.server.post('/users', userController.createUser.bind(userController));
    }
    
    public async start(port: number): Promise<void> {
        return new Promise((resolve, reject) => {
            this.server.listen(port, (err?: Error) => {
                if (err) {
                    reject(err);
                } else {
                    resolve();
                }
            });
        });
    }

    public async stop(): Promise<void> {
        return new Promise((resolve) => {
            this.server.close(() => {
                this.db.close();
                resolve();
            });
        });
    }
}
```

## 📝 모듈 네이밍 규칙

### 좋은 모듈명
- 짧고 명확함: `user`, `auth`, `config`
- 명사 사용: `controller`, `service`, `repository`
- 복수형 피함: `users` ❌ → `user` ✅

### 파일 네이밍
- 모듈 기능 중심: `user.ts`, `user.test.ts`
- 타입별 분리: `user.controller.ts`, `user.model.ts`, `user.repository.ts`

## 🧪 테스트 구조

```
src/
├── models/
│   ├── user.ts
│   └── user.test.ts
├── services/
│   ├── user.service.ts
│   ├── user.service.test.ts
│   └── __mocks__/
│       └── users.json
└── repositories/
    ├── user.repository.ts
    ├── user.repository.test.ts
    └── user.repository.integration.test.ts
```

### 테스트 파일 예시

```typescript
// src/services/user.service.test.ts
import { UserService } from './user.service';
import { UserRepository } from '../repositories/user.repository';
import { CreateUserRequest } from '../types/user.types';

// Mock the repository
jest.mock('../repositories/user.repository');

describe('UserService', () => {
    let userService: UserService;
    let mockUserRepository: jest.Mocked<UserRepository>;

    beforeEach(() => {
        mockUserRepository = new UserRepository() as jest.Mocked<UserRepository>;
        userService = new UserService(mockUserRepository);
    });

    describe('createUser', () => {
        const testCases = [
            {
                name: 'success',
                req: {
                    email: 'test@example.com',
                    name: 'Test User',
                } as CreateUserRequest,
                setup: () => {
                    mockUserRepository.getByEmail.mockResolvedValue(null);
                    mockUserRepository.create.mockResolvedValue({ id: 1, email: 'test@example.com', name: 'Test User' });
                },
                expectError: false,
            },
            {
                name: 'email already exists',
                req: {
                    email: 'existing@example.com',
                    name: 'Test User',
                } as CreateUserRequest,
                setup: () => {
                    mockUserRepository.getByEmail.mockResolvedValue({ id: 1, email: 'existing@example.com', name: 'Existing User' });
                },
                expectError: true,
            },
        ];

        testCases.forEach(({ name, req, setup, expectError }) => {
            it(name, async () => {
                setup();

                if (expectError) {
                    await expect(userService.createUser(req)).rejects.toThrow();
                } else {
                    const result = await userService.createUser(req);
                    expect(result).toBeDefined();
                    expect(result.email).toBe(req.email);
                }
            });
        });
    });
});
```

## 🔧 생성 스크립트

### 서비스 생성 자동화

```bash
#!/bin/bash
# scripts/new-service.sh

SERVICE_NAME=$1

if [ -z "$SERVICE_NAME" ]; then
    echo "Usage: $0 <service-name>"
    exit 1
fi

# 디렉토리 생성
mkdir -p src/models
mkdir -p src/services
mkdir -p src/repositories
mkdir -p src/controllers

# 기본 파일 생성
cat > src/models/${SERVICE_NAME}.model.ts << EOF
export interface ${SERVICE_NAME^} {
    id: string;
    name: string;
    createdAt: Date;
    updatedAt: Date;
}

export interface Create${SERVICE_NAME^}Request {
    name: string;
}

export interface Update${SERVICE_NAME^}Request {
    name?: string;
}
EOF

cat > src/repositories/${SERVICE_NAME}.repository.ts << EOF
import { ${SERVICE_NAME^}, Create${SERVICE_NAME^}Request, Update${SERVICE_NAME^}Request } from '../models/${SERVICE_NAME}.model';

export interface ${SERVICE_NAME^}Repository {
    create(data: Create${SERVICE_NAME^}Request): Promise<${SERVICE_NAME^}>;
    getById(id: string): Promise<${SERVICE_NAME^} | null>;
    update(id: string, data: Update${SERVICE_NAME^}Request): Promise<${SERVICE_NAME^}>;
    delete(id: string): Promise<void>;
}
EOF

echo "Generated TypeScript service structure for: $SERVICE_NAME"
```

## 💡 Best Practices

### 1. 네이밍 컨벤션

```typescript
// 좋은 예
class UserService { }
interface IUserRepository { }
type UserCreatePayload = { };
const getUserById = (id: string) => { };
const API_BASE_URL = 'https://api.example.com';

// 나쁜 예
class userservice { }
interface user_repository { }
type user_create_payload = { };
const get_user_by_id = (id: string) => { };
const api_base_url = 'https://api.example.com';
```

### 2. 폴더 구조 일관성

```
src/
├── controllers/
│   ├── __tests__/
│   ├── user.controller.ts
│   └── index.ts
├── services/
│   ├── __tests__/
│   ├── interfaces/
│   ├── user.service.ts
│   └── index.ts
└── repositories/
    ├── __tests__/
    ├── interfaces/
    ├── user.repository.ts
    └── index.ts
```

### 3. 에러 처리

```typescript
// 좋은 예: 구체적인 에러 타입
class UserService {
  async createUser(data: CreateUserDto): Promise<User> {
    const existingUser = await this.userRepo.findByEmail(data.email);
    if (existingUser) {
      throw new ConflictError('Email already exists');
    }
    
    try {
      return await this.userRepo.save(new User(data));
    } catch (error) {
      throw new DatabaseError('Failed to create user', error);
    }
  }
}

// 나쁜 예: 일반적인 에러
class UserService {
  async createUser(data: CreateUserDto): Promise<User> {
    const existingUser = await this.userRepo.findByEmail(data.email);
    if (existingUser) {
      throw new Error('User exists'); // 너무 일반적
    }
    
    return await this.userRepo.save(new User(data));
  }
}
```

### 4. 타입 안전성

```typescript
// 좋은 예: 엄격한 타입 정의
interface CreateUserRequest {
  email: string;
  name: string;
}

interface UserResponse {
  id: string;
  email: string;
  name: string;
  createdAt: string;
  updatedAt: string;
}

function createUser(req: CreateUserRequest): Promise<UserResponse> {
  // 타입 안전한 구현
}

// 나쁜 예: any 사용
function createUser(req: any): Promise<any> {
  // 타입 안전성 없음
}
```

이 가이드를 따라 TypeScript 프로젝트의 코드를 체계적으로 구조화할 수 있습니다!