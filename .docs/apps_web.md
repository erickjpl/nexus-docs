# Especificación: `apps/web`

Este módulo constituye el **Punto de Entrada de la Aplicación Web (React / Vite / Next.js)**. Su responsabilidad
exclusiva es montar la aplicación en el DOM, inicializar los adaptadores de infraestructura de cliente
(`FetchHttpClient`, `LocalStorageAdapter`), proveer las dependencias mediante React Context, configurar las rutas
del navegador y renderizar los Contenedores importados desde las librerías `ui/` de cada Bounded Context.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Cero Lógica de Negocio:** La aplicación `apps/web` no define componentes de formulario ni reglas
  de negocio. Es un ensamblador (*Composition Root*) que conecta rutas con los Contenedores (`*.container.tsx`)
  de `libs/{context}/ui`.
* **[ESTRICTO] Inyección de Infraestructura vía Context:** Las instancias de repositorios HTTP y clientes de red
  se construyen una sola vez en el `DependencyProvider` y se distribuyen al árbol de componentes mediante React Context.
* **[ESTRICTO] Dependencias Permitidas:**
  * `@monorepo/shared/domain` (`type:shared-domain`)
  * `@monorepo/shared/infrastructure/client` (`type:shared-infra-client`)
  * `@monorepo/{bounded_context}/domain` (`type:domain`)
  * `@monorepo/{bounded_context}/infrastructure/client` (`type:infra-client`)
  * `@monorepo/{bounded_context}/ui` (`type:ui`)
* **[ESTRICTO] Tag de Nx:** Debe estar configurado en `project.json` con `tags: ["type:app-frontend", "scope:web"]`.
* **[ESTRICTO] Prohibición de Servidor:** Prohibido importar módulos de `infrastructure/server`.

---

## 2. Estructura de Directorios

```text
apps/web/
├── src/
│   ├── app/
│   │   ├── di/
│   │   │   └── dependency-provider.tsx           # [ESTRICTO] Contenedor de DI para React
│   │   ├── routes/
│   │   │   └── app-router.tsx                    # [ESTRICTO] Declaración de rutas con React Router
│   │   └── app.tsx                               # [ESTRICTO] Componente raíz con Providers
│   ├── main.tsx                                  # [ESTRICTO] Bootstrap de React DOM (createRoot)
│   └── index.html
├── project.json
├── tsconfig.json
└── tsconfig.app.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `di/` (Inyección de Dependencias en React)

#### `dependency-provider.tsx`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Inicializa los adaptadores de cliente (`FetchHttpClient`, `HttpUserApiRepository`) utilizando
  las variables de entorno de la app web, y expone un hook `useDependencies()`.

```tsx
// apps/web/src/app/di/dependency-provider.tsx
import React, { createContext, useContext, useMemo } from 'react';
import {
  HttpClient,
  FetchHttpClient,
  KeyValueStorage,
  LocalStorageAdapter
} from '@monorepo/shared/infrastructure/client';
import { UserRepository } from '@monorepo/users/domain';
import { HttpUserApiRepository } from '@monorepo/users/infrastructure/client';

export interface AppDependencies {
  httpClient: HttpClient;
  storage: KeyValueStorage;
  userRepository: UserRepository;
}

const DependencyContext = createContext<AppDependencies | null>(null);

export const DependencyProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const dependencies = useMemo<AppDependencies>(() => {
    const apiUrl = (import.meta as any).env?.VITE_API_URL || 'http://localhost:3000/api';

    const storage = new LocalStorageAdapter();
    const httpClient = new FetchHttpClient(apiUrl);
    const userRepository = new HttpUserApiRepository(httpClient);

    return {
      httpClient,
      storage,
      userRepository
    };
  }, []);

  return (
    <DependencyContext.Provider value={dependencies}>
      {children}
    </DependencyContext.Provider>
  );
};

export function useDependencies(): AppDependencies {
  const context = useContext(DependencyContext);
  if (!context) {
    throw new Error('useDependencies must be used within a <DependencyProvider>');
  }
  return context;
}
```

---

### 3.2 Bloque: `routes/` (Enrutamiento Web)

#### `app-router.tsx`
* **Nivel:** **[ESTRICTO]**

```tsx
// apps/web/src/app/routes/app-router.tsx
import React from 'react';
import { BrowserRouter, Routes, Route, Link, useNavigate } from 'react-router-dom';
import { useDependencies } from '../di/dependency-provider';
import { RegisterUserContainer, UsersListContainer } from '@monorepo/users/ui';
import { CriteriaMother } from '@monorepo/shared/testing';

const UsersPage: React.FC = () => {
  const { userRepository } = useDependencies();
  const navigate = useNavigate();

  return (
    <div className="container">
      <nav>
        <Link to="/">Listado de Usuarios</Link> | <Link to="/register">Registrar Usuario</Link>
      </nav>
      <UsersListContainer
        repository={userRepository}
        initialCriteria={CriteriaMother.empty()}
      />
    </div>
  );
};

const RegisterUserPage: React.FC = () => {
  const { userRepository } = useDependencies();
  const navigate = useNavigate();

  return (
    <div className="container">
      <RegisterUserContainer
        repository={userRepository}
        onUserRegistered={() => navigate('/')}
      />
    </div>
  );
};

export const AppRouter: React.FC = () => {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<UsersPage />} />
        <Route path="/register" element={<RegisterUserPage />} />
      </Routes>
    </BrowserRouter>
  );
};
```

---

### 3.3 Bloque: `main.tsx` & `app.tsx` (Bootstrap de React)

#### `app.tsx`
* **Nivel:** **[ESTRICTO]**

```tsx
// apps/web/src/app/app.tsx
import React from 'react';
import { DependencyProvider } from './di/dependency-provider';
import { AppRouter } from './routes/app-router';

export const App: React.FC = () => {
  return (
    <DependencyProvider>
      <AppRouter />
    </DependencyProvider>
  );
};
```

---

#### `main.tsx`
* **Nivel:** **[ESTRICTO]**

```tsx
// apps/web/src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { App } from './app/app';

const root = ReactDOM.createRoot(document.getElementById('root') as HTMLElement);
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```
