# Especificación: `libs/shared/infrastructure/server`

Este módulo constituye la **Infraestructura de Servidor Compartida (Shared Server Infrastructure)**. Contiene las
implementaciones técnicas concretas de los buses (In-Memory y RabbitMQ), adaptadores de persistencia base (TypeORM
y MongoDB), serializadores de eventos de dominio, convertidores de Criteria, servicios de configuración tipados,
el sobre universal de respuesta **`ApiResponse`**, el sistema de **Logging estructurado contextual**, los **Guards de autorización de Nivel 1** y la integración con el contenedor de Inyección de Dependencias de NestJS.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Exclusividad de Servidor:** Este módulo utiliza APIs de Node.js, drivers de bases de datos y módulos de
  NestJS. **NUNCA** debe ser importado por aplicaciones de cliente (`apps/web`, `apps/mobile`, `apps/desktop`), ni por
  capas `ui/` o `infrastructure/client/`.
* **[ESTRICTO] Implementación de Puertos:** Toda clase aquí implementa interfaces definidas previamente en
  `@monorepo/shared/domain` o `@monorepo/shared/application`.
* **[ESTRICTO] Estandarización de Respuestas con `ApiResponse`:** Provee las utilidades para generar respuestas JSON homogéneas (`ApiResponse.success`, `ApiResponse.created`, `ApiResponse.error`).
* **[ESTRICTO] Formato de Log Estructurado:** Provee utilidades de logging que obligan al formato `[{ClassName}] {message}` y redactan información confidencial.
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:shared-infra-server", "scope:shared"]`.

---

## 2. Estructura de Directorios

```text
libs/shared/infrastructure/server/
├── src/
│   ├── http/
│   │   ├── api-response.ts                       # [ESTRICTO] Sobre universal de respuestas HTTP
│   │   └── guards/
│   │       └── action-permission.guard.ts        # [ESTRICTO] Guard de Nivel 1 para autorización por acción
│   │
│   ├── logging/
│   │   └── app-logger.service.ts                 # [ESTRICTO] Logger estructurado [{ClassName}] {message}
│   │
│   ├── bus/
│   │   ├── command/
│   │   │   ├── in-memory-command-bus.ts          # [ESTRICTO] Ejecución de comandos en memoria
│   │   │   └── command-handlers-information.ts   # [ESTRICTO] Registro y resolución de CommandHandlers
│   │   ├── query/
│   │   │   ├── in-memory-query-bus.ts            # [ESTRICTO] Ejecución de consultas en memoria
│   │   │   └── query-handlers-information.ts     # [ESTRICTO] Registro y resolución de QueryHandlers
│   │   ├── event/
│   │   │   ├── in-memory/
│   │   │   │   └── in-memory-async-event-bus.ts  # [ESTRICTO] EventBus local asíncrono
│   │   │   ├── rabbitmq/
│   │   │   │   └── rabbitmq-event-bus.ts         # [OPCIONAL/PROD] Publicador en RabbitMQ
│   │   │   └── serializer/
│   │   │       ├── domain-event-json-serializer.ts # [ESTRICTO] Serializa DomainEvent a JSON
│   │   │       └── domain-event-deserializer.ts   # [ESTRICTO] Reconstruye DomainEvent desde JSON
│   │
│   ├── persistence/
│   │   ├── typeorm/
│   │   │   └── typeorm-criteria-converter.ts     # [ESTRICTO] Convierte Criteria a TypeORM FindOptions
│   │   └── mongo/
│   │       └── mongo-criteria-converter.ts       # [OPCIONAL] Convierte Criteria a MongoDB Query Object
│   │
│   ├── config/
│   │   ├── env-config.interface.ts               # [ESTRICTO] Tipado estricto de variables de entorno
│   │   └── app-config.service.ts                 # [ESTRICTO] Servicio para acceso seguro a variables
│   │
│   ├── nest/
│   │   ├── tokens.ts                             # [ESTRICTO] Injection tokens para DI de NestJS
│   │   └── shared-infrastructure.module.ts       # [ESTRICTO] Módulo global de infraestructura NestJS
│   │
│   └── index.ts                                  # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `http/api-response.ts` (Sobre Homogéneo de API)

```typescript
// libs/shared/infrastructure/server/src/http/api-response.ts
export interface PaginationMeta {
  total: number;
  count: number;
  per_page: number;
  current_page: number;
  total_pages: number;
}

