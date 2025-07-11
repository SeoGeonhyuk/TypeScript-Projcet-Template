# TypeScript 프로젝트 초기 설정 가이드

새로운 TypeScript 프로젝트를 시작할 때 따라야 할 단계별 가이드입니다.

> 📚 **코딩 스타일**: 이 프로젝트는 ESLint + Prettier를 사용하여 일관된 코딩 스타일을 유지합니다.

## 📋 빠른 시작 체크리스트

```bash
# 1. 프로젝트 디렉토리 생성
mkdir myproject && cd myproject

# 2. Node.js 프로젝트 초기화
npm init -y

# 3. TypeScript 설정
npm install -D typescript @types/node
npx tsc --init

# 4. 기본 디렉토리 구조 생성
mkdir -p src tests docs scripts

# 5. 개발 도구 설치
npm install -D eslint prettier husky lint-staged

# 6. 기본 설정 파일 생성
touch .gitignore .eslintrc.js .prettierrc
```

## 🚀 단계별 상세 가이드

### 1. TypeScript 프로젝트 초기화

```bash
# package.json 생성
npm init -y

# TypeScript 설치
npm install -D typescript @types/node

# TypeScript 설정 파일 생성
npx tsc --init

# 추가 타입 정의 설치 (필요에 따라)
npm install -D @types/express @types/jest
```

### 2. 프로젝트 구조 생성

#### 최소 구조 (작은 프로젝트)
```
myproject/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts
│   └── index.test.ts
├── dist/
└── README.md
```

#### 표준 구조 (중간 규모 프로젝트)
```
myproject/
├── src/
│   ├── controllers/        # HTTP 컨트롤러
│   ├── services/          # 비즈니스 로직
│   ├── repositories/      # 데이터 액세스
│   ├── middlewares/       # 미들웨어
│   ├── models/            # 데이터 모델
│   ├── types/             # 타입 정의
│   ├── utils/             # 유틸리티 함수
│   ├── config/            # 설정 관리
│   └── app.ts             # 애플리케이션 진입점
├── tests/                 # 테스트 파일
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── docs/                  # 프로젝트 문서
├── scripts/               # 빌드, 배포 스크립트
├── public/                # 정적 파일 (웹 앱인 경우)
├── dist/                  # 컴파일된 파일
├── node_modules/          # 의존성
├── .gitignore
├── .eslintrc.js           # ESLint 설정
├── .prettierrc            # Prettier 설정
├── jest.config.js         # Jest 설정
├── tsconfig.json          # TypeScript 설정
├── Dockerfile
├── docker-compose.yml
├── package.json
└── README.md
```

### 3. .gitignore 설정

```gitignore
# Dependencies
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Production build
dist/
build/

# Runtime data
pids
*.pid
*.seed
*.pid.lock

# Coverage directory used by tools like istanbul
coverage/
.nyc_output

# Environment variables
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Logs
logs
*.log

# TypeScript
*.tsbuildinfo

# Optional npm cache directory
.npm

# Optional REPL history
.node_repl_history

# Output of 'npm pack'
*.tgz

# Yarn Integrity file
.yarn-integrity

# parcel-bundler cache (https://parceljs.org/)
.cache
.parcel-cache

# next.js build output
.next

# nuxt.js build output
.nuxt

# vuepress build output
.vuepress/dist

# Serverless directories
.serverless

# FuseBox cache
.fusebox/

# DynamoDB Local files
.dynamodb/
```

### 4. package.json 스크립트 설정

```json
{
  "name": "myproject",
  "version": "1.0.0",
  "description": "TypeScript project",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "build:watch": "tsc --watch",
    "start": "node dist/index.js",
    "dev": "tsx watch src/index.ts",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/**/*.ts",
    "lint:fix": "eslint src/**/*.ts --fix",
    "format": "prettier --write src/**/*.ts",
    "format:check": "prettier --check src/**/*.ts",
    "typecheck": "tsc --noEmit",
    "clean": "rm -rf dist",
    "prepare": "husky install",
    "docker:build": "docker build -t myproject:latest .",
    "docker:run": "docker run --rm -p 3000:3000 myproject:latest"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "husky": "^8.0.0",
    "jest": "^29.0.0",
    "lint-staged": "^15.0.0",
    "prettier": "^3.0.0",
    "tsx": "^4.0.0",
    "typescript": "^5.0.0"
  },
  "lint-staged": {
    "src/**/*.ts": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

### 5. 기본 index.ts 작성

```typescript
// src/index.ts
import { createServer } from './server';
import { config } from './config';
import { logger } from './utils/logger';

async function main(): Promise<void> {
  try {
    const server = createServer();
    
    // 서버 시작
    const app = server.listen(config.port, () => {
      logger.info(`Server running on port ${config.port}`);
    });

    // Graceful shutdown
    const gracefulShutdown = (signal: string) => {
      logger.info(`Received ${signal}. Shutting down gracefully...`);
      
      app.close(() => {
        logger.info('Server closed');
        process.exit(0);
      });
    };

    process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
    process.on('SIGINT', () => gracefulShutdown('SIGINT'));

  } catch (error) {
    logger.error('Failed to start server:', error);
    process.exit(1);
  }
}

// Handle unhandled promise rejections
process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Rejection at:', promise, 'reason:', reason);
  process.exit(1);
});

