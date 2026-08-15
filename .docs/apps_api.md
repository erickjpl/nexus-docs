# Especificación: `apps/api`

Este módulo constituye el **Punto de Entrada del Backend (NestJS API Gateway)**. Su única responsabilidad es arrancar
la aplicación de servidor (bootstrap), configurar middlewares globales, pipes de validación, **filtros globales de traducción de excepciones con sobre homogéneo (`ApiResponse`)**, orquestar **Guards de autorización de Nivel 1 (1 Acción = 1 Permiso vía Enum)** y ensamblar los módulos de infraestructura de cada Bounded Context.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Cero Lógica de Negocio:** La aplicación `apps/api` no contiene reglas de negocio, ni entidades, ni
  casos de uso. Es un mero orquestador de inicio (*Composition Root*) que ensambla las librerías de infraestructura.
* **[ESTRICTO] Controladores de Acción Única y Cero `try/catch` Manual:** Los controladores solo orquestan una única acción, construyen el DTO/Comando, delegan al servicio y retornan la respuesta. **Queda prohibido escribir bloques `try/catch` en los controladores**; el manejo de errores es 100% centralizado en el `DomainExceptionFilter`.
* **[ESTRICTO] Estandarización de Respuestas con Envelope `ApiResponse`:** Todas las respuestas HTTP (tanto de éxito como de error) respetan la estructura unificada `ApiResponse`.
* **[ESTRICTO] Autorización de Nivel 1 en Guards (1 Acción = 1 Permiso Dedicado):** Todo endpoint protegido comprueba que exista un usuario autenticado y que posea el `case` exacto del enum de permisos (`PermissionEnum.ACTION`). Prohibido usar cadenas mágicas o permisos genéricos compartidos.
* **[ESTRICTO] Documentación Viva con Bruno (`rest-client/api/`):** Cada endpoint expuesto debe contar obligatoriamente con su archivo `.bru` en la colección de Bruno, incluyendo su bloque `docs` en Markdown.
* **[ESTRICTO] Dependencias Permitidas:**
  * `@monorepo/shared/domain` (`type:shared-domain`)
  * `@monorepo/shared/infrastructure/server` (`type:shared-infra-server`)
  * `@monorepo/{bounded_context}/infrastructure/server` (`type:infra-server`)
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:app-backend", "scope:api"]`.

---

## 2. Estructura de Directorios

```text
apps/api/
├── src/
│   ├── filters/
│   │   └── domain-exception.filter.ts            # [ESTRICTO] Traduce errores de dominio al sobre unificado ApiResponse
│   │
│   ├── interceptors/
│   │   └── logging.interceptor.ts                # [ESTRICTO] Log estructurado [{ClassName}][L{line}] {message}
│   │
│   ├── app.module.ts                             # [ESTRICTO] Módulo raíz que ensambla los contextos
│   └── main.ts                                   # [ESTRICTO] Bootstrap de NestJS
├── rest-client/                                  # 🚀 Colección viva de Bruno
│   └── api/
│       └── users/
│           ├── register-user.bru
│           └── search-users.bru
├── project.json
├── tsconfig.json
└── tsconfig.app.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `filters/domain-exception.filter.ts` (Sobre Unificado `ApiResponse`)

```typescript
// apps/api/src/filters/domain-exception.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpStatus, BadRequestException, UnauthorizedException, HttpException } from '@nestjs/common';
import {
  DomainException,
  InvalidArgumentError,
  DomainNotFoundError,
  DomainConflictError,
  ForbiddenException,
} from '@monorepo/shared/domain';
import { ApiResponse, AppLoggerService } from '@monorepo/shared/infrastructure/server';

