# Reporte de auditoría de dependencias — Backend (Deckora-API)

- **Repositorio:** `deckora-api@1.0.0` — rama `dev`
- **Herramienta:** `npm audit`
- **Entorno:** Node v24.15.0 · npm 11.12.1
- **Fecha:** 2026-06-21

## Resumen

| Métrica | Valor |
|---|---|
| Dependencias totales | 409 (122 prod · 287 dev · 35 opcionales) |
| Vulnerabilidades | **2 moderadas** (0 críticas · 0 altas · 0 bajas) |

## Detalle de hallazgos

### 1. `uuid` < 11.1.1 — Severidad moderada
- **Advisory:** [GHSA-w5hq-g745-h8pq](https://github.com/advisories/GHSA-w5hq-g745-h8pq) — *Missing buffer bounds check in v3/v5/v6 when buf is provided* (falta de verificación de límites de buffer cuando se proporciona `buf`).
- **Versión instalada:** `uuid@8.3.2`
- **Origen:** dependencia **transitiva**, no directa.

### 2. `sequelize` — Severidad moderada (heredada)
- **Causa:** depende de una versión vulnerable de `uuid`.
- **Versión instalada:** `sequelize@6.37.8`
- **Cadena de dependencia:** `deckora-api` → `sequelize@6.37.8` → `uuid@8.3.2`

> Ambos hallazgos responden al **mismo problema de fondo**: el único `uuid` vulnerable ingresa a través de `sequelize`. No existen vulnerabilidades en dependencias directas del código de la aplicación.

## Remediación

`npm audit` solo ofrece un arreglo automático **destructivo**:

```
npm audit fix --force   → degradaría a sequelize@3.30.0 (BREAKING CHANGE)
```

**No se recomienda ejecutarlo**: bajar de Sequelize 6 a 3 rompería el ORM completo (modelos, migraciones, sintaxis de consultas). Opciones razonables:

1. **Aceptar y documentar el riesgo** (recomendado por ahora): severidad *moderada*, transitiva y solo explotable si se pasa un `buf` propio a `uuid`, patrón que Sequelize no utiliza de forma insegura. Bajo impacto real.
2. **Forzar resolución de `uuid`:** fijar `uuid@^11.1.1` mediante el campo `overrides` en `package.json` y verificar que Sequelize 6 siga operando con esa versión (requiere ejecutar la suite de pruebas).
3. **Vigilar** una versión de Sequelize 6.x que actualice su `uuid` interno y adoptarla cuando se publique.

## Conclusión

El backend no presenta vulnerabilidades críticas ni altas. Las 2 vulnerabilidades moderadas detectadas son transitivas y provienen de un único paquete (`uuid`) arrastrado por `sequelize`, con bajo impacto real en el contexto de uso del proyecto. Se mantiene como gate de auditoría dentro del pipeline de CI/CD.
