# Plan de Implementación — HU1: "The Snapshot"

## Resumen Ejecutivo

**Objetivo:** Implementar el componente central de Red-Nómada — el "Snapshot" técnico que permite decidir en < 5 segundos si un espacio sirve, mostrando únicamente 3 pilares con valores fijos (Conectividad, Energía, Ambiente). Sin puntuaciones, sin estrellas.

**Alcance:** Backend (Java/Spring Boot) + Frontend (React Native/Exopo) para el flujo base de visualización del Snapshot desde el mapa.

**Dependencias externas de la HU1:** Ninguna (es la HU base, todo depende de ella).

---

## FASE 0: Bootstrap del Monorepo

**Objetivo:** Crear la estructura base del proyecto monorepo según `arquitectura_stack.md`.

### Archivos a crear:

```
red-nomada/
├── apps/
│   ├── mobile/              ← (se crea en Fase 2)
│   └── api/
│       ├── build.gradle.kts
│       ├── settings.gradle.kts
│       └── src/
│           ├── main/
│           │   ├── java/com/rednomada/
│           │   │   └── RedNomadaApplication.java
│           │   └── resources/
│           │       └── application.yml
│           └── test/
├── docker-compose.yml
├── .gitignore
└── README.md (opcional, solo si se pide)
```

### Dependencias a instalar (Backend):

- **Gradle** con Kotlin DSL
- **Spring Boot 3.x** starter: `spring-boot-starter-web`
- **Spring Boot 3.x** starter: `spring-boot-starter-data-jpa`
- **Spring Boot 3.x** starter: `spring-boot-starter-validation`
- **PostgreSQL driver:** `org.postgresql:postgresql`
- **Flyway:** `org.flywaydb:flyway-core`
- **Lombok** (opcional pero recomendado): `org.projectlombok:lombok`

### docker-compose.yml:

- PostgreSQL 16 (puerto 5432)
- Redis 7 (puerto 6379)
- Volúmenes persistentes para datos

---

## FASE 1: Backend — Modelo de Datos + API del Snapshot

**Objetivo:** Entidades JPA, migraciones Flyway y endpoints que sirven los datos del Snapshot.

### Paso 1.1: Entidades JPA

Archivos a crear:

```
apps/api/src/main/java/com/rednomada/
├── model/
│   ├── Lugar.java          ← @Entity, campos: id(UUID), nombre, direccion, latitud(DECIMAL), longitud(DECIMAL), categoria(enum), activo(boolean), created_at, updated_at
│   ├── Reporte.java        ← @Entity, campos: id(UUID), usuario_id(FK), lugar_id(FK), conectividad(enum), energia(enum), ambiente(enum), created_at
│   └── enums/
│       ├── CategoriaLugar.java      ← CAFE, COWORKING, BIBLIOTECA, OTRO
│       ├── NivelConectividad.java   ← BAJA, MEDIA, ALTA
│       ├── NivelEnergia.java        ← POCOS, SUFICIENTES, MUCHOS
│       └── NivelAmbiente.java       ← SILENCIOSO, MODERADO, RUIDOSO
```

**Reglas clave en Reporte:**

- Índice compuesto `(lugar_id, created_at DESC)` para query rápida del reporte más reciente
- Restricción: 1 reporte por usuario por lugar cada 30 minutos (se valida en service layer)

### Paso 1.2: Migraciones Flyway

Archivos a crear:

```
apps/api/src/main/resources/db/migration/
├── V1__create_lugar.sql
├── V2__create_reporte.sql
├── V3__create_indexes.sql
└── V4__seed_lugares.sql       ← datos semilla de lugares MVP (CABA)
```

**Contenido de V4:** Insertar ~10-15 cafés/coworkings de Buenos Aires con coordenadas reales para pruebas.

### Paso 1.3: Repositories

Archivos a crear:

```
apps/api/src/main/java/com/rednomada/
└── repository/
    ├── LugarRepository.java     ← findByActivoTrue() + query geoespacial (PostGIS o lat/lng bounding box)
    └── ReporteRepository.java   ← findLatestByLugarId() + countByLugarIdAndCreatedAtAfter()
```

### Paso 1.4: DTOs del Snapshot

Archivos a crear:

```
apps/api/src/main/java/com/rednomada/
└── dto/
    ├── SnapshotDTO.java         ← conectividad, energia, ambiente, es_fresco(boolean), ultima_actualizacion, necesita_verificacion
    ├── LugarConSnapshotDTO.java ← Lugar + snapshot + distancia_metros
    └── LugarRequestParams.java  ← lat, lng, radio, categoria (query params)
```

**Lógica de frescura:**

- `es_fresco = (NOW() - reporte.created_at) < 2 horas`
- `necesita_verificacion = !es_fresco`
- Si no hay reportes → `snapshot = null`, el frontend muestra "Sin datos"

