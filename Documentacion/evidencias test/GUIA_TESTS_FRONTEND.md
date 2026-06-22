# Guia de Tests — Deckora-Web

> **Problema:** el coverage actual del frontend mide solo **10 de 125 archivos** (8%).
> La config de Vitest tiene un `include` que lista archivos puntuales, así que el
> 97% que reporta es solo sobre esos 10 archivos — no refleja la cobertura real.
>
> **Objetivo:** testear toda la aplicación frontend y que el coverage mida todo `src/`.

---

## 1. Corregir la config de coverage

En `vite.config.js`, cambiar el `include` restrictivo por uno que cubra todo `src/`:

```js
// ANTES (solo 10 archivos)
coverage: {
  provider: 'v8',
  reporter: ['text', 'html'],
  include: [
    'src/utils/validators.js',
    'src/utils/formatters.js',
    // ... solo 10 archivos
  ],
  thresholds: {
    statements: 95,
    branches: 80,
    functions: 95,
    lines: 95,
  },
},

// DESPUES (todo src/)
coverage: {
  provider: 'v8',
  reporter: ['text', 'html'],
  include: ['src/**/*.{js,jsx}'],
  exclude: ['src/main.jsx', 'src/styles/**'],
  thresholds: {
    statements: 60,
    branches: 50,
    functions: 60,
    lines: 60,
  },
},
```

> **Nota:** los thresholds van a bajar al principio. Ir subiéndolos a medida que
> se agregan tests. El objetivo final es llegar a 80%+ en todo.

---

## 2. Lo que YA está testeado (no repetir)

| Archivo | Tests existentes |
|---------|-----------------|
| `src/utils/validators.js` | `tests/unit/utils/validators.test.js` |
| `src/utils/formatters.js` | `tests/unit/utils/formatters.test.js` |
| `src/utils/deck-helpers.js` | `tests/unit/utils/deck-helpers.test.js` |
| `src/utils/torneos-helpers.js` | `tests/unit/utils/torneos-helpers.test.js` |
| `src/utils/errors.js` | `tests/unit/utils/errors.test.js` |
| `src/hooks/useDebounce.js` | `tests/unit/hooks/useDebounce.test.js` |
| `src/components/domain/CommanderBadge.jsx` | `tests/components/CommanderBadge.test.jsx` |
| `src/components/domain/EstadoBadge.jsx` | `tests/components/EstadoBadge.test.jsx` |
| `src/components/domain/FormatBadge.jsx` | `tests/components/FormatBadge.test.jsx` |
| `src/components/domain/RoleBadge.jsx` | `tests/components/RoleBadge.test.jsx` |
| `src/modules/mazos/components/PanelValidacion.jsx` | `tests/components/PanelValidacion.test.jsx` |
| Seguridad XSS | `tests/components/seguridad-xss.test.jsx` |
| ProtectedRoute | `tests/components/ProtectedRoute.test.jsx` |

---

## 3. Lo que FALTA testear (organizado por prioridad)

### Prioridad 1 — Utils y Hooks (lo más fácil, mayor impacto en coverage)

Son funciones puras o hooks simples, no necesitan mockear APIs.

| Archivo | Qué testear |
|---------|-------------|
| `src/utils/constants.js` | Que las constantes existan y tengan los valores esperados |
| `src/hooks/useFormValidation.js` | Manejo de valores, errores, touched, validadores |
| `src/hooks/useConfirmDialog.jsx` | Abrir/cerrar diálogo, confirmar/cancelar |
| `src/hooks/useGeolocation.js` | Mock de `navigator.geolocation`, éxito y error |

**Patrón a seguir** (ya establecido en el proyecto):

```jsx
// tests/unit/hooks/useFormValidation.test.js
import { renderHook, act } from '@testing-library/react';
import { useFormValidation } from '@/hooks/useFormValidation';

describe('useFormValidation', () => {
  it('inicializa con los valores por defecto', () => {
    const { result } = renderHook(() =>
      useFormValidation({ nombre: '' }, { nombre: (v) => !v ? 'Requerido' : null })
    );
    expect(result.current.values.nombre).toBe('');
    expect(result.current.errors).toEqual({});
  });

  it('valida campos al hacer submit', () => {
    // ...
  });
});
```

