# Documentación Arquitectónica y Estándares de Ingeniería

Este documento define la arquitectura, patrones, capas, principios y estructura de código que rigen este monorepo
en Nx con NestJS en el backend y soporte multiplataforma (Web, Mobile y Desktop). Su cumplimiento es obligatorio
para garantizar bajo acoplamiento, alta cohesión, aislamiento total del dominio y máxima escalabilidad.

---

## Índice

1. [Arquitectura y Paradigmas Globales](#1-arquitectura-y-paradigmas-globales)
   - [Arquitectura Cebolla (Onion Architecture)](#arquitectura-cebolla-onion-architecture)
   - [Event-Driven Architecture (EDA)](#event-driven-architecture-eda)
   - [CQRS (Command Query Responsibility Segregation)](#cqrs-command-query-responsibility-segregation)
   - [Vertical Slice Architecture](#vertical-slice-architecture)
   - [Bounded Context (Contexto Acotado)](#bounded-context-contexto-acotado)
   - [Transactional Outbox & Eventual Consistency](#transactional-outbox--eventual-consistency)
2. [DDD y Conceptos Tácticos](#2-ddd-y-conceptos-tácticos)
   - [DDD (Domain-Driven Design) & Ubiquitous Language](#ddd-domain-driven-design--ubiquitous-language)
   - [Entidad (Entity)](#entidad-entity)
   - [Objeto de Valor (Value Object)](#objeto-de-valor-value-object)
   - [Agregado y Raíz del Agregado](#agregado-y-raíz-del-agregado-aggregate--aggregate-root)
   - [Ciclo de Vida de Eventos de Dominio (Record & Pull)](#ciclo-de-vida-de-eventos-de-dominio-record--pull)
   - [Patrón Criteria (Specification Pattern)](#patrón-criteria-specification-pattern)
   - [Repositorio (Repository Port)](#repositorio-repository-port)
   - [Fábrica (Factory) y Métodos Primitivos](#fábrica-factory-y-métodos-primitivos)
3. [Clasificación y Jerarquía de Servicios](#3-clasificación-y-jerarquía-de-servicios)
   - [Domain Service (Servicio de Dominio)](#domain-service-servicio-de-dominio)
   - [Application Service (Servicio de Aplicación)](#application-service-servicio-de-aplicación)
   - [Infrastructure Service (Servicio de Infraestructura)](#infrastructure-service-servicio-de-infraestructura)
4. [Capas de la Arquitectura por Bounded Context](#4-capas-de-la-arquitectura-por-bounded-context)
   - [Capa 1: `domain/` (El Centro del Modelo)](#capa-1-domain-el-centro-del-modelo)
   - [Capa 2: `application/` (Casos de Uso y Orquestación)](#capa-2-application-casos-de-uso-y-orquestación)
   - [Capa 3: `infrastructure/server/` (Backend & NestJS)](#capa-3-infrastructureserver-adaptadores-backend--nestjs)
   - [Capa 4: `infrastructure/client/` (Frontend)](#capa-4-infrastructureclient-adaptadores-frontend--http-clients)
   - [Capa 5: `ui/` (Presentación Multiplataforma)](#capa-5-ui-presentación-multiplataforma)
5. [Principios de Diseño y Buenas Prácticas](#5-principios-de-diseño-y-buenas-prácticas)
   - [SOLID](#solid)
   - [YAGNI (You Aren't Gonna Need It)](#yagni-you-arent-gonna-need-it)
   - [DRY (Don't Repeat Yourself)](#dry-dont-repeat-yourself)
   - [Ley de Demeter (Principio de Menor Conocimiento)](#ley-de-demeter-principio-de-menor-conocimiento)
6. [Patrones Tácticos de Implementación](#6-patrones-tácticos-de-implementación)
   - [Inyección de Dependencias sin Contaminar el Core](#inyección-de-dependencias-sin-contaminar-el-core)
   - [Manejo de Errores: Either / Result Pattern](#manejo-de-errores-either--result-pattern)
   - [Wrappers (Empaquetadores de Terceros)](#wrappers-empaquetadores-de-terceros)
   - [Event Bus & Failover Publisher](#event-bus--failover-publisher)
   - [Criteria Converters](#criteria-converters)
   - [Nx Module Boundaries & Tags](#nx-module-boundaries--tags)
7. [Estrategia de Testing Multi-Capa](#7-estrategia-de-testing-multi-capa)
   - [TDD (Test-Driven Development)](#tdd-test-driven-development)
   - [Object Mother Pattern & Mocking Semántico](#object-mother-pattern--mocking-semántico)
   - [Pruebas Unitarias (Unit Testing)](#pruebas-unitarias-unit-testing)
   - [Pruebas de Integración (Integration Testing)](#pruebas-integración-integration-testing)
   - [Pruebas de Aceptación / E2E (End-to-End Testing)](#pruebas-de-aceptación--e2e-end-to-end-testing)
8. [Estructura Definitiva del Monorepo](#8-estructura-definitiva-del-monorepo)

---

## 1. Arquitectura y Paradigmas Globales

### Arquitectura Cebolla (Onion Architecture)
Patrón arquitectónico concéntrico basado en la regla de dependencia estricta: las capas exteriores conocen
a las interiores, pero el centro (Dominio) es 100% agnóstico al exterior.
* **Por qué existe:** Evita que los cambios en frameworks, bases de datos o librerías externas fuercen modificaciones
  en las reglas de negocio.
* **Comportamiento exigido:** Las dependencias siempre apuntan hacia adentro. Dominio no depende de nadie;
  Aplicación solo depende de Dominio; Infraestructura implementa los puertos de Dominio y Aplicación.
> **Nota clave:** *Cualquier import de un framework o librería externa (NestJS, React, TypeORM, Prisma, Axios)
> dentro de `domain/` o `application/` rompe la arquitectura y será rechazado automáticamente por el Linter.*

### Event-Driven Architecture (EDA)
Paradigma en el que los componentes de software se comunican de forma asíncrona mediante la emisión y recepción
de eventos de negocio, sin llamadas directas bloqueantes entre módulos.
* **Por qué existe:** Desacopla emisores de receptores en tiempo y espacio, permitiendo que múltiples sistemas
  reaccionen a un cambio de estado sin generar dependencias directas.
* **Comportamiento exigido:** EDA es el único mecanismo permitido para sincronizar estados entre distintos
  Bounded Contexts. El emisor no conoce quién consume el evento ni qué efecto secundario causará.

### CQRS (Command Query Responsibility Segregation)
Patrón que separa estrictamente las operaciones de modificación de estado (Commands) de las operaciones de
lectura/consulta de datos (Queries).
* **Por qué existe:** Los modelos optimizados para escribir (consistencia transaccional, invariantes de agregados)
  son ineficientes para consultar, y viceversa.
* **Comportamiento exigido:**
  * **Commands:** Modifican estado, validan agregados, emiten eventos y **no retornan datos** (salvo identificadores
    o Result de éxito/falla).
  * **Queries:** Consultan datos optimizados (Read Models / DTOs / Proyecciones) y **jamás modifican el estado**
    del sistema ni emiten eventos de dominio.

### Vertical Slice Architecture
Organización del código en "rebanadas" verticales basadas en funcionalidades específicas de negocio (*features* o
casos de uso) en lugar de carpetas técnicas horizontales masivas.
* **Por qué existe:** Agrupa todo lo que cambia junto por una misma razón de negocio, facilitando la navegación,
  el mantenimiento y la eliminación de código obsoleto.
* **Comportamiento exigido:** Modificar o eliminar una funcionalidad debe requerir tocar archivos únicamente dentro
  de su carpeta Slice correspondiente (`application/slices/{use-case}/`), garantizando cero efectos colaterales.

### Bounded Context (Contexto Acotado)
Frontera explícita dentro de la cual un modelo de dominio tiene un significado único, delimitado y totalmente
aislado de otros contextos.
* **Por qué existe:** Evita el antipatrón del "Modelo Único Universal" donde un concepto (ej. `User` o `Order`)
  se sobrecarga con responsabilidades dispares de ventas, facturación, autenticación y soporte.
* **Comportamiento exigido:** Un Bounded Context jamás debe importar directamente las entidades o repositorios
  de otro contexto; la integración entre ellos ocurre exclusivamente a través de Eventos de Dominio o DTOs
  compartidos mediante contratos públicos.

### Transactional Outbox & Eventual Consistency
Patrón para garantizar que la persistencia del agregado y la publicación de sus eventos de dominio sean atómicas
(ambas se ejecutan o ambas fallan).
* **Por qué existe:** En sistemas distribuidos, si se guarda en la base de datos pero el broker de mensajería
  está caído, se pierden eventos y se produce inconsistencia de datos (Dual-Write Problem).
* **Comportamiento exigido:** Los eventos se persisten transaccionalmente junto con el estado del agregado en una
  tabla/colección Outbox, o se despachan mediante un `FailoverPublisher` que reintenta en caso de fallo de red.

---

## 2. DDD y Conceptos Tácticos

### DDD (Domain-Driven Design) & Ubiquitous Language
Enfoque de desarrollo donde la estructura del código refleja fielmente el modelo de negocio y el lenguaje hablado
por los expertos del dominio.
* **Por qué existe:** Cierra la brecha de comunicación entre negocio y tecnología, asegurando que el software
  resuelva el problema real sin ambigüedades.
* **Comportamiento exigido:** Si un término utilizado en el código no existe en el vocabulario del experto de
  negocio, el código está expresando la solución técnica y no el problema del dominio.

### Entidad (Entity)
Objeto del dominio que posee una identidad única y continua a lo largo del tiempo, más allá de sus atributos o
cambios de estado.
* **Por qué existe:** Permite rastrear la evolución de un objeto de negocio específico a través de diferentes
  estados en su ciclo de vida.
* **Comportamiento exigido:** Dos entidades son iguales únicamente si sus IDs son idénticos, independientemente
  de si el resto de sus propiedades difieren.

### Objeto de Valor (Value Object)
Objeto inmutable que carece de identidad conceptual y se define únicamente por el valor de sus atributos
(ej. `UserId`, `UserName`, `UserEmail`).
* **Por qué existe:** Elimina la obsesión por primitivos (*Primitive Obsession*), encapsula validaciones e
  invariantes en su constructor y garantiza que no existan valores inválidos en el dominio.
* **Comportamiento exigido:**
  * Son inmutables. Si cambia un valor, se crea una nueva instancia.
  * Se comparan por igualdad de valor (`equals()`).
  * Implementan métodos de conversión canónicos: `value` (getter del primitivo) y validación en su instanciación.

### Agregado y Raíz del Agregado (Aggregate & Aggregate Root)
Un Agregado es un conjunto de Entidades y Objetos de Valor tratados como una unidad de consistencia atómica.
La Raíz del Agregado (*Aggregate Root*) es la única entidad a través de la cual el mundo exterior interactúa
con el grupo.
* **Por qué existe:** Protege las invariantes y reglas de negocio complejas en un solo lugar, garantizando que el
  estado interno nunca quede inconsistente.
* **Comportamiento exigido:**
  * Ningún objeto externo puede modificar entidades internas de un Agregado directamente; todo cambio debe
    solicitarse al Aggregate Root.
  * Cada Agregado define métodos canónicos `toPrimitives()` (para serialización limpia) y `fromPrimitives()`
    (fábrica estática para reconstrucción sin saltarse reglas).

### Ciclo de Vida de Eventos de Dominio (Record & Pull)
Mecanismo mediante el cual el Agregado registra internamente que algo sucedió, y la Capa de Aplicación extrae
esos eventos para publicarlos en el Event Bus tras la persistencia exitosa.
* **Por qué existe:** Evita efectos secundarios prematuros. Si una operación falla antes de persistir, los eventos
  no deben haberse disparado al exterior.
* **Comportamiento exigido:**
  * El Agregado invoca `this.record(new UserRegisteredDomainEvent(...))` internamente.
  * El Caso de Uso (Application Service) ejecuta el guardado en el repositorio: `await repository.save(user)`.
  * El Caso de Uso extrae los eventos acumulados: `const events = user.pullDomainEvents()`.
  * El Caso de Uso publica los eventos en el bus: `await eventBus.publish(events)`.

### Patrón Criteria (Specification Pattern)
Abstracción de dominio para expresar filtros, ordenamientos y paginaciones dinámicas sin atarse al lenguaje de
consulta de ninguna base de datos específica.
* **Por qué existe:** Evita acoplar la capa de aplicación o controladores a operadores de SQL, TypeORM,
  MongoDB Query Filters o ElasticSearch DSL.
* **Comportamiento exigido:**
  * Se compone de `Filters` (campos, operadores como `=`, `!=`, `>`, `CONTAINS`, y valores), `Order` (campo y
    dirección `ASC`/`DESC`) y límites de paginación (`limit`, `offset`).
  * La capa de infraestructura traduce el objeto `Criteria` al dialecto correspondiente mediante un `CriteriaConverter`.

### Repositorio (Repository Port)
Interfaz pura definida en la capa de dominio que abstrae los métodos necesarios para guardar y recuperar
Agregados completos desde una fuente de persistencia.
* **Por qué existe:** Aplica el principio de Inversión de Dependencias (DIP), permitiendo cambiar de motor de base
  de datos sin tocar una sola línea de lógica de negocio.
* **Comportamiento exigido:**
  * Trabaja con Agregados completos, nunca con tablas ni filas sueltas.
  * En `domain/` solo vive la `interface`; la implementación con TypeORM/Prisma/Mongo vive estrictamente
    en `infrastructure/server/`.

### Fábrica (Factory) y Métodos Primitivos
Mecanismos encapsulados encargados de la creación o reconstrucción de Agregados complejos.
* **Por qué existe:** Asegura que sea imposible instanciar un Agregado en un estado inválido o incompleto para
  el dominio.
* **Comportamiento exigido:**
  * **Creación de Negocio:** Métodos de fábrica (ej. `User.create(id, name, email)`) que disparan eventos de creación.
  * **Reconstrucción de Persistencia:** Método estático `User.fromPrimitives(plainData)` que rehidrata el agregado
    desde base de datos sin disparar eventos de creación.

---

## 3. Clasificación y Jerarquía de Servicios

### Domain Service (Servicio de Dominio)
Operación o regla de negocio pura que no pertenece de forma natural a una sola Entidad u Objeto de Valor, o que
involucra la interacción y validación entre múltiples Agregados.
* **Ubicación:** `libs/{bounded_context}/domain/src/services/`
* **Por qué existe:** Evita forzar lógica de negocio en entidades que no son dueñas naturales de toda la operación
  (ej. verificar unicidad de email consultando un repositorio).
* **Comportamiento exigido:** Es TypeScript puro sin dependencias de infraestructura ni frameworks. No maneja
  transacciones de base de datos ni respuestas HTTP.

### Application Service (Servicio de Aplicación)
Orquestador del caso de uso. Recibe Comandos/Queries, recupera Agregados mediante Repositorios, invoca reglas
de negocio (o Domain Services), persiste cambios y publica Eventos de Dominio.
* **Ubicación:** `libs/{bounded_context}/application/src/slices/{feature}/` o `services/`
* **Por qué existe:** Coordina el flujo de ejecución entre el mundo exterior y el dominio, actuando como la
  fachada del caso de uso.
* **Comportamiento exigido:** No contiene reglas de negocio. **No utiliza decoradores de NestJS (`@Injectable`)**
  para mantenerse 100% portable y testeable en milisegundos.

### Infrastructure Service (Servicio de Infraestructura)
Implementación concreta de operaciones técnicas que interactúan con tecnologías externas (envío de emails via
SendGrid, almacenamiento en AWS S3, pasarelas de pago, logs).
* **Ubicación:** `libs/{bounded_context}/infrastructure/server/services/`
* **Por qué existe:** Encapsula detalles técnicos volátiles detrás de un Puerto (interface) definido previamente
  por el Dominio o la Capa de Aplicación.
* **Comportamiento exigido:** Implementa estrictamente el contrato del puerto y traduce excepciones de terceros a
  tipos `Failure` o excepciones del dominio.

---

## 4. Capas de la Arquitectura por Bounded Context

```text
+-------------------------------------------------------------------------+
|                              CAPA 5: UI                                 |
|             (React Web, React Native Mobile, Electron Desktop)          |
+------------------------------------+------------------------------------+
|   CAPA 4: INFRASTRUCTURE/CLIENT    |    CAPA 3: INFRASTRUCTURE/SERVER   |
| (HTTP Adapters, API SDKs, Storage) | (NestJS Controllers, TypeORM/Mongo)|
+------------------------------------+------------------------------------+
|                         CAPA 2: APPLICATION                             |
|          (Vertical Slices, Command/Query Handlers, DTOs)                |
+-------------------------------------------------------------------------+
|                           CAPA 1: DOMAIN                                |
|    (Aggregate Roots, Value Objects, Domain Events, Criteria, Ports)     |
+-------------------------------------------------------------------------+
```

### Capa 1: `domain/` (El Centro del Modelo)
* **Contenido:** Agregados, Entidades, Value Objects, Domain Events, Domain Services, Factories y Puertos.
* **Regla estricta:** Cero dependencias externas. 100% TypeScript puro.

### Capa 2: `application/` (Casos de Uso y Orquestación)
* **Contenido:** Vertical Slices (Command Handlers, Query Handlers, Event Handlers), DTOs de entrada y respuestas.
* **Regla estricta:** Solo depende de `domain/` y `shared/domain`. Cero decoradores o librerías de frameworks.

### Capa 3: `infrastructure/server/` (Adaptadores Backend & NestJS)
* **Contenido:** Controladores HTTP de NestJS, módulos de NestJS (`*.module.ts`), Providers / Factories de DI,
  Mappers de persistencia, Repositorios TypeORM/Prisma/Mongo, Consumers de RabbitMQ.
* **Regla estricta:** Contiene todo lo que requiere entorno Node.js o el runtime de NestJS. Nunca se importa en
  aplicaciones frontend.

### Capa 4: `infrastructure/client/` (Adaptadores Frontend & HTTP Clients)
* **Contenido:** Adaptadores HTTP (Fetch/Axios wrappers) que implementan puertos de consulta/comando para ser
  consumidos por Web/Mobile, almacenamiento local (AsyncStorage/LocalStorage).
* **Regla estricta:** Solo contiene código compatible con navegadores y React Native. Nunca importa TypeORM,
  NestJS ni librerías de servidor.

### Capa 5: `ui/` (Presentación Multiplataforma)
* **Contenido:** Componentes visuales (React / React Native), Custom Hooks con Result Pattern, contenedores de estado.
* **Regla estricta:** La UI nunca ejecuta reglas de negocio; captura eventos del usuario y delega a los casos de
  uso o adaptadores de cliente.

---

## 5. Principios de Diseño y Buenas Prácticas

### SOLID
1. **Single Responsibility (SRP):** Cada clase, slice o módulo tiene una única razón para cambiar.
2. **Open/Closed (OCP):** El sistema se extiende agregando nuevos adaptadores o handlers sin modificar el core.
3. **Liskov Substitution (LSP):** Cualquier implementación de un Repositorio o Adapter debe ser intercambiable.
4. **Interface Segregation (ISP):** Puertos pequeños y específicos en lugar de interfaces gigantescas.
5. **Dependency Inversion (DIP):** Dominio y Aplicación definen contratos; Infraestructura implementa contratos.

### YAGNI (You Aren't Gonna Need It)
No implementar código, configuraciones o abstracciones basadas en especulaciones de necesidades futuras.

### DRY (Don't Repeat Yourself)
Asegurar que cada regla de negocio tenga una representación única e inequívoca en el sistema.
> **Aclaración clave:** *Duplicar código accidental entre dos Bounded Contexts es preferible a crear una dependencia
> acoplada entre ellos.*

### Ley de Demeter (Principio de Menor Conocimiento)
Un objeto solo debe interactuar con sus colaboradores directos sin indagar en las estructuras internas de terceros.

---

## 6. Patrones Tácticos de Implementación

### Inyección de Dependencias sin Contaminar el Core
Para no contaminar la capa de aplicación con `@Injectable()` ni `@Inject()` de NestJS:
1. Las clases de aplicación se declaran como clases TypeScript puras con constructores estándar.
2. En `infrastructure/server/` se crea un NestJS Module que define los providers utilizando `useFactory`:

```typescript
// libs/{context}/infrastructure/src/server/nest/user.module.ts
@Module({
  controllers: [UserPutController],
  providers: [
    TypeOrmUserRepository,
    {
      provide: 'UserRepository',
      useExisting: TypeOrmUserRepository,
    },
    {
      provide: UserRegistrar,
      useFactory: (repository: UserRepository, eventBus: EventBus) => {
        return new UserRegistrar(repository, eventBus);
      },
      inject: ['UserRepository', 'EventBus'],
    },
  ],
  exports: [UserRegistrar],
})
export class UserNestModule {}
```

### Manejo de Errores: Either / Result Pattern
Patrón funcional donde las funciones devuelven explícitamente un tipo `Result<Success, Failure>` en lugar de
lanzar excepciones incontroladas (`throw`).

### Wrappers (Empaquetadores de Terceros)
Encapsula cualquier SDK o librería externa detrás de una interfaz propia de la organización.

### Event Bus & Failover Publisher
* **In-Memory Bus:** Utilizado para testing y frontend.
* **RabbitMQ / Redis Bus:** Utilizado en producción backend con topología de colas por suscriptor, colas de
  reintento (`retry.*`) y colas de descarte (`dead_letter.*`).
* **Failover Publisher:** Si la conexión con el broker falla, los eventos se persisten en base de datos para su
  posterior reintento automático.

### Criteria Converters
Clases en `infrastructure/` que reciben un objeto `Criteria` y lo transforman en el formato nativo de la BD:
* `TypeOrmCriteriaConverter`: Construye `FindOptionsWhere` y `FindOptionsOrder`.
* `MongoCriteriaConverter`: Construye filtros `{ $and: [...] }` y proyecciones de orden.
* `ElasticCriteriaConverter`: Construye queries con `bodybuilder`.

### Nx Module Boundaries & Tags
Reglas en `.eslintrc.json` / `eslint.config.js` que validan estáticamente las importaciones permitidas:

| Tag de Librería | Puede Importar | Prohibido Importar |
| :--- | :--- | :--- |
| `type:domain` | `type:shared-domain` | `type:application`, `type:infra-*`, `type:ui` |
| `type:application` | `type:domain`, `type:shared-domain`, `type:shared-application` | `type:infra-*`, `type:ui` |
| `type:infra-server` | `type:domain`, `type:application`, `type:shared-*` | `type:ui`, `type:infra-client` |
| `type:infra-client` | `type:domain`, `type:application`, `type:shared-domain` | `type:infra-server` |
| `type:ui` | `type:application`, `type:infra-client`, `type:shared-*` | `type:infra-server` |

---

## 7. Estrategia de Testing Multi-Capa

### TDD (Test-Driven Development)
Ciclo de desarrollo: **Red** (test que falla) $\rightarrow$ **Green** (código mínimo) $\rightarrow$ **Refactor**.

### Object Mother Pattern & Mocking Semántico
Patrón para la creación de datos de prueba deterministas y aleatorios mediante clases dedicadas:
* **Object Mother:** Genera instancias válidas de Value Objects y Agregados (ej. `UserMother.create()`).
* **Mocks Semánticos:** Mocks con aserciones de negocio (ej. `repository.assertSaveHaveBeenCalledWith(user)`).

### Pruebas Unitarias (Unit Testing)
* **Objetivo:** Probar Agregados, Value Objects, Domain Services y Handlers de Aplicación.
* **Regla:** Cero acceso a red, disco o base de datos. Se ejecutan en milisegundos sin levantar frameworks.

### Pruebas de Integración (Integration Testing)
* **Objetivo:** Probar implementaciones de infraestructura (Repositorios TypeORM/Mongo, Event Bus RabbitMQ).
* **Regla:** Utilizan bases de datos efímeras (Docker / Testcontainers) para validar Mappers y CriteriaConverters.

### Pruebas de Aceptación / E2E (End-to-End Testing)
* **Objetivo:** Validar flujos completos desde los endpoints HTTP o interfaz de usuario hasta la base de datos.
* **Regla:** Prueban los caminos críticos de negocio (*Happy Paths*) y validan códigos de estado HTTP y contratos.

---

## 8. Estructura Definitiva del Monorepo

```text
my-monorepo/
├── apps/
│   ├── api/                              # Backend NestJS (Entry Point HTTP/WS/Microservicios)
│   │   ├── src/
│   │   │   ├── config/                   # Configuración del entorno del servidor API
│   │   │   ├── app.module.ts             # Módulo raíz que ensambla los módulos de infraestructura
│   │   │   └── main.ts                   # Bootstrap de NestJS
│   │   └── tsconfig.json
│   │
│   ├── web/                              # React App (Web Entry Point)
│   │   ├── src/
│   │   │   ├── app/
│   │   │   └── main.tsx
│   │   └── tsconfig.json
│   │
│   ├── mobile/                           # React Native / Expo (Mobile Entry Point)
│   │   ├── src/
│   │   │   └── index.ts
│   │   └── tsconfig.json
│   │
│   └── desktop/                          # Electron App (Desktop Entry Point)
│       ├── src/
│       │   ├── main/
│       │   ├── preload/
│       │   └── renderer/
│       └── tsconfig.json
│
├── libs/
│   ├── shared/                           # NÚCLEO COMPARTIDO ENTRE CONTEXTOS
│   │   ├── domain/                       # 1. SHARED DOMAIN KERNEL (0 dependencias / TypeScript Puro)
│   │   │   ├── src/
│   │   │   │   ├── aggregate/            # AggregateRoot, Entity base
│   │   │   │   ├── value-object/         # ValueObject base, StringValueObject, Uuid, NumberValueObject
│   │   │   │   ├── event/                # DomainEvent, DomainEventSubscriber, DomainEventClass
│   │   │   │   ├── criteria/             # Criteria, Filter, FilterField, FilterOperator, Order, OrderType
│   │   │   │   ├── bus/                  # Interfaces de CommandBus, QueryBus, EventBus
│   │   │   │   ├── result/               # Either, Result, Failure
│   │   │   │   └── index.ts
│   │   │   └── tsconfig.json
│   │   │
│   │   ├── application/                  # 2. SHARED APPLICATION (Abstracciones de orquestación)
│   │   │   ├── src/
│   │   │   │   ├── command/              # Command, CommandHandler
│   │   │   │   ├── query/                # Query, QueryHandler, Response
│   │   │   │   └── index.ts
│   │   │   └── tsconfig.json
│   │   │
│   │   ├── infrastructure/               # 3. SHARED INFRASTRUCTURE (Dividida en Server y Client)
│   │   │   ├── server/                   # Adaptadores de Servidor (Node.js / NestJS / RabbitMQ)
│   │   │   │   ├── src/
│   │   │   │   │   ├── bus/              # InMemoryCommandBus, RabbitMqEventBus, FailoverPublisher
│   │   │   │   │   ├── persistence/      # MongoRepository base, TypeOrmRepository base, CriteriaConverters
│   │   │   │   │   ├── config/           # AppConfigService, EnvConfig
│   │   │   │   │   ├── nest/             # Modulos y utilidades compartidas de NestJS
│   │   │   │   │   └── index.ts
│   │   │   │   └── tsconfig.json
│   │   │   │
│   │   │   └── client/                   # Adaptadores de Cliente (Browser / React Native)
│   │   │       ├── src/
│   │   │       │   ├── http/             # Axios/Fetch HTTP client wrapper
│   │   │       │   ├── storage/          # LocalStorage / AsyncStorage adapters
│   │   │       │   └── index.ts
│   │   │       └── tsconfig.json
│   │   │
│   │   └── testing/                      # 4. SHARED TESTING UTILITIES (Object Mothers, Mocks globales)
│   │       ├── src/
│   │       │   ├── mother/               # MotherCreator (Faker wrapper), UuidMother, StringMother
│   │       │   ├── mocks/                # EventBusMock, CommandBusMock
│   │       │   └── index.ts
│   │       └── tsconfig.json
│   │
│   └── {bounded_context}/                # CONTEXTO ACOTADO ESPECÍFICO (Ej: users, auth, billing)
│       │
│       ├── domain/                       # CAPA 1: DOMINIO DEL CONTEXTO (Puro TS)
│       │   ├── src/
│       │   │   ├── model/                # {Aggregate}.ts, {Aggregate}Id.ts, {Entity}.ts
│       │   │   ├── events/               # {Aggregate}RegisteredDomainEvent.ts
│       │   │   ├── exceptions/           # {DomainException}.ts
│       │   │   ├── factories/            # {Aggregate}Factory.ts
│       │   │   ├── services/             # Domain Services (Reglas de negocio complejas)
│       │   │   ├── ports/                # Puertos / Interfaces puras
│       │   │   │   ├── {Aggregate}Repository.ts # Interfaz del repositorio
│       │   │   │   └── {Service}Port.ts         # Contrato para servicios externos
│       │   │   └── index.ts
│       │   └── tsconfig.json
│       │
│       ├── application/                  # CAPA 2: APLICACIÓN / VERTICAL SLICES (Puro TS)
│       │   ├── src/
│       │   │   ├── slices/               # Vertical Slices por caso de uso
│       │   │   │   └── register-{aggregate}/
│       │   │   │       ├── register-{aggregate}.command.ts
│       │   │   │       ├── register-{aggregate}.handler.ts
│       │   │   │       ├── register-{aggregate}.handler.spec.ts
│       │   │   │       └── register-{aggregate}.dto.ts
│       │   │   ├── search-by-criteria/
│       │   │   │   ├── search-{aggregates}-by-criteria.query.ts
│       │   │   │   ├── search-{aggregates}-by-criteria.handler.ts
│       │   │   │   └── {aggregates}.response.ts
│       │   │   ├── event-handlers/       # Subscriptores a eventos de otros contextos
│       │   │   └── index.ts
│       │   └── tsconfig.json
│       │
│       ├── infrastructure/               # CAPA 3 Y 4: INFRAESTRUCTURA (Server & Client separados)
│       │   ├── server/                   # Adaptadores de Backend (NestJS, ORM, RabbitMQ)
│       │   │   ├── src/
│       │   │   │   ├── persistence/      # TypeOrm{Aggregate}Repository, Mongo{Aggregate}Repository
│       │   │   │   │   ├── entities/     # Esquemas de BD (TypeORM Entities / Mongo Schemas)
│       │   │   │   │   └── mappers/      # {Aggregate}Mapper (Dominio <-> Persistencia)
│       │   │   │   ├── http/             # Controladores NestJS
│       │   │   │   │   └── {aggregate}-put.controller.ts
│       │   │   │   ├── nest/             # {Context}NestModule.ts (Registro de Providers y Factories)
│       │   │   │   └── index.ts
│       │   │   └── tsconfig.json
│       │   │
│       │   └── client/                   # Adaptadores de Frontend (Consumo de API para Web/Mobile)
│       │       ├── src/
│       │       │   ├── api/              # Http{Aggregate}ApiRepository (Llamadas HTTP al backend)
│       │       │   └── index.ts
│       │       └── tsconfig.json
│       │
│       ├── ui/                           # CAPA 5: PRESENTACIÓN MULTIPLATAFORMA
│       │   ├── src/
│       │   │   ├── hooks/                # useRegister{Aggregate}.hook.ts
│       │   │   ├── components/           # Componentes visuales UI
│       │   │   └── index.ts
│       │   └── tsconfig.json
│       │
│       └── testing/                      # UTILIDADES DE TEST DEL CONTEXTO
│           ├── src/
│           │   ├── mother/               # {Aggregate}Mother.ts, {ValueObject}Mother.ts
│           │   ├── mocks/                # {Aggregate}RepositoryMock.ts
│           │   └── index.ts
│           └── tsconfig.json
│
├── tools/                                # Generadores Nx para crear Slices y Bounded Contexts
├── nx.json                               # Configuración de Nx Workspace y Cache
├── tsconfig.base.json                    # Path Aliases globales (@monorepo/{context}/...)
└── eslint.config.js                      # Reglas estrictas de Enforce Module Boundaries por Tags
```
