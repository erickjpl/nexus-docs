# Especificación: `libs/{bounded_context}/testing`

Este módulo constituye las **Utilidades de Testing del Bounded Context**. Proporciona generadores deterministas y
pseudoaleatorios (**Object Mothers**) para los Agregados, Value Objects, Comandos y Eventos específicos de este
contexto, así como **Mocks Semánticos** para los puertos de persistencia con aserciones declarativas de negocio.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Exclusividad de Testing:** Prohibido importar este paquete en código de producción. Solo se importa
  en archivos `*.spec.ts`, `*.test.ts` y tests de integración/aceptación.
* **[ESTRICTO] Dependencias Permitidas:**
  * `@monorepo/shared/domain` (`type:shared-domain`)
  * `@monorepo/shared/application` (`type:shared-application`)
  * `@monorepo/shared/testing` (`type:shared-testing`)
  * `@monorepo/{bounded_context}/domain` (`type:domain`)
  * `@monorepo/{bounded_context}/application` (`type:application`)
* **[ESTRICTO] Tag de Nx:** Debe estar configurado en `project.json` con `tags: ["type:testing", "scope:{context}"]`.
* **[ESTRICTO] Generación de Datos Válidos por Defecto:** Cada *Object Mother* debe ser capaz de construir una
  instancia 100% válida con datos aleatorios llamando a `.random()`, y permitir sobrescribir mediante `.create(...)`.

---

## 2. Estructura de Directorios

```text
libs/{bounded_context}/testing/
├── src/
│   ├── mother/
│   │   ├── {aggregate}-id.mother.ts              # [ESTRICTO] Generador de IDs del agregado
│   │   ├── {value_object}.mother.ts              # [ESTRICTO] Generador de cada VO del modelo
│   │   ├── {aggregate}.mother.ts                 # [ESTRICTO] Generador del agregado completo
│   │   ├── register-{aggregate}.command-mother.ts# [ESTRICTO] Generador de comandos para testear handlers
│   │   └── {aggregate}-registered.event-mother.ts# [ESTRICTO] Generador de eventos de dominio
│   │
│   ├── mocks/
│   │   └── {aggregate}.repository-mock.ts        # [ESTRICTO] Mock semántico del repositorio de dominio
│   │
│   └── index.ts                                  # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `mother/` (Object Mothers de Dominio y Aplicación)

#### `user-id.mother.ts` & Value Object Mothers
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/testing/src/mother/user-id.mother.ts
import { UuidMother } from '@monorepo/shared/testing';
import { UserId } from '@monorepo/users/domain';

export class UserIdMother {
  static create(value: string): UserId {
    return new UserId(value);
  }

  static random(): UserId {
    return this.create(UuidMother.random());
  }
}

// libs/users/testing/src/mother/user-name.mother.ts
import { StringMother } from '@monorepo/shared/testing';
import { UserName } from '@monorepo/users/domain';

export class UserNameMother {
  static create(value: string): UserName {
    return new UserName(value);
  }

  static random(): UserName {
    return this.create(StringMother.words(2));
  }
}

// libs/users/testing/src/mother/user-email.mother.ts
import { MotherCreator } from '@monorepo/shared/testing';
import { UserEmail } from '@monorepo/users/domain';

export class UserEmailMother {
  static create(value: string): UserEmail {
    return new UserEmail(value);
  }

  static random(): UserEmail {
    return this.create(MotherCreator.random().internet.email());
  }
}
```

---

#### `user.mother.ts` (Object Mother del Agregado)
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/testing/src/mother/user.mother.ts
import { User, UserId, UserName, UserEmail } from '@monorepo/users/domain';
import { UserIdMother } from './user-id.mother';
import { UserNameMother } from './user-name.mother';
import { UserEmailMother } from './user-email.mother';
import { RegisterUserCommand } from '@monorepo/users/application';

export class UserMother {
  static create(id?: UserId, name?: UserName, email?: UserEmail): User {
    return new User(
      id ?? UserIdMother.random(),
      name ?? UserNameMother.random(),
      email ?? UserEmailMother.random()
    );
  }

  static fromCommand(command: RegisterUserCommand): User {
    return new User(
      UserIdMother.create(command.id),
      UserNameMother.create(command.name),
      UserEmailMother.create(command.email)
    );
  }

  static random(): User {
    return this.create();
  }
}
```

---

#### `register-user.command-mother.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/testing/src/mother/register-user.command-mother.ts
import { UserId, UserName, UserEmail } from '@monorepo/users/domain';
import { RegisterUserCommand } from '@monorepo/users/application';
import { UserIdMother } from './user-id.mother';
import { UserNameMother } from './user-name.mother';
import { UserEmailMother } from './user-email.mother';

