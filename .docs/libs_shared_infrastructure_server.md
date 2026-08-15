# Especificación: `libs/shared/infrastructure/server`

Este módulo constituye la **Infraestructura de Servidor Compartida (Shared Server Infrastructure)**. Contiene las
implementaciones técnicas concretas de los buses (In-Memory y RabbitMQ), adaptadores de persistencia base (TypeORM
y MongoDB), serializadores de eventos de dominio, convertidores de Criteria, servicios de configuración tipados y la
integración con el contenedor de Inyección de Dependencias de NestJS.

---

## 1. Directrices Obligatorias de la Capa

* **[ESTRICTO] Exclusividad de Servidor:** Este módulo utiliza APIs de Node.js, drivers de bases de datos y módulos de
  NestJS. **NUNCA** debe ser importado por aplicaciones de cliente (`apps/web`, `apps/mobile`, `apps/desktop`), ni por
  capas `ui/` o `infrastructure/client/`.
* **[ESTRICTO] Implementación de Puertos:** Toda clase aquí implementa interfaces definidas previamente en
  `@monorepo/shared/domain` o `@monorepo/shared/application`.
* **[ESTRICTO] Tag de Nx:** Configurado en `project.json` con `tags: ["type:shared-infra-server", "scope:shared"]`.
* **[ESTRICTO] Tokens de Inyección Inmutables:** Para evitar colisiones y mantener TypeScript puro en el core, la
  inyección en NestJS se basa en constantes/símbolos (`Tokens`) exportados desde este módulo.

---

## 2. Estructura de Directorios

```text
libs/shared/infrastructure/server/
├── src/
│   ├── bus/
│   │   ├── command/
│   │   │   ├── in-memory-command-bus.ts          # [ESTRICTO] Ejecución de comandos en memoria
│   │   │   └── command-handlers-information.ts   # [ESTRICTO] Registro y resolución de CommandHandlers
│   │   ├── query/
│   │   │   ├── in-memory-query-bus.ts            # [ESTRICTO] Ejecución de consultas en memoria
│   │   │   └── query-handlers-information.ts     # [ESTRICTO] Registro y resolución de QueryHandlers
│   │   ├── event/
│   │   │   ├── in-memory/
│   │   │   │   └── in-memory-async-event-bus.ts  # [ESTRICTO] EventBus local asíncrono
│   │   │   ├── rabbitmq/
│   │   │   │   ├── rabbitmq-event-bus.ts         # [OPCIONAL/PROD] Publicador en RabbitMQ
│   │   │   │   ├── rabbitmq-connection.ts        # [OPCIONAL/PROD] Wrapper de conexión amqplib
│   │   │   │   ├── rabbitmq-configurer.ts        # [OPCIONAL/PROD] Topología de colas y retries
│   │   │   │   ├── rabbitmq-consumer.ts          # [OPCIONAL/PROD] Consumidor base de eventos
│   │   │   │   └── rabbitmq-queue-formatter.ts   # [OPCIONAL/PROD] Formateador de nombres de cola
│   │   │   ├── serializer/
│   │   │   │   ├── domain-event-json-serializer.ts # [ESTRICTO] Serializa DomainEvent a JSON
│   │   │   │   └── domain-event-deserializer.ts   # [ESTRICTO] Reconstruye DomainEvent desde JSON
│   │   │   └── failover/
│   │   │       └── domain-event-failover-publisher.ts # [OPCIONAL/PROD] Persiste eventos fallidos
│   │
│   ├── persistence/
│   │   ├── typeorm/
│   │   │   ├── typeorm.repository.ts             # [OPCIONAL] Repositorio base genérico para TypeORM
│   │   │   └── typeorm-criteria-converter.ts     # [OPCIONAL] Convierte Criteria a TypeORM FindOptions
│   │   └── mongo/
│   │       ├── mongo.repository.ts               # [OPCIONAL] Repositorio base genérico para MongoDB
│   │       ├── mongo-client-factory.ts           # [OPCIONAL] Conexión singleton a MongoDB
│   │       └── mongo-criteria-converter.ts       # [OPCIONAL] Convierte Criteria a MongoDB Query Object
│   │
│   ├── config/
│   │   ├── env-config.interface.ts               # [ESTRICTO] Tipado estricto de variables de entorno
│   │   └── app-config.service.ts                 # [ESTRICTO] Servicio para acceso seguro a variables
│   │
│   ├── nest/
│   │   ├── tokens.ts                             # [ESTRICTO] Injection tokens para DI de NestJS
│   │   └── shared-infrastructure.module.ts       # [ESTRICTO] Módulo global de infraestructura NestJS
│   │
│   └── index.ts                                  # [ESTRICTO] Barril público de exportación
├── project.json
├── tsconfig.json
└── tsconfig.lib.json
```

