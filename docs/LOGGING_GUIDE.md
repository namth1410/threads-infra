# Logging Best Practices Guide

Hướng dẫn cho developers về cách logging hiệu quả trong dự án Threads.

## 🎯 Logging Philosophy

**12-Factor App Principle**: Logs là event streams, ghi ra `stdout/stderr`, không quan tâm đến storage.

**Benefits:**
- ✅ Đơn giản: Không cần config log files, rotation, etc.
- ✅ Container-friendly: Docker/Kubernetes tự động thu thập stdout
- ✅ Centralized: Filebeat tự động gửi vào Elasticsearch
- ✅ Flexible: Dễ dàng thay đổi log destination

## 📝 Log Levels

Sử dụng đúng log level để dễ filter và troubleshoot:

| Level | Khi nào dùng | Ví dụ |
|-------|-------------|-------|
| **ERROR** | Lỗi cần xử lý ngay, ảnh hưởng đến user | Database connection failed, Payment processing error |
| **WARN** | Cảnh báo, cần theo dõi nhưng không critical | Deprecated API usage, High memory usage |
| **INFO** | Thông tin quan trọng về business logic | User logged in, Order created, File uploaded |
| **DEBUG** | Chi tiết cho development, không dùng production | Function parameters, SQL queries, API responses |

## 🔧 Backend (API) Logging

### Recommended: Structured JSON Logging

**Tại sao JSON?**
- ✅ Filebeat tự động parse
- ✅ Dễ query trong Kibana
- ✅ Có thể thêm metadata (userId, requestId, etc.)

### Node.js / NestJS Example

**Install Winston:**
```bash
npm install winston
```

**Configure Logger:**
```typescript
// src/config/logger.config.ts
import * as winston from 'winston';

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console()
  ]
});
```

**Usage Examples:**

```typescript
// ✅ GOOD: Structured logging
logger.info('User logged in', {
  userId: user.id,
  email: user.email,
  ip: req.ip,
  userAgent: req.headers['user-agent']
});

logger.error('Database query failed', {
  error: error.message,
  stack: error.stack,
  query: 'SELECT * FROM users',
  duration: 1234
});

logger.warn('High memory usage detected', {
  memoryUsage: process.memoryUsage(),
  threshold: '80%'
});

// ❌ BAD: Plain string (khó parse)
console.log('User logged in: ' + user.email);
```

**Output (JSON):**
```json
{
  "level": "info",
  "message": "User logged in",
  "timestamp": "2025-11-26T09:50:00.123Z",
  "userId": 123,
  "email": "user@example.com",
  "ip": "192.168.1.1",
  "userAgent": "Mozilla/5.0..."
}
```

### NestJS Integration

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { WinstonModule } from 'nest-winston';
import { logger } from './config/logger.config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: WinstonModule.createLogger({
      instance: logger,
    }),
  });
  
  await app.listen(3000);
}
```

**In Controllers/Services:**
```typescript
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class UserService {
  private readonly logger = new Logger(UserService.name);

  async createUser(dto: CreateUserDto) {
    this.logger.log('Creating new user', {
      email: dto.email,
      role: dto.role
    });
    
    try {
      const user = await this.userRepository.save(dto);
      
      this.logger.log('User created successfully', {
        userId: user.id,
        email: user.email
      });
      
      return user;
    } catch (error) {
      this.logger.error('Failed to create user', {
        error: error.message,
        stack: error.stack,
        dto
      });
      throw error;
    }
  }
}
```

## 🌐 Frontend (Web) Logging

### Strategy: Send Critical Errors to Backend

Frontend không nên ghi trực tiếp vào Elasticsearch (security risk). Thay vào đó, gửi errors về backend API.

### Error Logging Utility

```typescript
// src/utils/logger.ts
interface LogContext {
  userId?: string;
  page?: string;
  action?: string;
  [key: string]: any;
}

export class Logger {
  private static API_ENDPOINT = '/api/logs/frontend-error';

  static async error(error: Error, context?: LogContext) {
    // Log to console in development
    if (process.env.NODE_ENV === 'development') {
      console.error('[Frontend Error]', error, context);
    }

    // Send to backend in production
    if (process.env.NODE_ENV === 'production') {
      try {
        await fetch(this.API_ENDPOINT, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            message: error.message,
            stack: error.stack,
            context: {
              ...context,
              url: window.location.href,
              userAgent: navigator.userAgent,
              timestamp: new Date().toISOString(),
            },
          }),
        });
      } catch (e) {
        // Fail silently, don't break user experience
        console.error('Failed to send error log', e);
      }
    }
  }

  static info(message: string, context?: LogContext) {
    if (process.env.NODE_ENV === 'development') {
      console.log('[Frontend Info]', message, context);
    }
    // Don't send info logs to backend (too much data)
  }
}
```

### Usage in React Components

```typescript
// Component with error boundary
import { Logger } from '@/utils/logger';

