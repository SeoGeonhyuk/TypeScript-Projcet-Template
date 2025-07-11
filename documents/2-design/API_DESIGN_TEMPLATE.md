# API 설계 가이드

TypeScript/Node.js 프로젝트에서 RESTful API를 설계하고 구현하기 위한 가이드입니다.

## 📋 API 설계 원칙

### RESTful 설계 원칙
- **자원(Resource) 중심**: URL은 자원을 나타내며, 동사가 아닌 명사 사용
- **HTTP 메서드 활용**: GET, POST, PUT, DELETE 등 적절한 HTTP 메서드 사용
- **상태 비저장(Stateless)**: 각 요청은 독립적이어야 함
- **일관된 인터페이스**: 일관된 URL 구조와 응답 형식 사용

### API 버전 관리
- URL 경로에 버전 포함: `/api/v1/users`
- 하위 호환성 유지
- 버전 업그레이드 정책 문서화

## 🛠 API 구조 설계

### 1. URL 구조

```
GET    /api/v1/users           # 사용자 목록 조회
GET    /api/v1/users/:id       # 특정 사용자 조회
POST   /api/v1/users           # 사용자 생성
PUT    /api/v1/users/:id       # 사용자 전체 수정
PATCH  /api/v1/users/:id       # 사용자 부분 수정
DELETE /api/v1/users/:id       # 사용자 삭제

# 중첩 리소스
GET    /api/v1/users/:id/posts # 특정 사용자의 게시글 목록
POST   /api/v1/users/:id/posts # 특정 사용자의 게시글 생성
```

### 2. HTTP 상태 코드

```typescript
// 성공 응답
200 OK          // 조회, 수정 성공
201 Created     // 생성 성공
204 No Content  // 삭제 성공

// 클라이언트 오류
400 Bad Request      // 잘못된 요청
401 Unauthorized     // 인증 필요
403 Forbidden        // 권한 없음
404 Not Found        // 리소스 없음
409 Conflict         // 중복 리소스
422 Unprocessable Entity // 유효성 검사 실패

// 서버 오류
500 Internal Server Error // 서버 내부 오류
503 Service Unavailable   // 서비스 일시 중단
```

### 3. 요청/응답 형식

#### 표준 응답 형식

```typescript
// 성공 응답
interface ApiResponse<T> {
  success: true;
  data: T;
  meta?: {
    total?: number;
    page?: number;
    limit?: number;
  };
}

// 오류 응답
interface ApiError {
  success: false;
  error: {
    code: string;
    message: string;
    details?: Record<string, any>;
  };
}

// 페이지네이션 응답
interface PaginatedResponse<T> {
  success: true;
  data: T[];
  meta: {
    total: number;
    page: number;
    limit: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}
```

#### 요청 DTO (Data Transfer Object)

```typescript
// 사용자 생성 요청
interface CreateUserRequest {
  email: string;
  name: string;
  age?: number;
}

// 사용자 수정 요청
interface UpdateUserRequest {
  name?: string;
  age?: number;
}

// 사용자 검색 쿼리
interface UserSearchQuery {
  page?: number;
  limit?: number;
  search?: string;
  sort?: 'name' | 'created_at' | 'updated_at';
  order?: 'asc' | 'desc';
}
```

#### 응답 DTO

```typescript
// 사용자 응답
interface UserResponse {
  id: string;
  email: string;
  name: string;
  age: number;
  createdAt: string;
  updatedAt: string;
}

// 사용자 목록 응답
interface UserListResponse {
  users: UserResponse[];
  meta: {
    total: number;
    page: number;
    limit: number;
    totalPages: number;
  };
}
```

## 🔐 인증 및 보안

### 1. JWT 토큰 인증

```typescript
// 로그인 요청
interface LoginRequest {
  email: string;
  password: string;
}

// 로그인 응답
interface LoginResponse {
  accessToken: string;
  refreshToken: string;
  user: UserResponse;
}

// 토큰 갱신 요청
interface RefreshTokenRequest {
  refreshToken: string;
}
```

### 2. API 키 인증

```typescript
// 헤더 설정
interface ApiHeaders {
  'X-API-Key': string;
  'Authorization': string; // Bearer 토큰
  'Content-Type': 'application/json';
}
```

### 3. 권한 검사

```typescript
// 권한 레벨 정의
enum Permission {
  READ = 'read',
  WRITE = 'write',
  DELETE = 'delete',
  ADMIN = 'admin'
}

// 사용자 권한 정보
interface UserPermissions {
  userId: string;
  permissions: Permission[];
  roles: string[];
}
```

