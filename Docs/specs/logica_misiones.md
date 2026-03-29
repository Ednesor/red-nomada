# Lógica de Misiones y Progresión: Red-Nómada MVP

## Referencia a HU
- **HU3:** Misiones activas, puntos adicionales por "Confirmar lugar lleno", bonus por "Ser el primero en reportar"
- **HU3:** Métrica de impacto social — "Gracias a tu reporte, 12 personas evitaron ir a un lugar lleno"

---

## 1. Ciclo de Vida de una Misión

### Creación
- **MVP:** Misiones se pre-cargan en BD como datos semilla (Flyway migration)
- Son configurables por un admin (tabla `Mision`), no hardcodeadas en código
- Cada misión tiene: título, descripción, tipo, objetivo_cantidad, puntos_recompensa, activa

### Asignación
- Al consultar `GET /misiones`, el backend busca misiones activas (`Mision.activa = true`)
- Si no existe `MisionProgreso` para `(usuario_id, mision_id, fecha_actual)` → se crea automáticamente con `progreso_actual = 0`
- **Lazy creation:** no se pre-pueblan progresos al inicio del día; se crean on-demand

### Progreso
- Se actualiza **en cada acción relevante** del usuario (al crear reporte, al validar, al ser primero en reportar)
- Lógica backend: `MisionService.actualizarProgreso(usuario, tipo_accion, lugar_id)`
- Se incrementa `progreso_actual += 1` para cada misión del tipo correspondiente activa ese día

### Completitud
- Cuando `progreso_actual >= objetivo_cantidad`:
  1. `completada = true`
  2. Sumar `puntos_recompensa` a `Usuario.puntos_totales`
  3. Si la misión otorga insignia → crear registro en `Insignia`
  4. Retornar en response un flag `mision_completada` para que el frontend muestre celebración

### Reset Diario
- **No hay job de reset** — la separación es por campo `fecha` en `MisionProgreso`
- Cada día es un registro nuevo. La query siempre filtra por `fecha = CURRENT_DATE`
- Limpieza opcional: job semanal que archiva `MisionProgreso` de hace > 30 días

---

## 2. Catálogo de Misiones MVP

| Título | Tipo | Objetivo | Puntos | Insignia |
|---|---|---|---|---|
| Reportá 3 cafés hoy | REPORTAR | 3 reportes (categoría CAFE) | 50 | — |
| Reportá 5 lugares hoy | REPORTAR | 5 reportes (cualquier categoría) | 100 | "Nómada Activo" |
| Primer reporte del día | REPORTAR | 1 reporte | 20 | — |
| Validá 3 reportes hoy | VALIDAR | 3 validaciones | 60 | "Verificador" |
| Ser el primero | PRIMERO_EN_REPORTAR | 1 primer reporte | 40 | "Explorador Pionero" |

---

## 3. Reglas de Puntos

### Puntos por acción directa
- **Crear reporte:** 10 puntos (siempre, independiente de misiones)
- **Validar reporte:** 5 puntos
- **Ser primero en reportar un lugar:** 15 puntos (reemplaza los 10 base)
- **Confirmar lugar lleno:** +5 puntos bonus (sobre los 10 base)

### Puntos por misiones
- Al completar una misión → sumar `Mision.puntos_recompensa` a `Usuario.puntos_totales`

### Fórmula total
- `Usuario.puntos_totales` = Σ(puntos_acciones_directas) + Σ(puntos_misiones_completadas)
- Se actualiza atómicamente con cada acción (Redis INCRBY para performance, persistido a PostgreSQL)

---

## 4. Catálogo de Insignias MVP

| Nombre | Descripción | Condición | Icono |
|---|---|---|---|
| Primer Reporte | Tu primer reporte en Red-Nómada | Crear 1er reporte (lifetime) | `first_report` |
| Nómada Activo | Reportaste 5 lugares en un día | Completar misión "Reportá 5 lugares hoy" | `active_nomad` |
| Verificador | Validaste 3 reportes en un día | Completar misión "Validá 3 reportes hoy" | `verifier` |
| Explorador Pionero | Fuiste el primero en reportar un lugar | Completar misión "Ser el primero" | `pioneer` |
| 10 Reportes | Acumulaste 10 reportes | 10 reportes lifetime (trigger) | `ten_reports` |
| 50 Personas | Ayudaste a 50 personas | `personas_ayudadas_total >= 50` | `impact_50` |

- Insignias **no son revocables** (regla de negocio)
- Se otorgan una sola vez (UNIQUE en `usuario_id + nombre`)

---

## 5. Cálculo de Impacto Social

### Fórmula
- `personas_ayudadas` de un reporte = número de `ConsultaLugar` en ese lugar en las 2 horas posteriores al `Reporte.created_at`
- `personas_ayudadas_total` del usuario = Σ(personas_ayudadas de cada reporte del usuario)

### Registro de ConsultaLugar
- Cada vez que se ejecuta `GET /lugares/:id` → se inserta un `ConsultaLugar` con `lugar_id`, `usuario_id` (nullable para anónimos), `created_at`
- Para evitar inflar el conteo: un mismo `usuario_id` cuenta como 1 consulta por lugar por hora (deduplicación)

### Presentación del mensaje (post-reporte)
- Inmediatamente después de crear un reporte, el response incluye:
```json
{
  "reporte": { ... },
  "impacto": {
    "personas_ayudadas": 12,
    "mensaje": "Gracias a tu reporte, 12 personas evitaron ir a un lugar lleno"
  }
}
```
- El conteo es aproximado (no requiere precisión exacta)
- Si es el primer reporte de un lugar → el impacto se calcula después (0 por ahora, se actualiza async)

---

## 6. Lógica Backend

### Servicios necesarios
- `MisionService`: asignar misiones, actualizar progreso, detectar completitud, otorgar recompensas
- `ImpactoService`: registrar consultas, calcular impacto por reporte, totalizar por usuario
- `InsigniaService`: verificar condiciones de otorgamiento, crear insignias

### Eventos/Triggers para progreso
| Acción del usuario | Trigger de misión |
|---|---|
| `POST /lugares/:id/reportes` | Incrementar misiones REPORTAR, check PRIMERO_EN_REPORTAR |
| `POST /reportes/:id/validar` | Incrementar misiones VALIDAR |

### Performance
- `MisionProgreso`: índice compuesto `(usuario_id, fecha)` para query rápida
- Puntos totales: mantener en `Usuario.puntos_totales` actualizado (no recalcular)
- Impacto social: calcular async con Redis si la carga es alta; en MVP es síncrono

---

## 7. Reglas Anti-Abuso

- **Rate limit de reportes:** 1 reporte por lugar cada 30 minutos (ya existe en reglas_negocio.md)
- **Deduplicación de consultas:** un usuario cuenta como 1 consulta por lugar por hora
- **Validaciones propias:** no se puede validar propio reporte (ya existe)
- **Límite de misiones por día:** máximo 3 misiones activas simultáneamente por usuario
- **Detección futura (post-MVP):** alertar si un usuario crea > 10 reportes en 1 hora (posible bot)
