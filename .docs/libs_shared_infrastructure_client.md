# Especificación: `libs/shared/infrastructure/client`

Este módulo constituye la **Infraestructura de Cliente Compartida (Shared Client Infrastructure)**. Contiene las
implementaciones técnicas y adaptadores requeridos por aplicaciones frontend (Web, Mobile y Desktop) para
comunicarse con APIs remotas, almacenar datos locales en el dispositivo y gestionar buses reactivos en memoria para
la interfaz de usuario.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Compatibilidad Web & Mobile Total:** Prohibido utilizar módulos nativos de Node.js (`fs`, `path`,
  `crypto` de Node, `amqplib`, `mongodb`, `typeorm`, `@nestjs/*`). El código debe ser 100% compatible con
  navegadores modernos, Metro Bundler (React Native / Expo) y Electron Renderer.
* **[ESTRICTO] Dependencias Permitidas:** Solo puede depender de `@monorepo/shared/domain` (`type:shared-domain`).
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:shared-infra-client", "scope:shared"]`.
* **[ESTRICTO] Aislamiento de Librerías HTTP:** La UI nunca utiliza `fetch` o `axios` directamente. Toda petición
  pasa por la interfaz `HttpClient` para garantizar interceptores centralizados de autenticación, manejo de tokens
  y traducción de errores.

---

## 2. Estructura de Directorios

```text
libs/shared/infrastructure/client/
├── src/
│   ├── http/
│   │   ├── http-client.interface.ts              # [ESTRICTO] Contrato universal para clientes HTTP
│   │   ├── fetch-http-client.ts                  # [ESTRICTO] Implementación universal basada en fetch nativo
│   │   ├── http-error.ts                         # [ESTRICTO] Error tipado para respuestas HTTP 4xx/5xx
│   │   ├── http-request-options.interface.ts     # [ESTRICTO] Opciones de cabeceras, params y timeout
│   │   └── criteria-to-http-params.converter.ts  # [ESTRICTO] Serializa Criteria a Query Params URL
│   │
│   ├── storage/
│   │   ├── key-value-storage.interface.ts        # [ESTRICTO] Contrato universal de persistencia clave-valor
│   │   ├── in-memory-storage.adapter.ts          # [ESTRICTO] Adaptador en memoria para tests y SSR
│   │   └── local-storage.adapter.ts              # [OPCIONAL/WEB] Adaptador para localStorage de navegador
│   │
│   ├── bus/
│   │   └── client-event-bus.ts                   # [ESTRICTO] Bus de eventos local reactivo para la UI
│   │
│   └── index.ts                                  # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `http/`

#### `http-client.interface.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Desacopla los adaptadores de API de la librería de transporte HTTP subyacente. Permite cambiar
  `fetch` por `axios` o un mock sin tocar los repositorios del frontend.

```typescript
export interface HttpRequestOptions {
  headers?: Record<string, string>;
  params?: Record<string, string | number | boolean | undefined>;
  timeout?: number;
  signal?: AbortSignal;
}

export interface HttpClient {
  get<T>(url: string, options?: HttpRequestOptions): Promise<T>;
  post<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<T>;
  put<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<T>;
  patch<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<T>;
  delete<T>(url: string, options?: HttpRequestOptions): Promise<T>;
}
```

---

#### `fetch-http-client.ts` & `http-error.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Implementa `HttpClient` utilizando la API estándar `fetch` (disponible en navegadores,
  React Native y Node 18+).

