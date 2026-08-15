# Especificación: `libs/{bounded_context}/application`

Este módulo constituye la **Capa de Aplicación del Bounded Context** (la segunda capa de la Onion Architecture).
Su función exclusiva es orquestar los casos de uso del negocio organizados en **Vertical Slices**: recibe comandos
o consultas, recupera agregados, ejecuta métodos de dominio, persiste cambios y despacha eventos de dominio.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Cero Decoradores de Framework:** Prohibido el uso de `@Injectable()`, `@CommandHandler()`,
  `@QueryHandler()` de NestJS o cualquier paquete externo. Las clases son TypeScript puro.
* **[ESTRICTO] Dependencias Permitidas:** Únicamente puede importar:
  * `@monorepo/shared/domain` (`type:shared-domain`)
  * `@monorepo/shared/application` (`type:shared-application`)
  * `@monorepo/{bounded_context}/domain` (`type:domain`)
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:application", "scope:{context}"]`.
* **[ESTRICTO] Orquestación del Ciclo de Eventos:** El caso de uso es el responsable de extraer los eventos con
  `aggregate.pullDomainEvents()` y publicarlos en el `EventBus` **después** de haber persistido el agregado.
* **[ESTRICTO] Prohibición de Reglas de Negocio:** Esta capa no contiene lógica de negocio condicional ni validaciones
  de invariantes; solo orquesta puertos y agregados.

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
│   │   │   ├── find-{aggregate}.query-handler.ts        # [ESTRICTO] Handler de la consulta
│   │   │   ├── {aggregate}-finder.service.ts            # [ESTRICTO] Servicio de búsqueda y validación
│   │   │   └── {aggregate}.response.ts                  # [ESTRICTO] DTO inmutable de respuesta
│   │   │
│   │   └── search-{aggregates}-by-criteria/
│   │       ├── search-{aggregates}-by-criteria.query.ts         # [ESTRICTO] Consulta con filtros dinámicos
│   │       ├── search-{aggregates}-by-criteria.query-handler.ts # [ESTRICTO] Handler de búsqueda
│   │       ├── {aggregates}-by-criteria-searcher.service.ts     # [ESTRICTO] Orquestador de búsqueda
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
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** DTO inmutable que transporta datos primitivos hacia el caso de uso.

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

---

#### `user-registrar.service.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Es el orquestador del caso de uso.

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

---

#### `register-user.command-handler.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Conecta el `CommandBus` con el servicio `UserRegistrar`.

```typescript
// libs/users/application/src/slices/register-user/register-user.command-handler.ts
import { CommandHandler, CommandClass } from '@monorepo/shared/application';
import { UserId, UserName, UserEmail } from '@monorepo/users/domain';
import { RegisterUserCommand } from './register-user.command';
import { UserRegistrar } from './user-registrar.service';

export class RegisterUserCommandHandler implements CommandHandler<RegisterUserCommand> {
  constructor(private readonly userRegistrar: UserRegistrar) {}

  subscribedTo(): CommandClass<RegisterUserCommand> {
    return RegisterUserCommand;
  }

  async handle(command: RegisterUserCommand): Promise<void> {
    const id = new UserId(command.id);
    const name = new UserName(command.name);
    const email = new UserEmail(command.email);

    await this.userRegistrar.run({ id, name, email });
  }
}
```

---

### 3.2 Vertical Slice: Caso de Uso de Lectura (`search-users-by-criteria`)

#### `search-users-by-criteria.query.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/application/src/slices/search-users-by-criteria/search-users-by-criteria.query.ts
import { Query } from '@monorepo/shared/application';

export class SearchUsersByCriteriaQuery extends Query {
  constructor(
    readonly filters: Array<Map<string, string>>,
    readonly orderBy?: string,
    readonly orderType?: string,
    readonly limit?: number,
    readonly offset?: number
  ) {
    super();
  }
}
```

---

#### `users.response.ts` & `user.response.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/application/src/slices/search-users-by-criteria/users.response.ts
import { Response } from '@monorepo/shared/application';
import { User } from '@monorepo/users/domain';

export class UserResponse {
  readonly id: string;
  readonly name: string;
  readonly email: string;

  constructor(user: User) {
    this.id = user.id.value;
    this.name = user.name.value;
    this.email = user.email.value;
  }
}

export class UsersResponse implements Response {
  readonly users: Array<UserResponse>;

  constructor(users: Array<User>) {
    this.users = users.map((user) => new UserResponse(user));
  }
}
```

---

#### `users-by-criteria-searcher.service.ts` & Query Handler
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/application/src/slices/search-users-by-criteria/users-by-criteria-searcher.service.ts
import { Criteria, Filters, Order } from '@monorepo/shared/domain';
import { UserRepository } from '@monorepo/users/domain';
import { UsersResponse } from './users.response';

export class UsersByCriteriaSearcher {
  constructor(private readonly repository: UserRepository) {}

  async run(filters: Filters, order: Order, limit?: number, offset?: number): Promise<UsersResponse> {
    const criteria = new Criteria(filters, order, limit, offset);
    const users = await this.repository.matching(criteria);

    return new UsersResponse(users);
  }
}

// search-users-by-criteria.query-handler.ts
import { QueryHandler, QueryClass } from '@monorepo/shared/application';
import { Filters, Order } from '@monorepo/shared/domain';
import { SearchUsersByCriteriaQuery } from './search-users-by-criteria.query';
import { UsersResponse } from './users.response';
import { UsersByCriteriaSearcher } from './users-by-criteria-searcher.service';

export class SearchUsersByCriteriaQueryHandler
  implements QueryHandler<SearchUsersByCriteriaQuery, UsersResponse>
{
  constructor(private readonly searcher: UsersByCriteriaSearcher) {}

  subscribedTo(): QueryClass<SearchUsersByCriteriaQuery> {
    return SearchUsersByCriteriaQuery;
  }

  async handle(query: SearchUsersByCriteriaQuery): Promise<UsersResponse> {
    const filters = Filters.fromValues(query.filters);
    const order = Order.fromValues(query.orderBy, query.orderType);

    return this.searcher.run(filters, order, query.limit, query.offset);
  }
}
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/{bounded_context}/application/src/index.ts
export * from './slices/register-user/register-user.command';
export * from './slices/register-user/register-user.command-handler';
export * from './slices/register-user/user-registrar.service';

export * from './slices/search-users-by-criteria/search-users-by-criteria.query';
export * from './slices/search-users-by-criteria/search-users-by-criteria.query-handler';
export * from './slices/search-users-by-criteria/users-by-criteria-searcher.service';
export * from './slices/search-users-by-criteria/users.response';
```
