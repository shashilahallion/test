# NestJS Best Practices

A reference guide for building production-ready NestJS applications. This document covers architecture patterns, code organisation, security, performance, and testing conventions used in this project.

---

## Table of Contents

1. [Project Structure](#project-structure)
2. [Modules](#modules)
3. [Controllers](#controllers)
4. [Services & Business Logic](#services--business-logic)
5. [DTOs & Validation](#dtos--validation)
6. [Database & Repositories](#database--repositories)
7. [Authentication & Authorization](#authentication--authorization)
8. [Error Handling](#error-handling)
9. [Configuration Management](#configuration-management)
10. [Logging](#logging)
11. [Testing](#testing)
12. [Performance](#performance)
13. [Security](#security)
14. [API Documentation](#api-documentation)

---

## Project Structure

Organise code by **feature modules** rather than by type. Each feature owns its controller, service, DTOs, entities, and tests.

```
src/
├── app.module.ts
├── main.ts
├── common/                   # Shared utilities, guards, pipes, decorators
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   └── pipes/
├── config/                   # Configuration modules
└── modules/
    ├── users/
    │   ├── dto/
    │   │   ├── create-user.dto.ts
    │   │   └── update-user.dto.ts
    │   ├── entities/
    │   │   └── user.entity.ts
    │   ├── users.controller.ts
    │   ├── users.module.ts
    │   ├── users.service.ts
    │   └── users.service.spec.ts
    └── auth/
        ├── strategies/
        ├── guards/
        ├── auth.controller.ts
        ├── auth.module.ts
        └── auth.service.ts
```

---

## Modules

- Keep modules **small and cohesive** — one module per domain feature.
- Use `forRoot()` / `forRootAsync()` for globally shared modules (database, config).
- Mark truly global providers with `@Global()` sparingly; prefer explicit imports.
- Avoid circular dependencies — extract shared logic into a `common` or `shared` module.

```typescript
// users.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],   // Only export what other modules need
})
export class UsersModule {}
```

---

## Controllers

- Controllers should be **thin** — delegate all business logic to services.
- Use route-level decorators (`@Get`, `@Post`, etc.) and keep route paths lowercase and kebab-case.
- Always specify a consistent API prefix (e.g. `/api/v1`).
- Use `@HttpCode()` to return correct HTTP status codes.

```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get(':id')
  @HttpCode(HttpStatus.OK)
  findOne(@Param('id', ParseUUIDPipe) id: string): Promise<UserResponseDto> {
    return this.usersService.findOne(id);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createUserDto: CreateUserDto): Promise<UserResponseDto> {
    return this.usersService.create(createUserDto);
  }
}
```

---

## Services & Business Logic

- All business logic lives in services.
- Services should depend on **abstractions** (interfaces / repository tokens), not concrete implementations — aids testability.
- Return plain objects or typed DTOs from services; never expose raw database entities to controllers.
- Keep services **single-responsibility**: split large services into focused sub-services.

```typescript
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  async findOne(id: string): Promise<UserResponseDto> {
    const user = await this.userRepository.findOneBy({ id });
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return plainToInstance(UserResponseDto, user, { excludeExtraneousValues: true });
  }
}
```

---

## DTOs & Validation

- Use **class-validator** decorators on all DTOs.
- Enable `ValidationPipe` globally with `whitelist: true` and `forbidNonWhitelisted: true` to strip unknown properties.
- Use **class-transformer** with `@Expose()` on response DTOs to prevent over-fetching.
- Separate request DTOs (Create/Update) from response DTOs.

```typescript
// main.ts – global validation
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
    transformOptions: { enableImplicitConversion: true },
  }),
);

// create-user.dto.ts
export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsString()
  @IsOptional()
  name?: string;
}
```

---

## Database & Repositories

- Use the **Repository pattern** (TypeORM / Mongoose) and inject repositories through the DI container.
- Never write raw queries inside controllers.
- For complex queries, create custom repository classes that extend the base repository.
- Use **database transactions** for multi-step write operations.
- Add indexes on frequently queried fields.

```typescript
// Example: custom repository method with transaction
async transferBalance(fromId: string, toId: string, amount: number): Promise<void> {
  await this.dataSource.transaction(async (manager) => {
    await manager.decrement(Account, { id: fromId }, 'balance', amount);
    await manager.increment(Account, { id: toId }, 'balance', amount);
  });
}
```

---

## Authentication & Authorization

- Use **JWT** (Passport `passport-jwt`) for stateless API authentication.
- Store only a minimal payload in the token (user ID, roles) — never store passwords.
- Use `@UseGuards(JwtAuthGuard)` at the controller or route level.
- Implement **RBAC** with custom decorators and a `RolesGuard`.
- Rotate secrets regularly; set short token expiry and use refresh tokens.

```typescript
// roles.decorator.ts
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);

// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) return true;
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles.includes(role));
  }
}
```

---

## Error Handling

- Use NestJS **built-in HTTP exceptions** (`NotFoundException`, `BadRequestException`, etc.) in services.
- Create a global `HttpExceptionFilter` to standardise error response shapes.
- Never leak stack traces or internal details to clients in production.
- Log all unhandled exceptions centrally.

```typescript
// http-exception.filter.ts
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = exception.getStatus();
    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      message: exception.message,
    });
  }
}
```

---

## Configuration Management

- Use `@nestjs/config` with a **validation schema** (Joi or class-validator) to fail fast on missing env vars.
- Never hardcode secrets — load from environment variables or a secrets manager (AWS Secrets Manager, HashiCorp Vault).
- Define typed config interfaces; inject `ConfigService` rather than accessing `process.env` directly.

```typescript
// app.module.ts
ConfigModule.forRoot({
  isGlobal: true,
  validationSchema: Joi.object({
    NODE_ENV: Joi.string().valid('development', 'production', 'test').required(),
    PORT: Joi.number().default(3000),
    DATABASE_URL: Joi.string().required(),
    JWT_SECRET: Joi.string().min(32).required(),
  }),
})
```

---

## Logging

- Use NestJS's built-in `Logger` (or swap it for **Pino** / **Winston** in production).
- Attach a **correlation / request ID** to every log entry for distributed tracing.
- Log at the appropriate level: `verbose` for debug details, `log` for normal flow, `warn` for degraded states, `error` for exceptions.
- Never log sensitive data (passwords, tokens, PII).

```typescript
@Injectable()
export class UsersService {
  private readonly logger = new Logger(UsersService.name);

  async findOne(id: string): Promise<User> {
    this.logger.log(`Fetching user ${id}`);
    const user = await this.userRepository.findOneBy({ id });
    if (!user) {
      this.logger.warn(`User ${id} not found`);
      throw new NotFoundException(`User ${id} not found`);
    }
    return user;
  }
}
```

---

## Testing

- Write **unit tests** for every service method using Jest; mock all dependencies with `jest.fn()`.
- Write **integration tests** for controllers using `@nestjs/testing` `TestingModule`.
- Write **E2E tests** for critical user journeys using `supertest`.
- Aim for > 80 % coverage on business logic; 100 % is not always practical.
- Keep tests **isolated** — each test resets mocks and database state.

```typescript
// users.service.spec.ts
describe('UsersService', () => {
  let service: UsersService;
  let repo: jest.Mocked<Repository<User>>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: getRepositoryToken(User), useValue: { findOneBy: jest.fn() } },
      ],
    }).compile();

    service = module.get(UsersService);
    repo = module.get(getRepositoryToken(User));
  });

  it('throws NotFoundException when user does not exist', async () => {
    repo.findOneBy.mockResolvedValue(null);
    await expect(service.findOne('unknown-id')).rejects.toThrow(NotFoundException);
  });
});
```

---

## Performance

- Use **caching** (`@nestjs/cache-manager`, Redis) for frequently read, rarely changing data.
- Enable **compression** middleware (`compression` npm package) for HTTP responses.
- Use **BullMQ** or similar for long-running / background tasks — never block the event loop.
- Enable **HTTP/2** and keep-alive connections in production.
- Use **pagination** for all list endpoints; avoid returning unbounded datasets.

```typescript
// Paginated response pattern
async findAll(page = 1, limit = 20): Promise<PaginatedResponseDto<User>> {
  const [items, total] = await this.userRepository.findAndCount({
    skip: (page - 1) * limit,
    take: limit,
  });
  return { items, total, page, limit, totalPages: Math.ceil(total / limit) };
}
```

---

## Security

- Enable **Helmet** to set secure HTTP headers.
- Enable **CORS** with an explicit allow-list of origins.
- Apply **rate limiting** (`@nestjs/throttler`) to all public endpoints.
- Hash passwords with **bcrypt** (saltRounds ≥ 12); never store plaintext passwords.
- Validate and sanitise all user input (DTOs + `ValidationPipe`).
- Keep dependencies up to date; run `npm audit` regularly.

```typescript
// main.ts – security bootstrap
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.use(helmet());
  app.enableCors({ origin: process.env.ALLOWED_ORIGINS?.split(',') });
  app.useGlobalGuards(new ThrottlerGuard());
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }));
  app.useGlobalFilters(new HttpExceptionFilter());

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

---

## API Documentation

- Use **@nestjs/swagger** to auto-generate OpenAPI docs.
- Decorate all DTOs with `@ApiProperty()` and controllers with `@ApiTags()`, `@ApiOperation()`, and `@ApiResponse()`.
- Version your API from day one (`app.setGlobalPrefix('api/v1')`).
- Keep documentation in sync with code — do not maintain separate Postman collections as the source of truth.

```typescript
// main.ts – Swagger setup
const config = new DocumentBuilder()
  .setTitle('Command Center API')
  .setDescription('WhatsApp conversation management API')
  .setVersion('1.0')
  .addBearerAuth()
  .build();

const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('api/docs', app, document);
```

---

## Quick Reference Checklist

| Area | Key Action |
|------|-----------|
| Modules | One module per feature; export only what is needed |
| Controllers | Thin; delegate to services; use correct HTTP codes |
| Services | Single responsibility; return DTOs not entities |
| Validation | `ValidationPipe` globally with `whitelist: true` |
| Auth | JWT + Passport; short-lived tokens; RBAC guards |
| Errors | Global exception filter; no stack traces in production |
| Config | `@nestjs/config` with Joi validation; no hardcoded secrets |
| Logging | Structured logs; correlation ID; no sensitive data |
| Testing | Unit + integration + E2E; > 80 % coverage on logic |
| Security | Helmet + CORS + rate limiting + bcrypt |
| Docs | Swagger auto-generated from decorators |

---

> **Further reading**: [NestJS Official Docs](https://docs.nestjs.com) · [12-Factor App](https://12factor.net) · [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