### Prioridad 2 — Componentes UI (base del sistema de diseño)

Componentes reutilizables. Testear que renderizan con distintas props.

| Archivo | Qué testear |
|---------|-------------|
| `src/components/ui/Button.jsx` | Render, click, disabled, variantes |
| `src/components/ui/Modal.jsx` | Abrir, cerrar, contenido |
| `src/components/ui/Alert.jsx` | Variantes (success, error, warning), contenido |
| `src/components/ui/Input.jsx` | Render, onChange, error state |
| `src/components/ui/Select.jsx` | Opciones, selección, placeholder |
| `src/components/ui/Badge.jsx` | Variantes, texto |
| `src/components/ui/Card.jsx` | Render con children |
| `src/components/ui/Tabs.jsx` | Cambio de tab activo |
| `src/components/ui/Spinner.jsx` | Render básico |
| `src/components/ui/EmptyState.jsx` | Mensaje, ícono |
| `src/components/ui/ConfirmDialog.jsx` | Confirmar, cancelar, mensaje |
| `src/components/ui/Toast.jsx` | Tipos (success, error), auto-dismiss |
| `src/components/ui/Tooltip.jsx` | Hover muestra contenido |
| `src/components/ui/Textarea.jsx` | Render, onChange |
| `src/components/ui/Skeleton.jsx` | Render básico |
| `src/components/ui/ErrorBoundary.jsx` | Captura error y muestra fallback |
| `src/components/ui/ErrorChunk.jsx` | Render de error |

**Patrón a seguir:**

```jsx
// tests/components/ui/Button.test.jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from '@/components/ui';

describe('Button', () => {
  it('renderiza con el texto', () => {
    render(<Button>Guardar</Button>);
    expect(screen.getByRole('button', { name: 'Guardar' })).toBeInTheDocument();
  });

  it('ejecuta onClick al hacer click', async () => {
    const onClick = vi.fn();
    render(<Button onClick={onClick}>OK</Button>);
    await userEvent.click(screen.getByRole('button'));
    expect(onClick).toHaveBeenCalledOnce();
  });

  it('no ejecuta onClick cuando está disabled', async () => {
    const onClick = vi.fn();
    render(<Button onClick={onClick} disabled>OK</Button>);
    await userEvent.click(screen.getByRole('button'));
    expect(onClick).not.toHaveBeenCalled();
  });
});
```

### Prioridad 3 — Componentes de dominio (lógica de negocio visual)

| Archivo | Qué testear |
|---------|-------------|
| `src/components/domain/MTGCard.jsx` | Render de carta con imagen, nombre, set |
| `src/components/domain/DeckList.jsx` | Lista de cartas, agrupación |
| `src/components/domain/DeckStats.jsx` | Estadísticas del mazo |
| `src/components/domain/DeckBuilder.jsx` | Agregar/quitar cartas |
| `src/components/domain/TournamentCard.jsx` | Info del torneo |
| `src/components/domain/CartaDetalleModal.jsx` | Modal con detalles de carta |
| `src/components/domain/ManaCost.jsx` | Render de símbolos de maná |
| `src/components/domain/PodTable.jsx` | Tabla de pods |
| `src/components/domain/RoundView.jsx` | Vista de ronda |
| `src/components/domain/EstadisticasJugador.jsx` | Stats del jugador |
| `src/components/domain/MapaTiendas.jsx` | Mock de mapbox, render de tiendas |
| `src/components/domain/MiniMapaTienda.jsx` | Mapa pequeño |
| `src/components/domain/StorePin.jsx` | Pin de tienda en mapa |

### Prioridad 4 — Services (capa de API)

Mockear `fetch` o el cliente de Supabase. Verificar que llaman a los endpoints correctos.

