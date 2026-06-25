# Deckora

Plataforma web para la comunidad de **Magic: The Gathering**. Permite construir y gestionar mazos con asistencia de inteligencia artificial, organizar y jugar torneos, y explorar una biblioteca de ~32.500 cartas. Está diseñada para ser **multi-formato** (Commander, Standard, Modern, Pioneer, Legacy) y para jugadores tanto competitivos como casuales.

> Este repositorio contiene la **documentación y los entregables** del proyecto Deckora (informes, diagramas, evidencias de pruebas y el código fuente empaquetado). El código fuente vive en repositorios separados — ver [Repositorios](#repositorios).

---

## Descripción

Deckora tiene tres propósitos principales según el tipo de usuario:

- **Jugadores** — construir y gestionar mazos en cualquier formato, usar el **Asistente IA** para recibir recomendaciones de cartas y autocompletar mazos, inscribirse a torneos y consultar sus estadísticas e historial.
- **Organizadores y tiendas** — crear y administrar torneos, gestionar inscripciones, llevar el marcador de las rondas (Swiss, eliminación directa, final) y comunicarse con los participantes por correo. Las tiendas además aparecen geolocalizadas en un mapa.
- **Cualquier visitante** — explorar la cartelera pública de torneos, encontrar tiendas cercanas y navegar la biblioteca de cartas, sin necesidad de registrarse.

### Características destacadas

- **Asistente IA de mazos**: recomendaciones por similitud vectorial (embeddings + pgvector) y autocompletado de mazos mediante un LLM.
- **Gestión completa de torneos**: ciclo de vida (pendiente → en curso → finalizado), emparejamiento por rondas, sistema de puntos y tabla de posiciones con desempates.
- **Validación de mazos por formato**: reglas intercambiables por formato (patrón Strategy), aplicadas también al inscribirse a un torneo.
- **Notificaciones por email** transaccionales y no bloqueantes ante inscripciones, aprobaciones y rechazos.
- **Mapa de tiendas** geolocalizado con Mapbox.

---

## Tecnología

Arquitectura de **tres capas desacopladas** comunicadas por HTTP (REST/JSON + JWT). Todo el proyecto usa **JavaScript (ES Modules)**, sin TypeScript.

### Frontend (`Deckora-Web`)

| Tecnología | Rol |
|------------|-----|
| React 19 | Framework de interfaz (SPA) |
| Vite | Bundler y servidor de desarrollo |
| React Router DOM | Enrutamiento con lazy loading / code splitting |
| Bootstrap + React Bootstrap | Estilos y componentes base |
| Mapbox GL | Mapa interactivo de tiendas |
| Recharts | Gráficos de estadísticas |
| Lucide React | Íconos SVG |
| `@supabase/supabase-js` | Autenticación cliente y gestión de sesión |

### Backend (`Deckora-API`)

| Tecnología | Rol |
|------------|-----|
| Node.js (≥18, ESM) | Entorno de ejecución del servidor |
| Express 5 | Framework HTTP |
| Sequelize | ORM para PostgreSQL |
| pg / pg-hstore | Driver PostgreSQL |
| Zod | Validación de schemas de request |
| Resend | Envío de correos transaccionales |
| p-limit | Control de concurrencia (script de embeddings) |

El backend está organizado por módulos, cada uno con sus capas en el mismo orden: `routes → controller → service → repository`.

### Base de datos e infraestructura

| Servicio | Propósito |
|----------|-----------|
| PostgreSQL (Supabase) | Base de datos principal |
| pgvector | Búsqueda por similitud vectorial (Asistente IA) |
| Supabase Auth | Registro, login y emisión de JWT |
| Vercel | Hosting del frontend |
| Render.com | Hosting del backend |
| Nomic AI | Embeddings vectoriales (`nomic-embed-text-v1.5`, proceso offline) |
| DeepSeek | LLM (`deepseek-chat`) para explicaciones y autocompletado |
| Mapbox | Mapa de tiendas |
| Resend | Notificaciones por email |

### Diagrama general

```
┌──────────────────────────────────────────────┐
│  PRESENTACIÓN — React + Vite (Vercel)         │
└───────────────────────┬──────────────────────┘
                        │ REST/JSON + JWT
┌───────────────────────▼──────────────────────┐
│  NEGOCIO — Node.js + Express (Render)         │
└───────────────────────┬──────────────────────┘
                        │ Sequelize + pgvector
┌───────────────────────▼──────────────────────┐
│  DATOS — PostgreSQL (Supabase)                │
│  ~32.500 cartas MTG · embeddings vector(768)  │
└──────────────────────────────────────────────┘
```

---

## Repositorios

| Componente | Repositorio | Empaquetado en este repo |
|------------|-------------|--------------------------|
| Backend / API | https://github.com/VerdaMech/Deckora-API | [`Producto/Deckora-API-main.zip`](Producto/Deckora-API-main.zip) |
| Interfaz web | https://github.com/VerdaMech/Deckora-Web | [`Producto/Deckora-Web-main.zip`](Producto/Deckora-Web-main.zip) |

---

## Documentación

- [Documentación funcional](Documentacion/DOC_FUNCIONAL.md) — el **qué** y el **para qué**: roles, flujos de punta a punta y vistas por rol.
- [Documentación técnica](Documentacion/DOC_TECNICA.md) — el **cómo**: arquitectura, patrones de diseño, módulos, stack definitivo y despliegue.
- [Diagramas](Documentacion/) — arquitectura, casos de uso, componentes, infraestructura y diagramas As-Is / To-Be.
- [Evidencias de pruebas](Documentacion/evidencias%20test/) — cobertura de tests, auditoría de dependencias, reportes de Playwright y k6.
- Informes y presentaciones de las entregas parciales (EP2, EP3) en [`Documentacion/`](Documentacion/).

---

## Equipo

**Grupo N° 2**

- Anaís Carolina Palma Sánchez
- Vicente Joaquín Verdaguer Gaete

Proyecto académico — Duoc UC.
