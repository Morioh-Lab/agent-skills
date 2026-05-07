---
name: api-and-interface-design
description: Guides stable API and interface design. Use when designing APIs, module boundaries, or any public interface. Use when creating REST endpoints with NestJS controllers, defining DTOs and type contracts between modules, or establishing boundaries between frontend (Next.js) and backend (NestJS).
---

# API and Interface Design

## Overview

Design stable, well-documented interfaces that are hard to misuse. Good interfaces make the right thing easy and the wrong thing hard. This applies to REST APIs with NestJS controllers, GraphQL schemas, module boundaries, component props, and any surface where one piece of code talks to another.

## When to Use

- Designing new API endpoints with NestJS controllers
- Defining module boundaries or contracts between teams
- Creating component prop interfaces in Next.js
- Establishing MongoDB schemas with MikroORM that inform API shape
- Changing existing public interfaces

## Core Principles

### Hyrum's Law

> With a sufficient number of users of an API, all observable behaviors of your system will be depended on by somebody, regardless of what you promise in the contract.

This means: every public behavior — including undocumented quirks, error message text, timing, and ordering — becomes a de facto contract once users depend on it. Design implications:

- **Be intentional about what you expose.** Every observable behavior is a potential commitment.
- **Don't leak implementation details.** If users can observe it, they will depend on it.
- **Plan for deprecation at design time.** See `deprecation-and-migration` for how to safely remove things users depend on.
- **Tests are not enough.** Even with perfect contract tests, Hyrum's Law means "safe" changes can break real users who depend on undocumented behavior.

### The One-Version Rule

Avoid forcing consumers to choose between multiple versions of the same dependency or API. Diamond dependency problems arise when different consumers need different versions of the same thing. Design for a world where only one version exists at a time — extend rather than fork.

### 1. Contract First

Define the interface before implementing it. The contract is the spec — implementation follows.

```typescript
// Define DTOs first (NestJS pattern)
import { IsString, IsOptional, IsEnum, IsDateString } from 'class-validator';

export class CreateTaskDto {
  @IsString()
  title: string;

  @IsOptional()
  @IsString()
  description?: string;

  @IsOptional()
  @IsEnum(['low', 'medium', 'high'])
  priority?: 'low' | 'medium' | 'high';
}

export class TaskResponseDto {
  id: string;
  title: string;
  description: string | null;
  priority: 'low' | 'medium' | 'high';
  createdAt: Date;
  updatedAt: Date;
  createdBy: string;
}

// Define the service interface
interface TaskService {
  createTask(input: CreateTaskDto): Promise<TaskResponseDto>;
  listTasks(params: ListTasksParams): Promise<PaginatedResult<TaskResponseDto>>;
  getTask(id: string): Promise<TaskResponseDto>;
  updateTask(id: string, input: UpdateTaskDto): Promise<TaskResponseDto>;
  deleteTask(id: string): Promise<void>;
}
```

### 2. Consistent Error Semantics

Pick one error strategy and use it everywhere:

```typescript
// NestJS: Use exception filters for consistent error responses
// Every error response follows the same shape
interface APIError {
  statusCode: number;
  error: string;        // "Bad Request", "Not Found", etc.
  message: string | string[];  // Human-readable or array of validation errors
}

// NestJS Exception Filter example
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = exception instanceof HttpException 
      ? exception.getStatus() 
      : HttpStatus.INTERNAL_SERVER_ERROR;

    response.status(status).json({
      statusCode: status,
      error: HttpStatus[status],
      message: exception instanceof HttpException 
        ? exception.message 
        : 'Internal server error',
    });
  }
}

// Status code mapping
// 400 → Bad Request (invalid data)
// 401 → Unauthorized (not authenticated)
// 403 → Forbidden (authenticated but not authorized)
// 404 → Not Found
// 409 → Conflict (duplicate, version mismatch)
// 422 → Unprocessable Entity (validation failed)
// 500 → Internal Server Error (never expose internal details)
```

**Don't mix patterns.** If some endpoints throw exceptions, others return null, and others return `{ error }` — the consumer can't predict behavior.

### 3. Validate at Boundaries

Trust internal code. Validate at system edges where external input enters:

