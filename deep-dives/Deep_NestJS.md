# Deep NestJS — 40LPA Level
*Module System · DI Container · Lifecycle · Custom Decorators · Microservices · Testing*

---

# Chapter 1: How NestJS Works Under the Hood

## The Bootstrap Process

```
1. NestFactory.create(AppModule)
   ├── Scan AppModule and all imported modules (recursive)
   ├── Build a dependency graph of all providers
   ├── Instantiate providers in dependency order
   ├── Bind controllers to routes
   ├── Register middleware, guards, interceptors, pipes, filters
   └── Return the INestApplication instance

2. app.listen(3000)
   ├── Create Express/Fastify HTTP server
   ├── Register all routes from controllers
   ├── Start listening on the port
   └── Call onApplicationBootstrap lifecycle hook
```

## Module Resolution

```
@Module({
  imports: [DatabaseModule, AuthModule],
  controllers: [UsersController],
  providers: [UsersService, UsersRepository],
  exports: [UsersService],
})
export class UsersModule {}

What NestJS does:
  1. imports: make DatabaseModule and AuthModule's EXPORTED providers
     available inside UsersModule
  2. controllers: register UsersController's routes
  3. providers: instantiate UsersService and UsersRepository,
     make them available for injection WITHIN this module
  4. exports: make UsersService available to modules that IMPORT UsersModule

RULE: A provider is ONLY available within its own module
      unless it's exported AND the module is imported.
```

```
Module visibility:

AppModule
├── imports: [UsersModule, AuthModule]
│
├── UsersModule
│   ├── providers: [UsersService, UsersRepository]  ← private to UsersModule
│   ├── exports: [UsersService]                      ← visible to AppModule
│   └── imports: [AuthModule]                        ← AuthService available here
│
├── AuthModule
│   ├── providers: [AuthService, JwtService]
│   └── exports: [AuthService]

AppModule can use UsersService (exported by UsersModule).
AppModule CANNOT use UsersRepository (not exported).
UsersModule can use AuthService (imported AuthModule).
```

## DI Container — How Injection Works

```typescript
// NestJS uses a DI container (IoC container) based on metadata reflection.

// Step 1: You decorate a class with @Injectable()
@Injectable()
export class UsersService {
  constructor(
    private usersRepo: UsersRepository,    // NestJS reads this type via Reflect
    private authService: AuthService,
  ) {}
}

// Step 2: TypeScript's emitDecoratorMetadata compiler option adds metadata:
// Reflect.getMetadata('design:paramtypes', UsersService)
// → [UsersRepository, AuthService]

// Step 3: NestJS's container resolves the dependency graph:
//   UsersService needs → UsersRepository, AuthService
//   UsersRepository needs → DatabaseConnection
//   AuthService needs → JwtService, ConfigService
//
//   Resolution order: ConfigService → JwtService → AuthService
//                   → DatabaseConnection → UsersRepository
//                   → UsersService

// Step 4: All providers are SINGLETONS by default.
//   Same instance reused everywhere in the module.
```

### Provider Types

```typescript
// 1. Standard (class provider)
providers: [UsersService]
// Shorthand for: { provide: UsersService, useClass: UsersService }

// 2. Custom class (swap implementations)
{ provide: UsersService, useClass: MockUsersService }

// 3. Value provider
{ provide: 'API_KEY', useValue: process.env.API_KEY }
// Inject: @Inject('API_KEY') private apiKey: string

// 4. Factory provider (async initialization)
{
  provide: 'DATABASE',
  useFactory: async (config: ConfigService) => {
    const connection = await createConnection(config.get('DB_URL'));
    return connection;
  },
  inject: [ConfigService],    // dependencies for the factory function
}

// 5. Existing provider (alias)
{ provide: 'AliasedLogger', useExisting: LoggerService }
```

---

# Chapter 2: Lifecycle Hooks

```
Application lifecycle (in order):

1. onModuleInit()
   Called after the module's dependencies are resolved.
   Use: initialize connections, start background tasks.

2. onApplicationBootstrap()
   Called after ALL modules are initialized.
   Use: final setup, start consumers, register with service discovery.

--- Application is now running and handling requests ---

3. onModuleDestroy()
   Called when the module is being destroyed (app.close() or SIGTERM).
   Use: close database connections, stop consumers.

4. beforeApplicationShutdown(signal?)
   Called after onModuleDestroy, before the HTTP server closes.
   Use: stop accepting new connections, finish in-flight requests.

5. onApplicationShutdown(signal?)
   Called after the HTTP server is closed.
   Use: final cleanup, flush logs, deregister from service discovery.
```

```typescript
@Injectable()
export class KafkaService implements OnModuleInit, OnModuleDestroy {
  private consumer: Consumer;

  async onModuleInit() {
    this.consumer = kafka.consumer({ groupId: 'my-group' });
    await this.consumer.connect();
    await this.consumer.subscribe({ topic: 'events' });
    await this.consumer.run({
      eachMessage: async ({ message }) => {
        await this.handleMessage(message);
      },
    });
  }

  async onModuleDestroy() {
    await this.consumer.disconnect();
  }
}

// Enable shutdown hooks in main.ts
app.enableShutdownHooks();
```

