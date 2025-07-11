# JavaScript/TypeScript Development Guide

This guide provides specialized guidance for JavaScript and TypeScript development, focusing on common pitfalls, performance optimization, and type safety patterns derived from real-world project experience.

## Overview

JavaScript and TypeScript development involves unique challenges including numeric precision, type safety, runtime validation, and performance optimization. This guide captures solutions to common problems encountered in modern web development.

## JavaScript Precision and Numeric Issues

### Number Precision Problems

#### The Problem: MAX_SAFE_INTEGER Limitation
```javascript
// JavaScript precision limitation
console.log(Number.MAX_SAFE_INTEGER); // 9007199254740991 (2^53 - 1)

// Problem: Large integers lose precision
const bigNumber = 1234567890123456789;
console.log(bigNumber); // 1234567890123456800 (precision lost!)

// Common issue: Timestamp precision loss
const nanosecondTimestamp = Date.now() * 1000000; // Often exceeds MAX_SAFE_INTEGER
```

#### Solution Strategies

##### 1. String-Based Approach
```typescript
// Solution: Handle large numbers as strings
interface TimestampHandler {
  generateTimestamp(): string;
  validateTimestamp(timestamp: string): boolean;
}

class NanosecondTimestamp implements TimestampHandler {
  generateTimestamp(): string {
    // Generate as string to avoid precision loss
    const microseconds = Date.now() * 1000;
    const nanoseconds = microseconds * 1000;
    return nanoseconds.toString();
  }
  
  validateTimestamp(timestamp: string): boolean {
    // Validate string format
    return /^\d{19}$/.test(timestamp);
  }
}
```

##### 2. BigInt for Large Integer Operations
```typescript
// Solution: Use BigInt for large integer arithmetic
class BigIntTimestamp {
  static generate(): string {
    const now = BigInt(Date.now());
    const nanoseconds = now * 1000000n;
    return nanoseconds.toString();
  }
  
  static add(timestamp1: string, timestamp2: string): string {
    const big1 = BigInt(timestamp1);
    const big2 = BigInt(timestamp2);
    return (big1 + big2).toString();
  }
}
```

##### 3. Decimal.js for Precise Arithmetic
```typescript
import { Decimal } from 'decimal.js';

// Solution: Use Decimal.js for precise calculations
class PreciseCalculator {
  static multiply(a: string, b: string): string {
    const decimalA = new Decimal(a);
    const decimalB = new Decimal(b);
    return decimalA.mul(decimalB).toString();
  }
  
  static divide(a: string, b: string): string {
    const decimalA = new Decimal(a);
    const decimalB = new Decimal(b);
    return decimalA.div(decimalB).toString();
  }
}
```

### Testing Strategies for Precision Issues

#### Pattern Matching Instead of Exact Comparison
```typescript
// Bad: Exact comparison (fails due to precision)
test('should generate exact timestamp', () => {
  const timestamp = generateTimestamp();
  expect(timestamp).toBe('1234567890123456789');
});

// Good: Pattern matching
test('should generate valid timestamp format', () => {
  const timestamp = generateTimestamp();
  expect(timestamp).toMatch(/^\d{19}$/);
  expect(timestamp.length).toBe(19);
  expect(timestamp.startsWith('1234567890123456')).toBe(true);
});
```

#### Range-Based Validation
```typescript
// Good: Range validation for timestamps
test('should generate timestamp within valid range', () => {
  const timestamp = generateTimestamp();
  const timestampNum = BigInt(timestamp);
  const now = BigInt(Date.now()) * 1000000n;
  const fiveMinutesAgo = now - 300000000000n; // 5 minutes in nanoseconds
  
  expect(timestampNum).toBeGreaterThan(fiveMinutesAgo);
  expect(timestampNum).toBeLessThanOrEqual(now);
});
```

## TypeScript Type Safety Patterns

### Runtime Type Validation

#### Type Guard Functions
```typescript
// Interface definition
interface InitOptions {
  endpoint: string;
  serviceName: string;
  batchSize?: number;
  sendInterval?: number;
  debug?: boolean;
}

// Type guard implementation
function isInitOptions(obj: any): obj is InitOptions {
  return (
    typeof obj === 'object' &&
    obj !== null &&
    typeof obj.endpoint === 'string' &&
    typeof obj.serviceName === 'string' &&
    (obj.batchSize === undefined || typeof obj.batchSize === 'number') &&
    (obj.sendInterval === undefined || typeof obj.sendInterval === 'number') &&
    (obj.debug === undefined || typeof obj.debug === 'boolean')
  );
}

// Usage with validation
function createLogger(options: unknown): Logger {
  if (!isInitOptions(options)) {
    throw new Error('Invalid options provided');
  }
  
  // TypeScript now knows options is InitOptions
  return new Logger(options);
}
```

