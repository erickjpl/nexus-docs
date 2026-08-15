# Especificación: `apps/web`

Este módulo constituye el **Punto de Entrada de la Aplicación Web (React / Vite)**. Su responsabilidad
exclusiva es montar la aplicación en el DOM, inicializar los adaptadores de infraestructura de cliente mediante
las variables de entorno validadas (`env`), proveer las dependencias mediante React Context, configurar el enrutamiento
con React Router y renderizar las Páginas Orquestadoras (`pages/`) de cada Bounded Context.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Aislamiento de Entorno:** Queda **estrictamente prohibido invocar `import.meta.env` o `process.env`** en este módulo. Todo se consume desde la constante tipada y validada `env` de `@monorepo/shared/infrastructure/client`.
* **[ESTRICTO] Cero Lógica de Negocio y Presentación Local:** La aplicación `apps/web` no define formularios ni vistas de dominio. Es un ensamblador (*Composition Root*) que conecta rutas con las Páginas (`*-page.tsx`) de `libs/{context}/ui`.
* **[ESTRICTO] Dependencias Permitidas:**
  * `@monorepo/shared/domain` (`type:shared-domain`)
  * `@monorepo/shared/infrastructure/client` (`type:shared-infra-client`)
  * `@monorepo/{bounded_context}/domain` (`type:domain`)
  * `@monorepo/{bounded_context}/infrastructure/client` (`type:infra-client`)
  * `@monorepo/{bounded_context}/ui` (`type:ui`)
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:app-frontend", "scope:web"]`.
* **[ESTRICTO] Prohibición de Servidor:** Prohibido importar módulos de `infrastructure/server`.

---

## 2. Estructura de Directorios

```text
apps/web/
├── src/
│   ├── app/
│   │   ├── di/
│   │   │   └── dependency-provider.tsx           # [ESTRICTO] Inyección de dependencias cliente
│   │   ├── routes/
│   │   │   └── app-router.tsx                    # [ESTRICTO] Declaración de rutas con React Router
│   │   └── app.tsx                               # [ESTRICTO] Raíz de providers
│   ├── main.tsx                                  # [ESTRICTO] Bootstrap en el DOM
│   └── index.html
├── project.json
├── tsconfig.json
└── tsconfig.app.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `di/dependency-setup.tsx` (Inyección con Env Validado)

```tsx
// apps/web/src/app/di/dependency-setup.tsx
import React, { useMemo } from 'react';
import {
  FetchHttpClient,
  LocalStorageAdapter,
  env,
  RepositoryProvider
} from '@monorepo/shared/infrastructure/client';
import { HttpUserApiRepository } from '@monorepo/users/infrastructure/client';

export const DependencySetup: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const repositories = useMemo(() => {
    // Uso exclusivo de la configuración validada con Zod (Fail-Fast)
    const storage = new LocalStorageAdapter();
    const httpClient = new FetchHttpClient(
      env.VITE_API_URL,
      () => storage.get('auth_token'),
    );
    const userRepository = new HttpUserApiRepository(httpClient);

    return {
      storage,
      httpClient,
      userRepository,
    };
  }, []);

  return (
    <RepositoryProvider repositories={repositories}>
      {children}
    </RepositoryProvider>
  );
};
```

---

### 3.2 Bloque: `routes/app-router.tsx` & `routes/protected-route.tsx` (Enrutamiento con Páginas de UI)

```tsx
// apps/web/src/app/routes/protected-route.tsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuthPermissions } from '@monorepo/shared/infrastructure/client';

interface ProtectedRouteProps {
  permission?: string;
  children: React.ReactNode;
  fallback?: string;
}

export function ProtectedRoute({ permission, children, fallback = '/unauthorized' }: ProtectedRouteProps) {
  const { isAuthenticated, hasPermission } = useAuthPermissions();

  if (!isAuthenticated) return <Navigate to="/login" replace />;
  if (permission && !hasPermission(permission)) return <Navigate to={fallback} replace />;

  return <>{children}</>;
}
```

```tsx
// apps/web/src/app/routes/app-router.tsx
import React, { Suspense } from 'react';
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';
import { ProtectedRoute } from './protected-route';
import { UserPermissionEnum } from '@monorepo/users/domain';

const UsersPage = React.lazy(() => import('@monorepo/users/ui').then(m => ({ default: m.UsersPage })));

export const AppRouter: React.FC = () => {
  return (
    <BrowserRouter>
      <header className="app-navigation">
        <nav>
          <Link to="/users">Gestión de Usuarios</Link>
        </nav>
      </header>

      <Suspense fallback={<div>Cargando...</div>}>
        <Routes>
          <Route path="/users" element={
            <ProtectedRoute permission={UserPermissionEnum.USER_LIST}>
              <UsersPage />
            </ProtectedRoute>
          } />
          <Route path="/" element={
            <ProtectedRoute permission={UserPermissionEnum.USER_LIST}>
              <UsersPage />
            </ProtectedRoute>
          } />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
};
```

---

### 3.3 Bloque: `app.tsx` & `main.tsx`

```tsx
// apps/web/src/app/app.tsx
import React from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { DependencySetup } from './di/dependency-setup';
import { AppRouter } from './routes/app-router';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 5 * 60 * 1000, retry: 1 },
  },
});

export const App: React.FC = () => {
  return (
    <QueryClientProvider client={queryClient}>
      <DependencySetup>
        <AppRouter />
      </DependencySetup>
    </QueryClientProvider>
  );
};
```

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
