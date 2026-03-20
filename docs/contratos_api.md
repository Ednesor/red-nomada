# Contratos de API: Red-Nómada

## 📡 Base URL
```
http://localhost:3000/api
```

---

## 🏢 Endpoints: Places (Historia 1 - Snapshot)

### GET /places/:id
**Descripción:** Obtener snapshot técnico de un lugar.

**Path Params:**
- `id` (UUID) - ID del lugar

**Response (200 OK):**
```json
{
  "id": "uuid-123",
  "name": "Café Nómada",
  "latitude": 37.7749,
  "longitude": -122.4194,
  "connectivity": "alta",
  "energy": "suficientes",
  "environment": "moderado",
  "last_verified": "2026-03-20T10:30:00Z",
  "verification_count": 5
}
```

**Error (404 Not Found):**
```json
{
  "error": "Place not found",
  "code": "PLACE_NOT_FOUND"
}
```

**Historia Cubierta:** #1 (Snapshot <5s)  
**Restricción Clave:** Cero notas generales, solo 3 pilares

---

### POST /places
**Descripción:** Crear un nuevo lugar.

**Body:**
```json
{
  "name": "Café Nómada",
  "latitude": 37.7749,
  "longitude": -122.4194,
  "connectivity": "alta",
  "energy": "suficientes",
  "environment": "moderado"
}
```

**Response (201 Created):**
```json
{
  "id": "uuid-123",
  "name": "Café Nómada",
  "latitude": 37.7749,
  "longitude": -122.4194,
  "connectivity": "alta",
  "energy": "suficientes",
  "environment": "moderado",
  "created_by": "user-456",
  "created_at": "2026-03-20T10:30:00Z"
}
```

**Historia Cubierta:** #3 (Crear nuevos lugares para misión)

---

### GET /places/nearby?lat=:lat&lon=:lon&radius=:radius
**Descripción:** Buscar lugares cercanos (radio en km).

**Query Params:**
- `lat` (number) - Latitud
- `lon` (number) - Longitud
- `radius` (number, default: 5) - Radio en km

**Response (200 OK):**
```json
{
  "places": [
    {
      "id": "uuid-123",
      "name": "Café Nómada",
      "distance_km": 0.5,
      "connectivity": "alta",
      "energy": "suficientes",
      "environment": "moderado",
      "last_verified": "2026-03-20T10:30:00Z"
    }
  ]
}
```

**Historia Cubierta:** #1 (Descubrir espacios cercanos)

---

## ✅ Endpoints: Validations (Historia 2 - Verificación)

### POST /validations/:placeId
**Descripción:** Confirmar/actualizar estado de un lugar.

**Path Params:**
- `placeId` (UUID) - ID del lugar

**Body:**
```json
{
  "connectivity": "alta",
  "energy": "suficientes",
  "environment": "moderado"
}
```

**Response (201 Created):**
```json
{
  "id": "validation-uuid",
  "place_id": "place-uuid",
  "user_id": "user-uuid",
  "connectivity": "alta",
  "energy": "suficientes",
  "environment": "moderado",
  "people_saved": 12,
  "created_at": "2026-03-20T10:35:00Z"
}
```

**Historia Cubierta:** #2 (Input 1-tap), #3 (Impacto inmediato)  
**Restricción Clave:** Solo cuando app está abierta (sin background)

---

### GET /places/:placeId/status
**Descripción:** Obtener estado de urgencia de un lugar (¿necesita verificación?)

**Path Params:**
- `placeId` (UUID) - ID del lugar

**Response (200 OK):**
```json
{
  "id": "place-uuid",
  "name": "Café Nómada",
  "last_verified": "2026-03-19T08:30:00Z",
  "hours_since_verification": 26,
  "needs_verification": true,
  "urgent_message": "Este lugar necesita verificación AHORA"
}
```

**Historia Cubierta:** #2 (Banner "AHORA" si >2h)

---

## 🔐 Endpoints: Permissions (Historia 2 - Permisos)

### POST /permissions
**Descripción:** Guardar preferencia de ubicación del usuario.

**Body:**
```json
{
  "location_permission": "always" | "while_using" | "never"
}
```

**Response (201 Created):**
```json
{
  "user_id": "user-uuid",
  "location_permission": "always",
  "updated_at": "2026-03-20T10:30:00Z"
}
```

**Historia Cubierta:** #2 (Gestión de permisos granulares)  
**Restricción Clave:** Pregunta obligatoria en primer launch

---

### GET /permissions
**Descripción:** Obtener permisos actuales del usuario.

