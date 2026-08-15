# Especificación: `apps/desktop`

Este módulo constituye el **Punto de Entrada de la Aplicación de Escritorio (Electron + React)**. Su responsabilidad
es inicializar el proceso principal de Electron (**Main Process**), exponer un puente de comunicación seguro
(**Preload Script**) y renderizar la interfaz de usuario en el proceso de renderizado (**Renderer Process**)
reutilizando los Contenedores y Hooks de `libs/{context}/ui`.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Seguridad Estricta de Electron:** Prohibido habilitar `nodeIntegration: true`. El proceso de
  renderizado debe ejecutarse con `contextIsolation: true` y comunicarse con el sistema operativo exclusivamente
  mediante `contextBridge` en `preload.ts`.
* **[ESTRICTO] Reutilización del Dominio y UI:** La interfaz de escritorio renderiza exactamente los mismos Smart
  Components (`*.container.tsx`) de `libs/{context}/ui` e interactúa con la API mediante `HttpUserApiRepository`.
* **[ESTRICTO] Dependencias Permitidas:**
  * `@monorepo/shared/domain` (`type:shared-domain`)
  * `@monorepo/shared/infrastructure/client` (`type:shared-infra-client`)
  * `@monorepo/{bounded_context}/domain` (`type:domain`)
  * `@monorepo/{bounded_context}/infrastructure/client` (`type:infra-client`)
  * `@monorepo/{bounded_context}/ui` (`type:ui`)
* **[ESTRICTO] Tag de Nx:** Debe estar configurado en `project.json` con `tags: ["type:app-desktop", "scope:desktop"]`.

---

## 2. Estructura de Directorios

```text
apps/desktop/
├── src/
│   ├── main/
│   │   ├── main.ts                               # [ESTRICTO] Proceso principal de Electron (BrowserWindow)
│   │   └── ipc-handlers.ts                       # [OPCIONAL] Listeners IPC para diálogos nativos del SO
│   │
│   ├── preload/
│   │   └── preload.ts                            # [ESTRICTO] Exposición segura de APIs nativas a la UI
│   │
│   └── renderer/
│       ├── di/
│       │   └── desktop-dependency-provider.tsx   # [ESTRICTO] Proveedor de DI para el entorno desktop
│       ├── app.tsx                               # [ESTRICTO] Componente raíz del renderer
│       ├── main.tsx                              # [ESTRICTO] Bootstrap de React en Electron
│       └── index.html
├── project.json
├── tsconfig.json
└── tsconfig.app.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `main/` (Main Process de Electron)

#### `main.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// apps/desktop/src/main/main.ts
import { app, BrowserWindow, shell } from 'electron';
import * as path from 'path';

let mainWindow: BrowserWindow | null = null;

function createWindow(): void {
  mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      preload: path.join(__dirname, '../preload/preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true
    }
  });

  if (process.env.NODE_ENV === 'development') {
    mainWindow.loadURL('http://localhost:4200');
  } else {
    mainWindow.loadFile(path.join(__dirname, '../renderer/index.html'));
  }

  mainWindow.webContents.setWindowOpenHandler((details) => {
    shell.openExternal(details.url);
    return { action: 'deny' };
  });

  mainWindow.on('closed', () => {
    mainWindow = null;
  });
}

app.whenReady().then(createWindow);

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

app.on('activate', () => {
  if (mainWindow === null) {
    createWindow();
  }
});
```

---

### 3.2 Bloque: `preload/` (Puente Seguro IPC)

#### `preload.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// apps/desktop/src/types/desktop.d.ts
interface DesktopAPI {
  notify: (message: string) => void;
  getAppVersion: () => Promise<string>;
}

declare global {
  interface Window {
    desktopAPI: DesktopAPI;
  }
}

// apps/desktop/src/preload/preload.ts
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('desktopAPI', {
  notify: (message: string) => {
    ipcRenderer.send('app:notify', message);
  },
  getAppVersion: () => ipcRenderer.invoke('app:version')
});
```

#### `ipc-handlers.ts`
* **Nivel:** **[OPCIONAL]**

```typescript
// apps/desktop/src/main/ipc-handlers.ts
import { ipcMain, Notification } from 'electron';

export function registerIpcHandlers(): void {
  ipcMain.on('app:notify', (_event, message: string) => {
    new Notification({ title: 'App', body: message }).show();
  });
}
```

---

### 3.3 Bloque: `renderer/` (Interfaz de Usuario en Desktop)

#### `desktop-dependency-provider.tsx` & `main.tsx`
* **Nivel:** **[ESTRICTO]**

```tsx
// apps/desktop/src/renderer/di/desktop-dependency-provider.tsx
import React, { useMemo } from 'react';
import {
  FetchHttpClient,
  LocalStorageAdapter,
  RepositoryProvider
} from '@monorepo/shared/infrastructure/client';
import { HttpUserApiRepository } from '@monorepo/users/infrastructure/client';

export const DesktopDependencyProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const repositories = useMemo(() => {
    const apiUrl = 'http://localhost:3000/api';
    const storage = new LocalStorageAdapter();
    const httpClient = new FetchHttpClient(apiUrl, () => storage.get('auth_token'));
    const userRepository = new HttpUserApiRepository(httpClient);

    return { httpClient, storage, userRepository };
  }, []);

  return (
    <RepositoryProvider repositories={repositories}>
      {children}
    </RepositoryProvider>
  );
};
```

```tsx
// apps/desktop/src/renderer/app.tsx
import React from 'react';
import { UsersPage } from '@monorepo/users/ui';

export const App: React.FC = () => {
  return (
    <div className="desktop-window">
      <header className="title-bar">
        <h1>Mi Aplicación de Escritorio</h1>
      </header>
      <main>
        <UsersPage />
      </main>
    </div>
  );
};
```

```tsx
// apps/desktop/src/renderer/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { DesktopDependencyProvider } from './di/desktop-dependency-provider';
import { App } from './app';

const root = ReactDOM.createRoot(document.getElementById('root') as HTMLElement);
root.render(
  <React.StrictMode>
    <DesktopDependencyProvider>
      <App />
    </DesktopDependencyProvider>
  </React.StrictMode>
);
```
