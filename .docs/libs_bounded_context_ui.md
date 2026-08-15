# Especificación: `libs/{bounded_context}/ui`

Este módulo constituye la **Capa de Presentación e Interfaz de Usuario del Bounded Context** (la capa exterior visual).
Su responsabilidad es estructurar las pantallas mediante una **jerarquía estricta de 4 niveles** (`pages/` → `views/` → `components/` → `schemas/`), validar todos los formularios obligatoriamente con **Zod**, consumir los adaptadores cliente mediante **Custom Hooks**, utilizar **UI Wrappers** agnósticos y gobernar la visibilidad de acciones con la regla **"1 Acción = 1 Permiso Dedicado"**.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Jerarquía de Presentación de 4 Niveles:**
  1. **`pages/` (Páginas Orquestadoras):** Conectan rutas, estado global (Zustand), mutaciones/consultas asíncronas (TanStack Query) y Custom Hooks.
  2. **`views/` (Vistas Contenedoras):** Agrupan y orquestan componentes visuales dependientes entre sí (ej. modales, secciones de listados).
  3. **`components/` (Componentes Puros de Feature):** Componentes visuales específicos de la funcionalidad (ej. `UserCard`, `UserForm`).
  4. **`schemas/` (Esquemas Zod de Validación):** Definición aislada y tipada con Zod de los esquemas de validación para todos los formularios.
* **[ESTRICTO] Validación Zod Obligatoria en el 100% de Formularios:** Todo formulario DEBE definir su esquema Zod en `schemas/` y consumirlo mediante React Hook Form (`@hookform/resolvers/zod`).
* **[ESTRICTO] Desacoplamiento Visual mediante UI Wrappers (`@/shared/components/ui`):** Queda estrictamente **prohibido importar librerías UI de terceros directamente en las features** (ej. `antd`, `mui`, `chakra`). Toda la UI debe construirse con los adaptadores encapsulados (`AppButton`, `AppCard`, `AppForm`, `AppModal`, `AppTable`, `useAppNotification`).
* **[ESTRICTO] Control de Permisos Atómicos en UI (Cero Falsos Positivos):** Toda acción, botón o elemento que dispare una operación debe consultar el permiso atómico exacto vía Enum (`hasPermission(PermissionEnum.USER_REGISTER)`). Prohibido usar roles globales o strings mágicos para condicionar visibilidad.
* **[ESTRICTO] Importaciones Absolutas:** Prohibido el uso de rutas relativas multinivel (`../../..`). Toda importación utiliza los alias canónicos `@/*` o `@monorepo/{context}/*`.
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:ui", "scope:{context}"]`.
* **[ESTRICTO] Prohibición de Servidor:** Prohibido importar módulos de `infrastructure/server` o librerías de Node.js / TypeORM.

---

## 2. Estructura Canónica de Directorios

```text
libs/{bounded_context}/ui/
├── src/
│   ├── hooks/
│   │   ├── use-register-{aggregate}.hook.ts      # Hook para mutaciones y casos de uso
│   │   └── use-search-{aggregates}.hook.ts       # Hook para consultas reactivas con Criteria
│   │
│   ├── pages/                                    # 📄 Nivel 1: Páginas Orquestadoras de Alto Nivel
│   │   └── {aggregates}-page.tsx                 # Conecta rutas, estado y orquestación
│   │
│   ├── views/                                    # 🖼️ Nivel 2: Vistas Contenedoras Interdependientes
│   │   ├── register-{aggregate}-modal.view.tsx   # Contenedor de modal y orquestación visual
│   │   └── {aggregates}-list-section.view.tsx    # Contenedor de listado y filtros
│   │
│   ├── components/                               # 🧩 Nivel 3: Componentes Puros y Átomos
│   │   ├── {aggregate}-form.component.tsx        # Formulario desacoplado con UI Wrappers
│   │   └── {aggregate}-card.component.tsx        # Tarjeta visual atómica
│   │
│   ├── schemas/                                  # 📐 Nivel 4: Esquemas de Validación Zod
│   │   └── {aggregate}-form.schema.ts            # Esquema Zod e inferencia de tipos
│   │
│   └── index.ts                                  # Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada y Ejemplo Canónico

### 3.1 Nivel 4: Esquema Zod Obligatorio (`schemas/user-form.schema.ts`)

```typescript
// libs/users/ui/src/schemas/user-form.schema.ts
import { z } from 'zod';

