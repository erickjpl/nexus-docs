# Especificación: `libs/{bounded_context}/infrastructure/server`

Este módulo constituye la **Capa de Infraestructura de Servidor del Bounded Context**. Contiene las implementaciones
técnicas concretas que requieren el entorno de ejecución Node.js y el framework NestJS: persistencia en base de datos
(TypeORM/Mongo), esquemas de tablas/documentos, mappers de datos, controladores HTTP REST protegidos con la regla **"1 Acción = 1 Permiso Dedicado"**, consumidores de mensajería (RabbitMQ) y la configuración de inyección de dependencias de NestJS mediante **Factory Providers**.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Exclusividad de Backend:** Prohibido importar este paquete en aplicaciones frontend (`apps/web`,
  `apps/mobile`, `apps/desktop`) o librerías de UI. Contiene dependencias de Node.js, TypeORM y NestJS.
* **[ESTRICTO] Controladores de Acción Única y Cero `try/catch` Manual:** Los controladores no contienen bloques `try/catch`. Delegan directamente al CommandBus / QueryBus o al Servicio de Aplicación, y envuelven la salida en el sobre estandarizado `ApiResponse`.
* **[ESTRICTO] Autorización de Nivel 1 (1 Acción = 1 Permiso vía Enum):** Todo controlador de acción sensible declara explícitamente `@RequirePermission(PermissionEnum.ACTION)` y `@UseGuards(ActionPermissionGuard)`.
* **[ESTRICTO] Mappers Bidireccionales:** Ninguna consulta SQL/ORM debe devolver directamente esquemas de persistencia
  a la capa de aplicación. Todo dato se transforma al Agregado mediante `Aggregate.fromPrimitives()`, y todo agregado
  que entra se extrae con `aggregate.toPrimitives()`.
* **[ESTRICTO] Registro Limpio en NestJS:** Los casos de uso de `application/` se registran en los módulos de NestJS
  mediante `useFactory` y tokens de inyección, garantizando que el núcleo permanezca libre de `@Injectable()`.
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:infra-server", "scope:{context}"]`.

---

## 2. Estructura de Directorios

```text
libs/{bounded_context}/infrastructure/server/
├── src/
│   ├── persistence/
│   │   ├── typeorm/
│   │   │   ├── {aggregate}.entity-schema.ts      # [ESTRICTO/ORM] Esquema de tabla TypeORM (EntitySchema puro)
│   │   │   ├── {aggregate}.mapper.ts             # [ESTRICTO] Mapper bidireccional (DB Schema <-> Dominio)
│   │   │   └── typeorm-{aggregate}.repository.ts # [ESTRICTO] Implementación del puerto del repositorio
│   │   └── mongo/
│   │       ├── {aggregate}.mongo-schema.ts       # [OPCIONAL] Tipado de documento MongoDB
│   │       └── mongo-{aggregate}.repository.ts   # [OPCIONAL] Implementación con driver nativo de Mongo
│   │
│   ├── http/
│   │   ├── dto/
│   │   │   └── register-{aggregate}-http.dto.ts  # [ESTRICTO] DTO de entrada HTTP con class-validator
│   │   ├── {aggregate}-put.controller.ts         # [ESTRICTO] Endpoint HTTP para registro/actualización
│   │   └── {aggregates}-get.controller.ts        # [ESTRICTO] Endpoint HTTP para búsquedas con Criteria
│   │
│   ├── messaging/
│   │   └── rabbitmq-{event}.consumer.ts          # [OPCIONAL] Consumidor de eventos de RabbitMQ
│   │
│   ├── nest/
│   │   ├── {context}.tokens.ts                   # [ESTRICTO] Tokens de inyección para el contexto
│   │   └── {context}-server.module.ts            # [ESTRICTO] Módulo de NestJS que ensambla el contexto
│   │
│   └── index.ts                                  # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `persistence/typeorm/`

#### `user.entity-schema.ts` (Esquema desacoplado)
```typescript
// libs/users/infrastructure/server/src/persistence/typeorm/user.entity-schema.ts
import { EntitySchema } from 'typeorm';

export interface UserDatabaseSchema {
  id: string;
  name: string;
  email: string;
  created_at: Date;
  updated_at: Date;
}