function PaymentButton() {
  const handlePayment = async () => {
    try {
      await processPayment();
    } catch (error) {
      Logger.error(error as Error, {
        userId: currentUser.id,
        page: 'checkout',
        action: 'process_payment',
        amount: 100.00
      });
      
      // Show user-friendly error
      toast.error('Payment failed. Please try again.');
    }
  };

  return <button onClick={handlePayment}>Pay Now</button>;
}
```

### Global Error Handler

```typescript
// src/App.tsx
import { useEffect } from 'react';
import { Logger } from '@/utils/logger';

function App() {
  useEffect(() => {
    // Catch unhandled errors
    window.addEventListener('error', (event) => {
      Logger.error(event.error, {
        type: 'unhandled_error',
        filename: event.filename,
        lineno: event.lineno,
        colno: event.colno
      });
    });

    // Catch unhandled promise rejections
    window.addEventListener('unhandledrejection', (event) => {
      Logger.error(new Error(event.reason), {
        type: 'unhandled_rejection'
      });
    });
  }, []);

  return <YourApp />;
}
```

### Backend API Endpoint

```typescript
// src/logs/logs.controller.ts
import { Controller, Post, Body } from '@nestjs/common';
import { Logger } from '@nestjs/common';

@Controller('logs')
export class LogsController {
  private readonly logger = new Logger('FrontendLogs');

  @Post('frontend-error')
  async logFrontendError(@Body() errorData: any) {
    // Log to stdout → Filebeat → Elasticsearch
    this.logger.error('Frontend error received', {
      source: 'frontend',
      ...errorData
    });

    return { status: 'logged' };
  }
}
```

## 📊 What to Log

### ✅ DO Log:

**Business Events:**
```typescript
logger.info('User registered', { userId, email });
logger.info('Order placed', { orderId, userId, amount });
logger.info('File uploaded', { fileId, userId, size, mimeType });
```

**Errors:**
```typescript
logger.error('Payment failed', { error, userId, orderId, amount });
logger.error('Database connection lost', { error, retryCount });
```

**Performance Issues:**
```typescript
logger.warn('Slow query detected', { query, duration: 5234 });
logger.warn('High memory usage', { usage: '85%', threshold: '80%' });
```

**Security Events:**
```typescript
logger.warn('Failed login attempt', { email, ip, attempts: 5 });
logger.info('Password changed', { userId, ip });
```

### ❌ DON'T Log:

**Sensitive Data:**
```typescript
// ❌ BAD
logger.info('User logged in', { password: user.password });
logger.info('Payment processed', { creditCard: '1234-5678-9012-3456' });

// ✅ GOOD
logger.info('User logged in', { userId: user.id });
logger.info('Payment processed', { orderId, last4: '3456' });
```

**Too Much Detail in Production:**
```typescript
// ❌ BAD (production)
logger.debug('Function called', { params: allParams });
logger.debug('SQL query', { query, params });

// ✅ GOOD (only in development)
if (process.env.NODE_ENV === 'development') {
  logger.debug('Function called', { params });
}
```

## 🔍 Query Logs in Kibana

### Find Errors from Specific User

```
service: "api" AND level: "error" AND userId: "123"
```

### Find Slow Queries

```
service: "postgres" AND duration > 3000
```

### Find Failed Login Attempts

```
service: "api" AND message: *"Failed login"* AND attempts >= 5
```

### Find Frontend Errors

```
source: "frontend" AND level: "error"
```

## 📈 Monitoring & Alerts

### Create Alerts in Kibana

1. Vào **Stack Management** → **Rules and Connectors**
2. Click **Create rule**
3. Chọn **Elasticsearch query**
4. Configure:
   - **Index**: `filebeat-api-*`
   - **Query**: `level: "error"`
   - **Threshold**: Count > 10 in 5 minutes
5. Add action (Email, Slack, etc.)

### Example Alerts

**High Error Rate:**
```
Query: level: "error"
Threshold: Count > 50 in 5 minutes
Action: Send Slack notification
```

**Database Connection Issues:**
```
Query: message: *"database connection"* AND level: "error"
Threshold: Count > 1 in 1 minute
Action: Send email to DevOps team
```

## 🎓 Summary

1. **Use structured JSON logging** cho backend
2. **Log to stdout/stderr**, không log vào files
3. **Use appropriate log levels** (ERROR, WARN, INFO, DEBUG)
4. **Don't log sensitive data** (passwords, credit cards, etc.)
5. **Frontend errors** gửi về backend API
6. **Add context** (userId, requestId, etc.) vào mọi log
7. **Monitor logs** trong Kibana và setup alerts

Happy logging! 🚀
