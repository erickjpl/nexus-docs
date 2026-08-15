---
name: Bug Report
about: Reportar un error o comportamiento incorrecto en el sistema
title: "fix(<scope>): <título descriptivo del error>"
---

<!--
GUÍA DE LABELS PARA ESTA ISSUE:
- type::bug
- layer::[domain | application | infra-server | infra-client | ui]
- status::to-do
- priority::[critical | high | medium | low]
- severity::[blocker | major | minor]
- scope::[users | shared | api | web | mobile | desktop]
-->

## 1. Descripción del Error

<!--
Descripción concisa y clara del fallo o anomalía detectada.
-->

## 2. Pasos Exactos para Reproducir

1. Enviar una petición a `...` con los siguientes datos: `...`
2. Ejecutar el caso de uso `...`
3. Observar el error / excepción arrojada.

## 3. Comportamiento Esperado

<!--
¿Qué debería haber ocurrido según las reglas de negocio o especificación técnica?
-->

## 4. Comportamiento Actual Observado

<!--
¿Qué ocurrió realmente? (Ej. Error 500 no controlado, datos corruptos, etc.)
-->

## 5. Evidencia y Logs

```text
[Pegar aquí stack trace, logs de NestJS, respuesta HTTP o captura de pantalla]
```

## 6. Entorno y Contexto

- **Plataforma / App:** [API Backend / Web / Mobile / Desktop]
- **Versión o Commit:** [ej. v1.2.0 o commit sha]
- **Entorno:** [Desarrollo / Staging / Producción]

## 7. Checklist de Corrección

- [ ] Error reproducido de forma consistente con un test que falla (Red en TDD)
- [ ] Causa raíz identificada y solucionada en la capa correspondiente
- [ ] Test unitario o de integración agregado para prevenir regresiones futuras
- [ ] Rama de trabajo: `bugfix/<id_issue>-<nombre-kebab>`