---

## 3. Especificación Detallada por Componente

### 3.1 Bloque: `bus/command/`

#### `in-memory-command-bus.ts` & `command-handlers-information.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Implementa el puerto `CommandBus` de dominio permitiendo desacoplar el emisor del comando de
  su ejecutor físico.

```typescript
// command-handlers-information.ts
import { Command, CommandClass, CommandHandler } from '@monorepo/shared/application';

export class CommandHandlersInformation {
  private commandHandlersMap: Map<CommandClass<Command>, CommandHandler<Command>> = new Map();

  constructor(commandHandlers: Array<CommandHandler<Command>>) {
    commandHandlers.forEach((handler) => {
      this.commandHandlersMap.set(handler.subscribedTo(), handler);
    });
  }

  public search(command: Command): CommandHandler<Command> {
    const commandHandler = this.commandHandlersMap.get(command.constructor as CommandClass<Command>);
    if (!commandHandler) {
      throw new Error(`The command <${command.constructor.name}> has not a command handler associated`);
    }
    return commandHandler;
  }
}

// in-memory-command-bus.ts
import { Command, CommandBus } from '@monorepo/shared/domain';
import { CommandHandlersInformation } from './command-handlers-information';

export class InMemoryCommandBus implements CommandBus {
  constructor(private readonly commandHandlersInformation: CommandHandlersInformation) {}

  async dispatch(command: Command): Promise<void> {
    const handler = this.commandHandlersInformation.search(command);
    await handler.handle(command);
  }
}
```

---

### 3.2 Bloque: `bus/query/`

#### `in-memory-query-bus.ts` & `query-handlers-information.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Implementa el puerto `QueryBus` de dominio resolviendo la consulta sincrónicamente hacia su
  `QueryHandler` correspondiente y retornando su `Response`.

```typescript
// in-memory-query-bus.ts
import { Query, QueryBus, Response } from '@monorepo/shared/domain';
import { QueryHandlersInformation } from './query-handlers-information';

export class InMemoryQueryBus implements QueryBus {
  constructor(private readonly queryHandlersInformation: QueryHandlersInformation) {}

  async ask<R extends Response>(query: Query): Promise<R> {
    const handler = this.queryHandlersInformation.search(query);
    return (await handler.handle(query)) as R;
  }
}
```

---

### 3.3 Bloque: `bus/event/`

#### `domain-event-json-serializer.ts` & `domain-event-deserializer.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Para transmitir eventos de dominio a través de la red o guardarlos en base de datos, deben
  serializarse a un formato estándar canónico y poder reconstruirse fielmente.

```typescript
// domain-event-json-serializer.ts
import { DomainEvent } from '@monorepo/shared/domain';

export class DomainEventJsonSerializer {
  static serialize(event: DomainEvent): string {
    return JSON.stringify({
      data: {
        id: event.eventId,
        type: event.eventName,
        occurred_on: event.occurredOn.toISOString(),
        aggregate_id: event.aggregateId,
        attributes: event.toPrimitives()
      }
    });
  }
}

// domain-event-deserializer.ts
import { DomainEvent, DomainEventClass } from '@monorepo/shared/domain';

export class DomainEventDeserializer extends Map<string, DomainEventClass> {
  static configure(subscribers: Array<{ subscribedTo(): Array<DomainEventClass> }>): DomainEventDeserializer {
    const mapping = new DomainEventDeserializer();
    subscribers.forEach((subscriber) => {
      subscriber.subscribedTo().forEach((eventClass) => mapping.set(eventClass.EVENT_NAME, eventClass));
    });
    return mapping;
  }

  deserialize(eventString: string): DomainEvent {
    const { data } = JSON.parse(eventString);
    const { type, aggregate_id, attributes, id, occurred_on } = data;
    const eventClass = super.get(type);

    if (!eventClass) {
      throw new Error(`DomainEvent mapping not found for event <${type}>`);
    }

    return eventClass.fromPrimitives({
      aggregateId: aggregate_id,
      attributes,
      occurredOn: new Date(occurred_on),
      eventId: id
    });
  }
}
```