### Paso 1.5: Service Layer

Archivos a crear:

```
apps/api/src/main/java/com/rednomada/
└── service/
    ├── LugarService.java        ← listarCercanos(lat, lng, radio, categoria) → List<LugarConSnapshotDTO>
    │                               getDetalleConSnapshot(lugarId) → LugarConSnapshotDTO
    │                               calcularDistancia() (Haversine o PostGIS)
    └── SnapshotService.java     ← getSnapshot(lugarId) → SnapshotDTO
                                    calcularFrescura(reporte) → boolean
```

### Paso 1.6: Controllers REST

Archivos a crear:

```
apps/api/src/main/java/com/rednomada/
└── controller/
    └── LugarController.java
```

**Endpoints para HU1:**

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| `GET` | `/api/v1/lugares` | 🔓 Pública | Listar lugares cercanos con snapshot |
| `GET` | `/api/v1/lugares/:id` | 🔓 Pública | Detalle lugar + snapshot |

**Query params para `GET /lugares`:** `lat` (required), `lng` (required), `radio` (default 1000m), `categoria` (optional)

### Paso 1.7: Configuración

Archivos a crear/modificar:

```
apps/api/src/main/java/com/rednomada/
└── config/
    ├── CorsConfig.java          ← permitir requests del frontend Expo
    └── WebConfig.java           ← configuración adicional de web

apps/api/src/main/resources/
└── application.yml              ← datasource PostgreSQL, JPA config, Flyway enabled, server port 8080
```

---

## FASE 2: Frontend — Expo + Mapa + Snapshot

**Objetivo:** App React Native que muestra el mapa con marcadores de lugares y permite ver el Snapshot en < 5 segundos.

### Paso 2.1: Inicializar proyecto Expo

Comando a ejecutar:

```bash
npx create-expo-app@latest apps/mobile --template blank-typescript
```

### Dependencias a instalar (Frontend):

```
cd apps/mobile
npx expo install expo-router expo-location react-native-maps expo-secure-store @react-native-community/netinfo
npm install zustand axios nativewind tailwindcss
```

### Archivos de configuración a crear:

```
apps/mobile/
├── app.json                    ← configuración de Expo (Google Maps API key para Android)
├── tailwind.config.js          ← configuración NativeWind
├── babel.config.js             ← plugin nativewind/babel
├── metro.config.js             ← resolver de NativeWind
├── tsconfig.json               ← paths, strict mode
└── global.css                  ← imports de Tailwind
```

### Paso 2.2: Estructura de rutas (Expo Router)

Archivos a crear:

```
apps/mobile/app/
├── _layout.tsx                 ← root layout: providers, font loading
├── index.tsx                   ← redirect → /(tabs)/mapa
├── (tabs)/
│   ├── _layout.tsx             ← tab navigator: Mapa (placeholder), Misiones (placeholder), Perfil (placeholder)
│   └── mapa.tsx                ← PANTALLA PRINCIPAL DEL SNAPSHOT
└── lugar/
    └── [id].tsx                ← Detalle del lugar con snapshot completo
```

**Nota:** Las tabs de Misiones y Perfil son placeholders vacíos para HU1. Se implementan en HU3.

### Paso 2.3: Servicio API Client

Archivos a crear:

```
apps/mobile/services/
├── api.ts                      ← instancia Axios con baseURL, interceptor de auth (placeholder), timeout
└── lugares.ts                  ← getLugaresCercanos(lat, lng, radio?, categoria?)
                                   getLugarDetalle(id)
```

### Paso 2.4: Store de Zustand

Archivos a crear:

```
apps/mobile/stores/
├── lugarStore.ts               ← lugares[], lugarSeleccionado, fetchLugares(), filtros
└── authStore.ts                ← placeholder para HU2 (token, usuario)
```

### Paso 2.5: Componentes del Snapshot

Archivos a crear:

```
apps/mobile/components/
├── MapaConMarcadores.tsx       ← MapView + markers por categoría + indicador de frescura
├── BottomSheetLugar.tsx        ← preview del lugar al tap: nombre, dirección, 3 iconos de pilares, frescura, botones
├── PillarIcon.tsx              ← componente reutilizable: icono + etiqueta del pilar (ej: "📶 Alta")
├── FiltroBarra.tsx             ← barra de filtros horizontal (chips scrollables) — categoría + pilares
└── MarcadorLugar.tsx           ← marker custom por categoría (React.memo)
```

**PillarIcon — El componente CRÍTICO de la HU1:**

- 3 variantes: Conectividad (icono wifi/señal), Energía (icono batería/⚡), Ambiente (icono sonido/🔇)
- Solo muestra el valor textual: "Alta", "Suficientes", "Silencioso"
- SIN puntuación numérica, SIN estrellas, SIN colores de "bueno/malo"
- Diseño optimizado para scannear en < 2 segundos

