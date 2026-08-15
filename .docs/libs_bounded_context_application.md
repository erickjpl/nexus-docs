# Especificación: `libs/{bounded_context}/application`

Este módulo constituye la **Capa de Aplicación del Bounded Context** (la segunda capa de la Onion Architecture).
Su función exclusiva es orquestar los casos de uso del negocio organizados en **Vertical Slices**: recibe comandos
o consultas, recupera agregados, ejecuta métodos de dominio, valida **políticas de autorización de Nivel 2 (Alcance de Datos y Recursos)**, persiste cambios y despacha eventos de dominio.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Cero Decoradores de Framework:** Prohibido el uso de `@Injectable()`, `@CommandHandler()`,
  `@QueryHandler()` de NestJS o cualquier paquete externo. Las clases son TypeScript puro.
* **[ESTRICTO] Dependencias Permitidas:** Únicamente puede importar:
  * `@monorepo/shared/domain` (`type:shared-domain`)
  * `@monorepo/shared/application` (`type:shared-application`)
  * `@monorepo/{bounded_context}/domain` (`type:domain`)
* **[ESTRICTO] Autorización de Nivel 2 (Alcance de Datos en Dominio):** Aunque la petición haya superado el Nivel 1 (permiso de acción HTTP), el servicio de aplicación valida si el actor tiene derecho sobre la entidad o datos solicitados. Si se viola la política de acceso, lanza `ForbiddenException` de dominio. Para consultas masivas, delega el filtrado por alcance (*Query Scoping*) al repositorio.
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:application", "scope:{context}"]`.
* **[ESTRICTO] Orquestación del Ciclo de Eventos:** El caso de uso es el responsable de extraer los eventos con
  `aggregate.pullDomainEvents()` y publicarlos en el `EventBus` **después** de haber persistido el agregado.

---

## 2. Estructura de Directorios (Vertical Slices)

```text
libs/{bounded_context}/application/
├── src/
│   ├── slices/
│   │   ├── register-{aggregate}/
│   │   │   ├── register-{aggregate}.command.ts          # [ESTRICTO] Comando inmutable de entrada
│   │   │   ├── register-{aggregate}.command-handler.ts  # [ESTRICTO] Handler que conecta el bus con orquestador
│   │   │   └── {aggregate}-registrar.service.ts         # [ESTRICTO] Servicio que ejecuta el caso de uso
│   │   │
│   │   ├── find-{aggregate}/
│   │   │   ├── find-{aggregate}.query.ts                # [ESTRICTO] Consulta de búsqueda por ID
│   │   │   ├── find-{aggregate}.query-handler.ts        # [ESTRICTO] Handler de la consulta con Nivel 2
│   │   │   ├── {aggregate}-finder.service.ts            # [ESTRICTO] Servicio con validación de política
│   │   │   └── {aggregate}.response.ts                  # [ESTRICTO] DTO inmutable de respuesta
│   │   │
│   │   └── search-{aggregates}-by-criteria/
│   │       ├── search-{aggregates}-by-criteria.query.ts         # [ESTRICTO] Consulta con filtros dinámicos
│   │       ├── search-{aggregates}-by-criteria.query-handler.ts # [ESTRICTO] Handler de búsqueda
│   │       ├── {aggregates}-by-criteria-searcher.service.ts     # [ESTRICTO] Orquestador con Query Scoping
│   │       └── {aggregates}.response.ts                         # [ESTRICTO] DTO colección de respuesta
│   │
│   ├── event-handlers/
│   │   └── on-{other_context_event}-{action}.subscriber.ts      # [OPCIONAL] Reacción a eventos de otros contextos
│   │
│   └── index.ts                                          # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Vertical Slice: Caso de Uso de Escritura (`register-user`)

#### `register-user.command.ts`
```typescript
// libs/users/application/src/slices/register-user/register-user.command.ts
import { Command } from '@monorepo/shared/application';

export class RegisterUserCommand extends Command {
  readonly id: string;
  readonly name: string;
  readonly email: string;

  constructor(params: { id: string; name: string; email: string }) {
    super();
    this.id = params.id;
    this.name = params.name;
    this.email = params.email;
  }
}
```

#### `user-registrar.service.ts`
```typescript
// libs/users/application/src/slices/register-user/user-registrar.service.ts
import { EventBus } from '@monorepo/shared/domain';
import { User, UserId, UserName, UserEmail, UserRepository } from '@monorepo/users/domain';

export class UserRegistrar {
  constructor(
    private readonly repository: UserRepository,
    private readonly eventBus: EventBus
  ) {}

  async run(params: { id: UserId; name: UserName; email: UserEmail }): Promise<void> {
    // 1. Instanciación del agregado mediante fábrica de negocio (acumula eventos)
    const user = User.create(params.id, params.name, params.email);

    // 2. Persistencia en la base de datos a través del puerto
    await this.repository.save(user);

    // 3. Extracción de eventos acumulados
    const domainEvents = user.pullDomainEvents();

    // 4. Publicación en el EventBus
    await this.eventBus.publish(domainEvents);
  }
}
```

#### `register-user.command-handler.ts`
```typescript
// libs/users/application/src/slices/register-user/register-user.command-handler.ts
import { CommandHandler } from '@monorepo/shared/application';
import { RegisterUserCommand } from './register-user.command';
import { UserRegistrar } from './user-registrar.service';

export class RegisterUserCommandHandler implements CommandHandler<RegisterUserCommand> {
  constructor(private readonly registrar: UserRegistrar) {}