export interface ApiResponseMeta {
  code: number;
  timestamp: string;
  version: string;
  pagination?: PaginationMeta;
}

export interface ApiSuccessResponse<T> {
  success: true;
  message: string;
  data: T;
  meta: ApiResponseMeta;
}

export interface ApiErrorResponse {
  success: false;
  message: string;
  meta: ApiResponseMeta;
  errors?: Record<string, Array<string>>;
}

export class ApiResponse {
  private static readonly VERSION = '1.0';

  static success<T>(data: T, message = 'Operación completada exitosamente', code = 200): ApiSuccessResponse<T> {
    return {
      success: true,
      message,
      data,
      meta: {
        code,
        timestamp: new Date().toISOString(),
        version: this.VERSION,
      },
    };
  }

  static created<T>(data: T, message = 'Recurso creado exitosamente'): ApiSuccessResponse<T> {
    return this.success(data, message, 201);
  }

  static buildPaginationMeta(total: number, limit: number, offset: number): PaginationMeta {
    return {
      total,
      count: Math.min(limit, total - offset),
      per_page: limit,
      current_page: Math.floor(offset / limit) + 1,
      total_pages: Math.ceil(total / limit),
    };
  }

  static paginated<T>(data: Array<T>, pagination: PaginationMeta, message = 'Listado recuperado exitosamente'): ApiSuccessResponse<Array<T>> {
    return {
      success: true,
      message,
      data,
      meta: {
        code: 200,
        timestamp: new Date().toISOString(),
        version: this.VERSION,
        pagination,
      },
    };
  }

  static error(message: string, code = 400, errors?: Record<string, Array<string>>): ApiErrorResponse {
    return {
      success: false,
      message,
      meta: {
        code,
        timestamp: new Date().toISOString(),
        version: this.VERSION,
      },
      ...(errors ? { errors } : {}),
    };
  }
}
```

#### `require-permission.decorator.ts` y `action-permission.guard.ts`
```typescript
// libs/shared/infrastructure/server/src/http/decorators/require-permission.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const PERMISSION_KEY = 'required_permission';
export const RequirePermission = <T extends string>(permission: T) =>
  SetMetadata(PERMISSION_KEY, permission);

// libs/shared/infrastructure/server/src/http/guards/action-permission.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { PERMISSION_KEY } from '../decorators/require-permission.decorator';

@Injectable()
export class ActionPermissionGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermission = this.reflector.getAllAndOverride<string>(
      PERMISSION_KEY,
      [context.getHandler(), context.getClass()],
    );
    if (!requiredPermission) return true;

    const { user } = context.switchToHttp().getRequest();
    if (!user) {
      throw new ForbiddenException('Authentication required');
    }
    if (!user.permissions?.includes(requiredPermission)) {
      throw new ForbiddenException(`Missing permission: ${requiredPermission}`);
    }
    return true;
  }
}
```

---

### 3.2 Bloque: `logging/app-logger.service.ts` (Formato `[{ClassName}]`)

> **Nota:** La documentación arquitectónica original especificaba el formato `[{ClassName}][L{line}]`, pero el requerimiento de líneas `[L{line}]` se simplificó, ya que los números de línea manuales se desactualizan fácilmente y generan ruido innecesario en refactorizaciones.

```typescript
// libs/shared/infrastructure/server/src/logging/app-logger.service.ts
import { Injectable, LoggerService } from '@nestjs/common';