export const UserEntitySchema = new EntitySchema<UserDatabaseSchema>({
  name: 'User',
  tableName: 'users',
  columns: {
    id: {
      type: 'uuid',
      primary: true,
    },
    name: {
      type: 'varchar',
      length: 50,
    },
    email: {
      type: 'varchar',
      length: 100,
      unique: true,
    },
    created_at: {
      type: 'timestamp',
      createDate: true,
    },
    updated_at: {
      type: 'timestamp',
      updateDate: true,
    },
  },
});
```

#### `user.mapper.ts`
```typescript
// libs/users/infrastructure/server/src/persistence/typeorm/user.mapper.ts
import { User } from '@monorepo/users/domain';
import { UserDatabaseSchema } from './user.entity-schema';

export class UserMapper {
  static toDomain(raw: UserDatabaseSchema): User {
    return User.fromPrimitives({
      id: raw.id,
      name: raw.name,
      email: raw.email,
    });
  }

  static toPersistence(user: User): UserDatabaseSchema {
    const primitives = user.toPrimitives();
    return {
      id: primitives.id,
      name: primitives.name,
      email: primitives.email,
      created_at: new Date(),
      updated_at: new Date(),
    };
  }
}
```

#### `typeorm-user.repository.ts`
```typescript
// libs/users/infrastructure/server/src/persistence/typeorm/typeorm-user.repository.ts
import { Repository, DataSource, QueryFailedError } from 'typeorm';
import { Criteria, DomainConflictError } from '@monorepo/shared/domain';
import { TypeOrmCriteriaConverter } from '@monorepo/shared/infrastructure/server';
import { User, UserId, UserRepository } from '@monorepo/users/domain';
import { UserDatabaseSchema, UserEntitySchema } from './user.entity-schema';
import { UserMapper } from './user.mapper';

export class TypeOrmUserRepository implements UserRepository {
  private repository: Repository<UserDatabaseSchema>;
  private criteriaConverter = new TypeOrmCriteriaConverter();

  constructor(dataSource: DataSource) {
    this.repository = dataSource.getRepository<UserDatabaseSchema>(UserEntitySchema);
  }

  async save(user: User): Promise<void> {
    const raw = UserMapper.toPersistence(user);
    try {
      await this.repository.save(raw);
    } catch (error) {
      if (error instanceof QueryFailedError && error.message.includes('unique constraint')) {
        throw new DomainConflictError(`El usuario con el email ya existe`);
      }
      throw error;
    }
  }

  async search(id: UserId): Promise<User | null> {
    const raw = await this.repository.findOne({ where: { id: id.value } });
    return raw ? UserMapper.toDomain(raw) : null;
  }

  async searchAll(): Promise<Array<User>> {
    const raws = await this.repository.find();
    return raws.map((raw) => UserMapper.toDomain(raw));
  }

  async matching(criteria: Criteria): Promise<{ items: Array<User>; total: number }> {
    const options = this.criteriaConverter.convert<UserDatabaseSchema>(criteria);
    const [raws, total] = await this.repository.findAndCount(options);
    return { items: raws.map((raw) => UserMapper.toDomain(raw)), total };
  }
}
```

---

### 3.2 Bloque: `http/` (Controladores Protegidos y Limpios)

#### `register-user-http.dto.ts`
```typescript
// libs/users/infrastructure/server/src/http/dto/register-user-http.dto.ts
import { IsEmail, IsNotEmpty, IsString, IsUUID } from 'class-validator';

export class RegisterUserHttpDto {
  @IsUUID('4')
  readonly id!: string;

  @IsString()
  @IsNotEmpty()
  readonly name!: string;

  @IsEmail()
  readonly email!: string;
}
```

#### `user-put.controller.ts`
```typescript
// libs/users/infrastructure/server/src/http/user-put.controller.ts
import { Controller, Put, Param, Body, HttpCode, HttpStatus, Inject, UseGuards } from '@nestjs/common';
import { CommandBus } from '@monorepo/shared/domain';
import { Command } from '@monorepo/shared/application';
import { SHARED_TOKENS, ApiResponse } from '@monorepo/shared/infrastructure/server';
import { ActionPermissionGuard, RequirePermission } from '@monorepo/shared/infrastructure/server';
import { UserPermissionEnum } from '@monorepo/users/domain';
import { RegisterUserCommand } from '@monorepo/users/application';
import { RegisterUserHttpDto } from './dto/register-user-http.dto';

@Controller('users')
export class UserPutController {
  constructor(
    @Inject(SHARED_TOKENS.COMMAND_BUS)
    private readonly commandBus: CommandBus
  ) {}

