# Notificaciones Locales y Sistema de Alertas: Red-Nómada MVP

## Referencia a HU
- **HU2:** "Recibir alertas visuales pasivas para verificar lugares con información próxima a expirar"
- **HU2:** "Validación Pasiva en App: priorizando a los usuarios que estén utilizando la app en ese momento. Se prohíbe requerir el rastreo GPS en segundo plano o notificaciones push externas como única vía de funcionamiento"
- **HU2:** "Etiqueta de Urgencia: 'Este lugar necesita verificación AHORA'"
- **HU2:** "Input de un toque: El usuario debe poder confirmar el estado actual del lugar desde la alerta"
- **Restricción:** "Validación Pasiva: Las solicitudes de re-validación deben ser visuales y pasivas, apareciendo solo cuando el usuario tenga la app abierta"

---

## 1. Tipos de Alerta/Verificación

### A. Banner de urgencia (en detalle del lugar)
- **Dónde:** pantalla `/lugar/[id]`
- **Cuándo:** el lugar tiene datos >= 2 horas de antigüedad
- **Diseño:** banner naranja/rojo fijo en la parte superior de la pantalla
- **Texto:** `"Este lugar necesita verificación AHORA"`
- **Acción:** botón "Confirmar estado" que abre re-validación de un toque

### B. Alerta de verificación pasiva (proximidad)
- **Dónde:** pantalla principal (mapa) o tab de verificaciones
- **Cuándo:** el usuario tiene la app abierta y hay lugares obsoletos cerca (radio 500m)
- **Diseño:** card/banner deslizable desde arriba con el nombre del lugar y pregunta
- **Ejemplo:** `"¿Sigue tranquilo el Café Nómada?" [Sí, confirmar] [Ver detalle]`
- **Comportamiento:** aparece 1 vez, si se dismissa no reaparece en 30 minutos para ese lugar

### C. Badge de verificaciones pendientes
- **Dónde:** tab de navegación (ícono de mapa o sección de verificaciones)
- **Dónde más:** bottom sheet del marcador en el mapa
- **Cuándo:** cuando hay >= 1 lugar obsoleto cerca del usuario
- **Diseño:** badge numérico rojo sobre el ícono
- **Comportamiento:** se actualiza al mover el mapa o al abrir la app

---

## 2. Trigger de Alertas

### Cuándo se chequea
| Evento | Acción |
|---|---|
| Abrir la app | GET /verificaciones/pendientes con ubicación actual |
| Cambiar de pantalla (tab) | Re-chequear si pasaron > 5 min desde último chequeo |
| Centrar mapa en nueva ubicación | GET /verificaciones/pendientes para nueva zona |
| Pull-to-refresh | Forzar chequeo |

### Frecuencia
- Máximo 1 request a `/verificaciones/pendientes` cada 5 minutos
- No hay polling periódico — solo reacciona a eventos del usuario
- Se cachea la última respuesta por 5 minutos en memoria (no AsyncStorage)

### Priorización
- Si hay múltiples lugares obsoletos → mostrar el **más cercano** primero
- Si hay varios a similar distancia → mostrar el **más antiguo** primero
- Máximo 1 alerta pasiva visible a la vez (no spamear)

### Uso de ubicación
- Se usa la ubicación obtenida por `expo-location` (one-shot, no tracking)
- Si el usuario no tiene permisos de ubicación → no se muestran alertas de proximidad
- Fallback: alertas basadas en los últimos lugares consultados (sin geolocalización)

---

## 3. Diseño de la UI de Alertas