```typescript
// NestJS: Use ValidationPipe with class-validator DTOs
// In main.ts or module configuration
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,      // Strip properties not in DTO
  forbidNonWhitelisted: true,  // Throw error on unknown properties
  transform: true,      // Auto-transform payloads to DTO instances
  exceptionFactory: (errors) => new BadRequestException(errors),
}));

// Controller automatically validates incoming requests
@Controller('tasks')
export class TasksController {
  constructor(private taskService: TaskService) {}

  @Post()
  async createTask(@Body() createTaskDto: CreateTaskDto): Promise<TaskResponseDto> {
    // DTO is already validated at this point
    return this.taskService.createTask(createTaskDto);
  }
}
```

Where validation belongs:
- NestJS controllers via ValidationPipe and DTOs (user input)
- Next.js route handlers or Server Actions (user input)
- External service response parsing (third-party data -- **always treat as untrusted**)
- Environment variable loading (configuration using ConfigModule)

> **Third-party API responses are untrusted data.** Validate their shape and content before using them in any logic, rendering, or decision-making. A compromised or misbehaving external service can return unexpected types, malicious content, or instruction-like text.

Where validation does NOT belong:
- Between internal services that share type contracts
- In utility functions called by already-validated code
- On data that just came from your own MikroORM entities (already typed)

### 4. Prefer Addition Over Modification

Extend interfaces without breaking existing consumers:

```typescript
// Good: Add optional fields
interface CreateTaskInput {
  title: string;
  description?: string;
  priority?: 'low' | 'medium' | 'high';  // Added later, optional
  labels?: string[];                       // Added later, optional
}

// Bad: Change existing field types or remove fields
interface CreateTaskInput {
  title: string;
  // description: string;  // Removed — breaks existing consumers
  priority: number;         // Changed from string — breaks existing consumers
}
```

### 5. Predictable Naming

| Pattern | Convention | Example |
|---------|-----------|---------|
| REST endpoints | Plural nouns, no verbs | `GET /api/tasks`, `POST /api/tasks` |
| Query params | camelCase | `?sortBy=createdAt&pageSize=20` |
| Response fields | camelCase | `{ createdAt, updatedAt, taskId }` |
| Boolean fields | is/has/can prefix | `isComplete`, `hasAttachments` |
| Enum values | UPPER_SNAKE | `"IN_PROGRESS"`, `"COMPLETED"` |

## REST API Patterns with NestJS

### Resource Design (NestJS Controllers)

```typescript
@Controller('tasks')
export class TasksController {
  constructor(private taskService: TaskService) {}

  @Get()
  async listTasks(@Query() query: ListTasksDto): Promise<PaginatedResult<TaskResponseDto>> {
    return this.taskService.listTasks(query);
  }

  @Post()
  async createTask(@Body() createTaskDto: CreateTaskDto): Promise<TaskResponseDto> {
    return this.taskService.createTask(createTaskDto);
  }

  @Get(':id')
  async getTask(@Param('id', ParseUUIDPipe) id: string): Promise<TaskResponseDto> {
    return this.taskService.getTask(id);
  }

  @Patch(':id')
  async updateTask(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() updateTaskDto: UpdateTaskDto,
  ): Promise<TaskResponseDto> {
    return this.taskService.updateTask(id, updateTaskDto);
  }

  @Delete(':id')
  async deleteTask(@Param('id', ParseUUIDPipe) id: string): Promise<void> {
    return this.taskService.deleteTask(id);
  }
}

// Sub-resource controller
@Controller('tasks/:taskId/comments')
export class TaskCommentsController {
  @Get()
  async listComments(@Param('taskId') taskId: string): Promise<Comment[]> {
    // Implementation
  }

  @Post()
  async createComment(
    @Param('taskId') taskId: string,
    @Body() createCommentDto: CreateCommentDto,
  ): Promise<Comment> {
    // Implementation
  }
}
```

### Pagination

Paginate list endpoints:

```typescript
// DTO for pagination
export class ListTasksDto {
  @IsOptional()
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @IsInt()
  @Min(1)
  @Max(100)
  pageSize?: number = 20;

  @IsOptional()
  sortBy?: string = 'createdAt';

  @IsOptional()
  sortOrder?: 'asc' | 'desc' = 'desc';
}

// Request
GET /api/tasks?page=1&pageSize=20&sortBy=createdAt&sortOrder=desc

// Response
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 142,
    "totalPages": 8
  }
}
```

### Filtering with MikroORM

