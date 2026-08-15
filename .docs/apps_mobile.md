# Especificación: `apps/mobile`

Este módulo constituye el **Punto de Entrada de la Aplicación Móvil (React Native / Expo)**. Su responsabilidad es
inicializar el entorno nativo, configurar el adaptador de almacenamiento local móvil (`AsyncStorageAdapter`), proveer
las dependencias mediante React Context y orquestar las pantallas y la navegación con **React Navigation**, reutilizando
el 100% de la lógica de dominio y los adaptadores de cliente compartidos.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Reutilización del Núcleo:** La app móvil consume directamente las mismas interfaces de
  `@monorepo/shared/domain`, `@monorepo/shared/infrastructure/client` y los repositorios de
  `@monorepo/{bounded_context}/infrastructure/client`. No se duplica código de red ni modelos de dominio.
* **[ESTRICTO] Tag de Nx:** Debe estar configurado en `project.json` con `tags: ["type:app-mobile", "scope:mobile"]`.
* **[ESTRICTO] Prohibición de Dependencias de Servidor o DOM:** Prohibido importar módulos de `infrastructure/server`
  o paquetes exclusivos de navegador (`react-dom`, `localStorage`).

---

## 2. Estructura de Directorios

```text
apps/mobile/
├── src/
│   ├── app/
│   │   ├── di/
│   │   │   ├── async-storage.adapter.ts          # [ESTRICTO] Implementación de KeyValueStorage con AsyncStorage
│   │   │   └── mobile-dependency-provider.tsx    # [ESTRICTO] Inyección de dependencias para Mobile
│   │   ├── navigation/
│   │   │   └── app-navigator.tsx                 # [ESTRICTO] Configuración de React Navigation
│   │   └── app.tsx                               # [ESTRICTO] Componente raíz con NavigationContainer
│   └── index.ts                                  # [ESTRICTO] Entry point registrado con Expo/React Native
├── app.json
├── project.json
├── tsconfig.json
└── tsconfig.app.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `di/` (Adaptadores Nativos e Inyección de Dependencias)

#### `async-storage.adapter.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// apps/mobile/src/app/di/async-storage.adapter.ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { KeyValueStorage } from '@monorepo/shared/infrastructure/client';

export class AsyncStorageAdapter implements KeyValueStorage {
  async get(key: string): Promise<string | null> {
    return AsyncStorage.getItem(key);
  }

  async set(key: string, value: string): Promise<void> {
    await AsyncStorage.setItem(key, value);
  }

  async remove(key: string): Promise<void> {
    await AsyncStorage.removeItem(key);
  }

  async clear(): Promise<void> {
    await AsyncStorage.clear();
  }
}
```

---

#### `mobile-dependency-provider.tsx`
* **Nivel:** **[ESTRICTO]**

```tsx
// apps/mobile/src/app/di/mobile-dependency-provider.tsx
import React, { createContext, useContext, useMemo } from 'react';
import { HttpClient, FetchHttpClient, KeyValueStorage } from '@monorepo/shared/infrastructure/client';
import { UserRepository } from '@monorepo/users/domain';
import { HttpUserApiRepository } from '@monorepo/users/infrastructure/client';
import { AsyncStorageAdapter } from './async-storage.adapter';

export interface MobileDependencies {
  httpClient: HttpClient;
  storage: KeyValueStorage;
  userRepository: UserRepository;
}

const MobileDependencyContext = createContext<MobileDependencies | null>(null);

export const MobileDependencyProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const dependencies = useMemo<MobileDependencies>(() => {
    const apiUrl = 'https://api.tudominio.com/api';

    const storage = new AsyncStorageAdapter();
    const httpClient = new FetchHttpClient(apiUrl);
    const userRepository = new HttpUserApiRepository(httpClient);

    return {
      httpClient,
      storage,
      userRepository
    };
  }, []);

  return (
    <MobileDependencyContext.Provider value={dependencies}>
      {children}
    </MobileDependencyContext.Provider>
  );
};

export function useMobileDependencies(): MobileDependencies {
  const context = useContext(MobileDependencyContext);
  if (!context) {
    throw new Error('useMobileDependencies must be used within <MobileDependencyProvider>');
  }
  return context;
}
```

---

### 3.2 Bloque: `navigation/` (Navegación Móvil)

#### `app-navigator.tsx`
* **Nivel:** **[ESTRICTO]**

```tsx
// apps/mobile/src/app/navigation/app-navigator.tsx
import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { useMobileDependencies } from '../di/mobile-dependency-provider';
import { RegisterUserContainer, UsersListContainer } from '@monorepo/users/ui';
import { CriteriaMother } from '@monorepo/shared/testing';

export type RootStackParamList = {
  UsersList: undefined;
  RegisterUser: undefined;
};

const Stack = createNativeStackNavigator<RootStackParamList>();

function UsersListScreen() {
  const { userRepository } = useMobileDependencies();
  return (
    <UsersListContainer
      repository={userRepository}
      initialCriteria={CriteriaMother.empty()}
    />
  );
}

function RegisterUserScreen({ navigation }: any) {
  const { userRepository } = useMobileDependencies();
  return (
    <RegisterUserContainer
      repository={userRepository}
      onUserRegistered={() => navigation.goBack()}
    />
  );
}

export const AppNavigator: React.FC = () => {
  return (
    <Stack.Navigator initialRouteName="UsersList">
      <Stack.Screen name="UsersList" component={UsersListScreen} options={{ title: 'Usuarios' }} />
      <Stack.Screen name="RegisterUser" component={RegisterUserScreen} options={{ title: 'Registrar Usuario' }} />
    </Stack.Navigator>
  );
};
```

---

### 3.3 Bloque: `app.tsx` & `index.ts` (Bootstrap Móvil)

#### `app.tsx`
* **Nivel:** **[ESTRICTO]**

```tsx
// apps/mobile/src/app/app.tsx
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { MobileDependencyProvider } from './di/mobile-dependency-provider';
import { AppNavigator } from './navigation/app-navigator';

export const App: React.FC = () => {
  return (
    <MobileDependencyProvider>
      <NavigationContainer>
        <AppNavigator />
      </NavigationContainer>
    </MobileDependencyProvider>
  );
};
```

---

#### `index.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// apps/mobile/src/index.ts
import { registerRootComponent } from 'expo';
import { App } from './app/app';

registerRootComponent(App);
```