## 📊 데이터 검증

### 1. 요청 검증

```typescript
import { z } from 'zod';

// Zod 스키마 정의
const CreateUserSchema = z.object({
  email: z.string().email('유효한 이메일을 입력하세요'),
  name: z.string().min(2, '이름은 2자 이상이어야 합니다'),
  age: z.number().min(0).max(150).optional()
});

// 검증 미들웨어
import { Request, Response, NextFunction } from 'express';

const validateRequest = (schema: z.ZodSchema) => {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      schema.parse(req.body);
      next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          success: false,
          error: {
            code: 'VALIDATION_ERROR',
            message: '요청 데이터가 유효하지 않습니다',
            details: error.errors
          }
        });
      }
      next(error);
    }
  };
};
```

### 2. 응답 검증

```typescript
// 응답 스키마 정의
const UserResponseSchema = z.object({
  id: z.string(),
  email: z.string().email(),
  name: z.string(),
  age: z.number(),
  createdAt: z.string(),
  updatedAt: z.string()
});

// 응답 검증 함수
const validateResponse = <T>(schema: z.ZodSchema<T>, data: unknown): T => {
  return schema.parse(data);
};
```

## 🚀 API 구현 예시

### 1. Express.js 컨트롤러

```typescript
import { Request, Response, NextFunction } from 'express';
import { UserService } from '../services/user.service';
import { CreateUserRequest, UpdateUserRequest, UserSearchQuery } from '../types/user.types';

export class UserController {
  constructor(private userService: UserService) {}

  // 사용자 목록 조회
  async getUsers(req: Request, res: Response, next: NextFunction) {
    try {
      const query: UserSearchQuery = req.query;
      const result = await this.userService.getUsers(query);
      
      res.json({
        success: true,
        data: result.users,
        meta: result.meta
      });
    } catch (error) {
      next(error);
    }
  }

  // 사용자 조회
  async getUser(req: Request, res: Response, next: NextFunction) {
    try {
      const userId = req.params.id;
      const user = await this.userService.getUserById(userId);
      
      if (!user) {
        return res.status(404).json({
          success: false,
          error: {
            code: 'USER_NOT_FOUND',
            message: '사용자를 찾을 수 없습니다'
          }
        });
      }
      
      res.json({
        success: true,
        data: user
      });
    } catch (error) {
      next(error);
    }
  }

  // 사용자 생성
  async createUser(req: Request, res: Response, next: NextFunction) {
    try {
      const userData: CreateUserRequest = req.body;
      const user = await this.userService.createUser(userData);
      
      res.status(201).json({
        success: true,
        data: user
      });
    } catch (error) {
      next(error);
    }
  }

  // 사용자 수정
  async updateUser(req: Request, res: Response, next: NextFunction) {
    try {
      const userId = req.params.id;
      const updateData: UpdateUserRequest = req.body;
      const user = await this.userService.updateUser(userId, updateData);
      
      if (!user) {
        return res.status(404).json({
          success: false,
          error: {
            code: 'USER_NOT_FOUND',
            message: '사용자를 찾을 수 없습니다'
          }
        });
      }
      
      res.json({
        success: true,
        data: user
      });
    } catch (error) {
      next(error);
    }
  }

  // 사용자 삭제
  async deleteUser(req: Request, res: Response, next: NextFunction) {
    try {
      const userId = req.params.id;
      const deleted = await this.userService.deleteUser(userId);
      
      if (!deleted) {
        return res.status(404).json({
          success: false,
          error: {
            code: 'USER_NOT_FOUND',
            message: '사용자를 찾을 수 없습니다'
          }
        });
      }
      
      res.status(204).send();
    } catch (error) {
      next(error);
    }
  }
}
```

### 2. 라우트 정의

```typescript
import { Router } from 'express';
import { UserController } from '../controllers/user.controller';
import { validateRequest } from '../middleware/validation.middleware';
import { authenticate } from '../middleware/auth.middleware';
import { CreateUserSchema, UpdateUserSchema } from '../schemas/user.schema';

const router = Router();
const userController = new UserController();

// 인증 미들웨어 적용
router.use(authenticate);

// 사용자 라우트
router.get('/users', userController.getUsers.bind(userController));
router.get('/users/:id', userController.getUser.bind(userController));
router.post('/users', validateRequest(CreateUserSchema), userController.createUser.bind(userController));
router.put('/users/:id', validateRequest(UpdateUserSchema), userController.updateUser.bind(userController));
router.delete('/users/:id', userController.deleteUser.bind(userController));

export default router;
```