| Archivo | Qué testear |
|---------|-------------|
| `src/services/api.js` | Wrapper de fetch: headers, manejo de errores |
| `src/services/auth.service.js` | signup, login, logout, recuperar password |
| `src/services/cartas.service.js` | Búsqueda, detalle de cartas |
| `src/services/mazos.service.js` | CRUD de mazos, validación |
| `src/services/torneos.service.js` | CRUD de torneos, inscripciones |
| `src/services/biblioteca.service.js` | Biblioteca de cartas |
| `src/services/rondas.service.js` | Gestión de rondas |
| `src/services/enfrentamientos.service.js` | Resultados de enfrentamientos |
| `src/services/tiendas.service.js` | Listado de tiendas |
| `src/services/usuarios.service.js` | Perfil de usuario |
| `src/services/supabase.js` | Cliente de Supabase (verificar config) |

**Patrón:**

```js
// tests/unit/services/mazos.service.test.js
import { vi } from 'vitest';

// Mockear el módulo api antes de importar el service
vi.mock('@/services/api', () => ({
  apiFetch: vi.fn(),
}));

import { apiFetch } from '@/services/api';
import { obtenerMazos, crearMazo } from '@/services/mazos.service';

describe('mazos.service', () => {
  beforeEach(() => vi.clearAllMocks());

  it('obtenerMazos llama a GET /api/mazos', async () => {
    apiFetch.mockResolvedValue([{ id: 1, nombre: 'Mi Mazo' }]);
    const result = await obtenerMazos();
    expect(apiFetch).toHaveBeenCalledWith('/api/mazos', expect.objectContaining({ method: 'GET' }));
    expect(result).toHaveLength(1);
  });
});
```

### Prioridad 5 — Context providers

| Archivo | Qué testear |
|---------|-------------|
| `src/context/AuthContext.jsx` | Login actualiza usuario, logout limpia estado, rol se setea |
| `src/context/ToastContext.jsx` | Agregar toast, auto-dismiss, tipos |

**Patrón:**

```jsx
// tests/unit/context/AuthContext.test.jsx
import { render, screen, waitFor } from '@testing-library/react';
import { AuthProvider } from '@/context/AuthContext';
import { useAuth } from '@/hooks/useAuth';

vi.mock('@/services/supabase', () => ({
  supabase: {
    auth: {
      getSession: vi.fn().mockResolvedValue({ data: { session: null } }),
      onAuthStateChange: vi.fn().mockReturnValue({ data: { subscription: { unsubscribe: vi.fn() } } }),
    },
  },
}));

function TestComponent() {
  const { user, loading } = useAuth();
  if (loading) return <p>Cargando...</p>;
  return <p>{user ? user.email : 'Sin sesión'}</p>;
}

describe('AuthContext', () => {
  it('inicia sin usuario autenticado', async () => {
    render(<AuthProvider><TestComponent /></AuthProvider>);
    await waitFor(() => {
      expect(screen.getByText('Sin sesión')).toBeInTheDocument();
    });
  });
});
```

### Prioridad 6 — Componentes de módulos (pages y components complejos)

Estos requieren más mocking (router, context, services). Ir de a poco.