export const userFormSchema = z.object({
  name: z.string().min(2, 'El nombre debe tener al menos 2 caracteres').max(100),
  email: z.string().email('Ingrese un correo electrónico válido'),
  role: z.enum(['admin', 'user', 'manager'], {
    errorMap: () => ({ message: 'Seleccione un rol válido' }),
  }).optional().default('user'),
});

export type UserFormValues = z.infer<typeof userFormSchema>;
```

---

### 3.2 Nivel 3: Componente Visual Puro con UI Wrappers (`components/user-form.component.tsx`)

```tsx
// libs/users/ui/src/components/user-form.component.tsx
import React from 'react';
import { useForm, UseFormReturn } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { AppButton, AppForm } from '@monorepo/shared/infrastructure/client';
import { userFormSchema, UserFormValues } from '../schemas/user-form.schema';

export interface UserFormComponentProps {
  onSubmit: (values: UserFormValues, form: UseFormReturn<UserFormValues>) => void;
  isLoading?: boolean;
  initialValues?: Partial<UserFormValues>;
}

export const UserFormComponent: React.FC<UserFormComponentProps> = ({
  onSubmit,
  isLoading = false,
  initialValues,
}) => {
  const form = useForm<UserFormValues>({
    resolver: zodResolver(userFormSchema),
    defaultValues: initialValues,
  });

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = form;

  return (
    <AppForm onSubmit={handleSubmit((values) => onSubmit(values, form))}>
      <AppForm.Item label="Nombre Completo" error={errors.name?.message}>
        <input {...register('name')} placeholder="Ej. Juan Pérez" />
      </AppForm.Item>

      <AppForm.Item label="Correo Electrónico" error={errors.email?.message}>
        <input {...register('email')} type="email" placeholder="juan@ejemplo.com" />
      </AppForm.Item>

      <AppButton type="submit" variant="primary" loading={isLoading}>
        Guardar Usuario
      </AppButton>
    </AppForm>
  );
};
```

---

### 3.3 Nivel 2: Vista Contenedora (`views/register-user-modal.view.tsx`)

```tsx
// libs/users/ui/src/views/register-user-modal.view.tsx
import React from 'react';
import { UseFormReturn } from 'react-hook-form';
import { AppModal } from '@monorepo/shared/infrastructure/client';
import { UserFormComponent } from '../components/user-form.component';
import { UserFormValues } from '../schemas/user-form.schema';

export interface RegisterUserModalViewProps {
  isOpen: boolean;
  onClose: () => void;
  onSubmit: (values: UserFormValues, form: UseFormReturn<UserFormValues>) => void;
  isLoading?: boolean;
}

export const RegisterUserModalView: React.FC<RegisterUserModalViewProps> = ({
  isOpen,
  onClose,
  onSubmit,
  isLoading,
}) => {
  return (
    <AppModal title="Registrar Nuevo Usuario" open={isOpen} onClose={onClose}>
      <UserFormComponent onSubmit={onSubmit} isLoading={isLoading} />
    </AppModal>
  );
};
```

---

### 3.4 Nivel 1: Página Orquestadora con Permisos Atómicos (`pages/users-page.tsx`)

```tsx
// libs/users/ui/src/pages/users-page.tsx
import React, { useState } from 'react';
import { UseFormReturn } from 'react-hook-form';
import { AppButton, AppTable, useAppNotification, useAuthPermissions } from '@monorepo/shared/infrastructure/client';
import { UserPermissionEnum } from '@monorepo/users/domain';
import { useRegisterUser } from '../hooks/use-register-user.hook';
import { useSearchUsers } from '../hooks/use-search-users.hook';
import { RegisterUserModalView } from '../views/register-user-modal.view';
import { UserFormValues } from '../schemas/user-form.schema';