export class RegisterUserCommandMother {
  static create(id?: UserId, name?: UserName, email?: UserEmail): RegisterUserCommand {
    return new RegisterUserCommand({
      id: (id ?? UserIdMother.random()).value,
      name: (name ?? UserNameMother.random()).value,
      email: (email ?? UserEmailMother.random()).value
    });
  }

  static random(): RegisterUserCommand {
    return this.create();
  }
}
```

#### `user-registered.event-mother.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/testing/src/mother/user-registered.event-mother.ts
import { UserRegisteredDomainEvent } from '@monorepo/users/domain';
import { UuidMother } from '@monorepo/shared/testing';

export class UserRegisteredEventMother {
  static create(params?: Partial<{ id: string; name: string; email: string }>): UserRegisteredDomainEvent {
    return new UserRegisteredDomainEvent({
      aggregateId: params?.id ?? UuidMother.random(),
      name: params?.name ?? 'John Doe',
      email: params?.email ?? 'john@example.com',
    });
  }

  static random(): UserRegisteredDomainEvent {
    return this.create();
  }
}
```

---

### 3.2 Bloque: `mocks/` (Mock Semántico del Repositorio)

#### `user.repository-mock.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/testing/src/mocks/user.repository-mock.ts
import { Criteria } from '@monorepo/shared/domain';
import { User, UserId, UserRepository } from '@monorepo/users/domain';

export class UserRepositoryMock implements UserRepository {
  private saveMock = jest.fn();
  private searchMock = jest.fn();
  private searchAllMock = jest.fn();
  private matchingMock = jest.fn();

  async save(user: User): Promise<void> {
    this.saveMock(user);
  }

  async search(id: UserId): Promise<User | null> {
    return this.searchMock(id);
  }

  async searchAll(): Promise<Array<User>> {
    return this.searchAllMock();
  }

  async matching(criteria: Criteria): Promise<Array<User>> {
    return this.matchingMock(criteria);
  }

  assertSaveHaveBeenCalledWith(expectedUser: User): void {
    expect(this.saveMock).toHaveBeenCalledWith(expectedUser);
  }

  assertSaveHaveNotBeenCalled(): void {
    expect(this.saveMock).not.toHaveBeenCalled();
  }

  assertSearchHaveBeenCalledWith(expectedId: UserId): void {
    expect(this.searchMock).toHaveBeenCalledWith(expectedId);
  }

  returnOnSearch(user: User | null): void {
    this.searchMock.mockReturnValue(user);
  }

  returnOnMatching(users: Array<User>): void {
    this.matchingMock.mockReturnValue(users);
  }
}
```

---

### 3.3 Ejemplo de Test Unitario con Object Mothers

```typescript
// libs/users/application/src/slices/register-user/user-registrar.service.spec.ts
import { EventBusMock } from '@monorepo/shared/testing';
import { UserRegistrar } from './user-registrar.service';
import { UserMother, RegisterUserCommandMother, UserRepositoryMock } from '@monorepo/users/testing';
import { UserRegisteredDomainEvent } from '@monorepo/users/domain';

describe('UserRegistrar', () => {
  let repository: UserRepositoryMock;
  let eventBus: EventBusMock;
  let registrar: UserRegistrar;

  beforeEach(() => {
    repository = new UserRepositoryMock();
    eventBus = new EventBusMock();
    registrar = new UserRegistrar(repository, eventBus);
  });

  it('should register a valid user, persist it and publish domain event', async () => {
    const command = RegisterUserCommandMother.random();
    const user = UserMother.fromCommand(command);

    await registrar.run({
      id: user.id,
      name: user.name,
      email: user.email
    });

    repository.assertSaveHaveBeenCalledWith(user);
    eventBus.assertLastPublishedEventIs(UserRegisteredDomainEvent);
  });
});
```

> **Nota sobre Jest vs Vitest:** Los ejemplos aquí utilizan `jest.fn()` para los mocks. Si el equipo utiliza Vitest, se debe reemplazar por `vi.fn()`. Es recomendable crear una abstracción agnóstica para el test runner (ej. un `MockFactory`) si el proyecto los combina en distintos entornos.

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/{bounded_context}/testing/src/index.ts
export * from './mother/user-id.mother';
export * from './mother/user-name.mother';
export * from './mother/user-email.mother';
export * from './mother/user.mother';
export * from './mother/register-user.command-mother';
export * from './mother/user-registered.event-mother';

export * from './mocks/user.repository-mock';
```
