# Especificación: `libs/shared/infrastructure/client`

Este módulo constituye la **Infraestructura de Cliente Compartida (Shared Client Infrastructure)**. Contiene las
implementaciones técnicas, adaptadores y componentes base requeridos por aplicaciones frontend (Web, Mobile y Desktop) para
comunicarse con APIs remotas, almacenar datos locales en el dispositivo, gestionar buses reactivos, **validar variables de entorno centralizadas con Zod (`env.ts`)**, proveer **UI Wrappers** agnósticos y gestionar la **autorización atómica en frontend**.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Compatibilidad Web & Mobile Total:** Prohibido utilizar módulos nativos de Node.js (`fs`, `path`,
  `crypto` de Node, `amqplib`, `mongodb`, `typeorm`, `@nestjs/*`). El código debe ser 100% compatible con
  navegadores modernos, Metro Bundler (React Native / Expo) y Electron Renderer.
* **[ESTRICTO] Aislamiento Absoluto de Variables de Entorno (`src/config/env.ts`):** `import.meta.env` o `process.env` **NUNCA se invoca directamente fuera de este archivo**. Todas las variables del sistema se declaran, leen y validan mediante un esquema **Zod** al inicializar la aplicación (*Fail-Fast*).
* **[ESTRICTO] UI Wrappers Compartidos (`src/components/ui/`):** Prohibido el acoplamiento directo a librerías visuales en las features. Todo botón, modal, tabla o formulario se encapsula aquí (`AppButton`, `AppCard`, `AppForm`, `AppModal`, `AppTable`).
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
│   ├── config/
│   │   └── env.ts                                # [ESTRICTO] Único punto de acceso y validación Zod de variables de entorno
│   │
│   ├── di/
│   │   └── repository-registry.tsx               # [ESTRICTO] Registro de repositorios y hooks de DI
│   │
│   ├── components/
│   │   └── ui/                                   # [ESTRICTO] Wrappers agnósticos de componentes visuales
│   │       ├── app-button.tsx
│   │       ├── app-card.tsx
│   │       ├── app-form.tsx
│   │       ├── app-modal.tsx
│   │       ├── app-table.tsx
│   │       ├── app-notification.ts
│   │       └── index.ts
│   │
│   ├── auth/
│   │   └── use-auth-permissions.hook.ts          # [ESTRICTO] Hook de frontend para verificación de permisos atómicos
│   │
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
│   ├── sockets/
│   │   ├── socket-client.interface.ts            # [ESTRICTO] Contrato de cliente WebSocket
│   │   └── echo-socket.adapter.ts                # [OPCIONAL] Adaptador WebSocket (Socket.io / native WS)
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

### 3.1 Bloque: `config/env.ts` (Aislamiento de Entorno con Zod)

```typescript
// libs/shared/infrastructure/client/src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  VITE_API_URL: z.string().url('VITE_API_URL debe ser una URL válida'),
  VITE_APP_ENV: z.enum(['development', 'staging', 'production']).default('development'),
  VITE_WS_HOST: z.string().default('localhost'),
  VITE_WS_PORT: z.coerce.number().default(8080),
  VITE_TIMEOUT_MS: z.coerce.number().default(10000),
});

// ÚNICA lectura permitida de variables de entorno en toda la aplicación cliente
const envSource = {
  VITE_API_URL: (import.meta as any).env?.VITE_API_URL ?? 
    (typeof process !== 'undefined' ? process.env?.VITE_API_URL : undefined),
  VITE_APP_ENV: (import.meta as any).env?.VITE_APP_ENV ?? 
    (typeof process !== 'undefined' ? process.env?.VITE_APP_ENV : undefined),
  VITE_WS_HOST: (import.meta as any).env?.VITE_WS_HOST ?? 
    (typeof process !== 'undefined' ? process.env?.VITE_WS_HOST : undefined),
  VITE_WS_PORT: (import.meta as any).env?.VITE_WS_PORT ?? 
    (typeof process !== 'undefined' ? process.env?.VITE_WS_PORT : undefined),
  VITE_TIMEOUT_MS: (import.meta as any).env?.VITE_TIMEOUT_MS ?? 
    (typeof process !== 'undefined' ? process.env?.VITE_TIMEOUT_MS : undefined),
};

const parsedEnv = envSchema.safeParse(envSource);

if (!parsedEnv.success) {
  console.error('❌ Variables de entorno cliente inválidas:', parsedEnv.error.format());
  throw new Error('Configuración de entorno inválida. Revise su archivo .env');
}

export const env = parsedEnv.data;
```

