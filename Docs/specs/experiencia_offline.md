# Experiencia Offline y Sincronización: Red-Nómada MVP

## Referencia a HU
- **HU2:** "Input de un toque: El usuario debe poder confirmar el estado actual del lugar desde la alerta de forma rápida para mantener el sistema vivo y actualizado en tiempo real"
- **Restricción:** "Autonomía y Conectividad: No asumas internet constante ni batería ilimitada"

---

## 1. Estados de Conectividad

### Definiciones
| Estado | Definición | API disponible |
|---|---|---|
| **Online** | Conexión estable detectada | Todas |
| **Offline** | Sin conexión | Solo lectura cacheada + cola |
| **Intermitente** | Conexión inestable (netinfo reporta `isInternetReachable: false` intermitente) | Lectura cacheada + cola, requests con retry |

### Detección
- `@react-native-community/netinfo` con `addEventListener` en el root de la app
- Chequeo al abrir la app + listener continuo para cambios
- **No polling periódico** — reactivo a eventos de red para ahorrar batería

---

## 2. Estrategia de Cache Local

### Qué se cachea

| Dato | Almacenamiento | TTL | Tamaño aprox |
|---|---|---|---|
| Lugares cercanos (snapshots) | AsyncStorage (`cache:lugares`) | 30 min | < 500KB |
| Detalle de lugar visitado | AsyncStorage (`cache:lugar:{id}`) | 30 min | < 50KB c/u |
| Perfil de usuario | AsyncStorage (`cache:perfil`) | 1 hora | < 10KB |
| Misiones del día | AsyncStorage (`cache:misiones`) | Hasta 00:00 del día siguiente | < 20KB |
| Mapa tiles (post-MVP) | FileSystem | 7 días | Variable |

### Qué NO se cachea
- **Tokens de auth** → van en `expo-secure-store` (encriptado)
- **Reportes pendientes** → van en cola separada (ver sección 3)
- **Datos de otros usuarios** → no se cachean

### Invalidación
- TTL expirado → se descarta y se fetch fresh
- Al crear un reporte o validar → invalidar cache de ese `lugar_id`
- Pull-to-refresh → invalidar y re-fetch
- Límite total de cache: **5MB** — al exceder, se eliminan los más antiguos primero

---

## 3. Cola de Reportes Pendientes

### Estructura en AsyncStorage
```json
// key: "queue:reportes"
[
  {
    "id": "uuid-local",
    "lugar_id": "uuid",
    "conectividad": "ALTA",
    "energia": "SUFICIENTES",
    "ambiente": "SILENCIOSO",
    "created_at": "2026-03-28T15:30:00Z",
    "intentos": 0
  }
]
```

### Reglas
- **Orden:** FIFO (primero en entrar, primero en salir)
- **Límite:** máximo 20 reportes en cola
- **Antigüedad:** si un reporte lleva > 2 horas encolado → se descarta automáticamente (ya no sería fresco)
- **UI:** badge "N reportes pendientes" visible en tab de mapa

### Procesamiento
- Al reconectarse → despachar cola uno por uno (serial, no paralelo)
- Si un envío falla → incrementar `intentos`, esperar 5 segundos, reintentar
- Máximo 3 intentos por reporte → si falla los 3, se descarta y se notifica al usuario
- Éxito → eliminar de la cola, actualizar cache del lugar

---

## 4. Cola de Validaciones Pendientes

### Estructura
```json
// key: "queue:validaciones"
[
  {
    "id": "uuid-local",
    "reporte_id": "uuid",
    "created_at": "2026-03-28T15:30:00Z",
    "intentos": 0
  }
]
```

### Reglas
- Similar a reportes: FIFO, máximo 10, 3 reintentos
- **Conflicto:** si al sincronizar el reporte ya fue validado por otro usuario → se descarta silenciosamente (no es error)
- **Conflicto:** si el reporte ya no existe → se descarta y se notifica

---

## 5. UI Offline

### Indicadores visuales
- **Banner persistente** en la parte superior: `"Sin conexión — mostrando datos guardados"` (fondo amarillo/naranja)
- **Badge de cola pendiente** en el tab de mapa: número de items encolados
- **Icono de nube tachada** en el header cuando está offline

### Acciones disponibles offline
| Acción | Disponible offline | Comportamiento |
|---|---|---|
| Ver lugares cacheados | ✅ | Muestra datos del cache con indicador de antigüedad |
| Ver detalle de lugar cacheado | ✅ | Snapshot con tag "datos pueden no estar actualizados" |
| Crear reporte | ✅ | Se encola, badge se actualiza |
| Validar reporte | ✅ | Se encola, badge se actualiza |
| Ver misiones | ✅ | Datos cacheados (progreso local) |
| Ver perfil | ✅ | Datos cacheados |
| Navegar en mapa | ✅ | Sin nuevos marcadores, solo cacheados |
| Actualizar datos (pull-to-refresh) | ❌ | Mensaje: "Sin conexión — no se pueden actualizar los datos" |

### Feedback al encolar
- Toast: `"Reporte guardado — se enviará cuando vuelva la conexión"`
- Animación de confirmación (checkmark) aunque esté offline
- El reporte se refleja localmente en el progreso de misiones

---

## 6. Reconexión y Sincronización

### Trigger
- **Automático:** `NetInfo.addEventListener` detecta `isInternetReachable: true` después de estar offline
- **Manual:** pull-to-refresh cuando hay conexión
- **Al abrir la app** con conexión

### Orden de procesamiento
1. Refrescar token de auth si es necesario
2. Procesar cola de validaciones (más rápidas)
3. Procesar cola de reportes
4. Invalidar cache y re-fetch datos principales (lugares, misiones, perfil)

### Manejo de errores
- Error 401 → intentar refresh token → si falla, redirigir a login
- Error 409 (conflicto) → descartar item de la cola silenciosamente
- Error 429 (rate limit) → esperar el tiempo indicado en el header `Retry-After`
- Error 500 → reintentar (cuenta en los 3 intentos)

### Notificación al usuario
- Sync completada sin errores → **no notificar** (silencioso)
- Sync con items descartados → Toast: `"Algunos datos no se pudieron sincronizar"`
- Sync fallida completamente → Banner persistente hasta próxima conexión

---

## 7. Consideraciones de Batería

- **No GPS en background** (restricción del proyecto)
- **No polling de red** — reactivo con `addEventListener`
- **Cache agresivo** para evitar requests innecesarios
- **Cola serial** (no paralela) para minimizar radio activo del dispositivo
- **TTL generoso** (30 min) para lugares — un nómada no necesita datos en tiempo real cada segundo
