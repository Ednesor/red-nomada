# Restricciones Clave del Dominio: Red-Nómada

## 🎯 Principio Central

Red-Nómada **elimina la incertidumbre técnica** para nómadas digitales en menos de 5 segundos mediante datos validados colectivamente. No es una red social ni un directorio subjetivo.

---

## 🚫 Restricciones Críticas de Negocio

### 1. The Snapshot (Decisión en <5 segundos)
**Regla:** El usuario debe poder tomar una decisión sobre un espacio en menos de 5 segundos visuales.

- **Prohibición absoluta:** Cero estrellas, cero notas generales (1-10, A-F, etc.)
- **Base:** Exclusivamente los 3 pilares técnicos:
  - **Conectividad:** Baja | Media | Alta (sin gradientes)
  - **Energía:** Pocos | Suficientes | Muchos (sin gradientes)
  - **Ambiente:** Silencioso | Moderado | Ruidoso (sin gradientes)
- **Latencia:** UI debe renderizar en <500ms depuis cache local
- **Implicación:** Diseño minimalista, sin análisis subjetivos, sin reseñas

---

### 2. Datos Obsoletos: Marca Crítica >2 horas
**Regla:** Información con más de 2 horas sin validación debe marcarse como **urgente de verificar**.

- **Trigger:** `lastVerified > 2 horas`
- **Visual:** Banner/alerta permanente: **"Este lugar necesita verificación AHORA"**
- **Acción:** Priorizar a este lugar en alertas de validación
- **Implicación:** Sistema de "versionado temporal" de datos, no histórico

---

### 3. Validación Pasiva: Solo en Foreground
**Regla:** Las solicitudes de re-validación **solo pueden ocurrir cuando el usuario tiene la app abierta**.

- **Prohibición:** No usar GPS en background
- **Prohibición:** No usar notificaciones push externas como único canal
- **Mecanismo:** Listener local que detecta `lastVerified > 2h` en SQLite
- **Transporte:** WebSocket (si existe sesión activa) o queue local
- **Implicación:** No drenaje de batería, respeto a privacidad del usuario

---

### 4. Gestión de Permisos: Granular Explícita
**Regla:** El usuario debe elegir explícitamente cómo se usa su ubicación.

**Opciones únicas:**
1. **"Siempre"** → app puede solicitar validaciones en background (si está en use)
2. **"Una vez"** → sin acceso recurrente a ubicación
3. **"Solo en uso"** → ubicación solo cuando app está en foreground

- **Momento:** Pregunta obligatoria en primer launch
- **Actualización:** Usuario puede cambiar en settings
- **Bloqueo:** Si elige "Nunca", ocultar todos features que requieran ubicación
- **Implicación:** Consentimiento real, auditable

---

### 5. Impacto Social Tangible: Métrica Directa
**Regla:** Todo reporte/validación debe mostrar **inmediatamente** cuántas personas fueron ayudadas.

- **Cálculo:** Número de usuarios que consultaron este lugar en últimas 24h
- **Timing:** Mostrar al completar validación (no diferido)
- **Formato:** *"Gracias a tu reporte, 12 personas evitaron ir a un lugar lleno"*
- **Auditoría:** Registrar en tabla `user_impact` para análisis posterior
- **Implicación:** Gamificación basada en hechos, no estimaciones

---

### 6. Misiones: Comportamientos Específicos Incentivados
**Regla:** Las misiones deben incentivar reportes y validaciones, no consumo pasivo.

**Ejemplos autorizados:**
- "Reportá 3 cafés hoy" (crear nuevos lugares)
- "Ser el primero en reportar" (bonus +25 puntos)
- "Confirmar un lugar lleno" (bonus +10 puntos)
- "7 validaciones esta semana" (insignias)

**Prohibición:**
- No premiar consumo pasivo de datos
- No gamificar rating subjetivo
- No crear competencias tóxicas

---

## 🔋 Restricciones Técnicas (Dominio)

### Autonomía vs. Conectividad
**Regla:** El sistema debe *asumir* que:
- Conexión a internet es **intermitente**, no garantizada
- Batería es **limitada**
- Datos celulares son **caros** (en algunos países)

**Implicación:**
- SQLite local es persistencia primaria
- Sincronización es eventual + lazy
- Batch requests (no micro-updates)
- Wi-Fi preferred, 4G fallback

---

### Precisión Temporal: TTL Estricto
**Regla:** Ningún dato debe considerarse válido más de 2 horas.

- **Cache local:** TTL 2h (luego marcar con banner)
- **Cache server:** TTL <30 min
- **Re-validación:** Distribuir entre usuarios según permiso de ubicación
- **Implicación:** No hay "datos históricos acumulados", solo snapshot actual

---

### Equidad en Solicitudes de Validación
**Regla:** Las alertas de re-validación deben distribuirse **equitativamente** entre usuarios.

- **Algoritmo:** Priorizar a usuarios con permisos Always/WhileUsing
- **Límite:** Max 3-5 validaciones/usuario/día (evitar spam)
- **Round-robin:** Distribuir solicitudes para evitar sesgo
- **Implicación:** Escalabilidad comunitaria sin agotamiento

---

## ✅ Criterios de Aceptación Generales

Toda funcionalidad debe cumplir **obligatoriamente** con:

1. ✓ **Snapshot <5s:** Si visualiza un lugar, ¿puede decidir en 5 segundos?
2. ✓ **Cero notas generales:** ¿Tiene estrellas o notas del 1-10? → RECHAZAR
3. ✓ **3 pilares únicamente:** ¿Visualiza Conectividad/Energía/Ambiente? → OK
4. ✓ **Datos obsoletos marcados:** ¿Lugar >2h sin validación muestra banner "AHORA"?
5. ✓ **Validación pasiva:** ¿La alerta sale solo si app está abierta?
6. ✓ **Permisos explícitos:** ¿Se preguntó al usuario Always/While/Never?
7. ✓ **Impacto calculado:** ¿Se mostró "X personas evitaron" inmediatamente?
8. ✓ **Sin GPS background:** ¿Hay rastreo en background? → RECHAZAR
9. ✓ **Funciona offline:** ¿Sincroniza sin internet? → OK
10. ✓ **Equidad comunal:** ¿Se distribuyen validaciones por usuario?

---

## 🔐 Caso de Rechazo: Violación de Reglas

**Ejemplo de feature que sería RECHAZADA:**

> *"Agregar puntuación de 1-5 estrellas para permitir que usuarios califiquen subjetivamente la experiencia"*

**Razón:** Viola regla #1 (Prohibición de notas generales) y contradice el principio central de Red-Nómada.

---

## 📊 Evolución Futura (Post-MVP)

Estas reglas se mantienen para MVP. Extensiones posibles (NO implementar aún):
- Historial de cambios en estado de pilares
- Tendencias por hora/día
- Reputación del validador
- Categorías de espacios (café, coworking, etc.)

**Restricción:** No pueden introducir notas generales ni puntuaciones subjetivas.

