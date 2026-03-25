# Manejo de Errores: Red-Nómada MVP

## Formato Estándar de Error

Todos los errores del API siguen esta estructura:

```json
{
  "error": {
    "codigo": "REPORTE_DUPLICADO",
    "mensaje": "Ya reportaste este lugar en los últimos 30 minutos",
    "campo": "lugar_id",
    "timestamp": "2026-03-25T14:30:00Z"
  }
}
```

---

## Catálogo de Errores por Dominio

### Autenticación
| Código | HTTP | Mensaje |
|---|---|---|
| `AUTH_CREDENCIALES_INVALIDAS` | 401 | Credenciales inválidas |
| `AUTH_TOKEN_EXPIRADO` | 401 | Token expirado, usa refresh |
| `AUTH_TOKEN_INVALIDO` | 401 | Token inválido |
| `AUTH_REFRESH_INVALIDO` | 401 | Refresh token inválido o expirado |
| `AUTH_EMAIL_DUPLICADO` | 409 | Este email ya está registrado |
| `AUTH_RATE_LIMIT` | 429 | Demasiados intentos, espera 15 minutos |

### Reportes
| Código | HTTP | Mensaje |
|---|---|---|
| `REPORTE_INCOMPLETO` | 400 | Los 3 pilares son obligatorios |
| `REPORTE_VALOR_INVALIDO` | 400 | Valor no permitido para {pilar} |
| `REPORTE_DUPLICADO` | 429 | Ya reportaste este lugar en los últimos 30 minutos |
| `LUGAR_NO_ENCONTRADO` | 404 | Lugar no encontrado |

### Validaciones
| Código | HTTP | Mensaje |
|---|---|---|
| `VALIDACION_PROPIA` | 403 | No podés validar tu propio reporte |
| `VALIDACION_DUPLICADA` | 409 | Ya validaste este reporte |
| `REPORTE_NO_ENCONTRADO` | 404 | Reporte no encontrado |

### Ubicación
| Código | HTTP | Mensaje |
|---|---|---|
| `UBICACION_REQUERIDA` | 400 | Se requiere latitud y longitud |
| `UBICACION_FUERA_RANGO` | 400 | Coordenadas fuera de rango válido |

---

## Manejo en Frontend (React Native)

### Estrategia por tipo de error
- **401 (Token expirado):** Refresh automático transparente (interceptor Axios), reintentar request original
- **401 (Refresh inválido):** Navegar a pantalla de Login, limpiar stores y expo-secure-store
- **400 (Validación):** Mostrar error inline en el campo correspondiente
- **403 (Prohibido):** Mostrar toast/snackbar con mensaje del servidor (react-native-toast-message)
- **404 (No encontrado):** Mostrar pantalla de "Lugar no encontrado"
- **429 (Rate limit):** Mostrar toast con tiempo de espera
- **500 (Error interno):** Mostrar mensaje genérico "Algo salió mal, intentá de nuevo"

### Errores de conectividad (offline)
- Detectar estado de red con `@react-native-community/netinfo`
- Mostrar banner persistente: **"Sin conexión — mostrando datos guardados"**
- Encolar reportes pendientes en AsyncStorage para enviar cuando vuelva la conexión
- Listener de reconexión (`NetInfo.addEventListener`) para despachar cola
- No intentar re-enviar automáticamente más de **3 veces**

---

## Logging (Backend — Spring Boot)

- **Logger:** SLF4J + Logback (incluido en Spring Boot)
- **Error 4xx:** log level `warn`, no alerta
- **Error 5xx:** log level `error`, alerta a monitoring
- **Manejo centralizado:** `@RestControllerAdvice` con `GlobalExceptionHandler`
- Incluir en logs: `requestId` (MDC), `userId`, `endpoint`, `errorCode`, `timestamp`
- **No loguear:** passwords, tokens, datos sensibles del usuario
