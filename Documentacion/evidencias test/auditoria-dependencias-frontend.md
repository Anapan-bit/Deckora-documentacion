# Reporte de auditoría de dependencias — Frontend (Deckora-Web)

- **Repositorio:** Deckora-Web — rama `test/cobertura-frontend-80`
- **Herramienta:** `npm audit`
- **Entorno:** Node v24.15.0 · npm 11.12.1
- **Fecha:** 2026-06-21

## Resumen

| Métrica | Valor |
|---|---|
| Dependencias totales | 514 (127 prod · 388 dev · 32 opcionales) |
| Vulnerabilidades | **7** → 5 altas · 1 moderada · 1 baja (0 críticas) |

## Detalle de hallazgos

| Paquete | Severidad | Rango vulnerable | Fix disponible |
|---|---|---|---|
| `react-router` | **alta** | 7.0.0 – 7.15.0 | sí (no breaking) |
| `react-router-dom` | **alta** | 7.0.0-pre.0 – 7.14.2 | sí (no breaking) |
| `undici` | **alta** | 7.0.0 – 7.27.2 | sí (no breaking) |
| `vite` | **alta** | 8.0.0 – 8.0.15 | sí (no breaking) |
| `ws` | **alta** | 8.0.0 – 8.20.1 | sí (no breaking) |
| `brace-expansion` | moderada | 5.0.2 – 5.0.5 | sí (no breaking) |
| `@babel/core` | baja | <= 7.29.0 | sí (no breaking) |

### Vulnerabilidades altas

- **`react-router` / `react-router-dom`**
  - [GHSA-8x6r-g9mw-2r78](https://github.com/advisories/GHSA-8x6r-g9mw-2r78) — DoS por expansión de rutas sin límite en el endpoint `__manifest`.
  - [GHSA-84g9-w2xq-vcv6](https://github.com/advisories/GHSA-84g9-w2xq-vcv6) — posible CSRF en peticiones de documento `PUT`/`PATCH`/`DELETE`.
  - `react-router-dom` hereda la vulnerabilidad de `react-router`.
- **`undici`** (7 advisories) — bypass de validación de certificado TLS vía SOCKS5, divulgación de información entre usuarios por caché compartida, inyección de cabeceras vía `Set-Cookie`, DoS en cliente WebSocket, enrutamiento cross-origin por reutilización de pool SOCKS5, envenenamiento de cola de respuestas por reutilización de socket keep-alive y degradación del atributo `SameSite`.
- **`vite`**
  - [GHSA-v6wh-96g9-6wx3](https://github.com/advisories/GHSA-v6wh-96g9-6wx3) — `launch-editor`: divulgación de hash NTLMv2 vía rutas UNC en Windows.
  - [GHSA-fx2h-pf6j-xcff](https://github.com/advisories/GHSA-fx2h-pf6j-xcff) — bypass de `server.fs.deny` en rutas alternas de Windows.
- **`ws`**
  - [GHSA-58qx-3vcg-4xpx](https://github.com/advisories/GHSA-58qx-3vcg-4xpx) — divulgación de memoria no inicializada.
  - [GHSA-96hv-2xvq-fx4p](https://github.com/advisories/GHSA-96hv-2xvq-fx4p) — agotamiento de memoria (DoS) por fragmentos y chunks diminutos.

### Vulnerabilidad moderada

- **`brace-expansion`** — [GHSA-jxxr-4gwj-5jf2](https://github.com/advisories/GHSA-jxxr-4gwj-5jf2): un rango numérico grande anula la protección documentada `max` contra DoS.

### Vulnerabilidad baja

- **`@babel/core`** — [GHSA-4x5r-pxfx-6jf8](https://github.com/advisories/GHSA-4x5r-pxfx-6jf8): lectura arbitraria de archivos vía comentario `sourceMappingURL`.

## Remediación

**Todas las vulnerabilidades tienen corrección no destructiva.** Varias (`undici`, `vite`, `ws`, `launch-editor`) afectan principalmente al servidor de desarrollo y a herramientas de build, no al bundle de producción; `react-router` sí afecta runtime.

```
npm audit fix
```

Este comando actualiza los paquetes dentro de rangos compatibles sin cambios mayores. Tras ejecutarlo se recomienda:

1. Correr la suite de pruebas y la cobertura del frontend para confirmar que no hay regresiones.
2. Reconstruir (`npm run build`) y verificar el arranque de la aplicación.
3. Volver a ejecutar `npm audit` para validar que el conteo queda en 0.

## Conclusión

El frontend no presenta vulnerabilidades críticas. Las 7 detectadas (5 altas, 1 moderada, 1 baja) son corregibles mediante `npm audit fix` sin cambios incompatibles, por lo que la remediación es de bajo riesgo. Se mantiene la auditoría de dependencias como gate dentro del pipeline de CI/CD.