#### Validation with Detailed Error Messages
```typescript
// Enhanced validation with specific error messages
function validateInitOptions(obj: any): asserts obj is InitOptions {
  if (typeof obj !== 'object' || obj === null) {
    throw new Error('Options must be an object');
  }
  
  if (typeof obj.endpoint !== 'string') {
    throw new Error('endpoint must be a string');
  }
  
  if (typeof obj.serviceName !== 'string') {
    throw new Error('serviceName must be a string');
  }
  
  if (obj.batchSize !== undefined && typeof obj.batchSize !== 'number') {
    throw new Error('batchSize must be a number');
  }
  
  if (obj.sendInterval !== undefined && typeof obj.sendInterval !== 'number') {
    throw new Error('sendInterval must be a number');
  }
  
  if (obj.debug !== undefined && typeof obj.debug !== 'boolean') {
    throw new Error('debug must be a boolean');
  }
}
```

### Advanced Type Patterns

#### Conditional Types for API Design
```typescript
// Conditional types for flexible API design
type LogLevel = 'debug' | 'info' | 'warn' | 'error';

interface LogOptions<T extends LogLevel> {
  level: T;
  message: string;
  metadata?: T extends 'error' ? { error: Error } : Record<string, any>;
}

// Usage ensures error level includes error metadata
function log<T extends LogLevel>(options: LogOptions<T>): void {
  if (options.level === 'error') {
    // TypeScript knows metadata includes error
    const error = (options.metadata as { error: Error }).error;
    console.error(options.message, error);
  } else {
    console.log(options.message, options.metadata);
  }
}
```

#### Branded Types for Type Safety
```typescript
// Branded types for preventing mixing of similar types
type TraceId = string & { readonly __brand: 'TraceId' };
type SpanId = string & { readonly __brand: 'SpanId' };

// Factory functions for branded types
function createTraceId(id: string): TraceId {
  if (!/^[a-f0-9]{32}$/.test(id)) {
    throw new Error('Invalid trace ID format');
  }
  return id as TraceId;
}

function createSpanId(id: string): SpanId {
  if (!/^[a-f0-9]{16}$/.test(id)) {
    throw new Error('Invalid span ID format');
  }
  return id as SpanId;
}

// Usage prevents mixing of IDs
interface SpanContext {
  traceId: TraceId;
  spanId: SpanId;
}

function createSpanContext(traceId: TraceId, spanId: SpanId): SpanContext {
  return { traceId, spanId };
}

// This would cause TypeScript error:
// createSpanContext(spanId, traceId); // Error: types don't match
```

#### Utility Types for Data Transformation
```typescript
// Utility types for safe data transformation
type AttributeValue = string | number | boolean;

interface RawAttributes {
  [key: string]: AttributeValue;
}

// Transform raw attributes to OTLP format
type OTLPAttributeValue = 
  | { stringValue: string }
  | { intValue: number }
  | { boolValue: boolean };

type OTLPAttributes = {
  [K in keyof RawAttributes]: OTLPAttributeValue;
};

// Type-safe transformation function
function convertAttributes(attributes: RawAttributes): OTLPAttributes {
  const result: Record<string, OTLPAttributeValue> = {};
  
  for (const [key, value] of Object.entries(attributes)) {
    if (typeof value === 'string') {
      result[key] = { stringValue: value };
    } else if (typeof value === 'number') {
      result[key] = { intValue: value };
    } else if (typeof value === 'boolean') {
      result[key] = { boolValue: value };
    }
  }
  
  return result as OTLPAttributes;
}
```

## Performance Optimization Patterns

### Efficient Object Creation

#### Object Pooling for High-Frequency Operations
```typescript
// Object pooling for frequently created objects
class LogRecordPool {
  private pool: LogRecord[] = [];
  private readonly maxSize = 100;
  
  acquire(): LogRecord {
    if (this.pool.length > 0) {
      return this.pool.pop()!;
    }
    return this.createNew();
  }
  
  release(record: LogRecord): void {
    if (this.pool.length < this.maxSize) {
      this.reset(record);
      this.pool.push(record);
    }
  }
  
  private createNew(): LogRecord {
    return {
      body: '',
      timestamp: 0,
      severityNumber: 0,
      severityText: '',
      attributes: {}
    };
  }
  
  private reset(record: LogRecord): void {
    record.body = '';
    record.timestamp = 0;
    record.severityNumber = 0;
    record.severityText = '';
    record.attributes = {};
  }
}
```

