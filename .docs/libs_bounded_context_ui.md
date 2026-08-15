# Especificación: `libs/{bounded_context}/ui`

Este módulo constituye la **Capa de Presentación e Interfaz de Usuario del Bounded Context** (la capa exterior visual).
Su responsabilidad es capturar las interacciones del usuario en Web, Mobile o Desktop, invocar los casos de uso o
repositorios de cliente mediante **Custom Hooks**, y renderizar componentes visuales puros siguiendo el patrón
**Container / Presentational**.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Cero Reglas de Negocio en la Vista:** Los componentes visuales (`JSX/TSX`) **NUNCA** ejecutan cálculos
  de negocio, validaciones de invariantes ni formateos arbitrarios de datos. Toda la lógica de presentación reside
  en los Custom Hooks y el dominio.
* **[ESTRICTO] Patrón Container / Presentational (Smart vs Dumb):**
  * **Containers (`*.container.tsx`):** Componentes inteligentes que consumen los hooks, gestionan el estado,
    conectan rutas y pasan datos y callbacks a las vistas.
  * **Views (`*.view.tsx`):** Componentes puramente visuales que reciben props y emiten eventos (`onClick`, `onSubmit`).
* **[ESTRICTO] Dependencias Permitidas:**
  * `@monorepo/shared/domain` (`type:shared-domain`)
  * `@monorepo/shared/application` (`type:shared-application`)
  * `@monorepo/shared/infrastructure/client` (`type:shared-infra-client`)
  * `@monorepo/{bounded_context}/domain` (`type:domain`)
  * `@monorepo/{bounded_context}/infrastructure/client` (`type:infra-client`)
* **[ESTRICTO] Tag de Nx:** Debe estar configurado en `project.json` con `tags: ["type:ui", "scope:{context}"]`.
* **[ESTRICTO] Prohibición de Servidor:** Prohibido importar módulos de `infrastructure/server`.

---

## 2. Estructura de Directorios

```text
libs/{bounded_context}/ui/
├── src/
│   ├── hooks/
│   │   ├── use-register-{aggregate}.hook.ts      # [ESTRICTO] Hook para mutaciones de estado con Result Pattern
│   │   └── use-search-{aggregates}.hook.ts       # [ESTRICTO] Hook para consultas reactivas con Criteria
│   │
│   ├── containers/
│   │   ├── register-{aggregate}.container.tsx    # [ESTRICTO] Smart Component que orquesta creación
│   │   └── {aggregates}-list.container.tsx       # [ESTRICTO] Smart Component que orquesta listado
│   │
│   ├── views/
│   │   ├── register-{aggregate}.view.tsx         # [ESTRICTO] Dumb Component para formulario visual
│   │   └── {aggregates}-list.view.tsx            # [ESTRICTO] Dumb Component para render de lista/grilla
│   │
│   ├── components/
│   │   └── {aggregate}-card.component.tsx        # [OPCIONAL] Componente visual atómico del contexto
│   │
│   └── index.ts                                  # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `hooks/` (Lógica de Presentación y Estado)

#### `use-register-user.hook.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/ui/src/hooks/use-register-user.hook.ts
import { useState } from 'react';
import { User, UserId, UserName, UserEmail, UserRepository } from '@monorepo/users/domain';

interface RegisterUserInput {
  id: string;
  name: string;
  email: string;
}

export function useRegisterUser(repository: UserRepository) {
  const [isLoading, setIsLoading] = useState<boolean>(false);
  const [error, setError] = useState<string | null>(null);
  const [isSuccess, setIsSuccess] = useState<boolean>(false);

  const registerUser = async (input: RegisterUserInput): Promise<boolean> => {
    setIsLoading(true);
    setError(null);
    setIsSuccess(false);

    try {
      const id = new UserId(input.id);
      const name = new UserName(input.name);
      const email = new UserEmail(input.email);

      const user = User.create(id, name, email);

      await repository.save(user);

      setIsSuccess(true);
      return true;
    } catch (err: any) {
      setError(err?.message || 'Error inesperado al registrar el usuario');
      return false;
    } finally {
      setIsLoading(false);
    }
  };

  return {
    registerUser,
    isLoading,
    error,
    isSuccess
  };
}
```

---

#### `use-search-users.hook.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// libs/users/ui/src/hooks/use-search-users.hook.ts
import { useState, useEffect, useCallback } from 'react';
import { Criteria } from '@monorepo/shared/domain';
import { User, UserRepository } from '@monorepo/users/domain';