  subscribedTo(): CommandClass<RegisterUserCommand> {
    return RegisterUserCommand;
  }

  async handle(command: RegisterUserCommand): Promise<void> {
    await this.registrar.run({
      id: command.id,
      name: command.name,
      email: command.email,
    });
  }
}
```

---

### 3.2 Vertical Slice: Caso de Uso con Autorización de Nivel 2 (`find-user`)

```typescript
// libs/users/application/src/slices/find-user/user-finder.service.ts
import { DomainNotFoundError, ForbiddenException } from '@monorepo/shared/domain';
import { User, UserId, UserRepository } from '@monorepo/users/domain';
import { UserResponse } from './user.response';

export interface UserFinderActor {
  id: string;
  role: string;
  tenantId: string;
}

export class UserFinder {
  constructor(private readonly repository: UserRepository) {}

  async run(targetId: UserId, actor: UserFinderActor): Promise<UserResponse> {
    const user = await this.repository.search(targetId);

    if (!user) {
      throw new DomainNotFoundError(`El usuario con ID <${targetId.value}> no existe`);
    }

    // Autorización Nivel 2: Política de Dominio sobre el Recurso
    if (user.tenantId.value !== actor.tenantId) {
      throw new ForbiddenException('Cannot access user from another tenant');
    }

    const primitives = user.toPrimitives();
    return new UserResponse(primitives.id, primitives.name, primitives.email);
  }
}
```

#### `user.response.ts`
```typescript
// libs/users/application/src/slices/find-user/user.response.ts
import { Response } from '@monorepo/shared/application';

export class UserResponse implements Response {
  constructor(
    readonly id: string,
    readonly name: string,
    readonly email: string,
  ) {}
}
```

#### `find-user.query-handler.ts`
```typescript
// libs/users/application/src/slices/find-user/find-user.query-handler.ts
import { QueryHandler, QueryClass } from '@monorepo/shared/application';
import { FindUserQuery } from './find-user.query';
import { UserFinder } from './user-finder.service';
import { UserResponse } from './user.response';
import { UserId } from '@monorepo/users/domain';

export class FindUserQueryHandler implements QueryHandler<FindUserQuery, UserResponse> {
  constructor(private readonly finder: UserFinder) {}

  subscribedTo(): QueryClass<FindUserQuery> {
    return FindUserQuery;
  }

  async handle(query: FindUserQuery): Promise<UserResponse> {
    return this.finder.run(new UserId(query.id), { tenantId: query.tenantId });
  }
}
```

---

### 3.3 Vertical Slice: Caso de Uso de Lectura (`search-users-by-criteria`)

```typescript
import { Criteria, Filter, FilterField, FilterOperator, FilterValue, Operator, Filters, Order } from '@monorepo/shared/domain';
import { UserRepository } from '@monorepo/users/domain';
import { UsersResponse } from './users.response';
import { UserResponse } from '../find-user/user.response';

export interface UserSearcherActor {
  tenantId: string;
}

export class UsersByCriteriaSearcher {
  constructor(private readonly repository: UserRepository) {}

  async run(params: { actor: UserSearcherActor; filters?: Filter[]; order?: Order; limit?: number; offset?: number }): Promise<UsersResponse> {
    const scopeFilter = new Filter(
      new FilterField('tenant_id'),
      new FilterOperator(Operator.EQUAL),
      new FilterValue(params.actor.tenantId),
    );
    const scopedFilters = new Filters([scopeFilter, ...(params.filters || [])]);
    const criteria = new Criteria(scopedFilters, params.order ?? Order.none(), params.limit, params.offset);
    
    const { items, total } = await this.repository.matching(criteria);
    const userResponses = items.map(user => {
      const p = user.toPrimitives();
      return new UserResponse(p.id, p.name, p.email);
    });
    return new UsersResponse(userResponses, {
      total,
      count: userResponses.length,
      per_page: params.limit ?? 20,
      current_page: Math.floor((params.offset ?? 0) / (params.limit ?? 20)) + 1,
      total_pages: Math.ceil(total / (params.limit ?? 20)),
    });
  }
}
```

#### `users.response.ts`
```typescript
// libs/users/application/src/slices/search-users-by-criteria/users.response.ts
import { Response } from '@monorepo/shared/application';
import { PaginationMeta } from '@monorepo/shared/infrastructure/server';
import { UserResponse } from '../find-user/user.response';

export class UsersResponse implements Response {
  constructor(
    readonly users: UserResponse[],
    readonly pagination: PaginationMeta,
  ) {}
}
```

> **Transporte del Contexto de Actor:** El contexto del `actor` (como `id`, `role`, `tenantId`) debe extraerse del usuario autenticado en la petición HTTP (usualmente poblado por `JwtAuthGuard` en el backend) y pasarse explícitamente desde el Controlador al Comando o Query. La capa de aplicación no debe acceder directamente al request.

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/{bounded_context}/application/src/index.ts
export * from './slices/register-user/register-user.command';
export * from './slices/register-user/register-user.command-handler';
export * from './slices/register-user/user-registrar.service';

export * from './slices/find-user/user-finder.service';
export * from './slices/find-user/user.response';

export * from './slices/search-users-by-criteria/search-users-by-criteria.query';
export * from './slices/search-users-by-criteria/search-users-by-criteria.query-handler';
export * from './slices/search-users-by-criteria/users-by-criteria-searcher.service';
export * from './slices/search-users-by-criteria/users.response';
```