### 3. 서비스 레이어

```typescript
import { UserRepository } from '../repositories/user.repository';
import { CreateUserRequest, UpdateUserRequest, UserSearchQuery } from '../types/user.types';

export class UserService {
  constructor(private userRepository: UserRepository) {}

  async getUsers(query: UserSearchQuery) {
    const { page = 1, limit = 10, search, sort = 'created_at', order = 'desc' } = query;
    
    const offset = (page - 1) * limit;
    const users = await this.userRepository.findMany({
      offset,
      limit,
      search,
      sort,
      order
    });
    
    const total = await this.userRepository.count({ search });
    const totalPages = Math.ceil(total / limit);
    
    return {
      users,
      meta: {
        total,
        page,
        limit,
        totalPages,
        hasNext: page < totalPages,
        hasPrev: page > 1
      }
    };
  }

  async getUserById(id: string) {
    return await this.userRepository.findById(id);
  }

  async createUser(userData: CreateUserRequest) {
    // 이메일 중복 확인
    const existingUser = await this.userRepository.findByEmail(userData.email);
    if (existingUser) {
      throw new Error('이미 존재하는 이메일입니다');
    }
    
    return await this.userRepository.create(userData);
  }

  async updateUser(id: string, updateData: UpdateUserRequest) {
    const user = await this.userRepository.findById(id);
    if (!user) {
      return null;
    }
    
    return await this.userRepository.update(id, updateData);
  }

  async deleteUser(id: string) {
    const user = await this.userRepository.findById(id);
    if (!user) {
      return false;
    }
    
    await this.userRepository.delete(id);
    return true;
  }
}
```

## 📋 에러 처리

### 1. 글로벌 에러 핸들러

```typescript
import { Request, Response, NextFunction } from 'express';

export class AppError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public code: string,
    public details?: any
  ) {
    super(message);
    this.name = 'AppError';
  }
}

export const errorHandler = (
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  console.error(error);

  if (error instanceof AppError) {
    return res.status(error.statusCode).json({
      success: false,
      error: {
        code: error.code,
        message: error.message,
        details: error.details
      }
    });
  }

  // 예상치 못한 에러
  res.status(500).json({
    success: false,
    error: {
      code: 'INTERNAL_SERVER_ERROR',
      message: '서버 내부 오류가 발생했습니다'
    }
  });
};
```

### 2. 커스텀 에러 클래스

```typescript
export class ValidationError extends AppError {
  constructor(message: string, details?: any) {
    super(message, 400, 'VALIDATION_ERROR', details);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource}을(를) 찾을 수 없습니다`, 404, 'NOT_FOUND');
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409, 'CONFLICT');
  }
}

export class UnauthorizedError extends AppError {
  constructor(message: string = '인증이 필요합니다') {
    super(message, 401, 'UNAUTHORIZED');
  }
}
```

## 📄 API 문서화

### 1. OpenAPI/Swagger 설정

```typescript
import swaggerJSDoc from 'swagger-jsdoc';
import swaggerUi from 'swagger-ui-express';

const options = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'User API',
      version: '1.0.0',
      description: '사용자 관리 API'
    },
    servers: [
      {
        url: 'http://localhost:3000/api/v1',
        description: '개발 서버'
      }
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT'
        }
      }
    }
  },
  apis: ['./src/routes/*.ts', './src/schemas/*.ts']
};

const specs = swaggerJSDoc(options);

export { specs, swaggerUi };
```

### 2. API 문서 주석

```typescript
/**
 * @swagger
 * /users:
 *   get:
 *     summary: 사용자 목록 조회
 *     tags: [Users]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           default: 10
 *     responses:
 *       200:
 *         description: 성공
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 success:
 *                   type: boolean
 *                 data:
 *                   type: array
 *                   items:
 *                     $ref: '#/components/schemas/User'
 */
```

## 🧪 API 테스트

### 1. 통합 테스트

```typescript
import request from 'supertest';
import { app } from '../app';

