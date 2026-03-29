# Flujo de Reporte Completo: Red-Nómada MVP

## Referencia a HU
- **HU1:** "Pilares Obligatorios: Conectividad (Baja/Media/Alta), Energía (Pocos/Suficientes/Muchos), Ambiente (Silencioso/Moderado/Ruidoso)"
- **HU1:** "UI Eficiente: decisión en menos de 5 segundos"
- **HU3:** "Métrica de Impacto Social: Inmediatamente después de finalizar una re-validación o reporte, el sistema debe mostrar un mensaje de refuerzo"
- **HU3:** "Misiones: puntos adicionales por 'Confirmar un lugar lleno' y un bonus por 'Ser el primero en reportar este lugar'"

---

## 1. Puntos de Entrada al Reporte

| Origen | Navegación | Pre-condiciones |
|---|---|---|
| Mapa → tap marcador → "Reportar estado" | Bottom sheet → pantalla de reporte | Autenticado |
| Detalle lugar (`/lugar/[id]`) → botón "Reportar" | Navegación directa | Autenticado |
| Alerta de verificación → "Confirmar" | Acción directa desde alerta | Autenticado, ubicación activa (opcional) |

### Pre-chequeos antes de mostrar el formulario
1. ¿Usuario autenticado? → si no, redirigir a login
2. ¿Ya reportó este lugar en los últimos 30 min? → mostrar countdown y deshabilitar
3. ¿Offline? → permitir (se encola, ver experiencia_offline.md)

---

## 2. UI del Formulario de Reporte

### Diseño: un solo paso, 3 pilares simultáneos
- Pantalla/modal con los 3 pilares visibles al mismo tiempo
- Cada pilar es un **grupo de toggle buttons** (chips/botones seleccionables)

### Interacción por pilar
```
Conectividad:  [Baja]  [Media]  [Alta]
Energía:       [Pocos] [Suficientes] [Muchos]
Ambiente:      [Silencioso] [Moderado] [Ruidoso]
```
- Tap selecciona, tap de nuevo deselecciona
- Solo 1 valor por pilar (radio behavior)
- Valores seleccionados se destacan visualmente (color de fondo)

### Validación en tiempo real
- Si intenta enviar con pilares incompletos → highlight rojo en los faltantes
- Toast: "Seleccioná un valor para cada pilar"
- El botón "Enviar" se habilita **solo cuando los 3 están completos**

### Botón de envío
- Texto: "Enviar reporte"
- Posición: fijo en la parte inferior de la pantalla (siempre accesible)
- Estado loading al enviar: spinner + texto "Enviando..."

---

## 3. Confirmación Pre-Envío

- **No hay paso de confirmación** — el envío es directo al tocar "Enviar"
- Razón: HU1 exige rapidez (< 5 segundos). Un paso de confirmación agregaría fricción
- El usuario puede cambiar su selección antes de tocar enviar

---

## 4. Feedback Post-Envío

### Éxito (response 201)
1. **Animación de éxito:** checkmark animado + vibración háptica
2. **Mensaje de impacto social:** `"Gracias a tu reporte, X personas evitaron ir a un lugar lleno"` (del response)
3. **Progreso de misión:** si completó una misión → animación de celebración + puntos ganados visibles
4. **Botón:** "Volver al mapa" o timeout de 3 segundos y auto-navegar

### Actualización de datos
- El snapshot del lugar se actualiza inmediatamente en cache local
- Las misiones se refrescan (progreso actualizado)
- Los puntos_totales del perfil se incrementan

---

## 5. Manejo de Errores en el Flujo

### Error 429 — Ya reportó en 30 min
- Mostrar pantalla/modal con:
  - Mensaje: "Ya reportaste este lugar recientemente"
  - Countdown visible: "Podés volver a reportar en 24:30"
  - Botón: "Entendido" → volver atrás
- **El formulario NO se muestra** si el backend responde 429 al consultar eligibility

### Error 400 — Pilares incompletos o inválidos
- Highlight rojo en los campos con error
- Mensaje inline debajo de cada pilar inválido
- Botón "Enviar" permanece habilitado (para re-intentar)

### Error de red / Offline
- **Si offline al enviar:** encolar reporte, mostrar toast "Reporte guardado — se enviará cuando vuelva la conexión", animación de éxito local
- **Si error de red online:** toast "No se pudo enviar, reintentando..." + retry automático una vez
- **Si retry falla:** guardar en cola offline + toast "Se enviará cuando vuelva la conexión"

### Error 500 — Error interno
- Toast genérico: "Algo salió mal, intentá de nuevo"
- Botón "Reintentar" visible
- Log del error en Sentry/crashlytics (post-MVP)

---

## 6. Casos Especiales

### Primer reporte de un lugar
- Al enviar exitosamente → si es el primer reporte de ese lugar (lifetime):
  - Mensaje adicional: "¡Sos el primero en reportar este lugar! +15 puntos bonus"
  - Badge temporal en la animación de éxito

### Reporte que contradice uno anterior
- Se procesa normalmente (el backend reemplaza el anterior como estado actual)
- No se notifica al usuario que su reporte contradice otro
- El historial se conserva internamente

### Re-validación (un toque) vs reporte completo
- La re-validación desde una alerta NO abre el formulario
- Es un solo botón: "¿Sigue tranquilo este café?" → Sí/No
- "Sí" = confirma el estado actual del último reporte
- "No" = abre el formulario completo de reporte para corregir
- Flujo documentado en notificaciones_alertas.md

---

## 7. Animaciones y Micro-Interacciones

| Momento | Animación |
|---|---|
| Seleccionar pilar | Scale del chip seleccionado + cambio de color |
| Habilitar botón enviar | Fade-in del botón cuando los 3 pilares están completos |
| Enviar | Loading spinner en el botón |
| Éxito | Checkmark animado (Lottie o animación nativa) + vibración leve |
| Misión completada | Confetti/celebración overlay (2 segundos) |
| Error | Shake del formulario + toast |

---

## 8. Tracking y Analytics (post-MVP)

- Tiempo desde apertura del formulario hasta envío (debe ser < 5s para HU1)
- Tasa de abandono del formulario
- Pilares más frecuentemente seleccionados
- Lugares más reportados
