# Deckora — Documentación Técnica

Esta documentación explica el **cómo** del sistema: qué hace cada función, qué parámetros recibe, qué devuelve, qué reglas de negocio aplica y por qué el código está organizado como está. Está escrita para que cualquier desarrollador que entre al proyecto pueda entender la base de código sin tener que leerla línea por línea.

> **Nota sobre el alcance de Deckora**: la plataforma está pensada para toda la comunidad de Magic: The Gathering, independientemente del formato (Commander, Standard, Modern, Pioneer, Legacy) y del nivel de juego (competitivo o casual). No es exclusiva de Commander; ese formato recibe algunas reglas propias porque es el más jugado en la plataforma, pero el sistema fue diseñado desde el inicio para ser multi-formato.

---

## Índice

1. [Organización del código y arquitectura](#1-organización-del-código-y-arquitectura)
2. [Patrones de diseño utilizados](#2-patrones-de-diseño-utilizados)
3. [Módulo de IA — Recomendaciones vectoriales y Autocompletado](#3-módulo-de-ia--recomendaciones-vectoriales-y-autocompletado)
4. [Backend — Middlewares](#4-backend--middlewares)
5. [Backend — Módulo Auth](#5-backend--módulo-auth)
6. [Backend — Módulo Mazos](#6-backend--módulo-mazos)
7. [Backend — Módulo Torneos](#7-backend--módulo-torneos)
8. [Backend — Módulo Rondas](#8-backend--módulo-rondas)
9. [Backend — Módulo Enfrentamientos](#9-backend--módulo-enfrentamientos)
10. [Backend — Módulo Biblioteca](#10-backend--módulo-biblioteca)
11. [Backend — Módulo Cartas (búsqueda para el editor)](#11-backend--módulo-cartas-búsqueda-para-el-editor)
12. [Backend — Módulos de Perfiles](#12-backend--módulos-de-perfiles)
13. [Backend — Módulo Estadísticas](#13-backend--módulo-estadísticas)
14. [Backend — Módulo Notificaciones](#14-backend--módulo-notificaciones)
15. [Backend — Modelos y asociaciones Sequelize](#15-backend--modelos-y-asociaciones-sequelize)
16. [Frontend — Capa de servicios HTTP](#16-frontend--capa-de-servicios-http)
17. [Frontend — AuthContext y useAuth](#17-frontend--authcontext-y-useauth)
18. [Frontend — Hooks personalizados](#18-frontend--hooks-personalizados)
19. [Frontend — Utilidades (utils/)](#19-frontend--utilidades-utils)
20. [Frontend — Componentes de dominio](#20-frontend--componentes-de-dominio)
21. [Frontend — Componentes UI reutilizables](#21-frontend--componentes-ui-reutilizables)
22. [Frontend — Páginas principales](#22-frontend--páginas-principales)
23. [Frontend — Enrutamiento y code splitting](#23-frontend--enrutamiento-y-code-splitting)
24. [Variables de entorno y configuración](#24-variables-de-entorno-y-configuración)
25. [Tecnologías, lenguaje y dependencias (stack definitivo)](#25-tecnologías-lenguaje-y-dependencias-stack-definitivo)
26. [Configuración de servidor de producción (paso a paso)](#26-configuración-de-servidor-de-producción-paso-a-paso)
27. [Integraciones necesarias](#27-integraciones-necesarias)

---

## 1. Organización del código y arquitectura

### Arquitectura general del sistema

Deckora sigue una arquitectura de **tres capas desacopladas** comunicadas por HTTP:

```
┌─────────────────────────────────────────────────────────────────┐
│  CAPA DE PRESENTACIÓN                                           │
│  React + Vite (SPA)                                             │
│  Desplegado en Vercel                                           │
└────────────────────────┬────────────────────────────────────────┘
                         │ REST/JSON + JWT
┌────────────────────────▼────────────────────────────────────────┐
│  CAPA DE NEGOCIO                                                │
│  Node.js + Express                                              │
│  Desplegado en Render                                           │
└────────────────────────┬────────────────────────────────────────┘
                         │ Sequelize ORM + pgvector
┌────────────────────────▼────────────────────────────────────────┐
│  CAPA DE DATOS                                                  │
│  PostgreSQL (Supabase)                                          │
│  ~32.500 cartas MTG con embeddings vector(768)                  │
└─────────────────────────────────────────────────────────────────┘
```

La autenticación está delegada a **Supabase Auth** (servicio externo). El backend solo verifica los tokens JWT emitidos por Supabase, sin gestionar sesiones propias.

Los embeddings se generan con la **API de Nomic AI** (proceso offline). Las explicaciones en lenguaje natural y el autocompletado de mazos se generan con **DeepSeek** vía su API directa (en tiempo de request).

> **¿Por qué tres capas separadas?** Porque así se puede cambiar el frontend sin tocar el backend, o escalar el backend sin redeploy del frontend. Render y Vercel tienen ciclos de despliegue independientes, lo que en un equipo permite que dos personas trabajen en paralelo sin bloquearse.

---

### Backend — Arquitectura por módulos y capas

Cada módulo del backend tiene exactamente estas capas, siempre en el mismo orden de llamada:

```
routes.js  →  controller.js  →  service.js  →  repository.js
               (HTTP)            (reglas)        (base de datos)
```

- **`routes.js`**: registra las rutas Express y aplica los middlewares (`auth`, `requirePerfil`, `validate`). No tiene lógica de negocio. Es el único lugar donde se decide qué middlewares protegen cada endpoint.
- **`controller.js`**: extrae parámetros de `req` (body, params, usuario), llama al servicio y serializa la respuesta. Maneja errores con `next(err)`. No valida reglas de negocio; eso es trabajo del service.
- **`service.js`**: contiene **toda** la lógica de negocio: validaciones, permisos, orquestación entre repositorios. Lanza errores con `err.status` para que `errorHandler` los convierta en la respuesta HTTP correcta. Es el núcleo del módulo.
- **`repository.js`**: contiene únicamente las queries de Sequelize o SQL raw. No tiene lógica de negocio. Devuelve instancias de modelo o arrays de objetos planos.

```
src/
├── app.js                  ← registra todas las rutas y middlewares globales
├── modules/
│   ├── auth/               auth.routes → auth.controller → auth.service → (Supabase)
│   ├── mazos/              mazos.routes → mazos.controller → mazos.service → mazos.repository
│   │   └── estrategias/    commander.strategy.js, standard.strategy.js, ...
│   ├── torneos/
│   ├── rondas/
│   │   └── emparejadores/  swiss.emparejador.js, manual.emparejador.js, aleatorio.emparejador.js
│   ├── enfrentamientos/
│   ├── biblioteca/
│   ├── cartas/
│   ├── jugadores/
│   ├── organizadores/
│   ├── tiendas/
│   ├── usuarios/
│   ├── estadisticas/
│   └── notificaciones/     email.service.js + templates/
├── middleware/              auth.js, requirePerfil.js, validate.js, errorHandler.js
├── models/                 Sequelize models + associations (index.js centraliza todo)
├── utils/                  openrouter.js (Nomic + DeepSeek), haversine.js, scryfallMapper.js
├── config/                 db.js (Sequelize + pgvector), supabase.js
└── scripts/                generateEmbeddings.js (proceso offline), seedCartas.js, seedMazos.js
```

> **Tip para la presentación**: cuando alguien pregunte "¿dónde están las reglas de negocio?", la respuesta siempre es `*.service.js`. Cuando pregunte "¿dónde se toca la base de datos?", siempre es `*.repository.js`. Esta separación hace que cada capa sea testeable de forma independiente y que los bugs sean más fáciles de ubicar.

---

### Frontend — Arquitectura de módulos

```
src/
├── modules/
│   ├── mazos/
│   │   ├── components/   AsistenteIA, ModoEdicionMazo, BarraAgregarCarta,
│   │   │                 PanelValidacion, ImportarMazoModal
│   │   └── pages/        CrearMazoModal, DetalleMazo, MisMazos
│   ├── torneos/
│   │   ├── components/   BandejaInscripciones, FormularioTorneo, ListaInscritos,
│   │   │                 PanelInscripcion, ReportarResultadoModal, SnapshotMazoModal
│   │   └── pages/        Cartelera, DetalleTorneo, CrearTorneo, EditarTorneo,
│   │                     GestionTorneo, MisTorneos
│   ├── identidad/        Login, Registro, RecuperarPassword, Configuracion,
│   │                     PerfilJugador, PerfilOrganizador, PerfilTienda, PerfilRouter
│   ├── biblioteca/       Biblioteca (catálogo de ~32.500 cartas)
│   ├── dashboards/       Landing, DashboardJugador, DashboardOrganizador, DashboardTienda
│   └── mapa/             SeccionMapaTiendas
├── components/
│   ├── domain/           DeckBuilder, DeckList, DeckStats, MTGCard, RoundView,
│   │                     PodTable, TournamentCard, EstadisticasJugador, FormatBadge,
│   │                     EstadoBadge, ManaCost, CommanderBadge, MapaTiendas, ...
│   ├── ui/               Button, Modal, Alert, Spinner, Skeleton, Select, Tabs,
│   │                     Badge, Card, Input, Textarea, Toast, Tooltip,
│   │                     ConfirmDialog, ErrorBoundary, ErrorChunk, EmptyState
│   └── layout/           AppLayout, Navbar, Sidebar, Footer
├── services/             api.js + *.service.js por dominio
├── context/              AuthContext, ToastContext
├── hooks/                useAuth, useConfirmDialog, useDebounce, useFormValidation, useGeolocation
├── routes/               AppRoutes (con lazy loading), ProtectedRoute
└── utils/                constants.js, deck-helpers.js, formatters.js, validators.js, errors.js
```

### Frontend — Patrón de página

Cada página de React sigue este patrón:

```
useCallback(cargar) → useState(datos) + useState(cargando) + useState(error)
→ useEffect(cargar, [id])
→ if (cargando) return <Skeleton />
→ if (error)    return <Alert />
→ render con los datos
```

Los servicios (`*.service.js`) son simples wrappers de `api.js`. Toda la lógica de estado vive en los componentes de página.

---

## 2. Patrones de diseño utilizados

### Strategy — Validación de formatos de mazo

**Dónde:** `src/modules/mazos/estrategias/`

**Por qué se eligió**: sin Strategy habría un `switch` gigante en el servicio que crecería cada vez que se agregue un formato. Con Strategy, agregar "Vintage" o "Pauper" es crear un archivo nuevo y registrar una línea en el índice.

```js
// estrategias/index.js — el índice es la única fuente de verdad sobre qué formatos existen
export const estrategias = {
  COMMANDER: validarCommander,
  STANDARD:  validarStandard,
  MODERN:    validarModern,
  PIONEER:   validarPioneer,
  LEGACY:    validarLegacy,
};

// mazos.service.js — selección en runtime, sin if/switch
const estrategia = estrategias[mazo.formato];  // función correspondiente al formato
return estrategia({ formato: mazo.formato, cartas: cartasPlanas });
```

**Reglas por formato implementadas hoy:**

| Formato | Mínimo cartas | Máximo cartas | Copias por carta | Regla especial |
|---------|--------------|--------------|-----------------|----------------|
| COMMANDER | 100 exactas | 100 exactas | 1 (singleton) | Requiere 1 Legendary Creature como comandante |
| STANDARD | 60 | 250 | 4 | — |
| MODERN | 60 | sin límite | 4 | — |
| PIONEER | 60 | sin límite | 4 | — |
| LEGACY | 60 | sin límite | 4 | — |

Todas las estrategias eximen a las tierras básicas del límite de copias.

---

### Strategy — Emparejamiento de rondas

**Dónde:** `src/modules/rondas/emparejadores/`

**Por qué se eligió**: diferentes tipos de torneo necesitan lógicas de emparejamiento completamente distintas. El servicio de rondas no necesita saber cómo funciona cada algoritmo; solo invoca el emparejador correcto.

```js
// emparejadores/index.js
export const emparejadores = {
  swiss:                emparejarSwiss,
  eliminacion_directa:  emparejarManual,
  final:                emparejarManual,
};

// rondas.service.js
const emparejar = emparejadores[tipo_ronda];
const mesas = emparejar(inscripciones, asignaciones, torneo.formato);
```

Los tres emparejadores disponibles:

- **`swiss.emparejador.js`**: ordena por `puntos_acumulados` y agrupa. Para Commander usa el algoritmo `agruparCommander` (mesas de 3 y 4 jugadores con combinatoria óptima); para otros formatos hace 1v1 consecutivo.
- **`manual.emparejador.js`**: si se pasan `asignaciones` en el body las usa tal cual (el organizador asignó manualmente); si no, genera auto-bracket: el 1° juega contra el último, el 2° contra el penúltimo.
- **`aleatorio.emparejador.js`**: usa el algoritmo Fisher-Yates shuffle para aleatorizar y luego agrupa en pods de 4 (Commander) o 1v1 (otros formatos).

> **Tip para la presentación**: el patrón Strategy es uno de los más reconocibles del GoF. Puedes explicarlo mostrando `emparejadores/index.js` — en 10 líneas se ve perfectamente cómo el objeto actúa como "selector" de algoritmos sin condicionales.

---

### Repository — Acceso a datos

**Dónde:** Todos los `*.repository.js`

**Por qué se eligió**: si mañana se cambia Sequelize por Prisma, o PostgreSQL por MySQL, solo hay que tocar los repositorios. Los servicios no saben nada de ORM y no hay queries Sequelize esparcidas por el código.

```js
// mazos.service.js — sin ninguna referencia a Sequelize
const mazo = await repo.buscarPorId(mazoId);
const recomendaciones = await repo.buscarRecomendaciones(vector, excluirIds, formato);
```

---

### Layered Architecture (Arquitectura por capas)

**Dónde:** Todo el backend

Cada capa solo puede llamar a la capa inmediatamente inferior:

```
routes → controller → service → repository → BD
```

El controller nunca llama al repository directamente. El repository nunca llama al service.

---

### Middleware Chain (Cadena de responsabilidad)

**Dónde:** Express middleware pipeline

```js
router.post('/:id/cartas',
  auth,                            // ¿está autenticado?
  requirePerfil('jugador'),        // ¿tiene el rol correcto?
  validate(agregarCartaMazoSchema), // ¿el body es válido?
  mazosController.agregarCarta     // lógica de negocio
);
```

Si `auth` falla, los siguientes middlewares nunca se ejecutan. Esto es importante: la cadena corta en el primer fallo.

---

### DTO — Serialización de rondas

**Dónde:** `rondas.service.js → serializarRonda()`

Los objetos Sequelize tienen relaciones en PascalCase con profundidad variable. `serializarRonda()` convierte ese grafo al DTO plano que el frontend espera:

```js
// Entrada (Sequelize anidado):
ronda.enfrentamientos[0].participantes[0].Inscripcion.Jugador.Usuario.nombre_usuario

// Salida (DTO plano y predecible):
{ "participantes": [{ "nombre_usuario": "jugador1", "puntos": 3, "resultado": "ganador",
                      "mazo": { "nombre": "Mi Mazo", "comandante": "Atraxa" } }] }
```

La función hace una segunda query a `Inscripcion` (incluyendo `Jugador` y `Mazo`) en lugar de forzar un `include` profundo en Sequelize, que se vuelve difícil de mantener con muchos niveles de asociaciones.

---

### Facade — Capa de servicios HTTP del frontend

**Dónde:** `src/services/api.js`

`api.js` oculta toda la complejidad de autenticación y manejo de errores HTTP. El resto del código solo llama a `apiGet`, `apiPost`, etc.:

```js
// Cualquier componente:
const mazo = await apiGet('/mazos/123');

// api.js se encarga de:
// - obtener el token de Supabase (con timeout de 10s)
// - inyectar Authorization header
// - reintentar si el token expiró (refresh automático)
// - parsear el error del body JSON y lanzarlo como Error nativo de JS
```

> **Tip para la presentación**: muestra este archivo como ejemplo de centralización. Todo el código de red del frontend pasa por aquí, lo que significa que si hay un bug de autenticación, hay un solo lugar donde buscarlo.

---

### Observer — Escucha de cambios de sesión

**Dónde:** `AuthContext.jsx`

```js
supabase.auth.onAuthStateChange((_event, session) => {
  if (session) fetchMe(session.access_token);
  else { setUser(null); setRol(null); setPerfil(null); }
});
```

Esto hace que si el usuario cierra sesión en otra pestaña, la actual también se desconecta automáticamente, sin polling.

---

### Optimistic Update — Edición de mazo

**Dónde:** `ModoEdicionMazo.jsx`

Cuando el jugador agrega o elimina una carta, el frontend actualiza el estado local **inmediatamente** sin esperar la respuesta del servidor. Si la operación falla, revierte al estado anterior:

```js
// 1. Actualiza la UI al instante
setCartas(prev => [...prev, nuevaCarta]);

// 2. Llama al servidor en paralelo
try {
  await agregarCartaAMazo(mazoId, datos);
} catch {
  // 3. Si falla, revierte
  setCartas(snapshot);
  mostrarToast('No se pudo agregar la carta');
}
```

**Por qué**: la latencia de Render (hosting del backend) puede llegar a 800ms si el servidor está "en frío". Sin optimistic update la UI se sentiría lenta para algo tan frecuente como agregar cartas.

---

### Retry con backoff exponencial — Script de embeddings

**Dónde:** `scripts/generateEmbeddings.js → withRetry()`

```js
// En rate limit (429): espera 2s → 4s → 8s → 16s
// En otro error:       espera siempre 1s
const espera = err.message.includes('429')
  ? DELAY_BASE_MS * 2 ** intento
  : DELAY_BASE_MS;
```

---

## 3. Módulo de IA — Recomendaciones vectoriales y Autocompletado

El módulo de IA tiene **dos funcionalidades** independientes en `AsistenteIA.jsx`, ambas accesibles desde el panel lateral del DeckBuilder:

1. **Recomendaciones**: sugiere cartas similares al estilo del mazo actual usando búsqueda vectorial.
2. **Autocompletado**: completa automáticamente el mazo hasta el límite del formato usando un LLM.

---

### ¿Qué es un embedding?

Un **embedding** es una representación matemática del significado semántico de un texto. El modelo convierte texto descriptivo en un vector de 768 números, donde textos similares producen vectores matemáticamente cercanos.

```
Texto:   "Instant CMC:1 Colors:R"
Vector:  [-0.09, 0.014, -0.148, 0.071, ..., -0.017]  → 768 números

Texto similar:   "Instant CMC:2 Colors:R,B"
Vector:          [-0.08, 0.018, -0.130, 0.065, ..., -0.020]  → valores cercanos

Texto diferente: "Creature CMC:6 Colors:G"
Vector:          [0.31, -0.092, 0.204, -0.41, ..., 0.088]    → valores lejanos
```

---

### Modelo de embeddings: Nomic Embed Text v1.5

| Atributo | Valor |
|----------|-------|
| Modelo | `nomic-embed-text-v1.5` |
| Proveedor | Nomic AI (`api-atlas.nomic.ai`) |
| Dimensión de salida | 768 |
| Costo | Gratuito con API key |
| Task type | `search_document` |

El texto vectorizado por carta es intencionalmente minimalista:
```js
`${carta.tipo} CMC:${carta.cmc} Colors:${carta.colors?.join(',') ?? 'none'}`
// Ejemplo: "Instant CMC:1 Colors:R"
// Ejemplo: "Legendary Creature CMC:4 Colors:W,U,B,R,G"
```
Se usan tipo, CMC y colores porque determinan el "rol" de una carta. El texto oracle (efecto) no se incluye porque añade ruido al representar el rol mecánico dentro del mazo.

---

### Modelo de texto (LLM): DeepSeek Chat

| Atributo | Valor |
|----------|-------|
| Modelo | `deepseek-chat` |
| Proveedor | DeepSeek (`api.deepseek.com`) |
| Variable de entorno | `DEEPSEEK_API_KEY` |
| Usos | Explicaciones de recomendaciones + Autocompletado de mazos |
| Falla graceful | Sí — si DeepSeek falla, las recomendaciones se devuelven sin explicación |

> **Importante**: el archivo `src/utils/openrouter.js` conserva su nombre histórico pero actualmente usa la API de **DeepSeek**, no OpenRouter. Esto fue un cambio de proveedor LLM durante el desarrollo. El nombre del archivo no fue actualizado para evitar romper imports.

---

### Almacenamiento: pgvector + índice HNSW

```sql
-- Columna en la tabla cartas:
embedding vector(768)

-- Búsqueda por similitud coseno (operador <=>):
SELECT id, nombre, tipo, costo_mana, imagen_url, texto, cmc, colors, legalities
FROM cartas
WHERE embedding IS NOT NULL
  AND id NOT IN (:excluirIds)
  AND (legalities IS NULL
    OR legalities->>'commander' IS NULL
    OR legalities->>'commander' NOT IN ('banned', 'not_legal'))
ORDER BY embedding <=> :embedding::vector
LIMIT 10;
```

El operador `<=>` calcula la distancia coseno. Con el índice **HNSW** la búsqueda es O(log n) en lugar de O(n) (comparar contra las ~32.500 cartas una a una).

> **Nota sobre el filtro de legalidad**: el WHERE excluye solo cartas explícitamente `banned` o `not_legal`. Las cartas sin datos de legalidad (`NULL`) se incluyen en los resultados, lo que permite recomendar cartas con datos incompletos de Scryfall.

---

### Flujo técnico de una recomendación

```
GET /mazos/:id/recomendaciones
         ↓
mazos.controller.js → mazos.service.js → recomendarCartas()
         ↓
repo.buscarPorId(mazoId)
→ Mazo con MazoCartas → Carta (incluye campo embedding)
         ↓
Filtra cartas con embedding IS NOT NULL
Si length === 0 → 422 "Las cartas del mazo aún no tienen embeddings"
         ↓
Calcula vector promedio del mazo:
  vectorPromedio[i] = suma(embedding[i] de cada carta) / cantidadCartas
  → 768 operaciones de promedio
         ↓
repo.buscarRecomendaciones(vectorPromedio, excluirIds, formato)
→ SQL raw con ORDER BY embedding <=> vectorPromedio::vector LIMIT 10
→ Retorna las 10 cartas más similares al estilo del mazo
         ↓
generateExplanation(mazo.nombre, mazo.formato, recomendaciones)
→ POST a api.deepseek.com con deepseek-chat
→ Devuelve string con 3-4 bullets en español (o null si falla)
         ↓
{ recomendaciones: [...], explicacion: "..." }
```

---

### Flujo técnico del Autocompletado

```
POST /mazos/:id/autocompletar
         ↓
mazos.service.js → autocompletar()
         ↓
Calcula cuántas cartas faltan:
  OBJETIVO = { COMMANDER: 100, STANDARD: 60, MODERN: 60, PIONEER: 60, LEGACY: 60 }
  necesarias = OBJETIVO[formato] - totalActual
         ↓
Si formato === COMMANDER y mazo.comandante:
  Agrega primero el comandante si no está ya en el mazo
         ↓
generarListaMazo(nombre, formato, comandante, cartasExistentes, necesarias)
→ POST a api.deepseek.com con deepseek-chat
→ El prompt varía según formato:
    Commander: "genera X cartas singleton, sigue los colores del comandante"
    Otros:     "genera X nombres distintos para formato {formato}"
→ Devuelve lista en texto: "1 Sol Ring\n1 Command Tower\n..."
         ↓
Parsea cada línea → busca carta en BD local por nombre exacto o fuzzy
Para Commander: agrega con cantidad 1 (singleton forzado)
Para otros:     agrega con cantidad indicada en la lista
         ↓
Post-proceso Commander: si ninguna carta quedó marcada como comandante,
busca la primera Legendary Creature del mazo y la marca
         ↓
{ agregadas: [...], fallidas: [...] }
```

---

### `src/utils/openrouter.js` — Funciones de IA

| Función | Endpoint | Uso |
|---------|----------|-----|
| `generateEmbedding(text)` | `api-atlas.nomic.ai/v1/embedding/text` | Una carta individual (fallback del script) |
| `generateEmbeddingsBatch(texts[])` | `api-atlas.nomic.ai/v1/embedding/text` | Batch de hasta 50 cartas (script offline) |
| `generarListaMazo(nombre, formato, comandante, existentes, cantidad)` | `api.deepseek.com/chat/completions` | Genera lista para autocompletar el mazo |
| `generateExplanation(nombreMazo, formato, cartas)` | `api.deepseek.com/chat/completions` | Explicación de las recomendaciones vectoriales |

---

### `AsistenteIA.jsx` — Componente frontend

**Ruta:** `src/modules/mazos/components/AsistenteIA.jsx`

**Props:**

| Prop | Tipo | Descripción |
|------|------|-------------|
| `mazo` | Object | El mazo actual con `mazo.id` y `mazo.formato` |
| `cartas` | Array | Cartas actuales del mazo (para saber si está vacío) |
| `onAplicarSugerencia` | Function | Callback que recibe una carta recomendada para agregarla |
| `onAutocompletar` | Function | Callback que dispara el autocompletado en el service |
| `onMazoImportado` | Function | Callback que se llama tras importar una plantilla |

El componente tiene **dos secciones independientes**, cada una con su propio estado de máquina:

**Sección Recomendaciones:**
| Estado | Qué muestra |
|--------|------------|
| `inicial` | Descripción + botón "Pedir recomendaciones" + error (si hubo) |
| `cargando` | Spinner + "Analizando tu mazo..." |
| `resultado` | Explicación IA formateada + lista de cartas + botón "Nueva búsqueda" |

**Sección Autocompletado:**
| Estado | Qué muestra |
|--------|------------|
| `inicial` | Descripción + botón "Autocompletar con IA" + error (si hubo) |
| `cargando` | Spinner + "La IA está completando tu mazo..." |
| `resultado` | "Mazo completado con éxito" + botón "Volver a autocompletar" |

**Lógica especial cuando el mazo está vacío**: si `cartas.length === 0` y el formato tiene plantillas disponibles en `mazosPlantilla.json`, el botón "Autocompletar" en realidad carga una plantilla predefinida aleatoria en lugar de llamar al LLM. Esto evita que el LLM intente completar un mazo sin contexto.

---

### Script de generación de embeddings

**Archivo:** `scripts/generateEmbeddings.js`
**Comando:** `npm run embed:generate`

Idempotente: solo procesa cartas con `embedding IS NULL`. Puede interrumpirse y reanudarse.

```
Arquitectura:

pLimit(2)          → máximo 2 batches en paralelo simultáneamente
    │
    ├── batch 1 (50 cartas) ──┐
    └── batch 2 (50 cartas) ──┤→ generateEmbeddingsBatch(textos)
                              │     → POST api-atlas.nomic.ai
                              │        { model: "nomic-embed-text-v1.5",
                              │          texts: [50 strings],
                              │          task_type: "search_document" }
                              │     → { embeddings: [[768 floats], ...] }
                              │
                              └→ UPDATE cartas SET embedding = $1::vector WHERE id = $2
```

| Versión | Llamadas a la API | Tiempo estimado (35.000 cartas) |
|---------|------------------|--------------------------------|
| Sin batch (1 carta por llamada) | ~35.000 | 2–4 horas |
| Con batch (50 cartas por llamada) | ~700 | **10–20 minutos** |

---

## 4. Backend — Middlewares

### `auth.js`

Protege rutas que requieren usuario autenticado.

**Qué hace paso a paso:**
1. Lee el header `Authorization: Bearer <token>`
2. Llama a `supabase.auth.getUser(token)` — Supabase verifica la firma JWT
3. Busca el usuario en la tabla local `usuarios` por UUID
4. Verifica que la cuenta esté activa (`usuario.activo === true`)
5. Adjunta dos objetos al request:
   - `req.usuarioAuth` → objeto de Supabase (email, metadata)
   - `req.usuario` → instancia Sequelize con `id`, `nombre_usuario`, `rol`, `activo`

**Errores:** `401` en cualquier fallo de autenticación.

**Por qué verificar en BD local además de en Supabase**: Supabase solo sabe si el token es válido, pero no si el usuario fue desactivado en nuestra lógica de negocio. La tabla `usuarios` es la fuente de verdad de la aplicación.

---

### `requirePerfil.js`

Se usa después de `auth` para restringir un endpoint a roles específicos.

```js
// Acepta múltiples roles:
router.post('/:id/rondas', auth, requirePerfil('organizador', 'tienda'), ...);

// Internamente:
if (!roles.includes(req.usuario.rol)) → 403 Acceso denegado
```

---

### `validate.js`

Valida `req.body` contra un schema Zod antes de que llegue al controller.

```js
// Si la validación falla → 400 con { error, detalles: [...errores Zod] }
// Si pasa → req.body = resultado.data (ya parseado y tipado por Zod)
```

**Por qué Zod**: es la librería de validación más popular del ecosistema Node/TypeScript moderno, con mensajes de error muy descriptivos. Reemplaza los `if (!body.nombre) return res.status(400)...` manuales.

---

### `errorHandler.js`

Middleware global registrado al **final** de `app.js`. Recibe cualquier error lanzado con `next(err)`.

```js
// Patrón para errores controlados desde el service:
const err = new Error('El torneo no existe');
err.status = 404;
throw err;
// errorHandler lo convierte en: HTTP 404 { "error": "El torneo no existe" }
```

Lógica interna:
- `ZodError` → `400` con campos inválidos
- `err.status` o `err.statusCode` → usa ese código HTTP
- Cualquier otro error → `500 Internal Server Error`

---

## 5. Backend — Módulo Auth

### Signup — `auth.service.js → signup()`

Crea un usuario completo en 3 operaciones en orden:

1. **Crea en Supabase Auth** (`supabase.auth.signUp`)
2. **Crea en tabla `usuarios`** con el UUID devuelto por Supabase
3. **Crea el perfil extendido** según el rol:
   - `jugador` → tabla `jugadores` + `estadisticas` (contadores en 0)
   - `organizador` → tabla `organizadores`
   - `tienda` → tabla `tiendas`

**Rollback**: si los pasos 2 o 3 fallan, intenta eliminar el usuario de Supabase Auth. Esto evita cuentas "huérfanas" en Supabase que no tienen perfil en la BD propia.

El signup puede requerir **verificación de correo** (configuración de Supabase). En ese caso devuelve `{ requiresEmailVerification: true }` y el frontend muestra la pantalla de "verifica tu correo" sin intentar cargar el perfil.

---

### Login — `auth.service.js → login()`

Llama a `supabase.auth.signInWithPassword()`. Si falla lanza `401` con mensaje genérico (no distingue si el email no existe o la contraseña es incorrecta, por seguridad). Devuelve `{ access_token, usuario }`.

---

## 6. Backend — Módulo Mazos

### Función auxiliar `resolverCarta(identificador)`

Usada internamente por `agregarCarta`, `actualizarCarta`, `eliminarCarta` e `importarLista`:

```js
async function resolverCarta(identificador) {
  let carta = await cartasRepository.buscarPorScryfallId(identificador);
  if (!carta) carta = await cartasRepository.buscarPorId(identificador);
  if (!carta) → 404 "Carta no existe en la biblioteca local"
}
```

Intenta primero por `scryfall_id` (UUID de Scryfall), luego por `id` interno. Las cartas deben estar en la BD local; ya no se consulta Scryfall en tiempo real.

---

### Función auxiliar `verificarPropietario(mazo, jugadorId)`

Reutilizada en todas las funciones que modifican un mazo. Lanza `404` si el mazo no existe, `403` si el jugador que intenta operarlo no es su propietario.

---

### `crear(jugadorId, datos)`

Genera un `slug` único:
```
"Mi Mazo Dragones" → "mi-mazo-dragones-1715000000000"
```
El timestamp al final garantiza unicidad incluso con nombres idénticos. El slug se usa en URLs limpias y en búsquedas futuras.

---

### `validar(mazoId, jugadorId)`

Aplica el patrón Strategy. Las cartas del mazo se "aplanan" a objetos simples antes de pasarlas a la estrategia (sin referencia a Sequelize), para que las estrategias sean funciones puras sin dependencias externas.

---

### `importarLista(mazoId, jugadorId, lista, comandante)`

Importa un mazo desde texto. Soporta dos formatos de línea:

```
# Formato completo (Scryfall export):
1 Sol Ring (CMD) 236

# Formato simple:
1 Sol Ring
```

Para cada carta intenta en orden: buscar por `setCodigo + numeroColector`, luego por nombre exacto, luego por nombre parcial. Devuelve `{ importadas: [...], fallidas: [...] }` — las fallidas no abortan el proceso completo.

---

### `autocompletar(mazoId, jugadorId)`

Nueva funcionalidad de IA. Ver sección 3 para el flujo completo.

**Objetivo de cartas por formato:**
```js
const OBJETIVO_CARTAS = {
  COMMANDER: 100, STANDARD: 60, MODERN: 60, PIONEER: 60, LEGACY: 60,
};
```

---

### `recomendarCartas(mazoId, jugadorId)`

Ver sección 3 para el flujo completo.

---

## 7. Backend — Módulo Torneos

### `inscribir(torneoId, jugadorId, mazoId)`

Es la función más compleja del módulo. Realiza estas validaciones en orden:

1. El torneo existe
2. El torneo está en estado `pendiente`
3. El jugador no está ya inscrito
4. El mazo no está ya inscrito en este torneo
5. El mazo existe y pertenece al jugador
6. **El formato del mazo coincide con el formato del torneo** (nueva validación)
7. **El mazo pasa la validación del formato** usando el patrón Strategy (nueva validación)

Si todo pasa: crea la inscripción con `confirmado: false` y genera un **snapshot** de las cartas del mazo en ese momento. El snapshot preserva el estado del mazo al momento de inscripción aunque el jugador lo modifique después.

Las **notificaciones por correo** se envían de forma **no bloqueante** usando `.then()`:
```js
// No usa await → no retrasa la respuesta HTTP al jugador
Promise.all([...]).then(([organizador, jugador]) => {
  emailService.notificarSolicitudInscripcion({...}).catch(console.error);
}).catch(console.error);
```

**Por qué no bloqueante**: si el servicio de correo falla (Resend caído, límite de tasa) el flujo principal no debe romperse. La inscripción ya fue creada.

---

### `cambiarEstado(torneoId, usuarioId, { estado })`

**Al pasar a `en_curso`:**
```js
const MIN_JUGADORES = { COMMANDER: 3 };
const MIN_DEFAULT = 2;
// Commander necesita 3 porque el mínimo para una mesa válida es 3 jugadores
```

**Al pasar a `finalizado`**: verifica que no haya enfrentamientos pendientes, luego en una **transacción Sequelize** incrementa `torneos_participados` en las estadísticas de todos los inscritos confirmados. La transacción garantiza que si algo falla a mitad, ni los contadores ni el estado se modifican.

---

### `obtenerSnapshotInscripcion(torneoId, inscripcionId, usuarioId)`

Devuelve las cartas del mazo tal como estaban al momento de la inscripción. Solo accesible para el organizador del torneo. Permite verificar si un jugador usó el mazo declarado.

---

## 8. Backend — Módulo Rondas

### `crearRonda(torneoId, { tipo_ronda, asignaciones }, usuarioId)`

Validaciones en orden:
1. El torneo existe y el usuario es su organizador
2. **No hay enfrentamientos pendientes en la ronda actual** (no se puede crear la siguiente ronda si la anterior no está resuelta)
3. Hay suficientes inscripciones confirmadas (mínimo 3 para Commander, 2 para otros)
4. El `tipo_ronda` existe en el registro de emparejadores

Luego crea la ronda y, por cada "mesa" que devuelve el emparejador, crea un `Enfrentamiento` con un `EnfrentamientoParticipante` por cada inscripción.

**Por qué verificar enfrentamientos pendientes**: crear una nueva ronda mientras la anterior no está resuelta generaría datos incoherentes (los puntos acumulados para el emparejamiento swiss dependen de los resultados anteriores).

---

### Swiss para Commander vs 1v1

**Función `agruparCommander`**: el formato Commander requiere mesas de 3-4 jugadores. El algoritmo minimiza la cantidad de mesas de 3 (que son menos balanceadas que las de 4):

```
n % 3 == 0 → todas mesas de 3
n % 3 == 1 → 1 mesa de 4, resto de 3
n % 3 == 2 → 2 mesas de 4, resto de 3 (si n ≥ 8)
```

Para formatos 1v1 simplemente agrupa de a 2 en orden de ranking.

---

### `serializarRonda(ronda)`

Hace una segunda query a `Inscripcion` con `include` de `Jugador` y `Mazo` para enriquecer cada participante con datos legibles. Esto evita un `include` profundo de 4+ niveles en la query principal (que Sequelize maneja mal).

---

## 9. Backend — Módulo Enfrentamientos

### `registrarResultado(enfrentamientoId, participantes, usuarioId)`

Usa una **transacción Sequelize** para garantizar consistencia. Validaciones previas:
- El enfrentamiento no está ya `finalizado`
- El usuario es el organizador (verificado subiendo por `Enfrentamiento → Ronda → Torneo`)
- Los `inscripcion_id` del body coinciden exactamente con los participantes de la mesa
- No más de 1 ganador
- No puede haber ganador Y empates al mismo tiempo

**Sistema de puntos:**
```js
const PUNTOS = { ganador: 3, empate: 1, derrota: 0 };
```

**Mapeo resultado API → BD:**
```js
// La API acepta 'derrota', pero la BD guarda 'perdedor' (decisión de diseño heredada)
const RESULTADO_A_DB = { ganador: 'ganador', empate: 'empate', derrota: 'perdedor' };
```

Dentro de la transacción: actualiza resultado y puntos en `enfrentamiento_participantes`, actualiza estadísticas del jugador (usando `findOrCreate` para garantizar que exista el registro), y cambia el estado del enfrentamiento a `finalizado`.

---

## 10. Backend — Módulo Biblioteca

Sirve la biblioteca de cartas local sin autenticación.

### `GET /api/biblioteca/cartas`

Parámetros: `page` (default 1), `limit` (default 50, máximo 50), `set_codigo` (opcional).

Devuelve paginación completa:
```json
{ "data": [...cartas],
  "pagination": { "page": 1, "limit": 50, "total": 32500, "total_pages": 650 } }
```

### `GET /api/biblioteca/sets`

Devuelve sets agrupados por año, ordenados de más reciente a más antiguo, con conteo de cartas por set.

---

## 11. Backend — Módulo Cartas (búsqueda para el editor)

### `GET /api/cartas/buscar?q=&formato=`

**Filtros aplicados siempre:**
1. Nombre coincide con `q` (ILIKE, búsqueda parcial case-insensitive)
2. Excluye tokens: `numero_colector NOT LIKE 'T%'`
3. Excluye cartas de arte: `numero_colector NOT LIKE 'A%'`

**Filtro de legalidad si se pasa `formato`:**
```js
legalities->>'commander' = 'legal'  // (o el formato correspondiente)
```

**Si no se pasa `formato`:** acepta cartas legales en cualquiera de los 5 formatos soportados.

Devuelve 20 resultados máximo. Campos: `id, scryfall_id, nombre, tipo, costo_mana, imagen_url, set_codigo, es_tierra_basica, legalities`.

---

## 12. Backend — Módulos de Perfiles

### Tiendas
- `GET /api/tiendas/cercanas`: usa `haversine.js` para calcular distancias geográficas. La ruta `/cercanas` debe estar declarada **antes** de `/:id` en el router (si no, Express la interpreta como un ID).
- El perfil de tienda incluye `latitud` y `longitud` para el mapa de Mapbox.

### Organizadores
- `GET /organizadores/me` y `PATCH /organizadores/me`: requieren `rol === 'organizador'`. El campo `redes_sociales` es JSONB (links a Twitter, Instagram, etc.).

### Jugadores
- `GET /jugadores/:id/mazos`: solo devuelve mazos con `publico = true`. No requiere autenticación.

### Usuarios
- `GET /api/usuarios/:username`: perfil público por nombre de usuario.
- `GET /api/usuarios/:id/estadisticas`: estadísticas de un jugador por ID.
- `GET /api/usuarios/me/inscripciones`: inscripciones del usuario autenticado.
- `GET /api/usuarios/me/torneos`: torneos organizados por el usuario autenticado.

---

## 13. Backend — Módulo Estadísticas

### `GET /api/estadisticas/ranking`

Todos los jugadores ordenados por desempeño. Endpoint público, no requiere auth.

### `GET /api/estadisticas/mias`

Requiere auth + rol `jugador`. Devuelve `partidas_ganadas`, `partidas_perdidas`, `partidas_empatadas`, `torneos_participados`.

---

## 14. Backend — Módulo Notificaciones

`src/modules/notificaciones/email.service.js` centraliza el envío de correos con Resend. Tiene tres funciones:

| Función | Destinatario | Cuándo se llama |
|---------|-------------|-----------------|
| `notificarSolicitudInscripcion()` | Organizador | Al inscribirse un jugador |
| `notificarInscripcionAceptada()` | Jugador | Al aprobar la inscripción |
| `notificarInscripcionRechazada()` | Jugador | Al rechazar la inscripción |

Cada función llama a su template HTML en `templates/`. La variable `EMAIL_FROM` define el remitente; si no está configurada, usa `onboarding@resend.dev` (dominio de prueba de Resend que funciona sin verificación de dominio).

**Por qué se llama de forma no bloqueante**: las notificaciones de email son "nice to have". Si el servicio de Resend falla, la operación principal (inscripción, aprobación) no debe verse afectada.

---

## 15. Backend — Modelos y asociaciones Sequelize

### Regla crítica: aliases en `hasMany`

Cuando se define `as` en una asociación `hasMany`, **todos** los `include` que usen ese modelo deben especificar el mismo `as`:

```js
// models/index.js
Ronda.hasMany(Enfrentamiento, { foreignKey: 'ronda_id', as: 'enfrentamientos' });

// repository — CORRECTO:
Ronda.findAll({ include: [{ model: Enfrentamiento, as: 'enfrentamientos' }] });

// repository — ERROR EN RUNTIME:
Ronda.findAll({ include: [{ model: Enfrentamiento }] });
```

### Tabla de asociaciones completa

| Desde | Tipo | Hacia | FK | Alias |
|-------|------|-------|----|-------|
| Usuario | hasOne | Jugador | `usuario_id` | — |
| Usuario | hasOne | Organizador | `usuario_id` | — |
| Usuario | hasOne | Tienda | `usuario_id` | — |
| Usuario | hasMany | Torneo | `organizador_id` | `torneos_organizados` |
| Torneo | belongsTo | Usuario | `organizador_id` | `organizador` |
| Torneo | hasMany | Inscripcion | `torneo_id` | — |
| Torneo | hasMany | Ronda | `torneo_id` | — |
| Inscripcion | belongsTo | Jugador | `usuario_id` | — |
| Inscripcion | belongsTo | Mazo | `mazo_id` | — |
| Inscripcion | hasMany | EnfrentamientoParticipante | `inscripcion_id` | — |
| Inscripcion | hasMany | SnapshotMazoInscripcion | `inscripcion_id` | — |
| Ronda | hasMany | Enfrentamiento | `ronda_id` | `enfrentamientos` |
| Enfrentamiento | hasMany | EnfrentamientoParticipante | `enfrentamiento_id` | `participantes` |
| Mazo | hasMany | MazoCarta | `mazo_id` | `MazoCartas` |
| MazoCarta | belongsTo | Carta | `carta_id` | `Carta` |
| Jugador | hasOne | Estadistica | `usuario_id` | — |
| SnapshotMazoInscripcion | belongsTo | Carta | `carta_id` | `Carta` |

---

## 16. Frontend — Capa de servicios HTTP

### `api.js` — función base `apiFetch(path, options)`

Es el único lugar donde se maneja la autenticación HTTP en el frontend.

**Flujo:**
1. Llama `getAccessToken()` con timeout de 10s (red de seguridad para `getSession()` que puede bloquearse durante auto-refresh de Supabase)
2. Inyecta `Authorization: Bearer <token>` si hay token
3. Inyecta `Content-Type: application/json` si hay body
4. Si la respuesta es `204 No Content` → devuelve `null`
5. Si la respuesta es `401` → llama `supabase.auth.refreshSession()`, usa el token devuelto directamente (evita segunda llamada a `getSession()`) y reintenta la request
6. Si el refresh falla → `supabase.auth.signOut()` + lanza `Error('Sesión expirada')`
7. Si hay otro error HTTP → extrae el mensaje del body JSON y lanza `Error` con ese texto

**Exports:**
```js
export const apiGet    = (path)        => apiFetch(path, { method: 'GET' });
export const apiPost   = (path, body)  => apiFetch(path, { method: 'POST', body: JSON.stringify(body) });
export const apiPatch  = (path, body)  => apiFetch(path, { method: 'PATCH', body: JSON.stringify(body) });
export const apiPut    = (path, body)  => apiFetch(path, { method: 'PUT', body: JSON.stringify(body) });
export const apiDelete = (path)        => apiFetch(path, { method: 'DELETE' });
```

---

### Servicios disponibles

| Servicio | Funciones principales |
|----------|----------------------|
| `torneos.service.js` | `listarTorneos`, `obtenerTorneo`, `crearTorneo`, `actualizarTorneo`, `cambiarEstadoTorneo`, `inscribirseATorneo`, `listarInscripciones`, `listarPendientes`, `aprobarInscripcion`, `rechazarInscripcion`, `cancelarInscripcion`, `obtenerTablaPosiciones`, `obtenerSnapshotInscripcion` |
| `rondas.service.js` | `listarRondas`, `obtenerRonda`, `crearRonda`, `eliminarRonda` |
| `enfrentamientos.service.js` | `actualizarResultado`, `actualizarEstado` |
| `mazos.service.js` | `listarMazos`, `crearMazo`, `obtenerMazo`, `actualizarMazo`, `agregarCarta`, `actualizarCarta`, `eliminarCarta`, `eliminarMazo`, `validarMazo`, `getRecomendaciones`, `autocompletarMazo`, `importarMazo` |
| `cartas.service.js` | `buscarCartas(q, limit?, formato?)` |
| `biblioteca.service.js` | `listarCartas({ page, limit, set_codigo })`, `listarSets()` |
| `auth.service.js` | `login`, `signup`, `logout`, `getMe`, `recuperarPassword`, `actualizarPassword` |
| `usuarios.service.js` | `getPublicProfile(username)`, `obtenerEstadisticasJugador(usuarioId)`, `getMisTorneos()`, `getMisInscripciones()`, `actualizarPerfil()` |
| `tiendas.service.js` | `listarTiendasCercanas(lat, lng)`, `obtenerTiendaPorId()`, `actualizarTienda()` |

---

## 17. Frontend — AuthContext y useAuth

### `AuthContext.jsx`

```js
const [user, setUser]     = useState(null);   // objeto usuario de /auth/me
const [rol, setRol]       = useState(null);   // 'jugador' | 'organizador' | 'tienda'
const [perfil, setPerfil] = useState(null);   // datos del perfil extendido
const [loading, setLoading] = useState(true); // true mientras verifica la sesión inicial
```

**Inicialización:**
1. `supabase.auth.getSession()` → si hay sesión llama `fetchMe(token)`
2. `setLoading(false)` → las rutas protegidas ya pueden decidir si redirigir o no
3. `supabase.auth.onAuthStateChange()` → escucha cambios en tiempo real

**`fetchMe(token)`**: consulta `GET /auth/me` para obtener el usuario completo con rol y perfil. Si falla por error de red **no limpia el estado** (errores temporales no deben cerrar la sesión). Solo `onAuthStateChange` con `session = null` puede limpiar el estado.

**Manejo de verificación de email**: `signup()` comprueba si la respuesta tiene `requiresEmailVerification: true`. En ese caso no carga el usuario (la sesión no existe aún) y devuelve el flag para que la UI muestre la pantalla de verificación.

### `useAuth()`

Hook que expone `{ user, rol, perfil, loading, login, signup, logout }`. Lanza error si se usa fuera de `<AuthProvider>`.

---

## 18. Frontend — Hooks personalizados

| Hook | Archivo | Qué hace |
|------|---------|---------|
| `useAuth()` | `hooks/useAuth.js` | Re-exporta el contexto de autenticación |
| `useConfirmDialog()` | `hooks/useConfirmDialog.jsx` | Gestiona dialogs de confirmación con estado loading y campo de texto opcional |
| `useDebounce(value, delay)` | `hooks/useDebounce.js` | Debounce de un valor con delay configurable. Usado en `BarraAgregarCarta` para no disparar búsquedas en cada keystroke |
| `useFormValidation(initialState, validate)` | `hooks/useFormValidation.js` | Estado de formulario con campos touched, errores por campo y validación on-submit |
| `useGeolocation()` | `hooks/useGeolocation.js` | Accede a la geolocalización del navegador con manejo de errores y opción de auto-request |

---

## 19. Frontend — Utilidades (utils/)

### `constants.js`

Fuente de verdad de todos los enumerados de la aplicación. **Siempre importar de aquí**, nunca escribir strings mágicos como `'jugador'` o `'pendiente'` en los componentes:

```js
export const ROLES = { JUGADOR: 'jugador', ORGANIZADOR: 'organizador', TIENDA: 'tienda' };
export const FORMATOS = { COMMANDER: 'COMMANDER', STANDARD: 'STANDARD', MODERN: 'MODERN', PIONEER: 'PIONEER', LEGACY: 'LEGACY' };
export const FORMATO_LABELS = { COMMANDER: 'Commander', ... };
export const ESTADO_TORNEO = { PENDIENTE: 'pendiente', EN_CURSO: 'en_curso', FINALIZADO: 'finalizado', CANCELADO: 'cancelado' };
export const TIPO_RONDA = { SWISS: 'swiss', ELIMINACION_DIRECTA: 'eliminacion_directa', FINAL: 'final' };
export const RESULTADO_ENFRENTAMIENTO = { GANADOR: 'ganador', PERDEDOR: 'perdedor', EMPATE: 'empate', PENDIENTE: 'pendiente' };
```

### `deck-helpers.js`

Utilidades de análisis de mazos. **No depende de React** — son funciones puras:

| Función | Qué hace |
|---------|---------|
| `agruparPorTipo(cartas)` | Agrupa cartas en: comandante, criaturas, tierras, instantáneos, conjuros, artefactos, encantamientos, planeswalkers, otros |
| `calcularCurva(cartas)` | Retorna `[{cmc: '0', count}, ..., {cmc: '7+', count}]` para el gráfico de curva de maná. Excluye tierras. Parsea el CMC desde `carta.cmc` o desde `carta.costo_mana` si el primero no está disponible |
| `calcularDistribucionColor(cartas)` | Retorna `{W, U, B, R, G, C}` con conteo de cartas por color |
| `parseManaCost(cost)` | Convierte `"{2}{W}{U}"` en `['{2}', '{W}', '{U}']` |
| `contarCartasMazo(cartas)` | Suma todas las `cantidad` del mazo |

### `formatters.js`

Utilidades de formato de texto para la UI (fechas relativas, iniciales de usuario, cupos, etc.).

### `validators.js` / `errors.js` / `torneos-helpers.js`

Funciones de validación del lado cliente y utilidades específicas de torneos.

---

## 20. Frontend — Componentes de dominio

### `DeckBuilder.jsx`

Componente principal del editor de mazo. Orquesta:
- `BarraAgregarCarta` (búsqueda de cartas)
- `DeckList` (lista de cartas del mazo)
- `DeckStats` (curva de maná + distribución de colores)
- `AsistenteIA` (panel de IA)
- `PanelValidacion` (errores de formato)
- `ImportarMazoModal` (importar desde texto)

### `RoundView.jsx`

Renderiza una ronda completa. Props clave:

| Prop | Tipo | Descripción |
|------|------|-------------|
| `ronda` | Object | DTO de ronda con `enfrentamientos[]` |
| `editable` | boolean | Si es `true`, muestra botón "Reportar resultado" |
| `onReportarResultado` | Function? | Callback externo; si no se pasa, abre el modal interno |

Lee `ronda.enfrentamientos`. También acepta `ronda.mesas` como fallback de compatibilidad.

### `PodTable.jsx`

Renderiza una mesa individual. Usa `table-layout: fixed` con anchos porcentuales fijos. En mobile (≤640px) la columna "Mazo" se oculta con `display: none`. Mantiene estado local del enfrentamiento para actualizar a `finalizado` sin recargar del servidor.

### `BandejaInscripciones.jsx`

Panel para que el organizador apruebe/rechace inscripciones pendientes. Al aprobar/rechazar filtra la solicitud de la lista local y llama `onCambio?.()` para que `DetalleTorneo` recargue los inscritos confirmados.

Incluye botón "Ver mazo inscrito" que abre `SnapshotMazoModal` con las cartas exactas que el jugador declaró.

### `ReportarResultadoModal.jsx`

Busca participantes en: `enfrentamiento?.EnfrentamientoParticipantes ?? enfrentamiento?.jugadores ?? enfrentamiento?.participantes` (compatibilidad con distintos formatos de respuesta). Valida en el frontend antes de enviar: todos con resultado asignado, máximo 1 ganador.

### `FormatBadge.jsx` / `EstadoBadge.jsx` / `RoleBadge.jsx`

Badges de colores para mostrar formato, estado de torneo y rol de usuario respectivamente. Usan `constants.js` para mapear valores a labels y colores CSS.

### `ManaCost.jsx`

Renderiza símbolos de maná MTG desde una cadena como `{2}{W}{U}`. Parsea con `deck-helpers.parseManaCost`.

---

## 21. Frontend — Componentes UI reutilizables

Todos en `src/components/ui/`, exportados desde `index.js`.

| Componente | Props clave | Descripción |
|-----------|------------|-------------|
| `Button` | `variant`, `size`, `disabled`, `loading`, `onClick` | Variantes: `primary`, `secondary`, `ghost`, `danger` |
| `Modal` | `show`, `onHide`, `title`, `footer`, `size` | Wrapper de Bootstrap modal |
| `Alert` | `variant` | Variantes: `success`, `warning`, `danger`, `info` |
| `Skeleton` | `height`, `className` | Placeholder de carga animado |
| `EmptyState` | `icon`, `title`, `description` | Estado vacío con icono Lucide |
| `Spinner` | `size` | `size="sm"` para inline en botones |
| `Select` | `value`, `onChange`, `options` | `options=[{value, label}]` |
| `Tabs` | `activeKey`, `onSelect` | Con `Tabs.Tab` como children |
| `Badge` | `variant`, `size` | Pill de color para etiquetas |
| `Card` | `className` | Wrapper de tarjeta con sombra suave |
| `Input` | `label`, `error`, `...nativeProps` | Input con label y mensaje de error integrados |
| `Textarea` | `label`, `error`, `rows` | Textarea con label y error |
| `Toast` | `message`, `variant`, `visible` | Notificación flotante temporal |
| `Tooltip` | `text`, `children` | Tooltip al hover sobre un elemento |
| `ConfirmDialog` | `show`, `onConfirm`, `onCancel`, `requireText` | Dialog de confirmación, opcionalmente requiere escribir un texto |
| `ErrorBoundary` | `fallback` | Captura errores de render de React |
| `ErrorChunk` | — | Fallback para errores de lazy loading (chunk no cargado) |

---

## 22. Frontend — Páginas principales

### `GestionTorneo.jsx`

Página privada del organizador. Estado clave:
```js
const [torneo, setTorneo]
const [rondas, setRondas]
const [tabActivo, setTabActivo]         // key del tab: 'crear' | 'ronda-{id}'
const [tipoRonda, setTipoRonda]
const [confirmarEliminar, setConfirmarEliminar]
const [confirmarEstado, setConfirmarEstado]  // { nuevo, label }
```

`handleCrearRonda`: agrega la nueva ronda al array local y cambia el tab activo a esa ronda.
`handleEliminarRonda`: filtra la ronda del array y vuelve al tab `'crear'`.
`handleCambiarEstado`: actualiza solo el campo `estado` del torneo local sin recargar todo.

### `DetalleTorneo.jsx`

Vista pública que adapta su contenido según quién la ve:

```js
const esOrganizador =
  user != null &&
  (user.rol === ROLES.ORGANIZADOR || user.rol === ROLES.TIENDA) &&
  user.id === torneo.organizador_id;
```

`cargarInscripciones()` está separada de `cargarDatos()` y se pasa como `onCambio` a `BandejaInscripciones` para que la lista de inscritos se refresque en tiempo real sin recargar el torneo completo.

---

## 23. Frontend — Enrutamiento y code splitting

### `AppRoutes.jsx`

Todas las páginas excepto `Landing` usan `React.lazy()` + `Suspense`. Esto divide el bundle en chunks separados que solo se descargan cuando el usuario navega a esa ruta.

```js
const DetalleMazo = lazy(() => import('@/modules/mazos/pages/DetalleMazo'));
// Solo se descarga cuando el usuario navega a /mazos/:id
```

Cada ruta lazy está envuelta en `ConSuspense` que combina `Suspense` (muestra spinner durante la carga) con `ErrorBoundary` (muestra `ErrorChunk` si el chunk no se puede cargar, con botón de reintento).

**`ProtectedRoute`**: acepta `requireRol` como string o array de strings. Redirige a `/login` si no hay sesión, a `/forbidden` si hay sesión pero el rol no coincide.

**Alias `@/`**: apunta a `src/`. Siempre usar `@/` para imports que no sean del mismo directorio.

---

## 24. Variables de entorno y configuración

### Backend

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `PORT` | Puerto del servidor Express | `3001` |
| `NODE_ENV` | Entorno | `development` / `production` |
| `DATABASE_URL` | Connection string PostgreSQL (Supabase) | `postgresql://user:pass@host:5432/db` |
| `SUPABASE_URL` | URL del proyecto Supabase | `https://xxx.supabase.co` |
| `SUPABASE_SERVICE_KEY` | Service role key (privilegios admin) | `eyJ...` |
| `DEEPSEEK_API_KEY` | API key de DeepSeek para LLM (explicaciones + autocompletado) | `sk-...` |
| `NOMIC_API_KEY` | API key de Nomic AI para embeddings | `nk-...` |
| `RESEND_API_KEY` | API key de Resend para emails | `re_...` |
| `EMAIL_FROM` | Dirección de envío de correos | `no-reply@deckora.app` |

### Frontend

| Variable | Descripción |
|----------|-------------|
| `VITE_SUPABASE_URL` | URL del proyecto Supabase |
| `VITE_SUPABASE_ANON_KEY` | Anon key (pública) |
| `VITE_API_URL` | URL base de la API **incluyendo `/api`** (`https://deckora-api.onrender.com/api`) |
| `VITE_MAPBOX_TOKEN` | Token público de Mapbox |

**Importante**: `VITE_API_URL` ya incluye `/api`, por eso los servicios usan paths sin ese prefijo:
```js
apiGet('/torneos')
// → fetch('https://deckora-api.onrender.com/api/torneos')
```

Las variables `VITE_*` se incrustan en el bundle en tiempo de build. Deben configurarse en el dashboard de Vercel. El archivo `.env` local no se sube al repositorio.

---

## 25. Tecnologías, lenguaje y dependencias (stack definitivo)

### Frontend — `Deckora-Web`

| Tecnología | Versión | Rol |
|------------|---------|-----|
| React | ^19.2.5 | Framework UI |
| Vite | ^8.0.10 | Bundler y servidor de desarrollo |
| React Router DOM | ^7.14.2 | Enrutamiento SPA con lazy loading |
| React Bootstrap | ^2.10.10 | Componentes Bootstrap para React |
| Bootstrap | ^5.3.8 | CSS y utilidades |
| Lucide React | ^1.14.0 | Íconos SVG |
| Mapbox GL | ^3.23.1 | Mapa interactivo de tiendas |
| Recharts | ^3.8.1 | Gráficos de estadísticas |
| `@supabase/supabase-js` | ^2.105.1 | Auth cliente + gestión de sesión |

### Backend — `deckora-api`

| Tecnología | Versión | Rol |
|------------|---------|-----|
| Node.js | ≥18 (ESM) | Runtime (`"type": "module"`) |
| Express | ^5.2.1 | Framework HTTP |
| Sequelize | ^6.37.8 | ORM para PostgreSQL |
| pg / pg-hstore | ^8.20.0 | Driver PostgreSQL nativo |
| sequelize-cli | ^6.6.5 | Migraciones de base de datos |
| `@supabase/supabase-js` | ^2.105.1 | Admin SDK para verificar JWT |
| Zod | ^4.3.6 | Validación de schemas de request |
| Resend | ^6.12.3 | Envío de correos transaccionales |
| p-limit | ^7.3.0 | Control de concurrencia (script embeddings) |
| dotenv | ^17.4.2 | Variables de entorno |

### Infraestructura y servicios externos

| Servicio | Propósito |
|----------|-----------|
| **Vercel** | Hosting del frontend |
| **Render.com** | Hosting del backend (Node.js Web Service) |
| **Supabase** | PostgreSQL gestionado + Auth + pgvector |
| **Nomic AI** | Embeddings vectoriales (proceso offline, gratuito) |
| **DeepSeek** | LLM para explicaciones y autocompletado de mazos |
| **Mapbox** | Mapa interactivo de tiendas |
| **Resend** | Notificaciones de inscripción por email |

### Lenguaje

Todo el proyecto usa **JavaScript ES Modules** (sin TypeScript). `"type": "module"` en ambos `package.json`. Se usa `import/export` en todos los archivos.

---

## 26. Configuración de servidor de producción (paso a paso)

### Paso 1 — Supabase

1. Crear proyecto en [supabase.com](https://supabase.com).
2. Activar extensión `pgvector`:
   ```sql
   CREATE EXTENSION IF NOT EXISTS vector;
   ```
3. Anotar desde **Settings → API**: Project URL, anon key, service_role key, connection string.
4. En **Authentication → Settings**, agregar el dominio de Vercel a **Redirect URLs**.

### Paso 2 — Backend en Render.com

1. Nuevo **Web Service** → conectar `Deckora-API` rama `main`.
2. Build: `npm install` | Start: `npm start` | Node ≥18.
3. Variables de entorno: `NODE_ENV`, `DATABASE_URL`, `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `DEEPSEEK_API_KEY`, `NOMIC_API_KEY`, `RESEND_API_KEY`, `EMAIL_FROM`.
4. Ejecutar migraciones: `npm run db:migrate`
5. Poblar cartas: `node scripts/seedCartas.js` (~10-30 min)
6. Generar embeddings: `npm run embed:generate` (~10-20 min)

### Paso 3 — Frontend en Vercel

1. Importar `Deckora-Web` en [vercel.com](https://vercel.com). Vite se detecta automáticamente.
2. Variables de entorno: `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, `VITE_API_URL`, `VITE_MAPBOX_TOKEN`.
3. El `vercel.json` ya incluye el rewrite SPA:
   ```json
   { "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }] }
   ```

---

## 27. Integraciones necesarias

### 1. Supabase Auth
Autenticación de usuarios y emisión de JWT. El backend verifica tokens con la `service_role` key.
- Backend: `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`
- Frontend: `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`

### 2. Nomic AI — Embeddings vectoriales
Genera los vectores de 768 dimensiones para cada carta. Se ejecuta **una sola vez** en el script offline.
- Modelo: `nomic-embed-text-v1.5` | Endpoint: `api-atlas.nomic.ai/v1/embedding/text`
- Obtener API key: [atlas.nomic.ai](https://atlas.nomic.ai)
- Variable: `NOMIC_API_KEY`

### 3. DeepSeek — LLM
Genera explicaciones de recomendaciones y listas de mazos para autocompletado. En tiempo de request.
- Modelo: `deepseek-chat` | Endpoint: `api.deepseek.com/chat/completions`
- Obtener API key: [platform.deepseek.com](https://platform.deepseek.com)
- Variable: `DEEPSEEK_API_KEY`

### 4. Mapbox — Mapas interactivos
Renderiza el mapa de tiendas en la landing y el selector de ubicación en configuración de tienda.
- Variable: `VITE_MAPBOX_TOKEN` (token público embebido en el bundle, normal para Mapbox)
- Obtener: [mapbox.com](https://mapbox.com) → cuenta → Tokens

### 5. Resend — Email transaccional
Envía notificaciones de inscripción a torneos (solicitud, aprobación, rechazo).
- Capa gratuita: 3.000 emails/mes, dominio `@resend.dev` sin verificación para pruebas
- Variable: `RESEND_API_KEY`, `EMAIL_FROM`
- Obtener: [resend.com](https://resend.com)