@Catch()
export class DomainExceptionFilter implements ExceptionFilter {
  private readonly logger = new AppLoggerService(DomainExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();

    if (exception instanceof DomainNotFoundError) {
      response.status(HttpStatus.NOT_FOUND).json(
        ApiResponse.error(exception.message, HttpStatus.NOT_FOUND),
      );
      return;
    }

    if (exception instanceof InvalidArgumentError) {
      response.status(HttpStatus.BAD_REQUEST).json(
        ApiResponse.error(exception.message, HttpStatus.BAD_REQUEST),
      );
      return;
    }

    if (exception instanceof ForbiddenException) {
      response.status(HttpStatus.FORBIDDEN).json(
        ApiResponse.error(exception.message, HttpStatus.FORBIDDEN),
      );
      return;
    }

    if (exception instanceof DomainConflictError) {
      response.status(HttpStatus.CONFLICT).json(
        ApiResponse.error(exception.message, HttpStatus.CONFLICT),
      );
      return;
    }

    if (exception instanceof BadRequestException) {
      const res = exception.getResponse() as any;
      const messages: string[] = Array.isArray(res.message) ? res.message : [res.message];
      const errors: Record<string, string[]> = {};
      messages.forEach((msg) => {
        const match = msg.match(/^(\w+)\s/);
        const field = match ? match[1] : 'general';
        errors[field] = errors[field] || [];
        errors[field].push(msg);
      });
      response.status(HttpStatus.UNPROCESSABLE_ENTITY).json(
        ApiResponse.error('Validation failed', HttpStatus.UNPROCESSABLE_ENTITY, errors),
      );
      return;
    }

    if (exception instanceof UnauthorizedException) {
      response.status(HttpStatus.UNAUTHORIZED).json(
        ApiResponse.error('Authentication required', HttpStatus.UNAUTHORIZED),
      );
      return;
    }

    if (exception instanceof HttpException) {
      const status = exception.getStatus();
      response.status(status).json(
        ApiResponse.error(exception.message, status),
      );
      return;
    }

    this.logger.error('Unhandled Server Exception', exception);
    response.status(HttpStatus.INTERNAL_SERVER_ERROR).json(
      ApiResponse.error('Internal server error', HttpStatus.INTERNAL_SERVER_ERROR),
    );
  }
}
```

---

### 3.2 Bloque: `app.module.ts` (Ensamblaje del Servidor)

> [!NOTE]
> **Autenticación:** Antes de que `ActionPermissionGuard` pueda validar los permisos, debe existir un Guard de Autenticación (como `JwtAuthGuard`) registrado globalmente o a nivel de controlador que procese el token e inyecte el objeto `user` en el `request`.

```typescript
// apps/api/src/app.module.ts
import { Module } from '@nestjs/common';
import { SharedInfrastructureModule } from '@monorepo/shared/infrastructure/server';
import { UserServerModule } from '@monorepo/users/infrastructure/server';

@Module({
  imports: [
    SharedInfrastructureModule,
    UserServerModule,
  ],
})
export class AppModule {}
```

---

### 3.3 Bloque: Colección Bruno (`rest-client/api/users/register-user.bru`)

```text
environments/
  local.env:
    baseUrl: http://localhost:3000
    authToken: eyJhbGciOiJIUz...
  staging.env:
    baseUrl: https://api.staging.ejemplo.com
    authToken: eyJhbGciOiJIUz...

meta {
  name: Registrar Usuario
  type: http
  seq: 1
}

put {
  url: {{baseUrl}}/api/users/e4b1c2d3-4f5a-6b7c-8d9e-0f1a2b3c4d5e
  body: json
  auth: bearer
}

auth:bearer {
  token: {{authToken}}
}

body:json {
  {
    "id": "e4b1c2d3-4f5a-6b7c-8d9e-0f1a2b3c4d5e",
    "name": "Juan Pérez",
    "email": "juan.perez@ejemplo.com"
  }
}

docs {
  # Registrar Usuario
  
  Crea un nuevo agregado de Usuario en el sistema y despacha el evento de dominio `user.registered`.
  
  ### Autorización (Nivel 1)
  - **Permiso Requerido:** `UserPermissionEnum.USER_CREATE` (`user.create`)
  - **Autenticación:** Bearer Token obligatorio
  
  ### Respuestas
  - **201 Created:** `{ "success": true, "message": "Usuario registrado exitosamente", "data": { ... } }`
  - **400 Bad Request:** Formato de Value Object inválido
  - **403 Forbidden:** Usuario sin permiso `user.create`
  - **409 Conflict:** Email ya registrado
  - **422 Unprocessable Entity:** Payload incompleto
}
```

---

### 3.4 Bloque: `main.ts` (Bootstrap con Filtros y Pipes Globales)

```typescript
// apps/api/src/main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';
import { DomainExceptionFilter } from './filters/domain-exception.filter';
import { AppConfigService, AppLoggerService } from '@monorepo/shared/infrastructure/server';
// import { LoggingInterceptor } from './interceptors/logging.interceptor'; // Asumido que existe

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const logger = new AppLoggerService('Bootstrap');

  // 1. Prefijo Global de API
  app.setGlobalPrefix('api');

  // 2. Filtro Global de Excepciones del Dominio (con ApiResponse Envelope)
  app.useGlobalFilters(new DomainExceptionFilter());
  
  // Opcional: app.useGlobalInterceptors(new LoggingInterceptor());

  // 3. Pipe Global de Validación de DTOs
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    })
  );

  // 4. Habilitación de CORS
  app.enableCors({
    origin: '*',
    methods: 'GET,HEAD,PUT,PATCH,POST,DELETE,OPTIONS',
  });

  const config = app.get(AppConfigService);
  const port = config.get('PORT') || 3000;

  await app.listen(port);
  logger.log(`🚀 API Application is running on: http://localhost:${port}/api`);
}

bootstrap();
```