---

### 3.1.b Bloque: `di/repository-registry.tsx` (Inyección de Dependencias)

```tsx
// libs/shared/infrastructure/client/src/di/repository-registry.tsx
import { createContext, useContext, ReactNode } from 'react';

type RepositoryMap = Record<string, unknown>;

const RepositoryContext = createContext<RepositoryMap>({});

export function RepositoryProvider({
  repositories,
  children,
}: {
  repositories: RepositoryMap;
  children: ReactNode;
}) {
  return (
    <RepositoryContext.Provider value={repositories}>
      {children}
    </RepositoryContext.Provider>
  );
}

export function useRepository<T>(key: string): T {
  const repos = useContext(RepositoryContext);
  const repo = repos[key];
  if (!repo) {
    throw new Error(`Repository "${key}" not registered. Wrap your app with <RepositoryProvider>.`);
  }
  return repo as T;
}
```

---

### 3.2 Bloque: `components/ui/` (UI Wrappers Desacoplados)

#### `app-button.tsx`
```tsx
// libs/shared/infrastructure/client/src/components/ui/app-button.tsx
import React from 'react';

export interface AppButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
  loading?: boolean;
}

export const AppButton: React.FC<AppButtonProps> = ({
  children,
  variant = 'primary',
  loading = false,
  disabled,
  className = '',
  ...props
}) => {
  return (
    <button
      disabled={disabled || loading}
      className={`app-btn app-btn--${variant} ${className}`}
      {...props}
    >
      {loading ? <span className="spinner" /> : children}
    </button>
  );
};
```

#### `app-form.tsx`
```tsx
// libs/shared/infrastructure/client/src/components/ui/app-form.tsx
import React from 'react';

export interface AppFormProps extends React.FormHTMLAttributes<HTMLFormElement> {}

export const AppForm: React.FC<AppFormProps> & {
  Item: React.FC<{ label?: string; error?: string; children: React.ReactNode }>;
} = ({ children, ...props }) => {
  return <form {...props}>{children}</form>;
};

AppForm.Item = ({ label, error, children }) => (
  <div className="app-form-item">
    {label && <label className="app-form-label">{label}</label>}
    <div className="app-form-control">{children}</div>
    {error && <span className="app-form-error">{error}</span>}
  </div>
);
```

#### Otros Wrappers

```tsx
// libs/shared/infrastructure/client/src/components/ui/app-modal.tsx
import React from 'react';

export interface AppModalProps {
  title: string;
  open: boolean;
  onClose: () => void;
  children: React.ReactNode;
}

export const AppModal: React.FC<AppModalProps> = ({ title, open, onClose, children }) => {
  if (!open) return null;
  return (
    <div className="app-modal-overlay">
      <div className="app-modal">
        <header>
          <h2>{title}</h2>
          <button onClick={onClose}>X</button>
        </header>
        <main>{children}</main>
      </div>
    </div>
  );
};
```

```tsx
// libs/shared/infrastructure/client/src/components/ui/app-table.tsx
import React from 'react';

export interface AppTableProps<T = any> {
  dataSource: T[];
  columns: { title: string; dataIndex: keyof T }[];
  rowKey: keyof T;
  loading?: boolean;
}

export const AppTable = <T extends Record<string, any>>({ dataSource, columns, rowKey, loading }: AppTableProps<T>) => {
  if (loading) return <div>Cargando...</div>;
  return (
    <table className="app-table">
      <thead>
        <tr>{columns.map(col => <th key={String(col.dataIndex)}>{col.title}</th>)}</tr>
      </thead>
      <tbody>
        {dataSource.map(row => (
          <tr key={String(row[rowKey])}>
            {columns.map(col => <td key={String(col.dataIndex)}>{row[col.dataIndex]}</td>)}
          </tr>
        ))}
      </tbody>
    </table>
  );
};
```

```tsx
// libs/shared/infrastructure/client/src/components/ui/app-card.tsx
import React from 'react';

export const AppCard: React.FC<{ title?: string; children: React.ReactNode }> = ({ title, children }) => (
  <div className="app-card">
    {title && <div className="app-card-header">{title}</div>}
    <div className="app-card-body">{children}</div>
  </div>
);
```

```ts
// libs/shared/infrastructure/client/src/components/ui/app-notification.ts
export function useAppNotification() {
  return {
    message: {
      success: (msg: string) => console.log('✅', msg),
      error: (msg: string) => console.error('❌', msg),
    }
  };
}
```