---

# Chapter 3: Custom Decorators

```typescript
// Parameter decorator — extract data from request
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    return data ? user?.[data] : user;    // @CurrentUser('email') → user.email
  },
);

// Usage
@Get('me')
getMe(@CurrentUser() user: User) { return user; }

@Get('my-email')
getEmail(@CurrentUser('email') email: string) { return { email }; }
```

```typescript
// Method decorator — combine multiple decorators
import { applyDecorators, UseGuards, SetMetadata } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(JwtAuthGuard, RolesGuard),
    ApiBearerAuth(),                        // Swagger
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}

// Usage — one decorator instead of four
@Get('admin-panel')
@Auth(Role.ADMIN)
getAdminPanel() { ... }
```

```typescript
// Class decorator — apply to all routes in a controller
export function ApiController(prefix: string) {
  return applyDecorators(
    Controller(prefix),
    UseGuards(JwtAuthGuard),
    UseInterceptors(TransformInterceptor),
    UseFilters(HttpExceptionFilter),
    ApiTags(prefix),
  );
}

@ApiController('users')
export class UsersController { ... }
```

---

# Chapter 4: Execution Pipeline Deep Dive

```
Request arrives at NestJS:

  1. MIDDLEWARE (Express-level)
     ├── cors, helmet, morgan, body-parser
     ├── Runs on EVERY request (unless scoped to route)
     └── Has access to req, res, next (Express API)

  2. GUARDS (authorization)
     ├── Returns true/false: can this request proceed?
     ├── Has access to ExecutionContext (knows which handler will run)
     ├── Use for: auth, roles, rate limiting, feature flags
     └── If returns false → 403 Forbidden

  3. INTERCEPTORS (pre-processing)
     ├── Wraps the handler call with RxJS Observable
     ├── Can transform the request BEFORE the handler
     ├── Can transform the response AFTER the handler
     ├── Can measure timing, add headers, cache responses
     └── Use for: logging, caching, response transformation

  4. PIPES (validation/transformation)
     ├── Transform input data (e.g., string "5" → number 5)
     ├── Validate input data (e.g., check DTO with class-validator)
     ├── Throws 400 Bad Request if validation fails
     └── Use for: ValidationPipe, ParseIntPipe, ParseUUIDPipe

  5. HANDLER (your controller method)
     └── Runs your business logic

  6. INTERCEPTORS (post-processing)
     └── Transform the response (wrap in standard format, etc.)

  7. EXCEPTION FILTERS (error handling)
     ├── Catches any exception thrown during 2-6
     ├── Transforms it into an HTTP response
     └── Use for: consistent error format, logging, Sentry
```

```typescript
// Complete example: all layers working together

// Guard
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!roles) return true;
    const user = context.switchToHttp().getRequest().user;
    return roles.includes(user.role);
  }
}

// Interceptor
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const now = Date.now();
    const req = context.switchToHttp().getRequest();

    return next.handle().pipe(
      tap(() => console.log(`${req.method} ${req.url} — ${Date.now() - now}ms`)),
    );
  }
}

// Pipe
@Injectable()
export class ParseObjectIdPipe implements PipeTransform {
  transform(value: string): string {
    if (!Types.ObjectId.isValid(value)) {
      throw new BadRequestException(`Invalid ObjectId: ${value}`);
    }
    return value;
  }
}

// Exception filter
@Catch(MongoError)
export class MongoExceptionFilter implements ExceptionFilter {
  catch(exception: MongoError, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();

    if (exception.code === 11000) {
      response.status(409).json({ error: 'Duplicate entry' });
    } else {
      response.status(500).json({ error: 'Database error' });
    }
  }
}
```

---

# Chapter 5: Microservices Patterns

## Transport Layer Architecture

```typescript
// main.ts — Hybrid application (HTTP + Kafka)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Attach Kafka transport
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.KAFKA,
    options: {
      client: { brokers: ['kafka:9092'] },
      consumer: { groupId: 'api-group' },
    },
  });

  await app.startAllMicroservices();
  await app.listen(3000);
}

// Controller handles both HTTP and Kafka events
@Controller('orders')
export class OrdersController {
  // HTTP endpoint
  @Post()
  async createOrder(@Body() dto: CreateOrderDto) {
    const order = await this.ordersService.create(dto);
    this.kafkaClient.emit('order.created', order);    // fire event
    return order;
  }

  // Kafka event handler
  @EventPattern('payment.completed')
  async handlePaymentCompleted(@Payload() data: PaymentEvent) {
    await this.ordersService.markAsPaid(data.orderId);
  }

  // Kafka request-response
  @MessagePattern('order.get')
  async getOrder(@Payload() data: { orderId: string }) {
    return this.ordersService.findById(data.orderId);
  }
}
```

## CQRS Pattern in NestJS

