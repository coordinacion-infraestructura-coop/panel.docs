# Spec: Frontend React — Portal Ministerial

**Estado**: draft | **Versión**: 0.1.0

---

## 1. Propósito

SPA React que unifica el acceso a todos los subsistemas del ministerio. Muestra al ministro un dashboard ejecutivo consolidado y permite a cada secretaría operar su subsistema correspondiente desde una interfaz web consistente.

## 2. Stack frontend

| Tecnología | Versión | Uso |
|-----------|---------|-----|
| React | 18 | UI framework |
| Vite | 5 | Build tool |
| TypeScript | 5 | Type safety |
| React Router | 6 | Navegación SPA |
| Zustand | 4 | Estado global (auth, usuario) |
| TanStack Query | 5 | Fetching y caché de datos del servidor |
| Axios | 1 | HTTP client |
| Firebase JS SDK | 10 | Auth |
| Recharts | 2 | Gráficos dashboard ejecutivo |
| React Hook Form | 7 | Formularios |
| Zod | 3 | Validación frontend |

## 3. Estructura de rutas

```
/                           → redirect según rol
/login                      → login con Google Identity Platform
/dashboard                  → dashboard ejecutivo (rol: ministro)

/vivienda/                  → índice secretaría vivienda
/vivienda/beneficiarios     → listado
/vivienda/beneficiarios/nuevo
/vivienda/beneficiarios/:id
/vivienda/expedientes       → listado con filtros
/vivienda/expedientes/nuevo
/vivienda/expedientes/:id
/vivienda/programas         → estadísticas por programa

/infraestructura/
/infraestructura/proyectos
/infraestructura/proyectos/nuevo
/infraestructura/proyectos/:id

/territorial/
/territorial/cooperativas
/territorial/cooperativas/:id
/territorial/actividades
/territorial/actividades/nueva

/desarrollo/
/desarrollo/cooperativas    → (lectura desde UTN)
/desarrollo/cooperativas/:id

/gasifera/
/gasifera/escuelas
/gasifera/escuelas/:id
/gasifera/asesoramientos
/gasifera/asesoramientos/nuevo
/gasifera/creditos
/gasifera/creditos/nuevo
/gasifera/creditos/:id
/gasifera/cooperativas

/admin/                     → solo rol admin_sistema
/admin/usuarios
/admin/roles
```

## 4. Módulo `shared/auth`

### `useAuth()` hook

```typescript
interface AuthUser {
  uid: string;
  email: string;
  displayName: string;
  role: 'ministro' | 'secretario' | 'director' | 'operador' | 'admin_sistema';
  secretaria: string | null;
  direccion: string | null;
  token: string; // JWT para el API Gateway
}

function useAuth(): {
  user: AuthUser | null;
  loading: boolean;
  signIn: () => Promise<void>;
  signOut: () => Promise<void>;
}
```

### `<ProtectedRoute>` component

```typescript
// Redirige a /login si no autenticado
// Redirige a /no-autorizado si el rol no tiene acceso a la ruta
<ProtectedRoute roles={['secretario_vivienda', 'director_vivienda', 'ministro']}>
  <ExpedientesPage />
</ProtectedRoute>
```

## 5. Módulo `shared/api`

### Instancia axios

```typescript
// src/shared/api/client.ts
const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_GATEWAY_URL,
  timeout: 10000,
});

// Interceptor: agrega JWT en cada request
apiClient.interceptors.request.use(async (config) => {
  const token = await getFirebaseToken();
  config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Interceptor: manejo global de errores
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) signOut();
    if (error.response?.status === 403) navigate('/no-autorizado');
    return Promise.reject(error);
  }
);
```

## 6. Módulo dashboard ejecutivo

Solo accesible con rol `ministro`. Muestra KPIs consolidados de todos los subsistemas consultando BigQuery via un endpoint dedicado.

### Widgets del dashboard

| Widget | Datos | Fuente |
|--------|-------|--------|
| Expedientes vivienda por estado | Conteo por estado | svc-vivienda |
| Proyectos infraestructura activos | Lista + estado | svc-infraestructura |
| Cooperativas en fortalecimiento | Total + por departamento | svc-territorial |
| Créditos gasífera en mora | Listado + monto | svc-gasifera |
| Mapa de cobertura territorial | GeoJSON por departamento | BigQuery |
| Escuelas con gas | % avance del programa | svc-gasifera |

## 7. Variables de entorno

```bash
# .env.development
VITE_API_GATEWAY_URL=http://localhost:8000
VITE_FIREBASE_API_KEY=...
VITE_FIREBASE_AUTH_DOMAIN=gestorcooperativo.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=gestorcooperativo
VITE_ENV=development

# .env.production
VITE_API_GATEWAY_URL=https://api.ministerio-coop.gob.ar
VITE_FIREBASE_API_KEY=...
VITE_FIREBASE_AUTH_DOMAIN=gestorcooperativo.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=gestorcooperativo
VITE_ENV=production
```

## 8. Convenciones de componentes

```
modules/{secretaria}/
├── pages/
│   ├── {Recurso}ListPage.tsx      → listado con tabla y filtros
│   ├── {Recurso}DetailPage.tsx    → vista detalle / edición
│   └── {Recurso}NewPage.tsx       → formulario de alta
├── components/
│   ├── {Recurso}Table.tsx
│   ├── {Recurso}Form.tsx
│   ├── {Recurso}StatusBadge.tsx
│   └── {Recurso}Filters.tsx
├── hooks/
│   └── use{Recurso}.ts            → TanStack Query hooks
├── types/
│   └── {secretaria}.types.ts      → TypeScript interfaces
└── api/
    └── {secretaria}.api.ts        → funciones axios
```

## 9. Criterios de aceptación frontend

- [ ] Login con Google Identity Platform funcional
- [ ] Redirección automática por rol al hacer login
- [ ] Dashboard ejecutivo con widgets de los 5 subsistemas
- [ ] Módulo vivienda: CRUD beneficiarios + gestión expedientes
- [ ] Módulo infraestructura: listado y detalle de proyectos
- [ ] Módulo territorial: cooperativas y actividades
- [ ] Módulo desarrollo: lectura cooperativas UTN (solo lectura)
- [ ] Módulo gasífera: escuelas, asesoramientos, créditos y cuotas
- [ ] Responsive (desktop y tablet — no mobile en v1)
- [ ] Manejo de estados de carga y error en todos los fetches
- [ ] Acceso protegido por rol en cada ruta
