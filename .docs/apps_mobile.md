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
import React, { useMemo } from 'react';
import { FetchHttpClient, RepositoryProvider } from '@monorepo/shared/infrastructure/client';
import { HttpUserApiRepository } from '@monorepo/users/infrastructure/client';
import { AsyncStorageAdapter } from './async-storage.adapter';

export const MobileDependencyProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const repositories = useMemo(() => {
    const apiUrl = 'https://api.tudominio.com/api';

    const storage = new AsyncStorageAdapter();
    const httpClient = new FetchHttpClient(apiUrl, () => storage.get('auth_token'));
    const userRepository = new HttpUserApiRepository(httpClient);

    return {
      httpClient,
      storage,
      userRepository
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

### 3.2 Bloque: `navigation/` (Navegación Móvil)

#### `app-navigator.tsx`
* **Nivel:** **[ESTRICTO]**

```tsx
// apps/mobile/src/app/navigation/app-navigator.tsx
import React from 'react';
import { NativeStackScreenProps, createNativeStackNavigator } from '@react-navigation/native-stack';
import { UsersPage } from '@monorepo/users/ui';

export type RootStackParamList = {
  UsersList: undefined;
};

const Stack = createNativeStackNavigator<RootStackParamList>();

type UsersListScreenProps = NativeStackScreenProps<RootStackParamList, 'UsersList'>;

function UsersListScreen({ navigation }: UsersListScreenProps) {
  return <UsersPage />;
}

export const AppNavigator: React.FC = () => {
  return (
    <Stack.Navigator initialRouteName="UsersList">
      <Stack.Screen name="UsersList" component={UsersListScreen} options={{ title: 'Usuarios' }} />
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
import 'react-native-get-random-values';
import { registerRootComponent } from 'expo';
import { App } from './app/app';

registerRootComponent(App);
```

> **Nota sobre Polyfills:** React Native no implementa `crypto.getRandomValues()` por defecto, el cual es requerido por muchas implementaciones de UUID como `uuid` v4 que se usa en `@monorepo/shared/domain` para la generación de `UuidMother.random()`. Es obligatorio importar el polyfill antes que cualquier otra librería.

> **Nota sobre UI Interop:** Los componentes de `libs/{context}/ui` que se reutilicen en React Native deben utilizar primitivas agnósticas (por ejemplo utilizando librerías como `react-native-web` o Tamagui) o tener implementaciones separadas por plataforma, dado que los elementos del DOM tradicional (como `<div>` o `<span>`) causarán errores en React Native.