| Módulo | Archivos clave |
|--------|---------------|
| **Mazos** | `MisMazos.jsx`, `DetalleMazo.jsx`, `CrearMazoModal.jsx`, `ModoEdicionMazo.jsx`, `BarraAgregarCarta.jsx`, `AsistenteIA.jsx`, `ImportarMazoModal.jsx` |
| **Torneos** | `Cartelera.jsx`, `DetalleTorneo.jsx`, `GestionTorneo.jsx`, `CrearTorneo.jsx`, `EditarTorneo.jsx`, `MisTorneos.jsx`, `FormularioTorneo.jsx`, `PanelInscripcion.jsx`, `BandejaInscripciones.jsx`, `ListaInscritos.jsx`, `ReportarResultadoModal.jsx`, `SnapshotMazoModal.jsx` |
| **Identidad** | `Login.jsx`, `Registro.jsx`, `RecuperarPassword.jsx`, `Configuracion.jsx`, `PerfilJugador.jsx`, `PerfilOrganizador.jsx`, `PerfilTienda.jsx`, `PerfilRouter.jsx`, `SelectorRol.jsx`, `CuentaTab.jsx`, `MisEstadisticasTab.jsx`, `MisInscripcionesTab.jsx`, `MisTorneosTab.jsx`, `ProfileHeader.jsx`, `ConfiguracionOrganizadorTab.jsx`, `ConfiguracionTiendaTab.jsx` |
| **Dashboards** | `Landing.jsx`, `DashboardJugador.jsx`, `DashboardOrganizador.jsx`, `DashboardTienda.jsx`, `HeroLanding.jsx`, `FeaturesLanding.jsx`, `ProfilesLanding.jsx`, `CTALanding.jsx`, `BloqueResumen.jsx`, `StatsRapidas.jsx` |
| **Biblioteca** | `Biblioteca.jsx` |
| **Mapa** | `SeccionMapaTiendas.jsx` |
| **Layout** | `AppLayout.jsx`, `Navbar.jsx`, `Sidebar.jsx`, `Footer.jsx` |
| **Routing** | `AppRoutes.jsx` |
| **Pages** | `NotFound.jsx`, `Forbidden.jsx` |

**Patrón para pages (con mocking de router y context):**

```jsx
// tests/components/pages/NotFound.test.jsx
import { render, screen } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import NotFound from '@/pages/NotFound';

describe('NotFound', () => {
  it('muestra mensaje de página no encontrada', () => {
    render(
      <MemoryRouter>
        <NotFound />
      </MemoryRouter>
    );
    expect(screen.getByText(/no encontrad/i)).toBeInTheDocument();
  });
});
```

---

## 4. Estructura de carpetas para los tests

Seguir la estructura que ya existe y expandirla:

```
tests/
├── helpers/
│   └── setup.js              (ya existe)
├── unit/
│   ├── utils/                 (ya existen 5 tests)
│   ├── hooks/                 (ya existe useDebounce, agregar el resto)
│   ├── services/              (NUEVO — tests de cada service)
│   └── context/               (NUEVO — tests de AuthContext, ToastContext)
├── components/
│   ├── ui/                    (NUEVO — tests de componentes UI)
│   ├── domain/                (mover los existentes aquí + agregar nuevos)
│   ├── layout/                (NUEVO — Navbar, Sidebar, Footer)
│   └── modules/               (NUEVO — componentes de mazos, torneos, etc.)
└── pages/                     (NUEVO — tests de páginas completas)
```

---

## 5. Comandos útiles

```bash
# Correr todos los tests
npm test

# Correr con coverage (va a mostrar TODO src/ ahora)
npm run test:coverage

# Correr solo los tests de un archivo
npx vitest run tests/unit/hooks/useFormValidation.test.js

# Correr tests en modo watch (se re-ejecutan al guardar)
npm run test:watch

# Correr solo tests de componentes
npm run test:components
```

---

## 6. Tips

- **No mockear de más.** Si un componente usa un util puro (formatters, validators), dejarlo pasar sin mock.
- **Sí mockear:** servicios HTTP (`api.js`), Supabase, `react-router-dom` (useNavigate, useParams), mapbox-gl.
- **Usar `screen.getByRole`** antes que `getByTestId`. Es más accesible y no requiere agregar data-testid al código.
- **Testear comportamiento, no implementación.** "Al hacer click en Guardar se muestra el spinner" es mejor que "se llama a setState con loading=true".
- **El user-event ya está instalado** (`@testing-library/user-event`) pero no se usa en los tests actuales. Usarlo para simular clicks, typing, etc.

---

## 7. Resumen del estado actual vs objetivo

| Métrica | Ahora | Objetivo |
|---------|-------|----------|
| Archivos medidos | 10 / 125 | 125 / 125 |
| Tests | 117 | 300+ |
| Statements coverage (real) | ~8% estimado | 80%+ |
| Branches coverage (real) | ~5% estimado | 70%+ |