  @Put(':id')
  @HttpCode(HttpStatus.CREATED)
  @UseGuards(ActionPermissionGuard)
  @RequirePermission(UserPermissionEnum.USER_CREATE)
  async run(@Param('id') id: string, @Body() body: RegisterUserHttpDto) {
    const command = new RegisterUserCommand({
      id,
      name: body.name,
      email: body.email,
    });

    await this.commandBus.dispatch(command);

    return ApiResponse.created({
      id,
      name: body.name,
      email: body.email,
    }, 'Usuario registrado exitosamente');
  }
}
```

#### `users-get.controller.ts`
```typescript
// libs/users/infrastructure/server/src/http/users-get.controller.ts
import { Controller, Get, Query, Inject, UseGuards } from '@nestjs/common';
import { QueryBus } from '@monorepo/shared/domain';
import { SHARED_TOKENS, ApiResponse } from '@monorepo/shared/infrastructure/server';
import { ActionPermissionGuard, RequirePermission } from '@monorepo/shared/infrastructure/server';
import { UserPermissionEnum } from '@monorepo/users/domain';
import { SearchUsersByCriteriaQuery, UsersResponse } from '@monorepo/users/application';
import { SearchUsersQueryParamsDto } from './dto/search-users-query-params.dto'; // Asumido que existe

@Controller('users')
export class UsersGetController {
  constructor(
    @Inject(SHARED_TOKENS.QUERY_BUS) private readonly queryBus: QueryBus,
  ) {}

  @Get()
  @RequirePermission(UserPermissionEnum.USER_LIST)
  @UseGuards(ActionPermissionGuard)
  async run(@Query() queryParams: SearchUsersQueryParamsDto) {
    const response = await this.queryBus.ask<UsersResponse>(
      new SearchUsersByCriteriaQuery(
        queryParams.filters,
        queryParams.orderBy,
        queryParams.order,
        queryParams.limit,
        queryParams.offset,
      ),
    );
    return ApiResponse.paginated(response.users, response.pagination, 'Users retrieved');
  }
}
```

---

### 3.3 Bloque: `nest/` (Inyección de Dependencias Limpia)

#### `user.tokens.ts`
```typescript
// libs/users/infrastructure/server/src/nest/user.tokens.ts
export const USER_TOKENS = {
  REPOSITORY: Symbol('UserRepository'),
  FINDER: Symbol('UserFinder'),
  REGISTRAR: Symbol('UserRegistrar'),
} as const;
```

#### `user-server.module.ts`
```typescript
// libs/users/infrastructure/server/src/nest/user-server.module.ts
import { Module } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { EventBus } from '@monorepo/shared/domain';
import { SHARED_TOKENS } from '@monorepo/shared/infrastructure/server';
import { UserRegistrar, UserFinder, RegisterUserCommandHandler } from '@monorepo/users/application';
import { UserRepository } from '@monorepo/users/domain';
import { USER_TOKENS } from './user.tokens';
import { TypeOrmUserRepository } from '../persistence/typeorm/typeorm-user.repository';
import { UserPutController } from '../http/user-put.controller';
import { UsersGetController } from '../http/users-get.controller';

@Module({
  providers: [
    // Application Services
    {
      provide: USER_TOKENS.REGISTRAR,
      useFactory: (repository: UserRepository, eventBus: EventBus) => new UserRegistrar(repository, eventBus),
      inject: [USER_TOKENS.REPOSITORY, SHARED_TOKENS.EVENT_BUS],
    },
    {
      provide: USER_TOKENS.FINDER,
      useFactory: (repository: UserRepository) => new UserFinder(repository),
      inject: [USER_TOKENS.REPOSITORY],
    },
    // Command/Query Handlers
    {
      provide: RegisterUserCommandHandler,
      useFactory: (registrar: UserRegistrar) => new RegisterUserCommandHandler(registrar),
      inject: [USER_TOKENS.REGISTRAR],
    },
    // Repository
    {
      provide: USER_TOKENS.REPOSITORY,
      useFactory: (dataSource: DataSource) => new TypeOrmUserRepository(dataSource),
      inject: [DataSource],
    },
  ],
  controllers: [UserPutController, UsersGetController],
  exports: [RegisterUserCommandHandler],
})
export class UserServerModule {}
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/{bounded_context}/infrastructure/server/src/index.ts
export * from './persistence/typeorm/user.entity-schema';
export * from './persistence/typeorm/typeorm-user.repository';
export * from './persistence/typeorm/user.mapper';

export * from './http/user-put.controller';
export * from './http/users-get.controller';
export * from './http/dto/register-user-http.dto';

export * from './nest/user.tokens';
export * from './nest/user-server.module';
```