**Response (200 OK):**
```json
{
  "user_id": "user-uuid",
  "location_permission": "always" | "while_using" | "never"
}
```

**Historia Cubierta:** #2 (Validar scope de permisos)

---

## 🎮 Endpoints: Missions (Historia 3 - Gamificación)

### GET /missions
**Descripción:** Obtener misiones activas del usuario.

**Response (200 OK):**
```json
{
  "missions": [
    {
      "id": "mission-uuid",
      "title": "Reportá 3 cafés hoy",
      "description": "Crea 3 nuevos lugares",
      "target": 3,
      "current": 1,
      "completed": false,
      "reward_points": 50,
      "expires_at": "2026-03-20T23:59:00Z"
    },
    {
      "id": "mission-uuid-2",
      "title": "Ser el primero",
      "description": "Sé el primero en reportar un lugar",
      "target": 1,
      "current": 0,
      "completed": false,
      "reward_points": 100,
      "expires_at": "2026-03-27T23:59:00Z"
    }
  ]
}
```

**Historia Cubierta:** #3 (Misiones activas)

---

### POST /missions/:missionId/complete
**Descripción:** Marcar misión como completada (trigger automático desde validación o crear lugar).

**Path Params:**
- `missionId` (UUID) - ID de la misión

**Body:** (opcional, puede ser vacío)
```json
{}
```

**Response (200 OK):**
```json
{
  "id": "mission-uuid",
  "title": "Reportá 3 cafés hoy",
  "completed": true,
  "points_awarded": 50,
  "total_user_points": 250,
  "completed_at": "2026-03-20T10:40:00Z"
}
```

**Historia Cubierta:** #3 (Completar misión + otorgar puntos)

---

## 📊 Endpoints: Impact (Historia 3 - Impacto Social)

### GET /user/impact
**Descripción:** Obtener métricas de impacto del usuario.

**Response (200 OK):**
```json
{
  "user_id": "user-uuid",
  "total_validations": 8,
  "total_people_saved": 48,
  "average_people_per_validation": 6,
  "recent_impact": [
    {
      "validation_id": "val-uuid",
      "place_name": "Café Nómada",
      "people_saved": 12,
      "created_at": "2026-03-20T10:35:00Z"
    }
  ]
}
```

**Historia Cubierta:** #3 (Visualizar cuántas personas ayudé)

---

## 👤 Endpoints: Users (Setup)

### POST /users
**Descripción:** Crear usuario nuevo (signup).

**Body:**
```json
{
  "username": "nomada_123",
  "email": "user@example.com",
  "location_permission": "while_using"
}
```

**Response (201 Created):**
```json
{
  "id": "user-uuid",
  "username": "nomada_123",
  "email": "user@example.com",
  "location_permission": "while_using",
  "points": 0,
  "created_at": "2026-03-20T10:30:00Z"
}
```

---

### GET /users/:id
**Descripción:** Obtener perfil del usuario.

**Response (200 OK):**
```json
{
  "id": "user-uuid",
  "username": "nomada_123",
  "points": 250,
  "location_permission": "while_using",
  "total_validations": 8,
  "joined_at": "2026-03-15T10:30:00Z"
}
```

---

## 🔄 Flujo WebSocket: Verificación Pasiva (Historia 2)

**Evento enviado desde backend → client:**
```
Event: 'verification_request'
Payload: {
  "place_id": "place-uuid",
  "place_name": "Café Nómada",
  "message": "¿Sigue tranquilo este café?",
  "last_verified_hours_ago": 3
}
```

**Cliente responde:**
```
POST /validations/:placeId
```

---

## 📋 Resumen por Historia

| Historia | Endpoints | Flujo |
|---|---|---|
| **#1 Snapshot** | GET /places/:id, GET /places/nearby | Obtener 3 pilares en <500ms |
| **#2 Verificación** | POST /validations, GET /places/:id/status, POST/GET /permissions | Listener local → WebSocket → 1-tap confirm |
| **#3 Impacto** | GET /missions, POST /missions/:id/complete, GET /user/impact | Validar → Calcular people_saved → +Puntos |

---

## ✅ Validación de Contratos

Cada endpoint debe cumplir:
- ✓ Response con estructura consistente
- ✓ Status codes correctos (201 para POST, 404 para no encontrado)
- ✓ Sin enviar datos sensibles (GPS preciso, histórico completo)
- ✓ Timestamps en ISO 8601
- ✓ Enums fijos (connectivity, energy, environment, location_permission)
