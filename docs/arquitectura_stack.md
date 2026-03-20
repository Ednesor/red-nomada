# Arquitectura y Stack Tecnológico: Red-Nómada

## 📐 Visión General

Red-Nómada es un sistema **offline-first** que permite nómadas digitales compartir estado técnico de espacios en tiempo real, minimizando batería y tráfico de red.

**Principios:**
- Datos locales primero (SQLite) → Sincronización eventual
- Sin GPS en background / Sin push obligatorio
- Permisos explícitos: "siempre", "una vez", "solo en uso"
- Validación solo en foreground

---

## 🏗 Capas Arquitectónicas

```
┌─────────────────────────────────┐
│   Frontend (React Native)        │
│  📸 Snapshot | ⏰ Alerts | 🎮 Missions
└────────────┬────────────────────┘
             │ (HTTP REST + WebSocket)
┌────────────┴────────────────────┐
│   Backend (Node.js + Express)    │
│  • Places API                    │
│  • Verification Agent            │
│  • Gamification Logic            │
└────────────┬────────────────────┘
             │ (SQL + Cache)
┌────────────┴────────────────────┐
│   Data Layer                     │
│  • PostgreSQL (persistencia)     │
│  • Redis (cache + real-time)     │
│  • SQLite (local en app)         │
└─────────────────────────────────┘
```

---

## 📱 Frontend (React Native)

### Stack
- **Framework:** React Native (Expo o bare)
- **Estado Local:** SQLite (offline-first)
- **Sincronización:** Redux Saga + Axios
- **Ubicación:** expo-location (con granularidad de permisos)
- **UI:** React Native Paper / NativeBase

### Componentes Clave

**PlaceDetailCard (Historia 1: Snapshot)**
- Renderizar 3 pilares: Conectividad | Energía | Ambiente
- Cada pilar: 3 valores únicamente (Baja/Media/Alta, etc.)
- **SLA:** Latencia <500ms (desde cache local)
- ✅ Prohibición: Cero estrellas, cero notas generales

**VerificationAlert (Historia 2)**
- Modal que aparece **solo si app en foreground**
- Detectar localmente: `lastVerified > 2 horas`
- Mensaje: *"¿Sigue tranquilo este café?"* (1 tap confirm)
- No triggers: sin GPS background, sin push externa

**MissionsWidget (Historia 3)**
- Lista de retos activos: "Reportá 3 cafés hoy"
- Badge de impacto: *"Gracias a tu reporte, 12 personas evitaron..."*

### Flujo de Permisos (Historia 2)
```
App Launch
  ├─ Mostrar picker: "Ubicación: siempre / una vez / solo en uso"
  ├─ Store en SQLite + enviar a backend
  ├─ Always → Verificaciones pasivas continuas
  ├─ While Using → Solo si app activa
  └─ Never → Ocultar features de ubicación
```

---

## 🖥 Backend (Node.js + Express)

### Stack
- **Runtime:** Node.js 20+
- **Framework:** Express.js
- **ORM:** Sequelize o TypeORM
- **Validación:** Joi
- **WebSocket:** Socket.io
- **Scheduler:** node-cron

### Services

**Places Service**
```
GET    /api/places/:id           → Snapshot actual
POST   /api/places               → Crear lugar
GET    /api/places?lat&lon&r=    → Buscar cercanos
```

**Verification Service (Historia 2)**
```
POST   /api/validations/:placeId → Confirmar estado
GET    /api/places/:id/status    → Flag: "Verificación AHORA"
```

**Gamification Service (Historia 3)**
```
GET    /api/missions             → Misiones activas
POST   /api/missions/:id/complete → +puntos
GET    /api/user/impact          → {"peopleSaved": 12}
```

**Permission Service**
```
POST   /api/permissions          → Guardar preferencia (Always/While/Never)
GET    /api/permissions/:userId  → Validar scope
```

### Lógica: Verification Agent
- **Cron:** cada 10 min
- Busca lugares con `lastVerified > 2 horas`
- Selecciona 3-5 usuarios con permiso = "Always" o "WhileUsing"
- Envía evento WebSocket a app (solo si foreground)
- App muestra modal → confirma → POST validación

### Lógica: Impacto Social (Historia 3)
- POST validación → calcular: ¿Cuántos consultaron este lugar en 24h?
- Almacenar en tabla `user_impact` (auditoría)
- Retornar: `{"peopleSaved": 12}`
- App renderiza overlay: *"Gracias a tu reporte, 12 personas..."*

---

## 💾 Base de Datos

### PostgreSQL