---

#### `in-memory-async-event-bus.ts`
* **Nivel:** **[ESTRICTO]**
* **Por qué existe:** Permite publicar y procesar eventos dentro del mismo proceso sin infraestructura pesada.

---

### 3.4 Bloque: `persistence/`

#### Repositorios Base y `CriteriaConverter`
* **Nivel:** **[OPCIONAL según ORM seleccionado]**

```typescript
// typeorm-criteria-converter.ts
import { Criteria, Filter, Operator } from '@monorepo/shared/domain';
import { FindOptionsWhere, Equal, Not, LessThan, MoreThan, Like } from 'typeorm';

export class TypeOrmCriteriaConverter {
  convert<T>(criteria: Criteria): { where: FindOptionsWhere<T>; take?: number; skip?: number } {
    const where: any = {};
    criteria.filters.filters.forEach((filter) => {
      where[filter.field.value] = this.mapOperator(filter);
    });

    return {
      where,
      take: criteria.limit,
      skip: criteria.offset
    };
  }

  private mapOperator(filter: Filter) {
    switch (filter.operator.value) {
      case Operator.EQUAL: return Equal(filter.value.value);
      case Operator.NOT_EQUAL: return Not(Equal(filter.value.value));
      case Operator.GT: return MoreThan(filter.value.value);
      case Operator.LT: return LessThan(filter.value.value);
      case Operator.CONTAINS: return Like(`%${filter.value.value}%`);
      default: return Equal(filter.value.value);
    }
  }
}
```

---

### 3.5 Bloque: `nest/` (Integración con NestJS)

#### `tokens.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
export const SHARED_TOKENS = {
  COMMAND_BUS: Symbol('COMMAND_BUS'),
  QUERY_BUS: Symbol('QUERY_BUS'),
  EVENT_BUS: Symbol('EVENT_BUS'),
  CONFIG_SERVICE: Symbol('CONFIG_SERVICE')
} as const;
```

---

#### `shared-infrastructure.module.ts`
* **Nivel:** **[ESTRICTO]**

```typescript
import { Global, Module } from '@nestjs/common';
import { SHARED_TOKENS } from './tokens';
import { InMemoryCommandBus } from '../bus/command/in-memory-command-bus';
import { CommandHandlersInformation } from '../bus/command/command-handlers-information';
import { InMemoryQueryBus } from '../bus/query/in-memory-query-bus';
import { QueryHandlersInformation } from '../bus/query/query-handlers-information';
import { InMemoryAsyncEventBus } from '../bus/event/in-memory/in-memory-async-event-bus';
import { AppConfigService } from '../config/app-config.service';

@Global()
@Module({
  providers: [
    AppConfigService,
    {
      provide: SHARED_TOKENS.CONFIG_SERVICE,
      useExisting: AppConfigService
    },
    {
      provide: SHARED_TOKENS.COMMAND_BUS,
      useFactory: () => new InMemoryCommandBus(new CommandHandlersInformation([]))
    },
    {
      provide: SHARED_TOKENS.QUERY_BUS,
      useFactory: () => new InMemoryQueryBus(new QueryHandlersInformation([]))
    },
    {
      provide: SHARED_TOKENS.EVENT_BUS,
      useFactory: () => new InMemoryAsyncEventBus()
    }
  ],
  exports: [
    SHARED_TOKENS.CONFIG_SERVICE,
    SHARED_TOKENS.COMMAND_BUS,
    SHARED_TOKENS.QUERY_BUS,
    SHARED_TOKENS.EVENT_BUS
  ]
})
export class SharedInfrastructureModule {}
```

---

## 4. Barril de Exportación (`src/index.ts`)

```typescript
// libs/shared/infrastructure/server/src/index.ts
export * from './bus/command/in-memory-command-bus';
export * from './bus/command/command-handlers-information';

export * from './bus/query/in-memory-query-bus';
export * from './bus/query/query-handlers-information';

export * from './bus/event/in-memory/in-memory-async-event-bus';
export * from './bus/event/serializer/domain-event-json-serializer';
export * from './bus/event/serializer/domain-event-deserializer';

export * from './config/env-config.interface';
export * from './config/app-config.service';

export * from './nest/tokens';
export * from './nest/shared-infrastructure.module';
```
