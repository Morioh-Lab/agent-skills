---
name: security-and-hardening
description: Hardens code against vulnerabilities. Use when handling user input, authentication, data storage, or external integrations. Use when building any feature that accepts untrusted data, manages user sessions, or interacts with third-party services.
---

# Security and Hardening

## Overview

Security-first development practices for web applications with NestJS backend and NextJS frontend. Treat every external input as hostile, every secret as sacred, and every authorization check as mandatory. Security isn't a phase — it's a constraint on every line of code that touches user data, authentication, or external systems.

## When to Use

- Building anything that accepts user input
- Implementing authentication or authorization (NestJS Guards, NextJS Middleware)
- Storing or transmitting sensitive data (MongoDB encryption)
- Integrating with external APIs or services
- Adding file uploads, webhooks, or callbacks
- Handling payment or PII data

## The Three-Tier Boundary System

### Always Do (No Exceptions)

- **Validate all external input** at the system boundary (NestJS DTOs with class-validator, NextJS API route validation)
- **Parameterize all database queries** — MikroORM handles this automatically, never use raw queries with concatenation
- **Encode output** to prevent XSS (NextJS auto-escapes by default, don't use dangerouslySetInnerHTML)
- **Use HTTPS** for all external communication
- **Hash passwords** with bcrypt/scrypt/argon2 (never store plaintext)
- **Set security headers** (CSP, HSTS, X-Frame-Options, X-Content-Type-Options) via NextJS middleware or NestJS
- **Use httpOnly, secure, sameSite cookies** for sessions
- **Run `npm audit`** (or equivalent) before every release

### Ask First (Requires Human Approval)

- Adding new authentication flows or changing auth logic (NestJS AuthModule, JWT strategies)
- Storing new categories of sensitive data (PII, payment info) in MongoDB
- Adding new external service integrations
- Changing CORS configuration in NestJS
- Adding file upload handlers
- Modifying rate limiting or throttling (NestJS ThrottlerModule)
- Granting elevated permissions or roles

### Never Do

- **Never commit secrets** to version control (API keys, passwords, tokens, MongoDB connection strings)
- **Never log sensitive data** (passwords, tokens, full credit card numbers)
- **Never trust client-side validation** as a security boundary
- **Never disable security headers** for convenience
- **Never use `eval()` or `dangerouslySetInnerHTML`** with user-provided data
- **Never store sessions in client-accessible storage** (localStorage for auth tokens)
- **Never expose stack traces** or internal error details to users (use NestJS Exception Filters)

## OWASP Top 10 Prevention

### 1. Injection (SQL, NoSQL, OS Command)

```typescript
// BAD: NoSQL injection via query construction
const query = { $where: `this.username === '${username}'` };

// GOOD: MikroORM handles parameterization automatically
const user = await em.findOne(User, { username });

// GOOD: With conditions using object notation (safe)
const users = await em.find(User, { 
  status: 'active',
  role: { $in: ['user', 'admin'] } 
});

// BAD: Raw MongoDB query with concatenation
const collection = db.collection('users');
const result = await collection.find({ $where: `this.age > ${age}` }).toArray();

// GOOD: Use MikroORM's query builder or native driver safely
const qb = em.getQueryBuilder(User);
const result = await qb.where({ age: { $gt: age } }).getResult();
```

### 2. Broken Authentication

```typescript
// Password hashing in NestJS service
import { hash, compare } from 'bcrypt';

const SALT_ROUNDS = 12;
const hashedPassword = await hash(plaintext, SALT_ROUNDS);
const isValid = await compare(plaintext, hashedPassword);

// NestJS JWT Authentication Strategy
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: process.env.JWT_SECRET,  // From environment, not code
    });
  }

  async validate(payload: JwtPayload) {
    return { userId: payload.sub, username: payload.username };
  }
}

// Session management with httpOnly cookies
app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,     // Not accessible via JavaScript
    secure: true,       // HTTPS only
    sameSite: 'lax',    // CSRF protection
    maxAge: 24 * 60 * 60 * 1000,  // 24 hours
  },
}));
```

### 3. Cross-Site Scripting (XSS)

```typescript
// NextJS: Auto-escaping by default (GOOD)
return <div>{userInput}</div>;

// BAD: Bypassing NextJS protection
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// If you MUST render HTML, sanitize first
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(userInput);
<div dangerouslySetInnerHTML={{ __html: clean }} />
```

### 4. Broken Access Control

```typescript
// NestJS Guard for authorization
@Injectable()
export class OwnerGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private em: EntityManager,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const taskId = request.params.id;

    const task = await this.em.findOne(Task, taskId);
    
    // Check that the authenticated user owns this resource
    if (!task || task.ownerId !== user.id) {
      throw new ForbiddenException('Not authorized to modify this task');
    }

    return true;
  }
}

// Usage in controller
@Patch(':id')
@UseGuards(JwtAuthGuard, OwnerGuard)
async updateTask(@Param('id') id: string, @Body() dto: UpdateTaskDto) {
  return this.taskService.update(id, dto);
}
```

### 5. Security Misconfiguration

```typescript
// NextJS: Security headers in next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-DNS-Prefetch-Control', value: 'on' },
          { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
          { key: 'X-XSS-Protection', value: '1; mode=block' },
          { key: 'X-Frame-Options', value: 'SAMEORIGIN' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          {
            key: 'Content-Security-Policy',
            value: "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:;"
          },
        ],
      },
    ];
  },
};

// NestJS: CORS configuration
@Module({
  imports: [
    ConfigModule.forRoot(),
  ],
})
export class AppModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(cors({
        origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
        credentials: true,
        methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
        allowedHeaders: ['Content-Type', 'Authorization'],
      }))
      .forRoutes('*');
  }
}
```

### 6. Sensitive Data Exposure

```typescript
// NestJS: Exclude sensitive fields from responses
class UserResponseDto {
  @Expose()
  id: string;

  @Expose()
  username: string;

  @Expose()
  email: string;

  // passwordHash is never exposed
}

@Get('users/:id')
@UseInterceptors(ClassSerializerInterceptor)
async getUser(@Param('id') id: string): Promise<UserResponseDto> {
  const user = await this.userService.findById(id);
  return plainToInstance(UserResponseDto, user);
}

// Use environment variables for secrets
const API_KEY = process.env.STRIPE_API_KEY;
if (!API_KEY) throw new Error('STRIPE_API_KEY not configured');
```

## Input Validation Patterns

### NestJS DTOs with class-validator

```typescript
import { IsString, IsOptional, IsEnum, MaxLength, IsDateString } from 'class-validator';

export class CreateTaskDto {
  @IsString()
  @MaxLength(200)
  title: string;

  @IsString()
  @IsOptional()
  @MaxLength(2000)
  description?: string;

  @IsEnum(['low', 'medium', 'high'])
  @IsOptional()
  priority?: 'low' | 'medium' | 'high' = 'medium';

  @IsDateString()
  @IsOptional()
  dueDate?: string;
}

// Controller with automatic validation
@Controller('tasks')
export class TaskController {
  @Post()
  @UsePipes(new ValidationPipe({ transform: true, whitelist: true, forbidNonWhitelisted: true }))
  async createTask(@Body() dto: CreateTaskDto) {
    return this.taskService.create(dto);
  }
}
```

### File Upload Safety

```typescript
import { FileInterceptor } from '@nestjs/platform-express';
import { diskStorage } from 'multer';
import { extname } from 'path';

const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

@Post('upload')
@UseInterceptors(FileInterceptor('file', {
  storage: diskStorage({
    destination: './uploads',
    filename: (req, file, cb) => {
      const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
      cb(null, `${uniqueSuffix}${extname(file.originalname)}`);
    },
  }),
  limits: {
    fileSize: MAX_SIZE,
  },
  fileFilter: (req, file, cb) => {
    if (!ALLOWED_TYPES.includes(file.mimetype)) {
      return cb(new BadRequestException('File type not allowed'), false);
    }
    cb(null, true);
  },
}))
async uploadFile(@UploadedFile() file: Express.Multer.File) {
  // Don't trust the file extension — check magic bytes if critical
  return { filename: file.filename };
}
```

## Triaging npm audit Results

Not all audit findings require immediate action. Use this decision tree:

```
npm audit reports a vulnerability
├── Severity: critical or high
│   ├── Is the vulnerable code reachable in your app?
│   │   ├── YES --> Fix immediately (update, patch, or replace the dependency)
│   │   └── NO (dev-only dep, unused code path) --> Fix soon, but not a blocker
│   └── Is a fix available?
│       ├── YES --> Update to the patched version
│       └── NO --> Check for workarounds, consider replacing the dependency, or add to allowlist with a review date
├── Severity: moderate
│   ├── Reachable in production? --> Fix in the next release cycle
│   └── Dev-only? --> Fix when convenient, track in backlog
└── Severity: low
    └── Track and fix during regular dependency updates
```

**Key questions:**
- Is the vulnerable function actually called in your code path?
- Is the dependency a runtime dependency or dev-only?
- Is the vulnerability exploitable given your deployment context (e.g., a server-side vulnerability in a client-only app)?

When you defer a fix, document the reason and set a review date.

## Rate Limiting

```typescript
// NestJS ThrottlerModule
@Module({
  imports: [
    ThrottlerModule.forRoot([{
      ttl: 60000,         // 1 minute
      limit: 10,          // 10 requests per minute
    }]),
  ],
})
export class AppModule {}

// Apply to specific routes
@Controller('auth')
export class AuthController {
  @Post('login')
  @Throttle(5, 60)  // 5 requests per 60 seconds
  async login(@Body() dto: LoginDto) {
    return this.authService.login(dto);
  }
}

// NextJS: Rate limiting in API routes or middleware
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, "1 m"),
  analytics: true,
});

// In NextJS API route or middleware
export async function POST(req: Request) {
  const ip = req.headers.get("x-forwarded-for");
  const { success } = await ratelimit.limit(ip ?? "");
  
  if (!success) {
    return new Response("Too many requests", { status: 429 });
  }
  
  // Process request
}
```

## Secrets Management

```
.env files:
  ├── .env.example  → Committed (template with placeholder values)
  ├── .env          → NOT committed (contains real secrets)
  └── .env.local    → NOT committed (local overrides)

.gitignore must include:
  .env
  .env.local
  .env.*.local
  *.pem
  *.key
  .next/cache  # NextJS cache
```

**Always check before committing:**
```bash
# Check for accidentally staged secrets
git diff --cached | grep -i "password\|secret\|api_key\|token\|mongodb"
```

## Security Review Checklist

```markdown
### Authentication
- [ ] Passwords hashed with bcrypt/scrypt/argon2 (salt rounds ≥ 12)
- [ ] JWT tokens properly signed and validated (NestJS JwtModule)
- [ ] Session tokens are httpOnly, secure, sameSite
- [ ] Login has rate limiting (NestJS ThrottlerModule)
- [ ] Password reset tokens expire

### Authorization
- [ ] Every endpoint protected with guards (NestJS @UseGuards)
- [ ] Users can only access their own resources (OwnerGuard, RBAC)
- [ ] Admin actions require admin role verification

### Input
- [ ] All user input validated with DTOs (class-validator)
- [ ] MikroORM queries use object notation (no raw queries with concatenation)
- [ ] HTML output is encoded/escaped (NextJS auto-escaping)

### Data
- [ ] No secrets in code or version control (MongoDB connection strings, API keys)
- [ ] Sensitive fields excluded from API responses (ClassSerializerInterceptor)
- [ ] PII encrypted at rest if required (MongoDB field-level encryption)

### Infrastructure
- [ ] Security headers configured (NextJS next.config.js or middleware)
- [ ] CORS restricted to known origins (NestJS cors configuration)
- [ ] Dependencies audited for vulnerabilities
- [ ] Error messages don't expose internals (NestJS Exception Filters)
```
## See Also

For detailed security checklists and pre-commit verification steps, see `references/security-checklist.md`.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "This is an internal tool, security doesn't matter" | Internal tools get compromised. Attackers target the weakest link. |
| "We'll add security later" | Security retrofitting is 10x harder than building it in. Add it now. |
| "No one would try to exploit this" | Automated scanners will find it. Security by obscurity is not security. |
| "The framework handles security" | NestJS and NextJS provide tools, not guarantees. You still need to use them correctly. |
| "It's just a prototype" | Prototypes become production. Security habits from day one. |

## Red Flags

- User input passed directly to database queries, shell commands, or HTML rendering
- Secrets in source code or commit history (especially MongoDB connection strings)
- API endpoints without authentication or authorization checks (missing @UseGuards)
- Missing CORS configuration or wildcard (`*`) origins
- No rate limiting on authentication endpoints
- Stack traces or internal errors exposed to users (configure NestJS Exception Filters)
- Dependencies with known critical vulnerabilities

## Verification

After implementing security-relevant code:

- [ ] `npm audit` shows no critical or high vulnerabilities
- [ ] No secrets in source code or git history
- [ ] All user input validated at system boundaries (DTOs with class-validator)
- [ ] Authentication and authorization checked on every protected endpoint (@UseGuards)
- [ ] Security headers present in response (check with browser DevTools)
- [ ] Error responses don't expose internal details (Exception Filters configured)
- [ ] Rate limiting active on auth endpoints (ThrottlerModule)
- [ ] MikroORM queries use safe object notation (no raw queries)