### Banner de urgencia (detalle lugar)
```
┌─────────────────────────────────────────┐
│ ⚠️ Este lugar necesita verificación AHORA │
│                                           │
│ [Confirmar estado]         [Ver detalle]  │
└─────────────────────────────────────────┘
```
- Color de fondo: naranja (#F59E0B) con texto blanco
- Posición: sticky top, no se tapa con scroll del contenido
- Animación: slide-down al entrar a la pantalla

### Alerta pasiva (proximidad)
```
┌─────────────────────────────────────────┐
│ 📍 Cerca tuyo                            │
│                                           │
│ ¿Sigue tranquilo el Café Nómada?         │
│ Hace 2h 15m sin actualización             │
│                                           │
│ [Sí, confirmar]    [No, reportar]  [✕]   │
└─────────────────────────────────────────┘
```
- Color de fondo: blanco con borde naranja
- Posición: slide-down desde el top de la pantalla (por encima del contenido)
- Duración visible: permanece hasta que el usuario interactúa o pasan 10 segundos (se minimiza a un FAB)
- Si se cierra con [✕]: se marca como dismissada para ese lugar por 30 min

### Badge
- Número rojo, fondo rojo, texto blanco
- Posición: esquina superior derecha del ícono de tab
- Animación: pulse al cambiar el número

---

## 4. Flujo de Validación desde Alerta

### "Sí, confirmar" (un toque)
1. Usuario toca "Sí, confirmar"
2. Loading state en el botón (spinner)
3. POST /reportes/:reporteId/validar (body vacío)
4. Éxito → animación de checkmark + toast "¡Gracias! Tu confirmación ayuda a la comunidad"
5. Si la alerta era de proximidad → desaparece la card
6. Si era banner de urgencia → cambia a "Verificado ✓" en verde

### "No, reportar"
1. Usuario toca "No, reportar"
2. Navega a la pantalla de reporte completo para ese lugar
3. El formulario se abre pre-seleccionando el lugar

### Errores en la validación
- **403 (validó propio reporte):** toast "No podés validar tu propio reporte"
- **409 (ya validó):** toast "Ya validaste este reporte" + la alerta desaparece
- **404 (reporte no existe):** toast "Este reporte ya no está disponible" + la alerta desaparece
- **Red/offline:** encolar validación (ver experiencia_offline.md)

---

## 5. Permisos y Fallback

### Con permisos de ubicación
- Alertas de proximidad activas (lugares obsoletos cerca del usuario)
- Banner de urgencia funciona en cualquier lugar visitado

### Sin permisos de ubicación
- **No** se muestran alertas de proximidad
- **Sí** se muestra banner de urgencia al visitar un detalle de lugar obsoleto
- **Sí** se muestra badge si el usuario visitó recientemente lugares que ahora están obsoletos
- No se molesta al usuario pidiendo permisos repetidamente

### Solicitud de permisos desde contexto de alertas
- Si el usuario toca una alerta de proximidad y no tiene permisos:
  1. Modal explicativo: "Para mostrarte alertas de lugares cerca tuyo, necesitamos tu ubicación"
  2. Opciones: "Ahora no" | "Activar ubicación"
  3. Si acepta → solicitar permisos con expo-location
  4. Si rechaza → no volver a preguntar desde alertas

---

## 6. Notificaciones Locales (Expo)

### Cuándo se usan
- **Solo en MVP:** notificación local cuando la app pasa a background y hay verificaciones pendientes
- Ejemplo: usuario abre la app, hay 3 lugares obsoletos cerca, el usuario minimiza la app → notificación: "3 lugares cerca tuyo necesitan verificación"

### Implementación
- `expo-notifications` para notificaciones locales
- **No push notifications** (restricción del proyecto — no hay server push)
- Configurar solo cuando la app está en foreground o background cercano
- No configurar listeners cuando la app está cerrada (ahorro batería)

### Contenido de la notificación
- Título: "Red-Nómada necesita tu ayuda"
- Body: "X lugares cerca tuyo necesitan verificación"
- Al tocar → abre la app en el mapa centrado en la ubicación del usuario

---

## 7. Reglas Anti-Spam

| Regla | Detalle |
|---|---|
| Máximo alertas simultáneas | 1 alerta pasiva visible a la vez |
| Cooldown por lugar | 30 min después de dismissar una alerta para ese lugar |
| Cooldown global | Si el usuario dismissa 3 alertas seguidas → no mostrar más por 1 hora |
| Notificaciones locales | Máximo 1 por sesión de background |
| Frecuencia de chequeo | Máximo 1 request cada 5 minutos |

---

## 8. Consideraciones de Batería

- **No GPS continuo** — solo one-shot al abrir la app o mover el mapa
- **No polling** — reacciona a eventos del usuario
- **Cache de verificaciones** — 5 min en memoria, no hace request si el cache es fresco
- **Notificaciones locales** solo en background cercano, no en app cerrada