```typescript
import { HttpClient, HttpRequestOptions } from './http-client.interface';
import { HttpError } from './http-error';

export class FetchHttpClient implements HttpClient {
  constructor(
    private readonly baseUrl: string,
    private readonly defaultHeaders: () => Record<string, string> = () => ({})
  ) {}

  async get<T>(url: string, options?: HttpRequestOptions): Promise<T> {
    return this.request<T>(url, { method: 'GET', ...options });
  }

  async post<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<T> {
    return this.request<T>(url, {
      method: 'POST',
      body: JSON.stringify(body),
      ...options,
      headers: { 'Content-Type': 'application/json', ...options?.headers }
    });
  }

  async put<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<T> {
    return this.request<T>(url, {
      method: 'PUT',
      body: JSON.stringify(body),
      ...options,
      headers: { 'Content-Type': 'application/json', ...options?.headers }
    });
  }

  async patch<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<T> {
    return this.request<T>(url, {
      method: 'PATCH',
      body: JSON.stringify(body),
      ...options,
      headers: { 'Content-Type': 'application/json', ...options?.headers }
    });
  }

  async delete<T>(url: string, options?: HttpRequestOptions): Promise<T> {
    return this.request<T>(url, { method: 'DELETE', ...options });
  }

  private async request<T>(url: string, init: RequestInit & HttpRequestOptions): Promise<T> {
    const fullUrl = this.buildUrl(url, init.params);
    const headers = { ...this.defaultHeaders(), ...init.headers };

    const response = await fetch(fullUrl, { ...init, headers });

    if (!response.ok) {
      let errorBody: any;
      try {
        errorBody = await response.json();
      } catch {
        errorBody = await response.text();
      }
      throw new HttpError(response.status, response.statusText, errorBody);
    }

    if (response.status === 204) {
      return undefined as unknown as T;
    }

    return (await response.json()) as T;
  }

  private buildUrl(path: string, params?: Record<string, any>): string {
    const url = new URL(path, this.baseUrl);
    if (params) {
      Object.entries(params).forEach(([key, value]) => {
        if (value !== undefined && value !== null) {
          url.searchParams.append(key, String(value));
        }
      });
    }
    return url.toString();
  }
}

export class HttpError extends Error {
  constructor(
    readonly status: number,
    readonly statusText: string,
    readonly data: any
  ) {
    super(`HTTP Error ${status} (${statusText}): ${JSON.stringify(data)}`);
    this.name = 'HttpError';
  }
}
```

---

#### `criteria-to-http-params.converter.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Permite que un cliente frontend envíe un objeto `Criteria` al backend API mediante query
  parameters estandarizados.

```typescript
import { Criteria } from '@monorepo/shared/domain';

export class CriteriaToHttpParamsConverter {
  static convert(criteria: Criteria): Record<string, string | number> {
    const params: Record<string, string | number> = {};

    criteria.filters.filters.forEach((filter, index) => {
      params[`filters[${index}][field]`] = filter.field.value;
      params[`filters[${index}][operator]`] = filter.operator.value;
      params[`filters[${index}][value]`] = filter.value.value;
    });

    if (criteria.hasOrder()) {
      params['orderBy'] = criteria.order.orderBy.value;
      params['orderType'] = criteria.order.orderType.value;
    }

    if (criteria.limit !== undefined) {
      params['limit'] = criteria.limit;
    }

    if (criteria.offset !== undefined) {
      params['offset'] = criteria.offset;
    }

    return params;
  }
}
```

---

### 3.2 Bloque: `storage/`

#### `key-value-storage.interface.ts` & Adaptadores
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Abstrae el almacenamiento local de tokens de sesión, preferencias de usuario o caché.

```typescript
export interface KeyValueStorage {
  get(key: string): Promise<string | null>;
  set(key: string, value: string): Promise<void>;
  remove(key: string): Promise<void>;
  clear(): Promise<void>;
}

export class InMemoryStorageAdapter implements KeyValueStorage {
  private storage: Map<string, string> = new Map();

  async get(key: string): Promise<string | null> {
    return this.storage.get(key) ?? null;
  }

  async set(key: string, value: string): Promise<void> {
    this.storage.set(key, value);
  }

  async remove(key: string): Promise<void> {
    this.storage.delete(key);
  }

  async clear(): Promise<void> {
    this.storage.clear();
  }
}
```

---

### 3.3 Bloque: `bus/` (Client Event Bus)

#### `client-event-bus.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Implementa el puerto `EventBus` en el frontend para permitir que componentes de UI y Custom
  Hooks reaccionen a eventos de dominio locales.

```typescript
import { DomainEvent, DomainEventSubscriber, EventBus } from '@monorepo/shared/domain';

export class ClientEventBus implements EventBus {
  private subscribers: Array<DomainEventSubscriber<DomainEvent>> = [];

  addSubscribers(subscribers: Array<DomainEventSubscriber<DomainEvent>>): void {
    this.subscribers.push(...subscribers);
  }

  async publish(events: Array<DomainEvent>): Promise<void> {
    for (const event of events) {
      const matchedSubscribers = this.subscribers.filter((subscriber) =>
        subscriber.subscribedTo().some((eventClass) => eventClass.EVENT_NAME === event.eventName)
      );

      await Promise.all(matchedSubscribers.map((subscriber) => subscriber.on(event)));
    }
  }
}
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/shared/infrastructure/client/src/index.ts
export * from './http/http-client.interface';
export * from './http/http-request-options.interface';
export * from './http/fetch-http-client';
export * from './http/http-error';
export * from './http/criteria-to-http-params.converter';

export * from './storage/key-value-storage.interface';
export * from './storage/in-memory-storage.adapter';
export * from './storage/local-storage.adapter';

export * from './bus/client-event-bus';
```