---

### 3.3 Bloque: `auth/use-auth-permissions.hook.ts` (Permisos Atómicos en UI)

```typescript
// libs/shared/infrastructure/client/src/auth/use-auth-permissions.hook.ts
import { useCallback } from 'react';

export interface UserSessionState {
  userId: string;
  permissions: Array<string>;
}

export function useAuthPermissions<T extends string = string>() {
  // Simulación de lectura de estado global Zustand
  const userPermissions: Array<string> = ['user.create', 'user.view'];
  const isAuthenticated = true;

  const hasPermission = useCallback(
    (permission: T): boolean => {
      return userPermissions.includes(permission as string);
    },
    [userPermissions]
  );

  return {
    isAuthenticated,
    hasPermission,
    permissions: userPermissions,
  };
}
```

---

### 3.4 Bloque: `http/`

```typescript
// libs/shared/infrastructure/client/src/http/http-client.interface.ts
export interface PaginationMeta {
  total: number;
  perPage?: number;
  currentPage?: number;
  lastPage?: number;
}

export interface ApiResponse<T> {
  success: boolean;
  message: string;
  data: T;
  meta: { code: number; timestamp: string; version: string; pagination?: PaginationMeta };
  errors?: Record<string, string[]>;
}

export interface HttpRequestOptions {
  headers?: Record<string, string>;
  params?: Record<string, string | number | boolean | undefined>;
  timeout?: number;
  signal?: AbortSignal;
}

export interface HttpClient {
  get<T>(url: string, options?: HttpRequestOptions): Promise<ApiResponse<T>>;
  post<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<ApiResponse<T>>;
  put<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<ApiResponse<T>>;
  patch<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<ApiResponse<T>>;
  delete<T>(url: string, options?: HttpRequestOptions): Promise<ApiResponse<T>>;
}
```

```typescript
// libs/shared/infrastructure/client/src/http/http-error.ts
export class HttpError extends Error {
  constructor(
    readonly statusCode: number,
    message: string,
    readonly response?: any,
  ) {
    super(message);
    this.name = 'HttpError';
  }
}
```

```typescript
// libs/shared/infrastructure/client/src/http/fetch-http-client.ts
import { HttpClient, HttpRequestOptions, ApiResponse } from './http-client.interface';
import { HttpError } from './http-error';

export class FetchHttpClient implements HttpClient {
  constructor(
    private readonly baseUrl: string,
    private readonly getToken?: () => string | Promise<string | null> | null,
  ) {}

  async get<T>(url: string, options?: HttpRequestOptions): Promise<ApiResponse<T>> {
    return this.request<T>(url, { method: 'GET', ...options });
  }

  async post<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<ApiResponse<T>> {
    return this.request<T>(url, { method: 'POST', body: JSON.stringify(body), ...options });
  }

  async put<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<ApiResponse<T>> {
    return this.request<T>(url, { method: 'PUT', body: JSON.stringify(body), ...options });
  }

  async patch<T>(url: string, body?: any, options?: HttpRequestOptions): Promise<ApiResponse<T>> {
    return this.request<T>(url, { method: 'PATCH', body: JSON.stringify(body), ...options });
  }

  async delete<T>(url: string, options?: HttpRequestOptions): Promise<ApiResponse<T>> {
    return this.request<T>(url, { method: 'DELETE', ...options });
  }

  private async request<T>(url: string, init: RequestInit & HttpRequestOptions): Promise<ApiResponse<T>> {
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      ...init.headers,
    };

    const token = await this.getToken?.();
    if (token) {
      headers['Authorization'] = `Bearer ${token}`;
    }

    const queryString = init.params ? `?${new URLSearchParams(init.params as Record<string, string>)}` : '';

    const response = await fetch(`${this.baseUrl}${url}${queryString}`, {
      ...init,
      headers,
    });

    if (!response.ok) {
      const error = await response.json().catch(() => ({}));
      throw new HttpError(response.status, error.message || response.statusText, error);
    }

    const data = await response.json();
    return data as ApiResponse<T>;
  }
}
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/shared/infrastructure/client/src/index.ts
export * from './config/env';
export * from './di/repository-registry';

export * from './components/ui/app-button';
export * from './components/ui/app-card';
export * from './components/ui/app-form';
export * from './components/ui/app-modal';
export * from './components/ui/app-table';
export * from './components/ui/app-notification';

export * from './auth/use-auth-permissions.hook';

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