export const UsersPage: React.FC = () => {
  const [isModalOpen, setIsModalOpen] = useState(false);
  const { hasPermission } = useAuthPermissions();
  const { message } = useAppNotification();
  
  const { data: users, isLoading } = useSearchUsers();
  const { mutate: registerUser, isPending: isRegistering } = useRegisterUser();

  const handleRegisterSubmit = (values: UserFormValues, form: UseFormReturn<UserFormValues>) => {
    registerUser(values, {
      onSuccess: () => {
        message.success('Usuario registrado con éxito');
        setIsModalOpen(false);
      },
      onError: (error: any) => {
        message.error('Error al registrar el usuario');
        if (error.response?.meta?.code === 422 && error.response?.errors) {
          Object.entries(error.response.errors).forEach(([field, messages]) => {
            form.setError(field as keyof UserFormValues, {
              type: 'server',
              message: (messages as string[])[0],
            });
          });
        }
      }
    });
  };

  return (
    <div className="page-container">
      <header className="page-header">
        <h1>Gestión de Usuarios</h1>
        
        {/* Regla de Oro: 1 Acción = 1 Permiso Dedicado (Enum) */}
        {hasPermission(UserPermissionEnum.USER_CREATE) && (
          <AppButton variant="primary" onClick={() => setIsModalOpen(true)}>
            Nuevo Usuario
          </AppButton>
        )}
      </header>

      <main>
        <AppTable
          dataSource={users || []}
          loading={isLoading}
          rowKey="id"
          columns={[
            { title: 'ID', dataIndex: 'id' },
            { title: 'Nombre', dataIndex: 'name' },
            { title: 'Email', dataIndex: 'email' },
          ]}
        />
      </main>

      <RegisterUserModalView
        isOpen={isModalOpen}
        onClose={() => setIsModalOpen(false)}
        onSubmit={handleRegisterSubmit}
        isLoading={isRegistering}
      />
    </div>
  );
};
```

---

### 3.5 Hooks de Caso de Uso (`hooks/use-register-user.hook.ts`)

```typescript
// libs/users/ui/src/hooks/use-register-user.hook.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useRepository } from '@monorepo/shared/infrastructure/client';
import { UserRepository, User, UserId, UserName, UserEmail } from '@monorepo/users/domain';
import { UserFormValues } from '../schemas/user-form.schema';

export function useRegisterUser() {
  const repository = useRepository<UserRepository>('userRepository');
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (values: UserFormValues) => {
      const user = User.create(
        new UserId(crypto.randomUUID()),
        new UserName(values.name),
        new UserEmail(values.email)
      );
      await repository.save(user);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}
```

---

### 3.6 Hook de Consulta con TanStack Query (`hooks/use-search-users.hook.ts`)

```typescript
// libs/users/ui/src/hooks/use-search-users.hook.ts
import { useQuery } from '@tanstack/react-query';
import { useRepository } from '@monorepo/shared/infrastructure/client';
import { UserRepository, Criteria } from '@monorepo/users/domain';

export function useSearchUsers(criteria?: Criteria) {
  const repository = useRepository<UserRepository>('userRepository');

  return useQuery({
    queryKey: ['users', criteria],
    queryFn: () => repository.matching(criteria ?? Criteria.empty()),
  });
}
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/{bounded_context}/ui/src/index.ts
export * from './schemas/user-form.schema';
export * from './components/user-form.component';
export * from './views/register-user-modal.view';
export * from './pages/users-page';
export * from './hooks/use-register-user.hook';
export * from './hooks/use-search-users.hook';
```
