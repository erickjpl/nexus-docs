## 1. Resumen del Cambio

<!--
Describe brevemente el objetivo y alcance de este Merge Request.
-->

## 2. Issue Vinculado

<!--
Obligatorio: utiliza 'Closes' para auto-cerrar el issue al fusionar.
-->
Closes #

## 3. Tipo de Modificación

- [ ] ✨ `type::feature` - Nueva funcionalidad o caso de uso
- [ ] 🐛 `type::bug` - Corrección de error
- [ ] 🔧 `type::refactor` - Refactorización o limpieza de código
- [ ] 📝 `type::docs` - Actualización de documentación
- [ ] ⚡ `type::perf` - Optimización de rendimiento
- [ ] 🧪 `type::test` - Adición o corrección de pruebas
- [ ] 🚀 `type::hotfix` - Corrección crítica de producción

## 4. Checklist Arquitectónico y de Calidad (Obligatorio)

- [ ] **Onion Architecture:** El Dominio (`domain/`) no tiene dependencias de frameworks ni librerías externas.
- [ ] **Inyección Limpia:** No se utilizaron decoradores `@Injectable()` de NestJS en `domain/` ni `application/`.
- [ ] **Aislamiento de Cliente:** No se importaron paquetes de Node.js o TypeORM en `ui/` ni `infra-client/`.
- [ ] **Eventos de Dominio:** Los eventos se registran con `record()` y se publican con `pullDomainEvents()`.
- [ ] **Linter & Boundaries:** `npx nx affected -t lint` se ejecuta sin errores de `@nx/enforce-module-boundaries`.
- [ ] **Tests Unitarios:** `npx nx affected -t test` pasa al 100% con Object Mothers.
- [ ] **Formateo:** 2 espacios de identación y máximo 120 caracteres por línea.
- [ ] **Auto-eliminación:** Opción "Delete source branch when merge request is accepted" habilitada.

## 5. Instrucciones de Prueba Manual

<!--
Detalla los pasos para que el Reviewer pueda validar este cambio localmente o en entorno de pruebas.
-->
1. Ejecutar: `...`
2. Validar que: `...`