describe('User API', () => {
  let authToken: string;

  beforeAll(async () => {
    // 테스트용 토큰 생성
    authToken = await getTestToken();
  });

  afterAll(async () => {
    // 테스트 데이터 정리
    await cleanupTestData();
  });

  describe('GET /api/v1/users', () => {
    it('should return user list', async () => {
      const response = await request(app)
        .get('/api/v1/users')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      expect(response.body.success).toBe(true);
      expect(Array.isArray(response.body.data)).toBe(true);
    });

    it('should return paginated results', async () => {
      const response = await request(app)
        .get('/api/v1/users?page=1&limit=5')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      expect(response.body.meta.page).toBe(1);
      expect(response.body.meta.limit).toBe(5);
    });
  });

  describe('POST /api/v1/users', () => {
    it('should create a new user', async () => {
      const userData = {
        email: 'test@example.com',
        name: 'Test User',
        age: 30
      };

      const response = await request(app)
        .post('/api/v1/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send(userData)
        .expect(201);

      expect(response.body.success).toBe(true);
      expect(response.body.data.email).toBe(userData.email);
    });

    it('should return validation error for invalid data', async () => {
      const invalidData = {
        email: 'invalid-email',
        name: ''
      };

      const response = await request(app)
        .post('/api/v1/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send(invalidData)
        .expect(400);

      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('VALIDATION_ERROR');
    });
  });
});
```

### 2. Mock 테스트

```typescript
import { UserController } from '../controllers/user.controller';
import { UserService } from '../services/user.service';

jest.mock('../services/user.service');

describe('UserController', () => {
  let controller: UserController;
  let mockUserService: jest.Mocked<UserService>;

  beforeEach(() => {
    mockUserService = new UserService({} as any) as jest.Mocked<UserService>;
    controller = new UserController(mockUserService);
  });

  describe('getUsers', () => {
    it('should return users successfully', async () => {
      const mockUsers = [
        { id: '1', email: 'user1@example.com', name: 'User 1' }
      ];
      const mockMeta = { total: 1, page: 1, limit: 10, totalPages: 1 };
      
      mockUserService.getUsers.mockResolvedValue({
        users: mockUsers,
        meta: mockMeta
      });

      const mockReq = { query: {} } as any;
      const mockRes = {
        json: jest.fn(),
        status: jest.fn().mockReturnThis()
      } as any;
      const mockNext = jest.fn();

      await controller.getUsers(mockReq, mockRes, mockNext);

      expect(mockRes.json).toHaveBeenCalledWith({
        success: true,
        data: mockUsers,
        meta: mockMeta
      });
    });
  });
});
```

## 📊 성능 최적화

### 1. 페이지네이션

```typescript
// 커서 기반 페이지네이션
interface CursorPaginationQuery {
  cursor?: string;
  limit?: number;
}

// 오프셋 기반 페이지네이션
interface OffsetPaginationQuery {
  page?: number;
  limit?: number;
}
```

### 2. 캐싱

```typescript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

// 캐시 미들웨어
const cache = (ttl: number = 300) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    const key = `cache:${req.originalUrl}`;
    
    try {
      const cached = await redis.get(key);
      if (cached) {
        return res.json(JSON.parse(cached));
      }
      
      const originalJson = res.json;
      res.json = function(data) {
        redis.setex(key, ttl, JSON.stringify(data));
        return originalJson.call(this, data);
      };
      
      next();
    } catch (error) {
      next(error);
    }
  };
};
```

### 3. Rate Limiting

```typescript
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15분
  max: 100, // 최대 100회 요청
  message: {
    success: false,
    error: {
      code: 'RATE_LIMIT_EXCEEDED',
      message: '요청 한도를 초과했습니다'
    }
  }
});

app.use('/api/', limiter);
```

## 🔍 모니터링

### 1. 로깅

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' })
  ]
});

// 요청 로깅 미들웨어
const requestLogger = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info({
      method: req.method,
      url: req.originalUrl,
      status: res.statusCode,
      duration: `${duration}ms`,
      userAgent: req.get('User-Agent'),
      ip: req.ip
    });
  });
  
  next();
};
```

### 2. 헬스 체크

```typescript
// 헬스 체크 엔드포인트
app.get('/health', (req: Request, res: Response) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.npm_package_version
  });
});
```

## 💡 Best Practices

### 1. API 설계 원칙
- 명확하고 일관된 네이밍 컨벤션 사용
- 적절한 HTTP 상태 코드 사용
- 버전 관리 전략 수립
- 상세한 에러 메시지 제공

### 2. 보안 고려사항
- 입력 검증 및 출력 인코딩
- 적절한 인증/인가 구현
- Rate limiting 적용
- CORS 설정

### 3. 성능 최적화
- 적절한 캐싱 전략
- 데이터베이스 쿼리 최적화
- 페이지네이션 구현
- 압축 및 최적화

이 가이드를 따라 TypeScript로 견고하고 확장 가능한 API를 설계하고 구현할 수 있습니다!