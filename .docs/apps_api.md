# Especificación: `apps/api`

Este módulo constituye el **Punto de Entrada del Backend (NestJS API Gateway)**. Su única responsabilidad es arrancar
la aplicación de servidor (bootstrap), configurar middlewares, pipes de validación global, filtros de traducción de
excepciones de dominio a códigos HTTP, y ensamblar los módulos de infraestructura de cada Bounded Context.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Cero Lógica de Negocio:** La aplicación `apps/api` no contiene reglas de negocio, ni entidades, ni
  casos de uso. Es un mero orquestador de inicio (*Composition Root*) que junta las librerías de infraestructura.
* **[ESTRICTO] Dependencias Permitidas:**
  * `@monorepo/shared/domain` (`type:shared-domain`)
  * `@monorepo/shared/infrastructure/server` (`type:shared-infra-server`)
  * `@monorepo/{bounded_context}/infrastructure/server` (`type:infra-server`)
* **[ESTRICTO] Tag de Nx:** Debe estar configurado en `project.json` con `tags: ["type:app-backend", "scope:api"]`.
* **[ESTRICTO] Filtro Global de Excepciones de Dominio:** Ningún error de dominio (`InvalidArgumentError`, excepciones
  o `Failures`) debe escaparse como error 500 no controlado. Un filtro global (`DomainExceptionFilter`) debe capturarlas
  y responder con el código HTTP correspondiente (400 Bad Request, 404 Not Found, 409 Conflict).

---

## 2. Estructura de Directorios

```text
apps/api/
├── src/
│   ├── filters/
│   │   └── domain-exception.filter.ts            # [ESTRICTO] Traduce errores de dominio a HTTP status
│   │
│   ├── interceptors/
│   │   └── logging.interceptor.ts                # [OPCIONAL] Registro de tiempos de respuesta y peticiones
│   │
│   ├── app.module.ts                             # [ESTRICTO] Módulo raíz que ensambla los contextos
│   └── main.ts                                   # [ESTRICTO] Bootstrap de NestJS
├── project.json
├── tsconfig.json
└── tsconfig.app.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `filters/` (Traducción de Errores de Dominio)

#### `domain-exception.filter.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Evita que el dominio dependa de `@nestjs/common` o `HttpException`. Las clases de dominio
  lanzan errores de TypeScript puro (`InvalidArgumentError`), y este filtro de infraestructura los traduce a
  respuestas JSON semánticas con status HTTP 400/404/409.

```typescript
// apps/api/src/filters/domain-exception.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpStatus } from '@nestjs/common';
import { Response, Request } from 'express';
import {
  DomainException,
  InvalidArgumentError,
  DomainNotFoundError,
  DomainConflictError
} from '@monorepo/shared/domain';

@Catch()
export class DomainExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    // 1. Invariantes de Value Objects y validaciones de formato
    if (exception instanceof InvalidArgumentError) {
      response.status(HttpStatus.BAD_REQUEST).json({
        statusCode: HttpStatus.BAD_REQUEST,
        error: 'Bad Request',
        message: exception.message,
        timestamp: new Date().toISOString(),
        path: request.url
      });
      return;
    }

    // 2. Errores de entidad no encontrada
    if (exception instanceof DomainNotFoundError) {
      response.status(HttpStatus.NOT_FOUND).json({
        statusCode: HttpStatus.NOT_FOUND,
        error: 'Not Found',
        message: exception.message,
        timestamp: new Date().toISOString(),
        path: request.url
      });
      return;
    }

    // 3. Conflictos de regla de negocio o duplicados
    if (exception instanceof DomainConflictError) {
      response.status(HttpStatus.CONFLICT).json({
        statusCode: HttpStatus.CONFLICT,
        error: 'Conflict',
        message: exception.message,
        timestamp: new Date().toISOString(),
        path: request.url
      });
      return;
    }

    // 4. Otras excepciones de dominio no mapeadas específicamente
    if (exception instanceof DomainException) {
      response.status(HttpStatus.UNPROCESSABLE_ENTITY).json({
        statusCode: HttpStatus.UNPROCESSABLE_ENTITY,
        error: 'Unprocessable Entity',
        message: exception.message,
        timestamp: new Date().toISOString(),
        path: request.url
      });
      return;
    }

    // 5. Errores HTTP nativos de NestJS (HttpException)
    if (typeof exception === 'object' && exception !== null && 'getStatus' in exception && typeof (exception as any).getStatus === 'function') {
      const httpException = exception as { getStatus(): number; getResponse(): any };
      const status = httpException.getStatus();
      const res = httpException.getResponse();
      response.status(status).json(typeof res === 'object' ? res : { statusCode: status, message: res });
      return;
    }

    // 6. Errores no controlados (500 Internal Server Error)
    console.error('Unhandled Exception:', exception);
    response.status(HttpStatus.INTERNAL_SERVER_ERROR).json({
      statusCode: HttpStatus.INTERNAL_SERVER_ERROR,
      error: 'Internal Server Error',
      message: 'An unexpected internal error occurred',
      timestamp: new Date().toISOString(),
      path: request.url
    });
  }
}
```

---

### 3.2 Bloque: `app.module.ts` (Ensamblado de Bounded Contexts)

#### `app.module.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Es el módulo raíz que importa el `SharedInfrastructureModule` global y los módulos de
  servidor de cada Bounded Context (`UserServerModule`, `AuthServerModule`, etc.).

```typescript
// apps/api/src/app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { SharedInfrastructureModule, AppConfigService } from '@monorepo/shared/infrastructure/server';
import { UserServerModule, UserEntitySchema } from '@monorepo/users/infrastructure/server';

@Module({
  imports: [
    // 1. Núcleo Compartido
    SharedInfrastructureModule,

    // 2. Conexión a Base de Datos (TypeORM / PostgreSQL)
    TypeOrmModule.forRootAsync({
      inject: [AppConfigService],
      useFactory: (config: AppConfigService) => ({
        type: 'postgres',
        url: config.get('DATABASE_URL'),
        entities: [UserEntitySchema],
        synchronize: config.get('NODE_ENV') !== 'production',
        logging: config.get('NODE_ENV') === 'development'
      })
    }),

    // 3. Módulos de Servidor de los Bounded Contexts
    UserServerModule
  ]
})
export class AppModule {}
```

---

### 3.3 Bloque: `main.ts` (Bootstrap del Servidor)

#### `main.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
// apps/api/src/main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, Logger } from '@nestjs/common';
import { AppModule } from './app.module';
import { DomainExceptionFilter } from './filters/domain-exception.filter';
import { AppConfigService } from '@monorepo/shared/infrastructure/server';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const logger = new Logger('Bootstrap');

  // 1. Prefijo Global de API
  app.setGlobalPrefix('api');

  // 2. Filtro Global de Excepciones del Dominio
  app.useGlobalFilters(new DomainExceptionFilter());

  // 3. Pipe Global de Validación de DTOs
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true
    })
  );

  // 4. Habilitación de CORS
  app.enableCors({
    origin: '*',
    methods: 'GET,HEAD,PUT,PATCH,POST,DELETE,OPTIONS'
  });

  // 5. Lectura de Puerto desde ConfigService
  const config = app.get(AppConfigService);
  const port = config.get('PORT') || 3000;

  await app.listen(port);
  logger.log(`🚀 API Application is running on: http://localhost:${port}/api`);
}

bootstrap();
```
