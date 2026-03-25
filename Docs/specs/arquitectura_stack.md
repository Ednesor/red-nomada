# Arquitectura y Stack Tecnológico: Red-Nómada MVP

## Visión General
App móvil nativa para profesionales itinerantes que evalúan espacios de trabajo según 3 pilares técnicos (Conectividad, Energía, Ambiente). Diseñada para funcionar con conectividad intermitente y bajo consumo de batería.

---

## Frontend (App Móvil Nativa)

- **Framework:** React Native 0.76+ con Expo SDK 52+
- **Styling:** NativeWind v4 (TailwindCSS para React Native)
- **State Management:** Zustand (ligero, sin boilerplate, compatible RN)
- **Navegación:** Expo Router (file-based routing)
- **Almacenamiento local:** expo-secure-store (tokens), AsyncStorage (cache general)
- **Mapas:** react-native-maps (MapView nativo iOS/Android)
- **Geolocalización:** expo-location (permisos granulares nativos)
- **Networking:** Axios con interceptors para auth
- **Offline:** @react-native-community/netinfo + AsyncStorage para cola de reportes pendientes

### Justificación React Native + Expo
- Acceso nativo a GPS, permisos, notificaciones locales
- Performance nativo en animaciones del Snapshot (< 5s)
- Un solo codebase para iOS + Android
- Expo simplifica builds, OTA updates, y gestión de permisos
- NativeWind permite reusar conocimiento de TailwindCSS

---

## Backend (API REST)

- **Runtime:** Java 21 LTS
- **Framework:** Spring Boot 3.x
- **Seguridad:** Spring Security 6.x (JWT filter chain)
- **ORM:** Spring Data JPA + Hibernate
- **Migraciones:** Flyway
- **Validación:** Bean Validation (Jakarta `@Valid`, `@NotNull`, `@Pattern`)
- **Autenticación:** JWT (access + refresh tokens) con `jjwt` (io.jsonwebtoken)
- **Rate Limiting:** Bucket4j o Spring Cloud Gateway rate limiter
- **Build:** Gradle 8.x con Kotlin DSL
- **Documentación API:** SpringDoc OpenAPI (Swagger UI)

### Justificación Java + Spring Boot
- Ecosistema maduro y robusto para APIs REST
- Spring Security integrado (JWT, roles, CORS, CSRF)
- Tipado fuerte reduce bugs en reglas de negocio complejas
- Hibernate + Flyway para migraciones controladas
- Alto rendimiento bajo carga con pool de threads virtual (Java 21)

---

## Base de Datos

- **Principal:** PostgreSQL 16 (relacional, datos estructurados de lugares y reportes)
- **Cache:** Redis 7 (TTL nativo para expiración de datos a 2 horas, contadores de impacto en tiempo real)

### Justificación PostgreSQL + Redis
- PostgreSQL: consultas geoespaciales con PostGIS, ACID para integridad de reportes
- Redis: TTL automático para marcar datos como obsoletos, contadores atómicos para gamificación

---

## Infraestructura y Despliegue

- **Containerización:** Docker + Docker Compose (dev local)
- **CI/CD:** GitHub Actions
- **Hosting (MVP):**
  - Backend: Railway o Render (free tier para MVP)
  - PostgreSQL: Neon (serverless PostgreSQL, free tier)
  - Redis: Upstash (serverless Redis, free tier)
- **App Distribution:**
  - Expo EAS Build (builds en la nube)
  - Expo EAS Submit (publicación App Store / Google Play)
  - Expo EAS Update (OTA updates sin rebuild)

---

## Estructura de Carpetas (Monorepo)

```
red-nomada/
├── Docs/
│   └── specs/                  # Documentación SDD
├── apps/
│   ├── mobile/                 # Frontend React Native + Expo
│   │   ├── app/                # Expo Router (file-based routes)
│   │   │   ├── (tabs)/         # Tab navigation (mapa, misiones, perfil)
│   │   │   ├── lugar/[id].tsx  # Detalle lugar + Snapshot
│   │   │   └── auth/           # Login, Registro
│   │   ├── components/         # Componentes UI reutilizables
│   │   ├── features/           # Módulos por feature (snapshot, verification, gamification)
│   │   ├── hooks/              # Custom hooks (useLocation, usePermissions)
│   │   ├── services/           # API client, auth, offline queue
│   │   ├── stores/             # Zustand stores
│   │   ├── utils/              # Helpers, constantes de pilares
│   │   └── assets/             # Iconos, fuentes, imágenes
│   └── api/                    # Backend Spring Boot
│       └── src/
│           ├── main/
│           │   ├── java/com/rednomada/
│           │   │   ├── config/         # SecurityConfig, CorsConfig, RedisConfig
│           │   │   ├── controller/     # REST Controllers
│           │   │   ├── service/        # Lógica de negocio
│           │   │   ├── repository/     # Spring Data JPA Repositories
│           │   │   ├── model/          # Entidades JPA (@Entity)
│           │   │   ├── dto/            # DTOs de request/response
│           │   │   ├── security/       # JWT filter, AuthEntryPoint
│           │   │   ├── exception/      # GlobalExceptionHandler, custom exceptions
│           │   │   └── util/           # Helpers, constantes
│           │   └── resources/
│           │       ├── application.yml # Configuración Spring
│           │       └── db/migration/   # Scripts Flyway
│           └── test/                   # Tests unitarios e integración
```

---

## Decisiones Clave

| Decisión | Elección | Alternativa descartada | Razón |
|---|---|---|---|
| App type | React Native + Expo | PWA | Acceso nativo a GPS, permisos, mejor UX móvil |
| Styling | NativeWind | StyleSheet puro | Productividad, consistencia con Tailwind |
| Backend | Java/Spring Boot | Node.js/Express | Tipado fuerte, Spring Security integrado, robustez |
| ORM | JPA/Hibernate | JDBC puro | Productividad, migraciones con Flyway |
| DB | PostgreSQL | MongoDB | Datos relacionales, PostGIS, integridad |
| Cache/TTL | Redis | In-memory | TTL nativo para expiración de 2h |
| Mapas | react-native-maps | Leaflet | Componente nativo para RN, mejor performance |
| State | Zustand | Redux | Menos boilerplate, más simple para MVP |
| Navegación | Expo Router | React Navigation directo | File-based routing, deep linking automático |
