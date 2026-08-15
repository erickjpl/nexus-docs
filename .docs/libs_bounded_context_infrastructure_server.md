# Especificación: `libs/{bounded_context}/infrastructure/server`

Este módulo constituye la **Capa de Infraestructura de Servidor del Bounded Context**. Contiene las implementaciones
técnicas concretas que requieren el entorno de ejecución Node.js y el framework NestJS: persistencia en base de datos
(TypeORM/Mongo), esquemas de tablas/documentos, mappers de datos, controladores HTTP REST, consumidores de mensajería
(RabbitMQ) y la configuración de inyección de dependencias de NestJS mediante **Factory Providers**.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Exclusividad de Backend:** Prohibido importar este paquete en aplicaciones frontend (`apps/web`,
  `apps/mobile`, `apps/desktop`) o librerías de UI. Contiene dependencias de Node.js, TypeORM y NestJS.
* **[ESTRICTO] Dependencias Permitidas:**
  * `@monorepo/shared/domain` (`type:shared-domain`)
  * `@monorepo/shared/application` (`type:shared-application`)
  * `@monorepo/shared/infrastructure/server` (`type:shared-infra-server`)
  * `@monorepo/{bounded_context}/domain` (`type:domain`)
  * `@monorepo/{bounded_context}/application` (`type:application`)
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:infra-server", "scope:{context}"]`.
* **[ESTRICTO] Mappers Bidireccionales:** Ninguna consulta SQL/ORM debe devolver directamente esquemas de persistencia
  a la capa de aplicación. Todo dato se transforma al Agregado mediante `Aggregate.fromPrimitives()`, y todo agregado
  que entra se extrae con `aggregate.toPrimitives()`.
* **[ESTRICTO] Registro Limpio en NestJS:** Los casos de uso de `application/` se registran en los módulos de NestJS
  mediante `useFactory` y tokens de inyección, garantizando que el núcleo permanezca libre de `@Injectable()`.

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
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** En lugar de ensuciar las clases del Dominio con decoradores `@Entity()` y `@Column()`, se
  utiliza `EntitySchema` en la capa de infraestructura.

```typescript
// libs/users/infrastructure/server/src/persistence/typeorm/user.entity-schema.ts
import { EntitySchema } from 'typeorm';

export interface UserDatabaseSchema {
  id: string;
  name: string;
  email: string;
}

export const UserEntitySchema = new EntitySchema<UserDatabaseSchema>({
  name: 'User',
  tableName: 'users',
  columns: {
    id: {
      type: 'uuid',
      primary: true
    },
    name: {
      type: 'varchar',
      length: 50
    },
    email: {
      type: 'varchar',
      length: 100,
      unique: true
    }
  }
});
```

---

#### `user.mapper.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/infrastructure/server/src/persistence/typeorm/user.mapper.ts
import { User } from '@monorepo/users/domain';
import { UserDatabaseSchema } from './user.entity-schema';

export class UserMapper {
  static toDomain(raw: UserDatabaseSchema): User {
    return User.fromPrimitives({
      id: raw.id,
      name: raw.name,
      email: raw.email
    });
  }

  static toPersistence(user: User): UserDatabaseSchema {
    return user.toPrimitives();
  }
}
```

---

#### `typeorm-user.repository.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/infrastructure/server/src/persistence/typeorm/typeorm-user.repository.ts
import { Repository, DataSource } from 'typeorm';
import { Criteria } from '@monorepo/shared/domain';
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
    await this.repository.save(raw);
  }

  async search(id: UserId): Promise<User | null> {
    const raw = await this.repository.findOne({ where: { id: id.value } });
    return raw ? UserMapper.toDomain(raw) : null;
  }

  async searchAll(): Promise<Array<User>> {
    const raws = await this.repository.find();
    return raws.map((raw) => UserMapper.toDomain(raw));
  }

  async matching(criteria: Criteria): Promise<Array<User>> {
    const options = this.criteriaConverter.convert<UserDatabaseSchema>(criteria);
    const raws = await this.repository.find(options);
    return raws.map((raw) => UserMapper.toDomain(raw));
  }
}
```

---

### 3.2 Bloque: `http/` (Controladores NestJS)

#### `register-user-http.dto.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/infrastructure/server/src/http/dto/register-user-http.dto.ts
import { IsString, IsUUID, IsEmail, MinLength, MaxLength } from 'class-validator';

export class RegisterUserHttpDto {
  @IsUUID('4')
  id!: string;

  @IsString()
  @MinLength(3)
  @MaxLength(50)
  name!: string;

  @IsEmail()
  email!: string;
}
```

---

#### `user-put.controller.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/infrastructure/server/src/http/user-put.controller.ts
import { Controller, Put, Param, Body, HttpCode, HttpStatus, Inject } from '@nestjs/common';
import { CommandBus } from '@monorepo/shared/domain';
import { SHARED_TOKENS } from '@monorepo/shared/infrastructure/server';
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
  async run(@Param('id') id: string, @Body() body: RegisterUserHttpDto): Promise<void> {
    const command = new RegisterUserCommand({
      id,
      name: body.name,
      email: body.email
    });

    await this.commandBus.dispatch(command);
  }
}
```

---

### 3.3 Bloque: `nest/` (Inyección de Dependencias Limpia)

#### `user.tokens.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/infrastructure/server/src/nest/user.tokens.ts
export const USER_TOKENS = {
  REPOSITORY: Symbol('USER_REPOSITORY'),
  REGISTRAR: Symbol('USER_REGISTRAR'),
  SEARCHER: Symbol('USERS_SEARCHER')
} as const;
```

---

#### `user-server.module.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/infrastructure/server/src/nest/user-server.module.ts
import { Module } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { EventBus } from '@monorepo/shared/domain';
import { SHARED_TOKENS } from '@monorepo/shared/infrastructure/server';
import { UserRegistrar, RegisterUserCommandHandler } from '@monorepo/users/application';
import { UserRepository } from '@monorepo/users/domain';
import { USER_TOKENS } from './user.tokens';
import { TypeOrmUserRepository } from '../persistence/typeorm/typeorm-user.repository';
import { UserPutController } from '../http/user-put.controller';

@Module({
  controllers: [UserPutController],
  providers: [
    {
      provide: USER_TOKENS.REPOSITORY,
      useFactory: (dataSource: DataSource): UserRepository => {
        return new TypeOrmUserRepository(dataSource);
      },
      inject: [DataSource]
    },
    {
      provide: USER_TOKENS.REGISTRAR,
      useFactory: (repository: UserRepository, eventBus: EventBus): UserRegistrar => {
        return new UserRegistrar(repository, eventBus);
      },
      inject: [USER_TOKENS.REPOSITORY, SHARED_TOKENS.EVENT_BUS]
    },
    {
      provide: RegisterUserCommandHandler,
      useFactory: (registrar: UserRegistrar) => new RegisterUserCommandHandler(registrar),
      inject: [USER_TOKENS.REGISTRAR]
    }
  ],
  exports: [USER_TOKENS.REPOSITORY, USER_TOKENS.REGISTRAR]
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
export * from './http/dto/register-user-http.dto';

export * from './nest/user.tokens';
export * from './nest/user-server.module';
```
