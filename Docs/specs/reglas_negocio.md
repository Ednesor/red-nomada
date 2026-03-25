# Reglas de Negocio: Red-Nómada MVP

## 1. Los 3 Pilares (The Snapshot)

### Valores permitidos (enum estricto)
- **Conectividad:** `BAJA` | `MEDIA` | `ALTA`
- **Energía:** `POCOS` | `SUFICIENTES` | `MUCHOS`
- **Ambiente:** `SILENCIOSO` | `MODERADO` | `RUIDOSO`

### Restricciones
- **Prohibido** mostrar puntuaciones generales (estrellas, notas 1-10, promedios numéricos)
- Los 3 pilares son **obligatorios** en cada reporte; no se permite enviar un reporte incompleto
- El Snapshot debe permitir decisión en **< 5 segundos** (diseño UI optimizado)

---

## 2. Expiración y Frescura de Datos

### Regla de las 2 horas
- Un reporte es **fresco** si tiene menos de 2 horas de antigüedad
- Un reporte es **obsoleto** si tiene 2 horas o más
- Reportes obsoletos muestran: **"Este lugar necesita verificación AHORA"**

### Cálculo de estado actual de un lugar
- El estado visible de un lugar se basa en el **reporte más reciente** que sea fresco
- Si no hay reportes frescos → el lugar se marca como **"Sin datos recientes"** + banner de verificación
- Los reportes obsoletos **no se eliminan**, se conservan como histórico pero no se muestran como estado actual

---

## 3. Reportes y Validaciones

### Crear un reporte
- El usuario debe seleccionar un valor para **cada uno de los 3 pilares**
- Se registra automáticamente: timestamp, ID del usuario, ID del lugar
- Ubicación del usuario al momento del reporte (si tiene permisos activos)

### Re-validación (confirmar estado existente)
- Un usuario puede confirmar un reporte existente con **un solo toque**
- Confirmar extiende la frescura del reporte por 2 horas adicionales desde el momento de confirmación
- Un usuario **no puede** re-validar su propio reporte

### Reportes contradictorios
- Si un nuevo reporte contradice al anterior (ej: de `SILENCIOSO` a `RUIDOSO`), el nuevo reporte **reemplaza** al anterior como estado actual
- Se conserva el historial completo para análisis

---

## 4. Gestión de Ubicación y Permisos

### Permisos de geolocalización
- El sistema **siempre** debe preguntar: `"siempre"` | `"una vez"` | `"solo en uso"`
- **Prohibido:** rastreo GPS en segundo plano sin consentimiento explícito
- **Prohibido:** notificaciones push externas como vía principal de interacción

### Validación pasiva
- Las solicitudes de re-validación solo aparecen **cuando la app está abierta**
- Se priorizan usuarios que están **físicamente cerca** del lugar a verificar (si tienen permisos activos)
- Si el usuario no otorga permisos de ubicación → puede usar la app normalmente pero no recibe solicitudes de re-validación basadas en proximidad

---

## 5. Gamificación

### Misiones
- Retos diarios con objetivos concretos (ej: "Reportá 3 cafés hoy")
- Bonus por acciones de alto valor:
  - **"Primer reporte"** de un lugar: puntos extra
  - **"Confirmar lugar lleno/vacío"**: puntos de verificación
- Las misiones se resetean diariamente a las **00:00 hora local** del usuario

### Insignias
- Se otorgan al completar misiones o alcanzar hitos
- No son revocables una vez otorgadas

### Métrica de impacto social
- Después de cada reporte/validación, mostrar: **"Gracias a tu reporte, X personas evitaron ir a un lugar lleno"**
- `X` se calcula como el número de usuarios que **consultaron** ese lugar en las 2 horas siguientes al reporte
- El conteo es aproximado (no requiere precisión exacta)

---

## 6. Lugares (Espacios de Trabajo)

### Datos de un lugar
- Nombre, dirección, coordenadas GPS
- Categoría: `CAFE` | `COWORKING` | `BIBLIOTECA` | `OTRO`
- Estado actual = reporte más reciente fresco (o "Sin datos recientes")

### Alta de lugares
- **MVP:** los lugares se pre-cargan desde una base de datos inicial
- **Post-MVP:** los usuarios podrán sugerir nuevos lugares (requiere moderación)

---

## 7. Usuarios

### Roles
- **Explorador (default):** puede consultar lugares, reportar, re-validar, completar misiones
- **Admin:** gestión de lugares, moderación de reportes, configuración de misiones

### Restricciones
- Un usuario no puede enviar más de **1 reporte por lugar cada 30 minutos** (anti-spam)
- Un usuario no puede re-validar su propio reporte