// Handle uncaught exceptions
process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception:', error);
  process.exit(1);
});

main();
```

### 6. 기본 테스트 파일

```typescript
// src/index.test.ts
import { createServer } from './server';
import request from 'supertest';

describe('App', () => {
  let app: any;

  beforeAll(() => {
    app = createServer();
  });

  afterAll(() => {
    app.close();
  });

  it('should respond with 200 for health check', async () => {
    const response = await request(app)
      .get('/health')
      .expect(200);
    
    expect(response.body).toEqual({ status: 'OK' });
  });

  it('should handle 404 for unknown routes', async () => {
    await request(app)
      .get('/unknown')
      .expect(404);
  });
});
```

### 7. README.md 템플릿

```markdown
# Project Name

프로젝트에 대한 간단한 설명을 여기에 작성합니다.

## 시작하기

### 필요 조건

- Node.js 18.0 이상
- npm

### 설치

\```bash
# 저장소 클론
git clone https://github.com/username/projectname.git
cd projectname

# 의존성 설치
npm install
\```

### 빌드 및 실행

\```bash
# 빌드
npm run build

# 실행
npm start

# 또는 개발 모드 실행
npm run dev
\```

### 테스트

\```bash
# 테스트 실행
npm test

# 커버리지 확인
npm run test:coverage
\```

## 프로젝트 구조

\```
.
├── src/           # 소스 코드
├── tests/         # 테스트 파일
├── dist/          # 빌드 결과물
├── docs/          # 문서
└── ...
\```

## 기여하기

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 라이선스

이 프로젝트는 MIT 라이선스를 따릅니다. 자세한 내용은 [LICENSE](LICENSE) 파일을 참조하세요.
```

### 8. 개발 도구 설치

```bash
# 기본 개발 도구
npm install -D typescript @types/node

# 린터 및 포맷터
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
npm install -D prettier

# 테스트 도구
npm install -D jest @types/jest ts-jest
npm install -D supertest @types/supertest

# 개발 서버 (핫 리로드)
npm install -D tsx nodemon

# Git 훅
npm install -D husky lint-staged

# 타입 체크
npm install -D tsc-watch

# 추가 유틸리티
npm install -D rimraf # 크로스 플랫폼 rm -rf
npm install -D concurrently # 동시 스크립트 실행
```

### 9. ESLint 설정

```javascript
// .eslintrc.js
module.exports = {
  parser: '@typescript-eslint/parser',
  extends: [
    'eslint:recommended',
    '@typescript-eslint/recommended',
    'prettier'
  ],
  plugins: ['@typescript-eslint'],
  parserOptions: {
    ecmaVersion: 2020,
    sourceType: 'module',
    project: './tsconfig.json'
  },
  rules: {
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    '@typescript-eslint/explicit-function-return-type': 'warn',
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/no-non-null-assertion': 'error',
    'prefer-const': 'error',
    'no-var': 'error'
  },
  env: {
    node: true,
    es2020: true
  },
  overrides: [
    {
      files: ['**/*.test.ts', '**/*.spec.ts'],
      env: {
        jest: true
      },
      rules: {
        '@typescript-eslint/no-explicit-any': 'off'
      }
    }
  ]
};
```

### 10. VS Code 설정 (선택사항)

`.vscode/settings.json`:
```json
{
  "typescript.preferences.importModuleSpecifier": "relative",
  "typescript.suggest.autoImports": true,
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true,
    "source.organizeImports": true
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "eslint.validate": [
    "typescript",
    "javascript"
  ],
  "typescript.preferences.quoteStyle": "single",
  "files.exclude": {
    "node_modules": true,
    "dist": true,
    "coverage": true
  }
}

.vscode/extensions.json:
{
  "recommendations": [
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-typescript-next",
    "dbaeumer.vscode-eslint",
    "bradlc.vscode-tailwindcss",
    "ms-vscode.vscode-json"
  ]
}
```

## 🎯 다음 단계

프로젝트 초기 설정이 완료되면:

1. `1-requirements/REQUIREMENTS.md`에서 요구사항 정의
2. `3-implementation/TDD_GUIDE.md`를 참고하여 테스트 작성
3. `3-implementation/CODE_STRUCTURE.md`를 참고하여 코드 구조화
4. `2-design/ARCHITECTURE.md`를 참고하여 아키텍처 설계

## 💡 팁

- 처음에는 단순하게 시작하고 필요에 따라 구조를 확장하세요
- 타입 안전성을 최대한 활용하세요 (any 사용 지양)
- 외부 의존성은 신중하게 선택하세요
- 테스트를 먼저 작성하는 습관을 들이세요 (TDD)
- `npm audit`를 정기적으로 실행하여 보안 취약점을 확인하세요
- `tsconfig.json`에서 strict 모드를 활성화하세요

### 추가 설정 파일들

#### TypeScript 설정 (tsconfig.json)
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "removeComments": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noImplicitThis": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "moduleResolution": "node",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "resolveJsonModule": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

#### Prettier 설정 (.prettierrc)
```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

#### Jest 설정 (jest.config.js)
```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  transform: {
    '^.+\.ts$': 'ts-jest',
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.test.ts',
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  setupFilesAfterEnv: ['<rootDir>/tests/setup.ts'],
};
```

이 가이드를 따라 TypeScript 프로젝트를 체계적으로 시작할 수 있습니다!