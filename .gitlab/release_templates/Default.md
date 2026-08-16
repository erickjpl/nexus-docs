## Versión: v<MAJOR>.<MINOR>.<PATCH>

**Fecha de Publicación:** AAAA-MM-DD  
**Rama Origen:** `release/v<MAJOR>.<MINOR>.<PATCH>`  
**Tag Inmutable:** `v<MAJOR>.<MINOR>.<PATCH>`  

---

### 📝 Notas de la Versión

#### ✨ Nuevas Funcionalidades (Added)
- `scope`: Descripción de la funcionalidad implementada ([#issue_id](link))

#### 🐛 Corrección de Errores (Fixed)
- `scope`: Descripción del bug solucionado ([#issue_id](link))

#### 🔧 Refactorizaciones y Mejoras (Changed)
- `scope`: Descripción de la mejora técnica interna ([#issue_id](link))

#### ⚠️ Breaking Changes / Migraciones Requeridas
- Detalle de cambios que requieren migraciones de base de datos o variables de entorno nuevas.

---

### ✅ Checklist de Verificación de Release

- [ ] Todas las features planificadas están mergeadas en la rama de release.
- [ ] La Release Candidate (`vX.Y.Z-rc.N`) fue validada y aprobada por QA en Staging.
- [ ] Cero incidencias críticas (`severity::blocker` o `severity::major`) abiertas.
- [ ] Pipeline completo de CI/CD (`nx run-many -t lint test build`) exitoso al 100%.
- [ ] Archivo `CHANGELOG.md` actualizado en la raíz del proyecto.
- [ ] Merge Request aprobado hacia `main` y ejecutado con Tag.
- [ ] Retro-merge completado hacia `develop` para sincronizar cambios y parches.