#### Lazy Property Initialization
```typescript
// Lazy initialization for expensive properties
class Logger {
  private _exporter?: LogExporter;
  private _processor?: LogProcessor;
  
  get exporter(): LogExporter {
    if (!this._exporter) {
      this._exporter = new CustomLogExporter(this.options.endpoint);
    }
    return this._exporter;
  }
  
  get processor(): LogProcessor {
    if (!this._processor) {
      this._processor = new BatchLogRecordProcessor(this.exporter, {
        maxExportBatchSize: this.options.batchSize,
        scheduledDelayMillis: this.options.sendInterval
      });
    }
    return this._processor;
  }
}
```

### Memory Management

#### Weak References for Cache Management
```typescript
// Weak references for automatic cleanup
class LoggerCache {
  private cache = new WeakMap<object, Logger>();
  
  get(key: object): Logger | undefined {
    return this.cache.get(key);
  }
  
  set(key: object, logger: Logger): void {
    this.cache.set(key, logger);
  }
  
  // Automatic cleanup when key is garbage collected
}
```

#### Efficient String Concatenation
```typescript
// Efficient string building for large messages
class LogMessageBuilder {
  private parts: string[] = [];
  
  add(part: string): this {
    this.parts.push(part);
    return this;
  }
  
  build(): string {
    return this.parts.join('');
  }
  
  reset(): void {
    this.parts.length = 0;
  }
}

// Usage
const builder = new LogMessageBuilder();
const message = builder
  .add('Error in ')
  .add(functionName)
  .add(': ')
  .add(errorMessage)
  .build();
```

## Error Handling Patterns

### Structured Error Handling

#### Custom Error Classes with Context
```typescript
// Custom error hierarchy
abstract class BaseError extends Error {
  abstract readonly code: string;
  readonly timestamp: Date;
  readonly context: Record<string, any>;
  
  constructor(message: string, context: Record<string, any> = {}) {
    super(message);
    this.name = this.constructor.name;
    this.timestamp = new Date();
    this.context = context;
  }
}

class ValidationError extends BaseError {
  readonly code = 'VALIDATION_ERROR';
}

class NetworkError extends BaseError {
  readonly code = 'NETWORK_ERROR';
}

class ConfigurationError extends BaseError {
  readonly code = 'CONFIG_ERROR';
}

// Usage with context
function validateEndpoint(endpoint: string): void {
  if (!endpoint) {
    throw new ValidationError('Endpoint is required', { endpoint });
  }
  
  if (!endpoint.startsWith('http')) {
    throw new ValidationError('Endpoint must be HTTP URL', { endpoint });
  }
}
```

#### Result Pattern for Error Handling
```typescript
// Result pattern for avoiding exceptions
type Result<T, E = Error> = 
  | { success: true; data: T }
  | { success: false; error: E };

// Helper functions
function success<T>(data: T): Result<T, never> {
  return { success: true, data };
}

function failure<E>(error: E): Result<never, E> {
  return { success: false, error };
}

// Usage in async operations
async function exportLogs(logs: LogRecord[]): Promise<Result<ExportResult, NetworkError>> {
  try {
    const response = await fetch(endpoint, {
      method: 'POST',
      body: JSON.stringify(logs)
    });
    
    if (!response.ok) {
      return failure(new NetworkError(`HTTP ${response.status}`, { 
        status: response.status,
        statusText: response.statusText 
      }));
    }
    
    return success({ code: ExportResultCode.SUCCESS });
  } catch (error) {
    return failure(new NetworkError('Network request failed', { error }));
  }
}

// Error handling without exceptions
const result = await exportLogs(logs);
if (result.success) {
  console.log('Export successful:', result.data);
} else {
  console.error('Export failed:', result.error);
}
```

## Asynchronous Programming Patterns

### Promise Management

