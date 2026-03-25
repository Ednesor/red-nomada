# Modelo de Datos: Red-Nómada MVP

## Diagrama de Relaciones

```
Usuario 1──N Reporte N──1 Lugar
Usuario 1──N Validacion N──1 Reporte
Usuario 1──N MisionProgreso N──1 Mision
Usuario 1──N Insignia
Lugar 1──N Reporte
Lugar 1──N ConsultaLugar
```

---

## Entidades

### Usuario
| Campo | Tipo | Restricción |
|---|---|---|
| id | UUID | PK |
| email | VARCHAR(255) | UNIQUE, NOT NULL |
| nombre | VARCHAR(100) | NOT NULL |
| password_hash | VARCHAR(255) | NOT NULL |
| rol | ENUM(`EXPLORADOR`, `ADMIN`) | DEFAULT `EXPLORADOR` |
| puntos_totales | INT | DEFAULT 0 |
| permiso_ubicacion | ENUM(`SIEMPRE`, `UNA_VEZ`, `SOLO_EN_USO`, `NINGUNO`) | DEFAULT `NINGUNO` |
| created_at | TIMESTAMP | NOT NULL |
| updated_at | TIMESTAMP | NOT NULL |

---

### Lugar
| Campo | Tipo | Restricción |
|---|---|---|
| id | UUID | PK |
| nombre | VARCHAR(200) | NOT NULL |
| direccion | VARCHAR(500) | NOT NULL |
| latitud | DECIMAL(10,8) | NOT NULL |
| longitud | DECIMAL(11,8) | NOT NULL |
| categoria | ENUM(`CAFE`, `COWORKING`, `BIBLIOTECA`, `OTRO`) | NOT NULL |
| activo | BOOLEAN | DEFAULT true |
| created_at | TIMESTAMP | NOT NULL |
| updated_at | TIMESTAMP | NOT NULL |

---

### Reporte
| Campo | Tipo | Restricción |
|---|---|---|
| id | UUID | PK |
| usuario_id | UUID | FK → Usuario, NOT NULL |
| lugar_id | UUID | FK → Lugar, NOT NULL |
| conectividad | ENUM(`BAJA`, `MEDIA`, `ALTA`) | NOT NULL |
| energia | ENUM(`POCOS`, `SUFICIENTES`, `MUCHOS`) | NOT NULL |
| ambiente | ENUM(`SILENCIOSO`, `MODERADO`, `RUIDOSO`) | NOT NULL |
| es_fresco | BOOLEAN | Calculado: `created_at > NOW() - 2h` |
| created_at | TIMESTAMP | NOT NULL |

> **Índice compuesto:** `(lugar_id, created_at DESC)` para obtener rápidamente el reporte más reciente de un lugar.

> **Restricción:** Un usuario no puede crear más de 1 reporte por lugar en 30 minutos.

---

### Validacion
| Campo | Tipo | Restricción |
|---|---|---|
| id | UUID | PK |
| reporte_id | UUID | FK → Reporte, NOT NULL |
| usuario_id | UUID | FK → Usuario, NOT NULL |
| created_at | TIMESTAMP | NOT NULL |

> **Restricción:** `usuario_id ≠ Reporte.usuario_id` (no puede validar su propio reporte).

> **Efecto:** Al crear una validación, se extiende la frescura del reporte por 2 horas desde `Validacion.created_at`.

---

### Mision
| Campo | Tipo | Restricción |
|---|---|---|
| id | UUID | PK |
| titulo | VARCHAR(200) | NOT NULL |
| descripcion | TEXT | NOT NULL |
| tipo | ENUM(`REPORTAR`, `VALIDAR`, `PRIMERO_EN_REPORTAR`) | NOT NULL |
| objetivo_cantidad | INT | NOT NULL (ej: 3 para "Reportá 3 cafés") |
| puntos_recompensa | INT | NOT NULL |
| activa | BOOLEAN | DEFAULT true |

---

### MisionProgreso
| Campo | Tipo | Restricción |
|---|---|---|
| id | UUID | PK |
| usuario_id | UUID | FK → Usuario, NOT NULL |
| mision_id | UUID | FK → Mision, NOT NULL |
| progreso_actual | INT | DEFAULT 0 |
| completada | BOOLEAN | DEFAULT false |
| fecha | DATE | NOT NULL (día en que aplica la misión) |
| created_at | TIMESTAMP | NOT NULL |

> **Índice único:** `(usuario_id, mision_id, fecha)` — una misión por usuario por día.

---

### Insignia
| Campo | Tipo | Restricción |
|---|---|---|
| id | UUID | PK |
| usuario_id | UUID | FK → Usuario, NOT NULL |
| nombre | VARCHAR(100) | NOT NULL |
| descripcion | VARCHAR(500) | |
| icono | VARCHAR(100) | Nombre del icono/asset |
| otorgada_at | TIMESTAMP | NOT NULL |

> Las insignias **no son revocables**.

---

### ConsultaLugar
| Campo | Tipo | Restricción |
|---|---|---|
| id | UUID | PK |
| lugar_id | UUID | FK → Lugar, NOT NULL |
| usuario_id | UUID | FK → Usuario (nullable, visitantes anónimos) |
| created_at | TIMESTAMP | NOT NULL |

> Se usa para calcular la **métrica de impacto social** ("X personas consultaron este lugar").

---

## Notas de Implementación

- **Frescura dinámica:** `es_fresco` no se almacena como campo real. Se calcula en queries comparando `created_at` (o la última `Validacion.created_at` asociada) contra `NOW() - INTERVAL '2 hours'`
- **PostGIS:** Usar extensión para consultas de proximidad (`ST_DWithin` para encontrar lugares cercanos al usuario)
- **Soft delete:** Los lugares se desactivan (`activo = false`), nunca se eliminan