### Paso 2.6: Geolocalización

Archivos a crear:

```
apps/mobile/hooks/
├── useLocation.ts              ← wrapper de expo-location: pedir permisos (siempre/una vez/solo en uso), obtener posición one-shot, fallback a default
└── useNetInfo.ts               ← wrapper de @react-native-community/netinfo
```

### Paso 2.7: Utils y Constantes

Archivos a crear:

```
apps/mobile/utils/
├── pilares.ts                  ← constantes: CONECTIVIDAD_LABELS, ENERGIA_LABELS, AMBIENTE_LABELS
├── frescura.ts                 ← esFresco(timestamp), tiempoDesde(timestamp), necesitaVerificacion(timestamp)
└── distancias.ts               ← calcularDistancia(lat1, lng1, lat2, lng2) — Haversine
```

---

## FASE 3: Integración y Validación

**Objetivo:** Conectar frontend con backend y validar que el Snapshot cumple los criterios de aceptación.

### Paso 3.1: Variables de entorno

```
apps/mobile/.env
EXPO_PUBLIC_API_URL=http://localhost:8080/api/v1
```

### Paso 3.2: Probar flujo completo

1. Levantar backend con Docker Compose (PostgreSQL + Spring Boot)
2. Ejecutar migraciones Flyway (seed de lugares)
3. Levantar Expo
4. Verificar: mapa carga con marcadores de lugares seed
5. Verificar: tap en marcador → bottom sheet con Snapshot (3 iconos, valores)
6. Verificar: "Ver detalle" → pantalla `/lugar/[id]` con Snapshot completo
7. Verificar: Sin puntuaciones visibles en ningún lado
8. Verificar: filtro por categoría funciona

---

## Orden Lógico de Ejecución

```
Fase 0 (Bootstrap)     →  Fase 1 (Backend)           →  Fase 2 (Frontend)        →  Fase 3 (Integración)
├─ docker-compose       ├─ Entidades + Enums           ├─ Expo init + deps         ├─ .env config
├─ Gradle + Spring init ├─ Migraciones Flyway          ├─ Rutas Expo Router        ├─ Test manual full flow
├─ application.yml      ├─ Repositories                ├─ API client               └─ Validación CA
└─ project structure    ├─ DTOs + Services             ├─ Zustand store
                        ├─ Controllers                 ├─ Componentes Snapshot
                        └─ CorsConfig                  ├─ Geolocalización
                                                       └─ Utils
```

**Dependencia crítica:** Fase 1 debe completarse antes de Fase 2 (el frontend necesita la API para funcionar).

---

## Resumen de Archivos

| Fase | Archivos nuevos | Archivos modificados |
|------|----------------|---------------------|
| 0 - Bootstrap | 6 | 0 |
| 1 - Backend | ~15 | 0 |
| 2 - Frontend | ~20 | 0 |
| 3 - Integración | 1 | 0 |
| **Total** | **~42** | **0** |

---

## Librerías Exactas

### Backend (Gradle dependencies):

- `org.springframework.boot:spring-boot-starter-web:3.4.3`
- `org.springframework.boot:spring-boot-starter-data-jpa:3.4.3`
- `org.springframework.boot:spring-boot-starter-validation:3.4.3`
- `org.postgresql:postgresql:42.7.3`
- `org.flywaydb:flyway-core:10.12.0`
- `org.projectlombok:lombok:1.18.32`

### Frontend (npm dependencies):

- `expo@~52.0.0`, `expo-router`, `expo-location`, `expo-secure-store`
- `react-native-maps`
- `@react-native-community/netinfo`
- `zustand`, `axios`, `nativewind`, `tailwindcss`
- TypeScript incluido en template

---

## Riesgos Identificados

| Riesgo | Impacto | Mitigación |
|--------|---------|------------|
| PostGIS no disponible en Neon free tier | Medio | Usar bounding box con lat/lng en queries nativas (Haversine en service layer) — funcional para MVP |
| Google Maps API key para Android | Bajo | Usar Apple Maps en iOS (nativo), pedir key de Google solo si se necesita Android |
| Seed de lugares insuficiente | Bajo | Incluir 10-15 lugares reales de CABA en la migración V4 |

---

## Criterios de Aceptación Validables

- ✅ No hay puntuaciones generales en ningún componente
- ✅ 3 iconos de pilares visibles con valores fijos (Baja/Media/Alta, etc.)
- ✅ Snapshot visible en bottom sheet al tap de marcador
- ✅ Snapshot visible en pantalla de detalle `/lugar/[id]`
- ✅ Indicador de frescura en marcadores del mapa
- ✅ Filtros por categoría funcionales