#### Promise Pooling for Concurrent Operations
```typescript
// Promise pool for limiting concurrent operations
class PromisePool {
  private running = 0;
  private readonly queue: (() => Promise<any>)[] = [];
  
  constructor(private readonly maxConcurrency: number) {}
  
  async add<T>(promiseFactory: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push(async () => {
        try {
          const result = await promiseFactory();
          resolve(result);
        } catch (error) {
          reject(error);
        }
      });
      
      this.tryNext();
    });
  }
  
  private tryNext(): void {
    if (this.running >= this.maxConcurrency || this.queue.length === 0) {
      return;
    }
    
    this.running++;
    const promiseFactory = this.queue.shift()!;
    
    promiseFactory().finally(() => {
      this.running--;
      this.tryNext();
    });
  }
}
```

#### Timeout and Retry Patterns
```typescript
// Timeout wrapper for promises
function withTimeout<T>(promise: Promise<T>, timeoutMs: number): Promise<T> {
  return Promise.race([
    promise,
    new Promise<never>((_, reject) => {
      setTimeout(() => reject(new Error('Operation timed out')), timeoutMs);
    })
  ]);
}

// Retry with exponential backoff
async function withRetry<T>(
  operation: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  let lastError: Error;
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error as Error;
      
      if (attempt === maxRetries) {
        throw lastError;
      }
      
      const delay = baseDelay * Math.pow(2, attempt);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  
  throw lastError!;
}

// Usage
const result = await withRetry(
  () => withTimeout(exportLogs(logs), 5000),
  3,
  1000
);
```

## Testing Patterns

### TypeScript Testing Utilities

#### Type-Safe Mock Creation
```typescript
// Type-safe mock creation
function createMock<T>(): jest.Mocked<T> {
  return {} as jest.Mocked<T>;
}

// Usage
interface LogExporter {
  export(logs: LogRecord[]): Promise<ExportResult>;
  shutdown(): Promise<void>;
}

const mockExporter = createMock<LogExporter>();
mockExporter.export.mockResolvedValue({ code: ExportResultCode.SUCCESS });
```

#### Test Data Builders with Types
```typescript
// Type-safe test data builders
class LogRecordBuilder {
  private record: Partial<LogRecord> = {};
  
  withBody(body: string): this {
    this.record.body = body;
    return this;
  }
  
  withTimestamp(timestamp: number): this {
    this.record.timestamp = timestamp;
    return this;
  }
  
  withSeverity(number: number, text: string): this {
    this.record.severityNumber = number;
    this.record.severityText = text;
    return this;
  }
  
  build(): LogRecord {
    return {
      body: 'default message',
      timestamp: Date.now() * 1000000,
      severityNumber: 9,
      severityText: 'INFO',
      attributes: {},
      ...this.record
    };
  }
}

// Usage in tests
const logRecord = new LogRecordBuilder()
  .withBody('test message')
  .withSeverity(12, 'WARN')
  .build();
```

## Best Practices Summary

### Do's
- ✅ Use string handling for large numbers exceeding MAX_SAFE_INTEGER
- ✅ Implement runtime type validation for public APIs
- ✅ Use branded types for preventing type confusion
- ✅ Apply Result pattern for error handling in complex operations
- ✅ Use object pooling for high-frequency allocations
- ✅ Implement proper timeout and retry mechanisms
- ✅ Write type-safe tests with proper mock types

### Don'ts
- ❌ Rely on JavaScript number precision for large integers
- ❌ Skip runtime validation in library code
- ❌ Mix similar types without branded type safety
- ❌ Use exceptions for expected error conditions
- ❌ Create unnecessary objects in performance-critical code
- ❌ Ignore timeout and error handling in async operations
- ❌ Write tests without proper type safety

## Common Pitfalls and Solutions

### Pitfall: Precision Loss in Timestamps
```typescript
// Problem: Precision loss
const timestamp = Date.now() * 1000000; // May lose precision

// Solution: String-based timestamps
const timestamp = (Date.now() * 1000000).toString();
```

### Pitfall: Unsafe Type Assertions
```typescript
// Problem: Unsafe assertion
const options = userInput as InitOptions;

// Solution: Type guard validation
if (isInitOptions(userInput)) {
  const options = userInput; // Type-safe
}
```

### Pitfall: Memory Leaks in Event Handlers
```typescript
// Problem: Memory leak
element.addEventListener('click', handler);

// Solution: Proper cleanup
const controller = new AbortController();
element.addEventListener('click', handler, { signal: controller.signal });
// Later: controller.abort();
```

---

**Version**: 1.0  
**Last Updated**: 2025-07-11  
**Based on**: OpenTelemetry Front-End SDK Project Experience  
**Maintained by**: Claude AI Development Team