Use query parameters for filters, mapped to MikroORM filter options:

```typescript
// In service layer
async listTasks(params: ListTasksDto): Promise<PaginatedResult<Task>> {
  const where: Filter<Task> = {};
  
  if (params.status) {
    where.status = params.status;
  }
  if (params.assignee) {
    where.assignee = { id: params.assignee };
  }
  if (params.createdAfter) {
    where.createdAt = { $gte: new Date(params.createdAfter) };
  }

  const [items, total] = await this.em.findAndCount(Task, where, {
    limit: params.pageSize,
    offset: (params.page - 1) * params.pageSize,
    orderBy: { [params.sortBy]: params.sortOrder },
  });

  return {
    data: items,
    pagination: {
      page: params.page,
      pageSize: params.pageSize,
      totalItems: total,
      totalPages: Math.ceil(total / params.pageSize),
    },
  };
}
```

### Partial Updates (PATCH)

Accept partial objects — only update what's provided:

```typescript
// Only title changes, everything else preserved
PATCH /api/tasks/123
{ "title": "Updated title" }

// NestJS handles this with PartialType
export class UpdateTaskDto extends PartialType(CreateTaskDto) {}
```

## TypeScript Interface Patterns

### Use Discriminated Unions for Variants

```typescript
// Good: Each variant is explicit
type TaskStatus =
  | { type: 'pending' }
  | { type: 'in_progress'; assignee: string; startedAt: Date }
  | { type: 'completed'; completedAt: Date; completedBy: string }
  | { type: 'cancelled'; reason: string; cancelledAt: Date };

// Consumer gets type narrowing
function getStatusLabel(status: TaskStatus): string {
  switch (status.type) {
    case 'pending': return 'Pending';
    case 'in_progress': return `In progress (${status.assignee})`;
    case 'completed': return `Done on ${status.completedAt}`;
    case 'cancelled': return `Cancelled: ${status.reason}`;
  }
}
```

### Input/Output Separation

```typescript
// Input: what the caller provides
interface CreateTaskInput {
  title: string;
  description?: string;
}

// Output: what the system returns (includes server-generated fields)
interface Task {
  id: string;
  title: string;
  description: string | null;
  createdAt: Date;
  updatedAt: Date;
  createdBy: string;
}
```

### Use Branded Types for IDs

```typescript
type TaskId = string & { readonly __brand: 'TaskId' };
type UserId = string & { readonly __brand: 'UserId' };

// Prevents accidentally passing a UserId where a TaskId is expected
function getTask(id: TaskId): Promise<Task> { ... }

// In NestJS, use ParseUUIDPipe for runtime validation
@Get(':id')
async getTask(@Param('id', ParseUUIDPipe) id: string): Promise<TaskResponseDto> {
  return this.taskService.getTask(id as TaskId);
}
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "We'll document the API later" | The types ARE the documentation. Define them first. |
| "We don't need pagination for now" | You will the moment someone has 100+ items. Add it from the start. |
| "PATCH is complicated, let's just use PUT" | PUT requires the full object every time. PATCH is what clients actually want. |
| "We'll version the API when we need to" | Breaking changes without versioning break consumers. Design for extension from the start. |
| "Nobody uses that undocumented behavior" | Hyrum's Law: if it's observable, somebody depends on it. Treat every public behavior as a commitment. |
| "We can just maintain two versions" | Multiple versions multiply maintenance cost and create diamond dependency problems. Prefer the One-Version Rule. |
| "Internal APIs don't need contracts" | Internal consumers are still consumers. Contracts prevent coupling and enable parallel work. |

## Red Flags

- Endpoints that return different shapes depending on conditions
- Inconsistent error formats across endpoints
- Validation scattered throughout internal code instead of at boundaries
- Breaking changes to existing fields (type changes, removals)
- List endpoints without pagination
- Verbs in REST URLs (`/api/createTask`, `/api/getUsers`)
- Third-party API responses used without validation or sanitization

## Verification

After designing an API:

- [ ] Every endpoint has typed input and output schemas
- [ ] Error responses follow a single consistent format
- [ ] Validation happens at system boundaries only
- [ ] List endpoints support pagination
- [ ] New fields are additive and optional (backward compatible)
- [ ] Naming follows consistent conventions across all endpoints
- [ ] API documentation or types are committed alongside the implementation