export function useSearchUsers(repository: UserRepository, initialCriteria: Criteria) {
  const [users, setUsers] = useState<Array<User>>([]);
  const [isLoading, setIsLoading] = useState<boolean>(true);
  const [error, setError] = useState<string | null>(null);

  const fetchUsers = useCallback(async (criteria: Criteria) => {
    setIsLoading(true);
    setError(null);
    try {
      const result = await repository.matching(criteria);
      setUsers(result);
    } catch (err: any) {
      setError(err?.message || 'Error al cargar los usuarios');
    } finally {
      setIsLoading(false);
    }
  }, [repository]);

  useEffect(() => {
    fetchUsers(initialCriteria);
  }, [fetchUsers, initialCriteria]);

  return {
    users,
    isLoading,
    error,
    refetch: fetchUsers
  };
}
```

---

### 3.2 Bloque: `views/` (Componentes Presentacionales Puros)

#### `register-user.view.tsx`
* **Nivel:** **[ESTRICTO]**

```tsx
// libs/users/ui/src/views/register-user.view.tsx
import React, { useState } from 'react';

export interface RegisterUserFormData {
  id: string;
  name: string;
  email: string;
}

export interface RegisterUserViewProps {
  onSubmit: (data: RegisterUserFormData) => void;
  isLoading: boolean;
  errorMessage: string | null;
  isSuccess: boolean;
}

export const RegisterUserView: React.FC<RegisterUserViewProps> = ({
  onSubmit,
  isLoading,
  errorMessage,
  isSuccess
}) => {
  const [formData, setFormData] = useState<RegisterUserFormData>({
    id: '',
    name: '',
    email: ''
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    onSubmit(formData);
  };

  return (
    <form onSubmit={handleSubmit} className="user-form">
      <h2>Registrar Nuevo Usuario</h2>

      {errorMessage && <div className="error-alert">{errorMessage}</div>}
      {isSuccess && <div className="success-alert">¡Usuario registrado exitosamente!</div>}

      <div className="form-group">
        <label htmlFor="user-id">ID del Usuario (UUID):</label>
        <input
          id="user-id"
          type="text"
          value={formData.id}
          onChange={(e) => setFormData({ ...formData, id: e.target.value })}
          required
        />
      </div>

      <div className="form-group">
        <label htmlFor="user-name">Nombre:</label>
        <input
          id="user-name"
          type="text"
          value={formData.name}
          onChange={(e) => setFormData({ ...formData, name: e.target.value })}
          required
        />
      </div>

      <div className="form-group">
        <label htmlFor="user-email">Correo Electrónico:</label>
        <input
          id="user-email"
          type="email"
          value={formData.email}
          onChange={(e) => setFormData({ ...formData, email: e.target.value })}
          required
        />
      </div>

      <button type="submit" disabled={isLoading}>
        {isLoading ? 'Guardando...' : 'Registrar Usuario'}
      </button>
    </form>
  );
};
```

---

### 3.3 Bloque: `containers/` (Smart Components)

#### `register-user.container.tsx`
* **Nivel:** **[ESTRICTO]**

```tsx
// libs/users/ui/src/containers/register-user.container.tsx
import React from 'react';
import { UserRepository } from '@monorepo/users/domain';
import { useRegisterUser } from '../hooks/use-register-user.hook';
import { RegisterUserView, RegisterUserFormData } from '../views/register-user.view';

export interface RegisterUserContainerProps {
  repository: UserRepository;
  onUserRegistered?: () => void;
}

export const RegisterUserContainer: React.FC<RegisterUserContainerProps> = ({
  repository,
  onUserRegistered
}) => {
  const { registerUser, isLoading, error, isSuccess } = useRegisterUser(repository);

  const handleSubmit = async (data: RegisterUserFormData) => {
    const success = await registerUser(data);
    if (success && onUserRegistered) {
      onUserRegistered();
    }
  };

  return (
    <RegisterUserView
      onSubmit={handleSubmit}
      isLoading={isLoading}
      errorMessage={error}
      isSuccess={isSuccess}
    />
  );
};
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/{bounded_context}/ui/src/index.ts
export * from './hooks/use-register-user.hook';
export * from './hooks/use-search-users.hook';

export * from './views/register-user.view';
export * from './views/users-list.view';

export * from './containers/register-user.container';
export * from './containers/users-list.container';
```
