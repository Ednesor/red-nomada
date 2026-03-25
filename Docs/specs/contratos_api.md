# Contratos API: Red-Nómada MVP

**Base URL:** `/api/v1`  
**Formato:** JSON  
**Autenticación:** Bearer Token (JWT)

---

## Autenticación

### `POST /auth/registro`
Registrar nuevo usuario.
- **Body:** `{ email, nombre, password }`
- **Response 201:** `{ token, refreshToken, usuario: { id, email, nombre, rol } }`
- **Errores:** 400 (validación), 409 (email duplicado)

### `POST /auth/login`
Iniciar sesión.
- **Body:** `{ email, password }`
- **Response 200:** `{ token, refreshToken, usuario: { id, email, nombre, rol } }`
- **Errores:** 401 (credenciales inválidas)

### `POST /auth/refresh`
Renovar access token.
- **Body:** `{ refreshToken }`
- **Response 200:** `{ token, refreshToken }`

---

## Lugares

### `GET /lugares`
Listar lugares cercanos. 🔓 Público
- **Query params:**
  - `lat` (required): latitud del usuario
  - `lng` (required): longitud del usuario
  - `radio` (optional): radio en metros, default 1000
  - `categoria` (optional): `CAFE | COWORKING | BIBLIOTECA | OTRO`
- **Response 200:**
```json
[
  {
    "id": "uuid",
    "nombre": "Café Nómada",
    "direccion": "Av. Corrientes 1234",
    "latitud": -34.603,
    "longitud": -58.381,
    "categoria": "CAFE",
    "distancia_metros": 250,
    "snapshot": {
      "conectividad": "ALTA",
      "energia": "SUFICIENTES",
      "ambiente": "MODERADO",
      "es_fresco": true,
      "ultima_actualizacion": "2026-03-25T14:30:00Z",
      "necesita_verificacion": false
    }
  }
]
```

### `GET /lugares/:id`
Detalle de un lugar con snapshot actual. 🔓 Público
- **Response 200:** Objeto lugar + snapshot + historial reciente de reportes
- **Side effect:** Registra una `ConsultaLugar` para métrica de impacto

---

## Reportes

### `POST /lugares/:lugarId/reportes`
Crear un reporte de estado. 🔒 Auth requerida
- **Body:**
```json
{
  "conectividad": "ALTA",
  "energia": "SUFICIENTES",
  "ambiente": "SILENCIOSO"
}
```
- **Response 201:** `{ reporte, impacto: { mensaje: "Gracias a tu reporte..." } }`
- **Errores:**
  - 400: pilares incompletos o valores inválidos
  - 429: ya reportó este lugar en los últimos 30 minutos

### `GET /lugares/:lugarId/reportes`
Historial de reportes de un lugar. 🔓 Público
- **Query params:** `limit` (default 10), `offset`
- **Response 200:** Lista de reportes con `es_fresco` calculado

---

## Validaciones

### `POST /reportes/:reporteId/validar`
Confirmar estado de un reporte existente. 🔒 Auth requerida
- **Body:** vacío (un toque)
- **Response 201:** `{ validacion, reporte_actualizado, impacto: { mensaje } }`
- **Errores:**
  - 403: no puede validar su propio reporte
  - 409: ya validó este reporte

---

## Verificación (Alertas Pasivas)

### `GET /verificaciones/pendientes`
Lugares cercanos que necesitan verificación. 🔒 Auth requerida
- **Query params:** `lat`, `lng`, `radio` (default 500m)
- **Response 200:**
```json
[
  {
    "lugar_id": "uuid",
    "nombre": "Biblioteca Central",
    "distancia_metros": 120,
    "ultimo_reporte_hace": "2h 15m",
    "pregunta": "¿Sigue tranquilo este lugar?"
  }
]
```

---

## Gamificación

### `GET /misiones`
Misiones activas del día para el usuario. 🔒 Auth requerida
- **Response 200:**
```json
[
  {
    "id": "uuid",
    "titulo": "Reportá 3 cafés hoy",
    "tipo": "REPORTAR",
    "objetivo_cantidad": 3,
    "progreso_actual": 1,
    "puntos_recompensa": 50,
    "completada": false
  }
]
```

### `GET /perfil/impacto`
Estadísticas de impacto del usuario. 🔒 Auth requerida
- **Response 200:**
```json
{
  "personas_ayudadas_total": 156,
  "reportes_realizados": 42,
  "validaciones_realizadas": 18,
  "puntos_totales": 780,
  "insignias": [
    { "nombre": "Primer Reporte", "icono": "first_report", "otorgada_at": "..." }
  ]
}
```

---

## Permisos de Ubicación

### `PUT /perfil/permisos-ubicacion`
Actualizar preferencia de ubicación. 🔒 Auth requerida
- **Body:** `{ permiso: "SIEMPRE" | "UNA_VEZ" | "SOLO_EN_USO" | "NINGUNO" }`
- **Response 200:** `{ permiso_ubicacion: "SOLO_EN_USO" }`

---

## Códigos de Error Comunes

| Código | Significado |
|---|---|
| 400 | Datos de entrada inválidos (validación Zod) |
| 401 | Token ausente o expirado |
| 403 | Acción no permitida (ej: validar propio reporte) |
| 404 | Recurso no encontrado |
| 409 | Conflicto (ej: email duplicado, ya validado) |
| 429 | Rate limit (ej: reporte duplicado en < 30min) |

---

## Headers Requeridos

```
Content-Type: application/json
Authorization: Bearer <jwt_token>  (en endpoints 🔒)
```
