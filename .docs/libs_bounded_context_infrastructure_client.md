# Especificación: `libs/{bounded_context}/infrastructure/client`

Este módulo constituye la **Capa de Infraestructura de Cliente del Bounded Context**. Contiene los adaptadores HTTP,
repositorios de API y estrategias de persistencia local (caché/offline) necesarios para que las interfaces de
usuario (Web, Mobile y Desktop) interactúen con los servicios backend de este contexto sin acoplarse a detalles
de transporte de red.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Compatibilidad Web & Mobile:** Prohibido importar módulos de Node.js (`fs`, `path`, `@nestjs/*`,
  `typeorm`, `mongodb`). Todo el código debe compilar limpiamente en navegadores y React Native (Metro).
* **[ESTRICTO] Dependencias Permitidas:**
  * `@monorepo/shared/domain` (`type:shared-domain`)
  * `@monorepo/shared/application` (`type:shared-application`)
  * `@monorepo/shared/infrastructure/client` (`type:shared-infra-client`)
  * `@monorepo/{bounded_context}/domain` (`type:domain`)
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:infra-client", "scope:{context}"]`.
* **[ESTRICTO] Consumo a través de `HttpClient`:** Prohibido el uso directo de `fetch` o `axios` crudo. Toda petición
  se realiza a través de la interfaz `HttpClient` inyectada en el constructor.
* **[ESTRICTO] Rehidratación hacia el Dominio:** Los datos recibidos del servidor se deserializan hacia Agregados o
  DTOs del dominio utilizando `Aggregate.fromPrimitives()`.

---

## 2. Estructura de Directorios

```text
libs/{bounded_context}/infrastructure/client/
├── src/
│   ├── api/
│   │   ├── dto/
│   │   │   ├── {aggregate}-api.response.ts       # [ESTRICTO] Tipado de la respuesta JSON del servidor
│   │   │   └── register-{aggregate}-api.body.ts  # [ESTRICTO] Formato del payload enviado a la API
│   │   └── http-{aggregate}-api.repository.ts    # [ESTRICTO] Implementación del repositorio vía HTTP
│   │
│   ├── cache/
│   │   └── {aggregate}-offline-storage.ts        # [OPCIONAL] Almacenamiento local clave-valor para offline
│   │
│   └── index.ts                                  # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `api/dto/`

#### `user-api.response.ts` & `register-user-api.body.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/infrastructure/client/src/api/dto/user-api.response.ts
export interface UserApiResponse {
  id: string;
  name: string;
  email: string;
}

// libs/users/infrastructure/client/src/api/dto/register-user-api.body.ts
export interface RegisterUserApiBody {
  id: string;
  name: string;
  email: string;
}
```

---

### 3.2 Bloque: `api/` (Repositorio HTTP)

#### `http-user-api.repository.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Implementa la interfaz pura `UserRepository` del dominio en el lado cliente. Realiza las
  llamadas HTTP hacia el backend y rehidrata los agregados con `User.fromPrimitives()`.

```typescript
// libs/users/infrastructure/client/src/api/http-user-api.repository.ts
import { Criteria } from '@monorepo/shared/domain';
import { HttpClient, CriteriaToHttpParamsConverter, HttpError } from '@monorepo/shared/infrastructure/client';
import { User, UserId, UserRepository, DomainConflictError, InvalidArgumentError } from '@monorepo/users/domain';
import { UserApiResponse, RegisterUserApiBody } from './dto/user-api.response';

export class HttpUserApiRepository implements UserRepository {
  private readonly endpoint = '/users';

  constructor(private readonly httpClient: HttpClient) {}

  private mapHttpError(error: HttpError): never {
    switch (error.statusCode) {
      case 409: throw new DomainConflictError('User already exists');
      case 422: throw new InvalidArgumentError(error.message);
      default: throw error;
    }
  }

  async save(user: User): Promise<void> {
    const body: RegisterUserApiBody = user.toPrimitives();
    try {
      await this.httpClient.put<void>(`${this.endpoint}/${user.id.value}`, body);
    } catch (error) {
      if (error instanceof HttpError) this.mapHttpError(error);
      throw error;
    }
  }

  async search(id: UserId): Promise<User | null> {
    try {
      const response = await this.httpClient.get<UserApiResponse>(`${this.endpoint}/${id.value}`);
      return User.fromPrimitives(response.data);
    } catch (error) {
      if (error instanceof HttpError && error.statusCode === 404) {
        return null;
      }
      if (error instanceof HttpError) this.mapHttpError(error);
      throw error;
    }
  }

  async searchAll(): Promise<Array<User>> {
    const response = await this.httpClient.get<UserApiResponse[]>(this.endpoint);
    return response.data.map((item) => User.fromPrimitives(item));
  }

  async matching(criteria: Criteria): Promise<{ items: User[]; total: number }> {
    const params = CriteriaToHttpParamsConverter.convert(criteria);
    const response = await this.httpClient.get<UserApiResponse[]>(this.endpoint, { params });

    return {
      items: response.data.map(User.fromPrimitives),
      total: response.meta.pagination?.total ?? 0,
    };
  }
}
```

> **Nota:** La interfaz `HttpClient` extrae automáticamente la propiedad `.data` del `ApiResponse` homogéneo. Para endpoints con paginación, puedes definir el tipo de retorno esperado que incluya la metadata.

---

### 3.3 Bloque: `cache/` (Soporte Offline / Local)

#### `user-offline-storage.ts`
* **Nivel:** **[OPCIONAL]**

```typescript
// libs/users/infrastructure/client/src/cache/user-offline-storage.ts
import { KeyValueStorage } from '@monorepo/shared/infrastructure/client';
import { User } from '@monorepo/users/domain';

export class UserOfflineStorage {
  private readonly storageKey = 'offline_users';

  constructor(private readonly storage: KeyValueStorage) {}

  async saveAll(users: Array<User>): Promise<void> {
    const serialized = JSON.stringify(users.map((u) => u.toPrimitives()));
    await this.storage.set(this.storageKey, serialized);
  }

  async getAll(): Promise<Array<User>> {
    const raw = await this.storage.get(this.storageKey);
    if (!raw) return [];

    const parsed = JSON.parse(raw);
    return parsed.map((item: any) => User.fromPrimitives(item));
  }
}
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/{bounded_context}/infrastructure/client/src/index.ts
export * from './api/dto/user-api.response';
export * from './api/dto/register-user-api.body';
export * from './api/http-user-api.repository';
export * from './cache/user-offline-storage';
```
