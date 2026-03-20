# Modelo de Datos: Red-Nómada

## 📊 Entidades Principales

### 1. **Users** (Usuarios)
Representa un usuario de la plataforma.

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username VARCHAR(255) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE,
  location_permission VARCHAR(20), -- 'always' | 'while_using' | 'never'
  points INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Relaciones:** 1 → Many (Validations, Missions, UserImpact)

---

### 2. **Places** (Lugares)
Representa un espacio laboral (café, coworking, etc.)

```sql
CREATE TABLE places (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  latitude DECIMAL(10, 8) NOT NULL,
  longitude DECIMAL(10, 8) NOT NULL,
  
  -- Los 3 pilares técnicos (Historia 1)
  connectivity VARCHAR(20), -- 'baja' | 'media' | 'alta'
  energy VARCHAR(20),       -- 'pocos' | 'suficientes' | 'muchos'
  environment VARCHAR(20),  -- 'silencioso' | 'moderado' | 'ruidoso'
  
  last_verified TIMESTAMP, -- Para marcar si >2h (Historia 2)
  verification_count INT DEFAULT 0,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Relaciones:** 1 → Many (Validations, UserImpact)

---

### 3. **Validations** (Validaciones)
Registro cada vez que un usuario confirma/actualiza el estado de un lugar.

```sql
CREATE TABLE validations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  place_id UUID NOT NULL REFERENCES places(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id),
  
  -- Valores confirmados por el usuario
  connectivity VARCHAR(20),
  energy VARCHAR(20),
  environment VARCHAR(20),
  
  created_at TIMESTAMP DEFAULT NOW()
);
```

**Índices recomendados:**
- `INDEX (place_id, created_at)` → Buscar últimas validaciones de un lugar
- `INDEX (user_id, created_at)` → Historial por usuario

**Relaciones:** Many → 1 (Places, Users)

---

### 4. **Missions** (Misiones/Retos - Historia 3)
Retos activos para incentivar comportamientos.

```sql
CREATE TABLE missions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  title VARCHAR(255) NOT NULL, -- "Reportá 3 cafés hoy"
  description TEXT,
  target INT NOT NULL,         -- Meta (ej: 3)
  current INT DEFAULT 0,       -- Progreso actual
  completed BOOLEAN DEFAULT FALSE,
  
  reward_points INT,           -- Puntos al completar
  created_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP         -- Fecha de expiración
);
```

**Relaciones:** Many → 1 (Users)

---

### 5. **UserImpact** (Métricas de Impacto - Historia 3)
Auditoría de "cuántas personas ayudé con mi reporte".

```sql
CREATE TABLE user_impact (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  validation_id UUID NOT NULL REFERENCES validations(id),
  
  -- Calculado en el momento de la validación
  people_saved INT, -- Número de usuarios que consultaron este lugar en 24h
  
  created_at TIMESTAMP DEFAULT NOW()
);
```

**Relaciones:** Many → 1 (Users, Validations)

---

## 🔗 Relaciones ER (Entidad-Relación)

```
Users (1)
  ├─── (Many) Validations
  ├─── (Many) Missions
  └─── (Many) UserImpact

Places (1)
  ├─── (Many) Validations
  └─── (Many) UserImpact

Validations (Many)
  ├─── (1) Users
  ├─── (1) Places
  └─── (1) UserImpact
```

---

## ⚡ Flujos de Datos por Historia

### Historia 1: Snapshot
```
GET /api/places/:id
  ↓
SELECT * FROM places WHERE id = :id
  ↓
Retorna: connectivity, energy, environment
  ↓
UI renderiza 3 pilares en <500ms
```

### Historia 2: Verificación + Datos Obsoletos
```
Listener local: detecta place.last_verified > 2 horas
  ↓
Backend cron: SELECT places WHERE last_verified < NOW() - INTERVAL '2 hours'
  ↓
WebSocket → muestra modal al usuario (foreground)
  ↓
POST /api/validations/:placeId
  ↓
INSERT INTO validations (place_id, user_id, connectivity, energy, environment)
  ↓
UPDATE places SET last_verified = NOW(), verification_count = verification_count + 1
```

### Historia 3: Impacto Social
```
POST /api/validations/:placeId
  ↓
Calcular: COUNT(validations) WHERE place_id = :placeId AND created_at > NOW() - INTERVAL '24 hours'
  ↓
INSERT INTO user_impact (user_id, validation_id, people_saved)
  ↓
POST /api/missions/:missionId/complete → +puntos
  ↓
Retorna: {"peopleSaved": 12, "pointsAwarded": 50}
```

---

## 📋 Tabla Resumen

| Entidad | Propósito | Clave Foránea | Índices |
|---|---|---|---|
| **Users** | Identidad de usuario | — | `(username)` |
| **Places** | Espacios laborales | `created_by` → Users | `(latitude, longitude)` |
| **Validations** | Actualizaciones de estado | `place_id`, `user_id` | `(place_id, created_at)`, `(user_id, created_at)` |
| **Missions** | Retos gamificados | `user_id` | `(user_id, expires_at)` |
| **UserImpact** | Auditoría de impacto | `user_id`, `validation_id` | `(user_id, created_at)` |

---

## 🚫 Decisiones de Diseño

| Decisión | Razón |
|---|---|
| **No tabla de "ratings"** | Prohibición absoluta de notas generales (regla negocio) |
| **Los 3 pilares en Places** | Snapshot actual, dato único (no histórico) |
| **Validations separada** | Auditoría de cambios + calcular impacto |
| **UserImpact separada** | Métricas desacopladas, fácil análisis posterior |
| **last_verified timestamp** | Gatillo para marcar "AHORA" si >2h |
| **location_permission en Users** | Auditoría de permisos otorgados |

---

## ✅ Validaciones (Constraints)

```sql
-- No permitir lugares sin pilares
ALTER TABLE places 
ADD CONSTRAINT check_pillars 
CHECK (connectivity IS NOT NULL 
  AND energy IS NOT NULL 
  AND environment IS NOT NULL);

-- Valores solo permitidos (enum-like)
ALTER TABLE places 
ADD CONSTRAINT check_connectivity 
CHECK (connectivity IN ('baja', 'media', 'alta'));

-- El usuario no puede validar el mismo lugar >1 vez/minuto
-- (Implementar en backend con query de tiempo)
```

---

## 📦 Índices para Performance

```sql
-- Búsqueda por proximidad geográfica
CREATE INDEX idx_places_geom ON places USING GIST(
  ll_to_earth(latitude, longitude)
);

-- Búsqueda de validaciones recientes por lugar
CREATE INDEX idx_validations_place_created ON validations(place_id, created_at DESC);

-- Búsqueda de validaciones por usuario
CREATE INDEX idx_validations_user_created ON validations(user_id, created_at DESC);

-- Misiones pendientes de expiración
CREATE INDEX idx_missions_user_expires ON missions(user_id, expires_at DESC);

-- Impacto por usuario (para dashboards)
CREATE INDEX idx_user_impact_created ON user_impact(user_id, created_at DESC);
```

---

## 🔄 Versionado (Datos Obsoletos)

**No usar tabla de historial completo.** Usar:
- `validations` → registro inmutable de cada actualización
- `places.last_verified` → timestamp único (no histórico)
- TTL 2h → Lógica de negocio, no DB

**Implicación:** Data fresh, sin bloat de historial completo.