@Injectable()
export class AppLoggerService implements LoggerService {
  private static readonly REDACTED_KEYS = ['password', 'token', 'authorization', 'secret', 'credit_card'];

  constructor(private readonly context: string) {}

  log(message: string, data?: Record<string, unknown>): void {
    console.log(`[${this.context}] ${message}`, data ? this.redact(data) : '');
  }

  error(message: string, error?: unknown): void {
    console.error(`[${this.context}] ${message}`, error instanceof Error ? error.stack : error);
  }

  warn(message: string, data?: Record<string, unknown>): void {
    console.warn(`[${this.context}] ${message}`, data ? this.redact(data) : '');
  }

  private redact(data: Record<string, unknown>): Record<string, unknown> {
    const redacted = { ...data };
    for (const key of Object.keys(redacted)) {
      if (AppLoggerService.REDACTED_KEYS.some(k => key.toLowerCase().includes(k))) {
        redacted[key] = '[REDACTED]';
      }
    }
    return redacted;
  }
}
```

---

### 3.3 Bloque: `bus/` (CommandBus, QueryBus, EventBus)

```typescript
// libs/shared/infrastructure/server/src/bus/command/command-handlers-information.ts
import { CommandHandler } from '@monorepo/shared/application';

export class CommandHandlersInformation {
  private readonly handlers: Map<string, CommandHandler<any>>;

  constructor(handlers: CommandHandler<any>[]) {
    this.handlers = new Map();
    for (const handler of handlers) {
      const commandClass = handler.subscribedTo();
      this.handlers.set(commandClass.name, handler);
    }
  }

  search(commandName: string): CommandHandler<any> {
    const handler = this.handlers.get(commandName);
    if (!handler) {
      throw new Error(`No handler registered for command: ${commandName}`);
    }
    return handler;
  }
}

// libs/shared/infrastructure/server/src/bus/command/in-memory-command-bus.ts
import { Injectable } from '@nestjs/common';
import { CommandBus } from '@monorepo/shared/domain';
import { CommandHandlersInformation } from './command-handlers-information';

@Injectable()
export class InMemoryCommandBus implements CommandBus {
  constructor(private readonly info: CommandHandlersInformation) {}

  async dispatch<C>(command: C): Promise<void> {
    const commandName = (command as any).constructor.name;
    const handler = this.info.search(commandName);
    await handler.handle(command as any);
  }
}

// libs/shared/infrastructure/server/src/bus/query/query-handlers-information.ts
import { QueryHandler } from '@monorepo/shared/application';

export class QueryHandlersInformation {
  private readonly handlers: Map<string, QueryHandler<any, any>>;

  constructor(handlers: QueryHandler<any, any>[]) {
    this.handlers = new Map();
    for (const handler of handlers) {
      const queryClass = handler.subscribedTo();
      this.handlers.set(queryClass.name, handler);
    }
  }

  search(queryName: string): QueryHandler<any, any> {
    const handler = this.handlers.get(queryName);
    if (!handler) throw new Error(`No handler for query: ${queryName}`);
    return handler;
  }
}

// libs/shared/infrastructure/server/src/bus/query/in-memory-query-bus.ts
import { Injectable } from '@nestjs/common';
import { QueryBus } from '@monorepo/shared/domain';
import { QueryHandlersInformation } from './query-handlers-information';

@Injectable()
export class InMemoryQueryBus implements QueryBus {
  constructor(private readonly info: QueryHandlersInformation) {}

  async ask<R>(query: unknown): Promise<R> {
    const queryName = (query as any).constructor.name;
    const handler = this.info.search(queryName);
    return handler.handle(query as any) as Promise<R>;
  }
}
```

---

### 3.4 Bloque: Registro de Handlers (DI en NestJS)

Para que los `CommandHandlers` y `QueryHandlers` sean detectados y registrados correctamente en el bus, se utiliza el patrón Factory de NestJS inyectando explícitamente los handlers.

```typescript
// libs/{context}/infrastructure/server/src/nest/context-server.module.ts
import { Module } from '@nestjs/common';
import { CommandHandlersInformation } from '@monorepo/shared/infrastructure/server';
import { CommandHandler } from '@monorepo/shared/application';
import { RegisterUserCommandHandler } from '../application/slices/register-user.command-handler';