```sql
-- Lugares
CREATE TABLE places (
  id UUID PRIMARY KEY,
  name VARCHAR(255),
  latitude DECIMAL(10,8),
  longitude DECIMAL(10,8),
  last_verified TIMESTAMP,
  connectivity VARCHAR(20),  -- Baja/Media/Alta
  energy VARCHAR(20),        -- Pocos/Suficientes/Muchos
  environment VARCHAR(20),   -- Silencioso/Moderado/Ruidoso
  created_at TIMESTAMP
);

-- Validaciones
CREATE TABLE validations (
  id UUID PRIMARY KEY,
  place_id UUID REFERENCES places,
  user_id UUID,
  connectivity VARCHAR(20),
  energy VARCHAR(20),
  environment VARCHAR(20),
  created_at TIMESTAMP
);

-- Usuarios
CREATE TABLE users (
  id UUID PRIMARY KEY,
  username VARCHAR(255) UNIQUE,
  location_permission VARCHAR(20), -- Always/WhileUsing/Never
  points INT,
  created_at TIMESTAMP
);

-- Misiones
CREATE TABLE missions (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users,
  title VARCHAR(255),  -- "Reportá 3 cafés hoy"
  target INT,
  current INT,
  completed BOOLEAN,
  reward_points INT,
  created_at TIMESTAMP
);

-- Impacto (auditoría)
CREATE TABLE user_impact (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users,
  validation_id UUID REFERENCES validations,
  people_saved INT,
  created_at TIMESTAMP
);
```

### SQLite (Local en App)
- Replica de `places` (snapshot actual + metadata)
- Cache local de validaciones propias
- Índice por `lastVerified` para detectar datos obsoletos

### Redis (Cache + Real-time)
- `place:{placeId}` → snapshot (TTL 2h)
- `verification:pending:{userId}` → alertas pendientes
- WebSocket channel para eventos de validación

---

## 🔄 Flujos Críticos

**Flujo 1: Ver Snapshot (Historia 1)**
```
Usuario abre lugar
  ├─ Consultar SQLite local (offline OK)
  ├─ Si no existe: GET /api/places/:id
  ├─ Cache en SQLite
  └─ Renderizar 3 pilares en <500ms
```

**Flujo 2: Re-validación Pasiva (Historia 2)**
```
1. App listener: detecta lugar con lastVerified > 2h en SQLite
2. Backend cron: identifica validaciones pendientes
3. WebSocket event → App muestra modal (solo foreground)
4. Usuario tap → POST /api/validations/:placeId
5. Backend actualiza + calcula impacto
6. Retorna: {"peopleSaved": 12}
7. App muestra badge/overlay
```

**Flujo 3: Misión Completada (Historia 3)**
```
Usuario completa validación
  ├─ POST /api/validations/:placeId
  ├─ Backend: GET impacto_count (quién consultó en 24h)
  ├─ POST /api/missions/:id/complete (+puntos)
  ├─ Retorna: {"peopleSaved": 12, "pointsAwarded": 50}
  └─ App muestra: "Gracias a tu reporte, 12 personas evitaron..."
```

---

## 🛡 Restricciones Mapeadas

| Restricción | Implementación |
|---|---|
| **Snapshot <5s** | Cache local SQLite + <500ms render |
| **No GPS background** | Listener solo en foreground |
| **Batería limitada** | Permisos granulares Always/While/Never |
| **Sin internet** | SQLite offline-first + sync eventual |
| **Sin push obligatorio** | WebSocket para validaciones (app abierta) |
| **Datos obsoletos >2h** | Flag: "Este lugar necesita verificación AHORA" |

---

## 📦 Dependencias

### Frontend
- `react-native` / `expo`
- `@react-navigation/native`
- `react-native-sqlite-storage`
- `redux` + `redux-saga`
- `axios`
- `expo-location`
- `socket.io-client`

### Backend
- `express`
- `pg` (PostgreSQL)
- `redis`
- `joi`
- `socket.io`
- `node-cron`
- `sequelize`

### Infrastructure
- PostgreSQL 14+
- Redis 7+
- Node.js 20 LTS

---

## 🚀 Orden de Implementación (MVP)

1. **Schema DB:** crear tablas en PostgreSQL
2. **Backend minimal:** Places CRUD + /validations endpoint
3. **Frontend UI:** PlaceDetailCard (Snapshot) con datos mock
4. **Local sync:** SQLite + Redux
5. **API integration:** GET /places/:id + cache
6. **Verification:** Listener local + WebSocket alert modal
7. **Gamification:** Missions engine + impact counter
8. **Refinamiento:** Permisos, percolación de datos obsoletos, pruebas

---

## ✅ Validación de Soluciones

Toda solución debe cumplir:
- ✓ Criterios de aceptación de cada historia
- ✓ No asumir internet constante
- ✓ Respetar "siempre/una vez/solo en uso"
- ✓ Solo validar cuando app activa
- ✓ Sin GPS background
- ✓ Snapshot <5 segundos
- ✓ Cero puntuaciones generales