```typescript
// Command (write operation)
export class CreateOrderCommand {
  constructor(
    public readonly userId: string,
    public readonly items: OrderItem[],
  ) {}
}

@CommandHandler(CreateOrderCommand)
export class CreateOrderHandler implements ICommandHandler<CreateOrderCommand> {
  async execute(command: CreateOrderCommand) {
    const order = await this.repo.create(command);
    this.eventBus.publish(new OrderCreatedEvent(order.id));
    return order;
  }
}

// Query (read operation)
export class GetOrdersQuery {
  constructor(public readonly userId: string) {}
}

@QueryHandler(GetOrdersQuery)
export class GetOrdersHandler implements IQueryHandler<GetOrdersQuery> {
  async execute(query: GetOrdersQuery) {
    return this.readRepo.findByUser(query.userId);
  }
}

// Controller
@Controller('orders')
export class OrdersController {
  constructor(
    private commandBus: CommandBus,
    private queryBus: QueryBus,
  ) {}

  @Post()
  create(@Body() dto: CreateOrderDto, @CurrentUser() user: User) {
    return this.commandBus.execute(new CreateOrderCommand(user.id, dto.items));
  }

  @Get()
  findAll(@CurrentUser() user: User) {
    return this.queryBus.execute(new GetOrdersQuery(user.id));
  }
}
```

---

# Chapter 6: Testing

```typescript
// Unit test — isolate the service, mock dependencies
describe('UsersService', () => {
  let service: UsersService;
  let mockRepo: jest.Mocked<UsersRepository>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: UsersRepository,
          useValue: {
            findById: jest.fn(),
            create: jest.fn(),
            save: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get(UsersService);
    mockRepo = module.get(UsersRepository);
  });

  it('should find a user by id', async () => {
    const user = { id: '1', name: 'Alice' };
    mockRepo.findById.mockResolvedValue(user);

    const result = await service.findById('1');

    expect(result).toEqual(user);
    expect(mockRepo.findById).toHaveBeenCalledWith('1');
  });

  it('should throw if user not found', async () => {
    mockRepo.findById.mockResolvedValue(null);
    await expect(service.findById('999')).rejects.toThrow(NotFoundException);
  });
});

// E2E test — test the full HTTP pipeline
describe('UsersController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = module.createNestApplication();
    app.useGlobalPipes(new ValidationPipe());
    await app.init();
  });

  it('POST /users should create a user', () => {
    return request(app.getHttpServer())
      .post('/users')
      .send({ name: 'Alice', email: 'alice@test.com' })
      .expect(201)
      .expect((res) => {
        expect(res.body.name).toBe('Alice');
        expect(res.body.id).toBeDefined();
      });
  });

  afterAll(() => app.close());
});
```

---

# Chapter 7: 🧩 40LPA Interview Questions

**Q: Explain NestJS's DI system. How does it resolve dependencies?**
A: NestJS uses constructor-based injection with TypeScript's `emitDecoratorMetadata`. When you mark a class `@Injectable()`, TypeScript emits metadata about its constructor parameters (their types). At startup, NestJS scans all modules, builds a dependency graph from this metadata, topologically sorts it, and instantiates providers in dependency order. Providers are singletons by default within their module scope. Circular dependencies are resolved using `forwardRef()`.

**Q: What's the difference between Guards and Middleware?**
A: Middleware is Express-level — it has access to `req, res, next` but doesn't know WHICH handler will run. Guards are NestJS-level — they receive `ExecutionContext` which knows the controller, handler, and metadata (like `@Roles()`). Middleware runs first, can modify the request, and can end the request. Guards run after middleware and return boolean (allow/deny). Use middleware for cross-cutting concerns (logging, CORS). Use guards for authorization logic.

**Q: How does the execution pipeline work?**
A: Request → Middleware → Guards → Interceptors (pre) → Pipes → Handler → Interceptors (post) → Exception Filters. Each layer has a specific purpose: middleware for request transformation, guards for authorization, interceptors for cross-cutting concerns (logging, caching, transformation), pipes for validation/transformation, and exception filters for error formatting. If any layer throws, control jumps to exception filters.

**Q: How would you implement multi-tenancy in NestJS?**
A: Three approaches. (1) Database-per-tenant: use a request-scoped provider that creates a DB connection based on `req.headers['x-tenant-id']`. (2) Schema-per-tenant: same DB, different schemas — a middleware sets the schema on the DB session. (3) Row-level: shared tables with a `tenantId` column — a global interceptor adds `tenantId` to all queries. Approach 3 is simplest for < 100 tenants. Approach 1 gives the best isolation. Implement tenant context via a request-scoped injectable: `@Injectable({ scope: Scope.REQUEST })`.

**Q: How do you handle circular dependencies?**
A: Use `forwardRef(() => ServiceName)` in both the `@Inject()` and the module's imports. But circular dependencies usually indicate a design problem — extract the shared logic into a third service that both depend on. In modules: `imports: [forwardRef(() => OtherModule)]`.