@Module({
  providers: [
    RegisterUserCommandHandler,
    {
      provide: CommandHandlersInformation,
      useFactory: (...handlers: CommandHandler<any>[]) => new CommandHandlersInformation(handlers),
      // Es vital listar cada handler en el array inject
      inject: [RegisterUserCommandHandler],
    },
  ],
})
export class ContextServerModule {}
```

---

### 3.5 Bloque: `persistence/typeorm/`

```typescript
// libs/shared/infrastructure/server/src/persistence/typeorm/typeorm-criteria-converter.ts
import { FindManyOptions, ILike, LessThan, MoreThan, Not, Equal } from 'typeorm';
import { Criteria, Filter, Operator, Order } from '@monorepo/shared/domain';

export class TypeOrmCriteriaConverter<T> {
  convert(criteria: Criteria): FindManyOptions<T> {
    const options: FindManyOptions<T> = {};

    if (criteria.hasFilters()) {
      options.where = this.buildWhere(criteria.filters.filters);
    }
    if (criteria.hasOrder()) {
      options.order = this.buildOrder(criteria.order) as any;
    }
    if (criteria.limit) {
      options.take = criteria.limit;
    }
    if (criteria.offset) {
      options.skip = criteria.offset;
    }

    return options;
  }

  private buildWhere(filters: ReadonlyArray<Filter>): Record<string, unknown> {
    const where: Record<string, unknown> = {};
    for (const filter of filters) {
      where[filter.field.value] = this.mapOperator(filter);
    }
    return where;
  }

  private mapOperator(filter: Filter): unknown {
    const value = filter.value.value;
    switch (filter.operator.value) {
      case Operator.EQUAL: return Equal(value);
      case Operator.NOT_EQUAL: return Not(Equal(value));
      case Operator.CONTAINS: return ILike(`%${value}%`);
      case Operator.NOT_CONTAINS: return Not(ILike(`%${value}%`));
      case Operator.GT: return MoreThan(value);
      case Operator.LT: return LessThan(value);
      default: return Equal(value);
    }
  }

  private buildOrder(order: Order): Record<string, 'ASC' | 'DESC'> {
    if (!order.hasOrder()) return {};
    return { [order.orderBy.value]: order.orderType.value.toUpperCase() as 'ASC' | 'DESC' };
  }
}
```

---

### 3.6 Bloque: `nest/` (Módulos de NestJS y Tokens)

```typescript
// libs/shared/infrastructure/server/src/nest/tokens.ts
export const SHARED_TOKENS = {
  COMMAND_BUS: Symbol('CommandBus'),
  QUERY_BUS: Symbol('QueryBus'),
  EVENT_BUS: Symbol('EventBus'),
  LOGGER: Symbol('AppLoggerService'),
} as const;
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/shared/infrastructure/server/src/index.ts
export * from './http/api-response';
export * from './http/decorators/require-permission.decorator';
export * from './http/guards/action-permission.guard';

export * from './logging/app-logger.service';

export * from './bus/command/in-memory-command-bus';
export * from './bus/command/command-handlers-information';
export * from './bus/query/in-memory-query-bus';
export * from './bus/query/query-handlers-information';
export * from './bus/event/in-memory/in-memory-async-event-bus';
export * from './bus/event/serializer/domain-event-json-serializer';
export * from './bus/event/serializer/domain-event-deserializer';

export * from './persistence/typeorm/typeorm-criteria-converter';

export * from './config/env-config.interface';
export * from './config/app-config.service';

export * from './nest/tokens';
export * from './nest/shared-infrastructure.module';
```
