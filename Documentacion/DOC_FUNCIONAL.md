# Deckora — Documentación Funcional

Esta documentación explica el **qué** y el **para qué** del sistema: quiénes son los usuarios, qué pueden hacer, cómo fluyen los procesos de punta a punta, y cómo se relacionan las distintas partes de la aplicación. No asume conocimiento técnico previo.

---

## Índice

1. [Qué es Deckora](#1-qué-es-deckora)
2. [Roles de usuario](#2-roles-de-usuario)
3. [Mapa de la aplicación](#3-mapa-de-la-aplicación)
4. [Flujo: Registro e inicio de sesión](#4-flujo-registro-e-inicio-de-sesión)
5. [Flujo: Gestión de mazos](#5-flujo-gestión-de-mazos)
6. [Flujo: Asistente IA — Recomendaciones y Autocompletado](#6-flujo-asistente-ia--recomendaciones-y-autocompletado)
7. [Flujo: Ciclo de vida de un torneo](#7-flujo-ciclo-de-vida-de-un-torneo)
8. [Flujo: Inscripción a un torneo](#8-flujo-inscripción-a-un-torneo)
9. [Flujo: Gestión de rondas](#9-flujo-gestión-de-rondas)
10. [Flujo: Reporte de resultados](#10-flujo-reporte-de-resultados)
11. [Flujo: Tabla de posiciones](#11-flujo-tabla-de-posiciones)
12. [Flujo: Biblioteca de cartas](#12-flujo-biblioteca-de-cartas)
13. [Diagrama de datos](#13-diagrama-de-datos)
14. [Diagrama de arquitectura](#14-diagrama-de-arquitectura)
15. [Vistas por rol — Resumen](#15-vistas-por-rol--resumen)
16. [Arquitectura de software y patrones de diseño](#16-arquitectura-de-software-y-patrones-de-diseño)
17. [Tecnologías, lenguaje y dependencias](#17-tecnologías-lenguaje-y-dependencias)
18. [Configuración de servidor de producción](#18-configuración-de-servidor-de-producción)
19. [Integraciones externas](#19-integraciones-externas)

---

## 1. Qué es Deckora

Deckora es una plataforma web para la comunidad de **Magic: The Gathering**. Está pensada para cualquier persona que disfrute del juego, ya sea de forma competitiva o casual, y sin importar el formato que prefiera (Commander, Standard, Modern, Pioneer, Legacy).

Tiene tres propósitos principales:

- **Para jugadores**: construir y gestionar mazos con asistencia de inteligencia artificial, inscribirse a torneos, ver su historial y estadísticas personales.
- **Para organizadores y tiendas**: crear y administrar torneos, gestionar inscripciones, llevar el marcador de las rondas y comunicarse con los participantes por correo.
- **Para cualquier visitante**: explorar la cartelera de torneos públicos, conocer tiendas cerca de su ubicación y navegar la biblioteca de cartas del juego (~32.500 cartas).

> **Nota importante**: aunque Commander es el formato más usado dentro de Deckora (y recibe algunas reglas especiales como la validación del comandante), la plataforma fue diseñada desde el inicio para ser **multi-formato**. Un torneo de Standard, un mazo de Pioneer o una colección de Legacy funcionan exactamente igual en el sistema.

---

## 2. Roles de usuario

```
┌──────────────────────────────────────────────────────────────────┐
│                        ROLES EN DECKORA                          │
├────────────────┬─────────────────────┬───────────────────────────┤
│   JUGADOR      │    ORGANIZADOR       │         TIENDA             │
├────────────────┼─────────────────────┼───────────────────────────┤
│ • Crear y      │ • Crear torneos      │ • Idéntico a organizador   │
│   gestionar    │ • Publicar torneos   │ • Tiene perfil de tienda   │
│   mazos en     │ • Gestionar rondas   │   con datos de ubicación   │
│   cualquier    │ • Aprobar/rechazar   │   y contacto               │
│   formato      │   inscripciones      │                            │
│                │ • Reportar           │                            │
│ • Usar el      │   resultados         │                            │
│   Asistente    │ • Cambiar estado     │                            │
│   IA para      │   del torneo         │                            │
│   mejorar o    │                      │                            │
│   completar    │                      │                            │
│   su mazo      │                      │                            │
│                │                      │                            │
│ • Inscribirse  │                      │                            │
│   a torneos    │                      │                            │
│                │                      │                            │
│ • Ver sus      │                      │                            │
│   estadísticas │                      │                            │
└────────────────┴─────────────────────┴───────────────────────────┘
```

**Regla importante:** un usuario tiene exactamente un rol y no puede cambiarlo. El rol se elige al momento del registro.

La **cartelera de torneos**, los **perfiles públicos** y la **biblioteca de cartas** son accesibles sin registrarse.

> **Tip para la presentación**: el rol de "tienda" existe porque las tiendas de Magic son las que típicamente organizan eventos. Tienen los mismos permisos que un organizador pero además aparecen en el mapa de Mapbox con su dirección física, permitiendo que jugadores las encuentren geográficamente.

---

## 3. Mapa de la aplicación

```
DECKORA WEB
│
├── / (Landing)                 Presentación pública con mapa de tiendas
│
├── /torneos                    Cartelera pública de torneos (con filtros)
│   └── /torneos/:id            Detalle de torneo (vista adaptada por rol)
│
├── /biblioteca                 Catálogo de cartas (~32.500 cartas MTG)
│
├── /u/:username                Perfil público de cualquier usuario
│
├── /login  /registro  /recuperar
│
└── [Zona autenticada]
    │
    ├── [Jugador]
    │   ├── /jugador            Dashboard personal con estadísticas
    │   ├── /mazos              Mis mazos (todos los formatos)
    │   ├── /mazos/:id          Detalle y edición de mazo (con Asistente IA)
    │   └── /configuracion      Configuración de cuenta y perfil
    │
    ├── [Organizador / Tienda]
    │   ├── /organizador        Dashboard con torneos activos
    │   ├── /organizador/torneos/nuevo         Crear torneo
    │   ├── /organizador/torneos/:id/editar    Editar torneo
    │   ├── /organizador/torneos/:id/gestion   Panel de gestión de rondas
    │   └── /mis-torneos        Historial de torneos organizados
    │
    └── /torneos/:id            (vista diferente si eres el organizador)
```

---

## 4. Flujo: Registro e inicio de sesión

### Registro de nuevo usuario

```
Usuario completa formulario:
  nombre de usuario, correo, contraseña, rol (jugador / organizador / tienda)
         ↓
  [Backend] Crea usuario en Supabase Auth
         ↓
  [Backend] Guarda en tabla 'usuarios' (id, nombre_usuario, correo, rol)
         ↓
  [Backend] Crea perfil extendido según rol:
    • jugador      → tabla 'jugadores' + tabla 'estadisticas' (contadores en 0)
    • organizador  → tabla 'organizadores'
    • tienda       → tabla 'tiendas'
         ↓
  Dependiendo de la configuración de Supabase:
    A) Verificación de correo desactivada → devuelve token → usuario puede operar
    B) Verificación de correo activada   → "Revisa tu correo para confirmar tu cuenta"
```

Si algo falla al crear el perfil extendido, el sistema intenta eliminar el usuario de Supabase Auth para no dejar cuentas incompletas ("huérfanas").

---

### Inicio de sesión

```
Usuario ingresa correo + contraseña
         ↓
  Supabase Auth verifica credenciales
         ↓
         ├── ✗ Credenciales incorrectas → mensaje de error genérico
         │                                (no distingue si el email no existe
         │                                 o la contraseña es incorrecta, por seguridad)
         └── ✓ Correcto → devuelve token JWT (válido 1 hora)
                    ↓
             Frontend carga /auth/me
             → obtiene usuario con rol y perfil extendido
                    ↓
             Redirige al dashboard del rol:
             • jugador      → /jugador
             • organizador  → /organizador
             • tienda       → /tienda
```

### Token expirado — renovación automática

```
Frontend hace cualquier request al servidor
         ↓
    Token expirado (1 hora de vida)
         ↓
  API devuelve HTTP 401
         ↓
  Frontend intenta renovar el token automáticamente
  (supabase.auth.refreshSession)
         ↓
         ├── ✓ Renovación exitosa → reintenta la request original sin que el usuario note nada
         └── ✗ No se pudo renovar → cierra sesión → redirige a /login
```

---

## 5. Flujo: Gestión de mazos

### Crear un mazo

Solo los jugadores pueden crear mazos.

```
Jugador en /mazos/nuevo:
  1. Ingresa nombre del mazo
  2. Elige formato (Commander, Standard, Modern, Legacy, Pioneer)
  3. Opcionalmente: agrega descripción, marca como público/privado
         ↓
  [Backend] Crea el mazo con un slug único generado del nombre
  Ej: "Mi Mazo Dragones" → slug: "mi-mazo-dragones-1715000000000"
         ↓
  Redirige a /mazos/:id para comenzar a agregar cartas
```

---

### Agregar cartas al mazo

```
Jugador busca una carta en el DeckBuilder:
         ↓
  BarraAgregarCarta espera 300ms (debounce) tras el último keystroke
  antes de disparar la búsqueda (evita requests por cada letra)
         ↓
  GET /cartas/buscar?q=nombre&formato=MODERN
         ↓
  [Backend] Filtra por nombre + legalidad en el formato del mazo
  Excluye automáticamente tokens (ej. Thopter Token) y cartas de arte
         ↓
  Jugador selecciona cantidad y si es comandante (solo en Commander)
         ↓
  [Frontend] Actualiza la UI INMEDIATAMENTE (optimistic update)
         ↓
  [Backend] Agrega la carta a la tabla mazo_cartas
         ↓
         ├── ✓ Éxito → la UI ya mostraba el cambio, todo bien
         └── ✗ Error  → la UI revierte la carta agregada + muestra toast de error
```

**Filtro de formato**: la búsqueda siempre pasa el formato del mazo. Esto evita que el jugador agregue cartas no permitidas en ese formato sin darse cuenta.

---

### Importar un mazo desde texto

```
Jugador abre "Importar mazo" y pega una lista de texto:
  1 Sol Ring (CMD) 236
  1 Command Tower
  4 Lightning Bolt
         ↓
  POST /mazos/:id/importar
         ↓
  [Backend] Parsea cada línea (soporta formato completo e.g. "1 Nombre (SET) 001"
  y formato simple e.g. "1 Nombre")
         ↓
  Por cada carta: busca primero por set+número, luego por nombre exacto,
  luego por nombre parcial
         ↓
  Devuelve { importadas: [...], fallidas: [...] }
  Las fallidas no abortan el proceso — se siguen importando las demás
```

---

### Validar el mazo

```
Jugador hace clic en "Validar mazo"
         ↓
  [Backend] Aplica las reglas del formato:

  ── COMMANDER ─────────────────────────────────────────────────────
  ✓ Total de 100 cartas exactamente
  ✓ Exactamente 1 comandante marcado
  ✓ El comandante es una Legendary Creature
  ✓ Máximo 1 copia de cada carta (excepto tierras básicas)

  ── STANDARD ──────────────────────────────────────────────────────
  ✓ Entre 60 y 250 cartas
  ✓ Máximo 4 copias por carta (excepto tierras básicas)

  ── MODERN / PIONEER / LEGACY ─────────────────────────────────────
  ✓ Mínimo 60 cartas
  ✓ Máximo 4 copias por carta (excepto tierras básicas)
         ↓
  Devuelve { valido: true/false, errores: ["...", "..."] }
         ↓
  Frontend muestra los errores específicos en el PanelValidacion
```

> **Decisión de diseño importante**: la validación del mazo también se ejecuta al **inscribirse a un torneo**. Esto impide que un jugador inscriba un mazo inválido, garantizando consistencia entre lo que el sistema considera válido y lo que se acepta para competir.

---

## 6. Flujo: Asistente IA — Recomendaciones y Autocompletado

El Asistente IA es una funcionalidad exclusiva del modo edición de mazo. Tiene dos capacidades distintas que se muestran en el mismo panel lateral del DeckBuilder.

---

### Capacidad 1: Recomendaciones vectoriales

Analiza las cartas del mazo y sugiere 10 cartas que tienen un estilo mecánico similar.

#### ¿Cómo funciona "recomendar por estilo"?

Un **embedding** es una representación matemática del "significado" de una carta. La IA convierte el texto descriptivo (tipo, costo de maná, colores) en una lista de 768 números. Cartas similares producen números similares:

```
"Instant CMC:1 Colors:R"  →  [-0.09, 0.014, -0.148, ...]  (768 números)
"Sorcery CMC:2 Colors:R"  →  [-0.08, 0.018, -0.130, ...]  (números cercanos → cartas similares)
"Creature CMC:5 Colors:G" →  [0.31, -0.092, 0.204, ...]   (números lejanos → cartas distintas)
```

#### Flujo de una recomendación

```
Jugador hace clic en "Pedir recomendaciones"
         ↓
  [Frontend] Muestra spinner "Analizando tu mazo..."
         ↓
  GET /mazos/:id/recomendaciones
         ↓
  [Backend] Carga el mazo con todas sus cartas y sus vectores
         ↓
  Calcula el VECTOR PROMEDIO del mazo
  (promedio matemático de todos los embeddings)
  Este vector representa el "estilo general" del mazo
         ↓
  [PostgreSQL + pgvector]
  Busca las 10 cartas de la BD más parecidas al vector promedio
  usando distancia coseno
  (excluye cartas ya en el mazo y las explícitamente baneadas)
         ↓
  [DeepSeek API]
  Genera 3-4 bullets de explicación en español sobre las recomendaciones
  (si falla, las recomendaciones se muestran igual sin explicación)
         ↓
  Frontend muestra:
  • Explicación en texto con nombres de cartas en negrita
  • Lista de cartas recomendadas con tipo, costo y extracto del texto
  • Botón "Agregar al mazo" por carta
```

#### ¿Cómo se generan los embeddings de las cartas?

Los vectores no se generan en tiempo real. Existe un proceso offline que se ejecuta una vez (y se puede re-ejecutar si se agregan nuevas cartas):

```
Script: npm run embed:generate

1. Consulta todas las cartas sin embedding en la BD
2. Las agrupa en lotes de 50
3. Envía cada lote a la API de Nomic AI (gratis)
4. Guarda los 768 números en la columna 'embedding' de la tabla 'cartas'
5. Si algo falla, reintenta hasta 5 veces con espera progresiva (1s → 2s → 4s → 8s → 16s)

Tiempo estimado para ~32.500 cartas: 10-20 minutos
```

---

### Capacidad 2: Autocompletado con IA

Rellena automáticamente las cartas que faltan para completar el mazo según el límite del formato.

```
Jugador hace clic en "Autocompletar con IA"
         ↓
  [Caso especial: mazo vacío]
  Si el mazo no tiene cartas y hay plantillas predefinidas para el formato,
  se carga una plantilla aleatoria en lugar de llamar al LLM
  (más rápido y predictible para mazos nuevos)
         ↓
  [Caso normal: mazo parcialmente lleno]
  POST /mazos/:id/autocompletar
         ↓
  [Backend] Calcula cuántas cartas faltan:
  • Commander: necesita llegar a 100
  • Standard/Modern/Pioneer/Legacy: necesita llegar a 60
         ↓
  [DeepSeek API]
  Envía prompt: "Genera X cartas para completar este mazo de formato F.
  Mazo: {nombre}. Comandante: {si aplica}. Cartas existentes: [...]"
         ↓
  DeepSeek devuelve una lista de nombres de cartas en inglés
         ↓
  [Backend] Busca cada carta en la BD local por nombre y la agrega al mazo
  Las cartas no encontradas se registran como "fallidas" pero no abortan el proceso
         ↓
  [Solo Commander] Si ninguna carta quedó marcada como comandante,
  busca la primera Legendary Creature del mazo y la marca automáticamente
         ↓
  [Frontend] Recarga el mazo completo con las cartas nuevas
```

---

## 7. Flujo: Ciclo de vida de un torneo

### Estados y transiciones

```
                    ┌──────────┐
                    │PENDIENTE │ ← Estado inicial al crear
                    └─────┬────┘
                          │ Organizador publica (si hay suficientes confirmados)
                          ▼
                    ┌──────────┐
                    │EN CURSO  │ ← El torneo está activo, se juegan rondas
                    └─────┬────┘
                          │ Organizador finaliza (si no hay enfrentamientos pendientes)
                          ▼
                    ┌──────────┐
                    │FINALIZADO│ ← Estado terminal
                    └──────────┘

    Desde PENDIENTE o EN CURSO → CANCELADO (estado terminal)
```

### Lo que cambia en cada estado

| Estado | Inscripciones | Rondas | Resultados |
|--------|--------------|--------|------------|
| `pendiente` | Abiertas | No se pueden crear | — |
| `en_curso` | Cerradas | Se pueden crear y gestionar | Se pueden reportar |
| `finalizado` | Cerradas | Solo lectura | Solo lectura |
| `cancelado` | Cerradas | Solo lectura | Solo lectura |

### Validación para iniciar el torneo

```
Organizador hace clic en "Publicar torneo" (cambiar a EN_CURSO)
         ↓
  [Backend] Cuenta inscripciones con confirmado = true:
    • Formato Commander → mínimo 3 jugadores confirmados (mínimo para una mesa)
    • Otros formatos    → mínimo 2 jugadores confirmados
         ↓
         ├── ✗ Menos del mínimo → error, el torneo no puede iniciarse
         └── ✓ Suficientes → estado cambia a EN_CURSO
                             El torneo aparece en la cartelera pública
```

### Al finalizar el torneo

```
Organizador hace clic en "Finalizar torneo"
         ↓
  [Backend] Verifica que no haya enfrentamientos sin resultado
         ↓
  En una TRANSACCIÓN (todo o nada):
    • Incrementa 'torneos_participados' en las estadísticas de cada jugador inscrito
    • Cambia el estado del torneo a 'finalizado'
  Si algo falla → rollback completo (ni los contadores ni el estado se modifican)
```

---

## 8. Flujo: Inscripción a un torneo

### Inscripción (jugador)

```
Jugador visita /torneos/:id (torneo en estado PENDIENTE)
         ↓
  PanelInscripcion muestra el formulario
         ↓
  Jugador selecciona uno de sus mazos
         ↓
  [Backend] Verifica en orden:
    ✓ El torneo existe y está pendiente
    ✓ El jugador no está ya inscrito
    ✓ El mazo no está ya inscrito en este torneo
    ✓ El mazo pertenece al jugador
    ✓ El formato del mazo coincide con el formato del torneo
    ✓ El mazo pasa la validación de reglas del formato
         ↓
  Crea la inscripción con confirmado: false (pendiente de aprobación)
         ↓
  [Backend] Guarda un SNAPSHOT del mazo:
  Copia todas las cartas tal como están en ese momento.
  Aunque el jugador modifique el mazo después, el snapshot preserva lo declarado.
         ↓
  [Email, no bloqueante] Notifica al organizador: "Jugador X quiere inscribirse con Mazo Y"
```

> **Por qué validar el mazo al inscribirse**: el sistema impide que un jugador inscriba un mazo que no cumple las reglas del formato del torneo. Así el organizador no recibe solicitudes de mazos inválidos que tendría que revisar manualmente.

### Aprobación de inscripciones (organizador)

```
Organizador en /torneos/:id (sección BandejaInscripciones)
         ↓
  Ve las solicitudes pendientes con nombre del jugador y mazo
  Puede hacer clic en "Ver mazo inscrito" para ver el snapshot exacto
         ↓
         ├── Aprobar → inscripción queda confirmada (confirmado: true)
         │              [Email] Notifica al jugador: "Tu inscripción fue aceptada"
         │              La lista de inscritos se refresca automáticamente
         │
         └── Rechazar → inscripción es eliminada
                         [Email] Notifica al jugador: "Tu inscripción fue rechazada"
                         El jugador puede intentar con otro mazo

Nota: solo las inscripciones confirmadas cuentan para el mínimo requerido
para iniciar el torneo.
```

---

## 9. Flujo: Gestión de rondas

El organizador gestiona las rondas desde `/organizador/torneos/:id/gestion`.

### Tipos de ronda

```
Organizador selecciona el tipo de ronda:
         │
         ├── SWISS ─────────────────────────────────────────────────
         │   Emparejamiento por puntos acumulados.
         │
         │   Para COMMANDER (mesas de 3-4 jugadores):
         │   Agrupa optimizando el número de mesas completas de 4.
         │   Ejemplo (8 jugadores: A=10pts, B=8pts, ... H=1pt):
         │     Mesa 1: [A, B, C, D]
         │     Mesa 2: [E, F, G, H]
         │
         │   Para OTROS FORMATOS (1v1):
         │   El 1° juega contra el 2°, el 3° contra el 4°, etc.
         │     Mesa 1: [A vs B]
         │     Mesa 2: [C vs D]
         │
         ├── ELIMINACIÓN DIRECTA ────────────────────────────────────
         │   Bracket: el mejor contra el peor.
         │   Si el organizador no asigna mesas manualmente, se genera auto-bracket.
         │
         │   Ejemplo con 8 jugadores ordenados:
         │     Mesa 1: [A vs H]    Mesa 2: [B vs G]
         │     Mesa 3: [C vs F]    Mesa 4: [D vs E]
         │
         └── FINAL ───────────────────────────────────────────────────
             Mismo mecanismo que eliminación directa.
             Para los jugadores clasificados al final del torneo.
```

### Crear una ronda

```
Organizador elige tipo y hace clic en "Crear ronda"
         ↓
  [Backend] Verifica que no haya enfrentamientos pendientes
  en la ronda anterior (no se puede avanzar con partidas sin resultado)
         ↓
  Calcula puntos acumulados de cada inscripción
  (suma de puntos de todos los enfrentamientos anteriores)
         ↓
  Aplica el emparejador correspondiente
         ↓
  Crea la ronda y las mesas (enfrentamientos) con todos los jugadores asignados
         ↓
  Aparece un nuevo tab en la vista de gestión con las mesas de esa ronda
```

### Eliminar una ronda

```
Organizador hace clic en "Eliminar ronda"
         ↓
  Modal de confirmación
         ↓
  [Backend] Verifica que NINGÚN enfrentamiento esté FINALIZADO
         ↓
         ├── ✗ Hay resultados registrados → error, no se puede eliminar
         └── ✓ Sin resultados → elimina participantes, enfrentamientos y la ronda
```

---

## 10. Flujo: Reporte de resultados

```
Organizador ve las mesas de la ronda actual
         ↓
  Hace clic en "Reportar resultado" en una mesa
         ↓
  Se abre ReportarResultadoModal con los jugadores de esa mesa
         ↓
  Organizador asigna resultado a cada jugador:
    • Ganador (3 puntos)
    • Empate  (1 punto)
    • Derrota (0 puntos)
         ↓
  Validaciones en el FRONTEND (antes de enviar):
    ✓ Todos los jugadores tienen resultado asignado
    ✓ No hay más de 1 ganador por mesa
         ↓
  POST /enfrentamientos/:id/resultado
         ↓
  [Backend] Valida las mismas reglas nuevamente (defensa en profundidad)
         ↓
  En una TRANSACCIÓN (todo o nada):
    1. Registra resultado y puntos de cada participante
    2. Actualiza estadísticas del jugador (partidas_ganadas / partidas_empatadas / partidas_perdidas)
    3. Cambia estado del enfrentamiento a FINALIZADO
         ↓
  La mesa queda marcada visualmente como finalizada
  Los puntos ya están disponibles para el emparejamiento de la próxima ronda
```

**Sistema de puntos:**

| Resultado | Puntos |
|-----------|--------|
| Ganador | 3 puntos |
| Empate | 1 punto |
| Derrota | 0 puntos |

---

## 11. Flujo: Tabla de posiciones

```
Cualquier visitante accede al detalle de un torneo en curso o finalizado
         ↓
  [Backend] Para cada inscripción:
    • Suma todos los puntos_obtenidos de sus participaciones → puntos_totales
    • Cuenta cuántas veces obtuvo resultado 'ganador' → victorias
         ↓
  Ordena por puntos_totales (mayor primero)
  Si hay empate de puntos → desempate por victorias
         ↓
  Asigna posición (1°, 2°, 3°...)
         ↓
  DetalleTorneo muestra la tabla:
  #  │ Jugador  │ Pts │ V │ D │ E
  ───┼──────────┼─────┼───┼───┼───
  1  │ jugador1 │ 12  │ 4 │ 0 │ 0
  2  │ jugador2 │  9  │ 3 │ 0 │ 0
  3  │ jugador3 │  4  │ 1 │ 1 │ 1
```

---

## 12. Flujo: Biblioteca de cartas

La biblioteca es pública y no requiere autenticación.

```
Visitante accede a /biblioteca
         ↓
  [Backend] Carga los sets disponibles (agrupados por año)
  [Backend] Carga la primera página de cartas (50 cartas)
         ↓
  Visitante ve un grid de cartas con imagen, nombre y set
         ↓
         ├── Filtro por set:
         │     Visitante selecciona un set del desplegable
         │     → La URL cambia: /biblioteca?set_codigo=DSK&page=1
         │     → Se recarga el grid con solo las cartas de ese set
         │
         ├── Paginación:
         │     Visitante hace clic en página siguiente
         │     → La URL cambia: /biblioteca?page=2
         │     → Se carga la página 2
         │
         └── Ver carta en detalle:
               Visitante hace clic en una carta
               → Abre CartaDetalleModal con imagen grande, texto y datos
```

**Estado en la URL**: los filtros y la página actual se guardan en la URL (`?set_codigo=DSK&page=2`), así si el visitante recarga la página ve el mismo resultado.

---

## 13. Diagrama de datos

### Entidades principales y sus relaciones

```
┌───────────┐          ┌───────────┐        ┌───────────┐
│  USUARIO  │──1:1───►  │  JUGADOR  │───1:N──│   MAZO    │
│           │          │           │        │           │
│ id        │          │ usuario_id│        │ nombre    │
│ nombre_usr│          │           │        │ formato   │
│ rol       │──1:1───►  │ORGANIZADOR│        │ usuario_id│
│ correo    │          │           │        └─────┬─────┘
│ activo    │──1:1───►  │  TIENDA   │              │ 1:N
└─────┬─────┘          └───────────┘        ┌─────▼─────┐
      │ 1:N                                 │MAZO_CARTA │
      │                                     │           │
      │                                     │ carta_id  │
      ▼                                     │ cantidad  │
┌───────────┐                              └─────┬─────┘
│  TORNEO   │                                    │ N:1
│           │                              ┌─────▼───────────────────┐
│ nombre    │                              │        CARTA             │
│ estado    │                              │                          │
│ formato   │                              │ nombre, tipo, imagen_url │
│ org_id    │                              │ embedding vector(768) ← IA│
└─────┬─────┘                              └──────────────────────────┘
      │ 1:N
      ▼
┌─────────────┐
│ INSCRIPCION │
│             │
│ torneo_id   │
│ usuario_id  │──── FK al Jugador
│ mazo_id     │
│ confirmado  │ (true = aprobado / false = pendiente)
└──────┬──────┘
       │ 1:N  (genera también snapshot de cartas)
       │
       ▼
┌──────────────────────────────────┐
│ SNAPSHOT_MAZO_INSCRIPCION        │
│ Foto del mazo al momento de      │
│ inscribirse (no varía aunque el  │
│ jugador cambie su mazo después)  │
└──────────────────────────────────┘

┌───────────┐
│  TORNEO   │──1:N──► ┌───────────┐
└───────────┘         │   RONDA   │
                      │           │
                      │ tipo_ronda│ (swiss / eliminacion_directa / final)
                      │ numero    │
                      └─────┬─────┘
                            │ 1:N
                            ▼
                      ┌──────────────────┐
                      │  ENFRENTAMIENTO  │
                      │  (Mesa)          │
                      │                  │
                      │ numero_mesa      │
                      │ estado           │ (pendiente / finalizado)
                      └────────┬─────────┘
                               │ 1:N
                               ▼
                      ┌────────────────────────────┐
                      │ ENFRENTAMIENTO_PARTICIPANTE │
                      │                             │
                      │ inscripcion_id  ────────────── FK a INSCRIPCION
                      │ resultado                   │
                      │ puntos_obtenidos            │
                      └─────────────────────────────┘
```

---

## 14. Diagrama de arquitectura

### Vista general del sistema

```
┌─────────────────────────────────────────────────────────────────────┐
│                             INTERNET                                 │
└──────┬──────────────────────────┬────────────────────┬──────────────┘
       │                          │                    │
       ▼                          ▼                    ▼
┌──────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
│ SUPABASE AUTH│   │    DECKORA WEB        │   │  NOMIC AI            │
│              │   │  (React + Vite)       │   │  api-atlas.nomic.ai  │
│ • JWT tokens │──►│  Vercel               │   │                      │
│ • Auth SDK   │   │                       │   │ Embeddings           │
└──────────────┘   └──────────┬────────────┘   │ gratuitos            │
                               │               └────────┬─────────────┘
                        HTTPS  │ + Bearer Token         │ API HTTPS
                               │                        │
                               ▼                        │
                  ┌────────────────────────┐           │
                  │   DECKORA API          │◄──────────┘
                  │  (Node.js + Express)   │
                  │  Render.com            │
                  │                        │──────► DeepSeek API
                  └───────────┬────────────┘        (LLM: explicaciones
                               │                     + autocompletado)
                        ORM    │ Sequelize + pgvector
                               ▼
                  ┌────────────────────────┐
                  │   POSTGRESQL           │
                  │   Supabase             │
                  │                        │
                  │  ~32.500 cartas MTG    │
                  │  con embeddings        │
                  │  vector(768)           │
                  │                        │
                  │  Usuarios, torneos,    │
                  │  mazos, rondas, etc.   │
                  └────────────────────────┘
```

### Flujo de una request autenticada

```
Usuario hace clic en "Inscribirse"
         ↓
React component → torneos.service.js → api.js
         │
         └── api.js: getAccessToken() con timeout 10s
             → inyecta Authorization: Bearer eyJ...
         ↓
POST /api/torneos/:id/inscripciones
Authorization: Bearer eyJ...token...
         ↓
  [Express] middleware auth.js:
    • Lee el token del header
    • Llama a supabase.auth.getUser(token)
    • Busca el usuario en tabla 'usuarios'
    • Verifica que la cuenta esté activa
    • Adjunta req.usuario al request
         ↓
  [Express] torneos.controller.js
  → llama a torneos.service.js
  → hace validaciones de negocio
  → llama a torneos.repository.js
  → ejecuta queries en PostgreSQL
         ↓
  Respuesta JSON al frontend
         ↓
React actualiza el estado local
Usuario ve la confirmación de inscripción
```

---

## 15. Vistas por rol — Resumen

### `/torneos/:id` — Detalle de torneo

Esta página adapta su contenido según quién la visita:

```
┌──────────────────────────────────────────────────────────────────┐
│                    /torneos/:id                                   │
│                                                                   │
│  ── Header ────────────────────────────────────────────────────  │
│  │  Nombre, formato, estado, fecha, ubicación, organizador    │  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                   │
│  Si es el ORGANIZADOR del torneo:                                 │
│  ── Administración ─────────────────────────────────────────── │ │
│  │  [Gestionar torneo]  [Publicar / Finalizar]  [Cancelar]   │  │
│  │  Solicitudes pendientes de inscripción:                    │  │
│  │  Jugador1 · MiMazo1  [Ver mazo] [Aprobar] [Rechazar]      │  │
│  │  Jugador2 · MiMazo2  [Ver mazo] [Aprobar] [Rechazar]      │  │
│  ───────────────────────────────────────────────────────────── │ │
│                                                                   │
│  Si es un JUGADOR (no organizador):                               │
│  ── Inscripción ─────────────────────────────────────────────    │
│  │  Seleccionar mazo: [dropdown]  [Inscribirse]               │  │
│  │  (o "Ya estás inscrito con Mazo X  [Cancelar inscripción]")│  │
│  ─────────────────────────────────────────────────────────────   │
│  ── Inscritos ────────────────────────────────────────────────   │
│  │  • Jugador1 · MazoA · hace 2 días                         │  │
│  │  • Jugador2 · MazoB · hace 1 día                          │  │
│  ─────────────────────────────────────────────────────────────   │
│                                                                   │
│  Si es ANÓNIMO:                                                   │
│  (muestra PanelInscripcion con aviso de "Iniciá sesión")          │
│                                                                   │
│  Para TODOS (si torneo en_curso o finalizado):                    │
│  ── Rondas ───────────────────────────────────────────────────   │
│  │  Ronda 1 (Swiss)                                          │  │
│  │  Mesa 1: Jugador A vs B vs C vs D                         │  │
│  │  [Reportar resultado] ← solo visible para el organizador  │  │
│  ─────────────────────────────────────────────────────────────   │
│  ── Tabla de posiciones ──────────────────────────────────────   │
│  │  # │ Jugador  │ Pts │ V │ D │ E                           │  │
│  │  1 │ jugador1 │ 12  │ 4 │ 0 │ 0                          │  │
│  ─────────────────────────────────────────────────────────────   │
└──────────────────────────────────────────────────────────────────┘
```

---

### `/mazos/:id` — Detalle y edición de mazo (con Asistente IA)

```
┌──────────────────────────────────────────────────────────────────┐
│  DECKBUILDER                                                      │
│                                                                   │
│  ── Buscar carta ──  ── Lista del mazo ──  ── Panel lateral ──   │
│  │  [nombre...] 🔍 │  │  [Carta 1 x1]   │  │ Stats           │  │
│  │                 │  │  [Carta 2 x2]   │  │ Curva de maná   │  │
│  │  Resultados:    │  │  [Carta 3 x1]   │  │                 │  │
│  │  • Sol Ring     │  │  ...            │  │ Colores         │  │
│  │  • Command Tow  │  └─────────────────┘  ├─────────────────┤  │
│  │  ...            │                        │ Validación      │  │
│  └─────────────────┘                        │ ✓ 100/100      │  │
│                                             ├─────────────────┤  │
│  [Importar mazo]                            │ Asistente IA ✨  │  │
│                                             │                 │  │
│                                             │ [Pedir          │  │
│                                             │  recomendac.]   │  │
│                                             │                 │  │
│                                             │ ─────────────   │  │
│                                             │ [Autocompletar  │  │
│                                             │  con IA]        │  │
│                                             └─────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### `/organizador/torneos/:id/gestion` — Panel de gestión

```
┌──────────────────────────────────────────────────────────────────┐
│  GESTIÓN: "Torneo Regional"    [Ver página pública →]             │
│  Commander  ·  EN CURSO        [Editar torneo]                    │
│                                                                   │
│  [Publicar torneo]  [Finalizar torneo]  [Cancelar torneo]         │
│                                                                   │
│  ── Tabs ──────────────────────────────────────────────────────  │
│  │  [Ronda 1] [Ronda 2] [Ronda 3] [+ Nueva ronda]            │  │
│  ─────────────────────────────────────────────────────────────   │
│                                                                   │
│  Ronda 2 · Swiss                                                  │
│  ── Mesa 1  ·  EN CURSO ───────────────────────────────────────  │
│  │  Jugador    │ Mazo/Comandante │ Resultado │ Pts          │   │
│  │  jugador1   │ MazoA / Atraxa  │    —      │  —           │   │
│  │  jugador2   │ MazoB / Muldrot │    —      │  —           │   │
│  │  jugador3   │ MazoC / Kenrith │    —      │  —           │   │
│  │  jugador4   │ MazoD / Edgar   │    —      │  —           │   │
│  │  [Reportar resultado]                                     │   │
│  ─────────────────────────────────────────────────────────────   │
│                                                                   │
│  [Eliminar ronda]                                                 │
└──────────────────────────────────────────────────────────────────┘
```

---

### Dashboards por rol

**Dashboard del Jugador** (`/jugador`):
- Estadísticas personales: partidas ganadas, perdidas, empatadas, torneos jugados, win rate
- Acceso rápido a sus mazos (todos los formatos)
- Próximos torneos disponibles para inscribirse

**Dashboard del Organizador** (`/organizador`):
- Sus torneos activos y pendientes
- Acceso rápido a crear nuevo torneo

**Dashboard de Tienda** (`/tienda`):
- Igual al organizador
- Perfil de tienda con información de contacto y ubicación en el mapa

---

## 16. Arquitectura de software y patrones de diseño

### Arquitectura en tres capas

El sistema está dividido en tres partes independientes que se comunican entre sí:

```
┌─────────────────────────────────┐
│  INTERFAZ DE USUARIO            │
│  Lo que ve el usuario           │
│  (navegador web)                │
└──────────────────┬──────────────┘
                   │ peticiones HTTP
┌──────────────────▼──────────────┐
│  SERVIDOR / LÓGICA              │
│  Las reglas del sistema         │
│  (validaciones, permisos)       │
└──────────────────┬──────────────┘
                   │ consultas
┌──────────────────▼──────────────┐
│  BASE DE DATOS                  │
│  Usuarios, torneos,             │
│  mazos, cartas, etc.            │
└─────────────────────────────────┘
```

---

### Patrones de diseño principales

#### Módulos por funcionalidad
Todo el código está organizado por área de la aplicación (torneos, mazos, identidad, etc.). Toda la lógica relacionada a un tema está junta: routes, controller, service y repository del mismo módulo viven en la misma carpeta.

#### Reglas de validación intercambiables (Strategy)
Las reglas de validación de mazos y el sistema de emparejamiento de rondas están diseñados de forma que agregar un nuevo formato o tipo de ronda no requiere modificar el código existente — solo se agrega una nueva "estrategia".

#### Una sola entrada para las peticiones HTTP (Facade)
Todas las comunicaciones del frontend hacia el servidor pasan por un único punto central (`api.js`) que maneja automáticamente la autenticación, los errores y la renovación de sesión.

#### Actualización optimista (UI)
Cuando el jugador agrega o elimina una carta, la UI se actualiza de forma inmediata sin esperar la respuesta del servidor. Si el servidor rechaza la operación, la UI se revierte al estado anterior. Hace la aplicación sentirse rápida incluso con latencia.

#### Capas de responsabilidad (backend)
Dentro del servidor, cada tipo de archivo tiene un rol fijo: las rutas dirigen, el controlador extrae datos del request, el servicio aplica reglas de negocio, el repositorio consulta la base de datos.

#### Carga diferida (Lazy Loading)
Las páginas del frontend se dividen en chunks separados que solo se descargan cuando el usuario navega a esa ruta. Esto reduce el tiempo de carga inicial de la aplicación.

> **Tip para la presentación**: si preguntan "¿cómo aseguran que el mazo cumpla las reglas al inscribirse?", la respuesta muestra dos patrones a la vez: Strategy (las reglas de validación son intercambiables por formato) y Defense in Depth (se valida tanto en el frontend como en el backend, y además al inscribirse en el torneo).

---

## 17. Tecnologías, lenguaje y dependencias

### Interfaz de usuario (Frontend)

| Tecnología | Para qué se usa |
|------------|----------------|
| **React 19** | Construir la interfaz visual |
| **Vite** | Herramienta de desarrollo y construcción del proyecto |
| **React Router** | Navegación entre páginas sin recargar el navegador, con lazy loading |
| **Bootstrap + React Bootstrap** | Estilos base y componentes de interfaz |
| **Mapbox GL** | Mapa interactivo de tiendas |
| **Recharts** | Gráficos de estadísticas del jugador |
| **Lucide React** | Íconos de la interfaz |

### Servidor (Backend)

| Tecnología | Para qué se usa |
|------------|----------------|
| **Node.js** | Entorno de ejecución del servidor |
| **Express** | Framework para manejar las peticiones HTTP |
| **Sequelize** | Traducir las operaciones del código a queries de base de datos |
| **Zod** | Validar que los datos que llegan al servidor tienen el formato correcto |
| **Resend** | Enviar correos de notificación de inscripciones |

### Base de datos e infraestructura

| Servicio | Para qué se usa |
|----------|----------------|
| **PostgreSQL (Supabase)** | Base de datos principal |
| **pgvector** | Extensión para búsqueda por similitud vectorial (Asistente IA) |
| **Supabase Auth** | Autenticación de usuarios (registro, login, tokens) |
| **Vercel** | Hosting de la interfaz web |
| **Render.com** | Hosting del servidor |

---

## 18. Configuración de servidor de producción

### Servicios necesarios (todos con plan gratuito disponible)

| Servicio | URL | Para qué |
|----------|-----|---------|
| Supabase | supabase.com | Base de datos y autenticación |
| Render | render.com | Hosting del servidor |
| Vercel | vercel.com | Hosting de la interfaz |
| Nomic AI | atlas.nomic.ai | Embeddings vectoriales (una sola vez) |
| DeepSeek | platform.deepseek.com | LLM para explicaciones y autocompletado |
| Mapbox | mapbox.com | Mapa de tiendas |
| Resend | resend.com | Correos de notificación |

### 1. Configurar Supabase

1. Crear proyecto en Supabase.
2. Activar extensión de búsqueda vectorial: `CREATE EXTENSION IF NOT EXISTS vector;`
3. Guardar las claves de acceso (Project URL, anon key, service_role key, connection string).
4. Agregar el dominio de Vercel en la lista de URLs permitidas (para recuperación de contraseña).

### 2. Desplegar el servidor (Render.com)

1. Crear Web Service conectado al repositorio `Deckora-API`, rama `main`.
2. Comando de inicio: `npm start`.
3. Agregar variables de entorno: claves de Supabase, DeepSeek, Nomic, Resend.
4. Ejecutar migraciones: `npm run db:migrate`.
5. Poblar cartas MTG: `node scripts/seedCartas.js`.
6. Generar embeddings: `npm run embed:generate` (~10-20 min).

### 3. Desplegar la interfaz (Vercel)

1. Importar `Deckora-Web` en Vercel. Detecta Vite automáticamente.
2. Agregar variables de entorno: URL de Supabase, URL del servidor, token de Mapbox.
3. Hacer deploy. Vercel construye y publica automáticamente en cada push a `main`.

---

## 19. Integraciones externas

### Supabase Auth — Autenticación

Gestiona el registro e inicio de sesión. Emite JWT que el backend verifica en cada request.

**Por qué**: implementar autenticación segura desde cero (contraseñas hasheadas, sesiones, refresh tokens, recuperación por email, verificación de cuenta) tomaría semanas y es propenso a errores de seguridad. Supabase lo resuelve en horas.

---

### Nomic AI — Embeddings vectoriales

Transforma el texto de cada carta en un vector de 768 números. Este proceso se ejecuta **una sola vez** en el script offline, no en tiempo de request.

**Por qué**: es el motor del Asistente IA de recomendaciones. Sin embeddings, la recomendación por similaridad no existe. El modelo `nomic-embed-text-v1.5` es gratuito y de alta calidad.

---

### DeepSeek — LLM

Genera dos cosas:
1. **Explicaciones** en lenguaje natural de las cartas recomendadas (en tiempo de request, falla gracefully).
2. **Listas de mazos** para autocompletar (en tiempo de request, el resultado se importa a la BD local).

**Por qué**: acceso a un LLM de alta capacidad con costo muy bajo por token. El sistema está diseñado para que si el LLM falla, las funcionalidades principales (recomendaciones, inscripciones, torneos) sigan funcionando.

---

### Mapbox — Mapas interactivos

Renderiza el mapa de tiendas en la landing page y el selector de ubicación al configurar el perfil de tienda.

**Por qué**: es el estándar de la industria para mapas web personalizables, con capa gratuita suficiente para el volumen del proyecto.

---

### Resend — Email transaccional

Envía automáticamente correos cuando:
- Un jugador solicita inscribirse a un torneo (avisa al organizador)
- El organizador aprueba una inscripción (avisa al jugador)
- El organizador rechaza una inscripción (avisa al jugador)

**Por qué**: mantiene a los participantes informados sin que deban revisar la plataforma constantemente. Los emails se envían de forma no bloqueante — si el servicio falla, la operación principal (inscripción, aprobación) no se ve afectada.
