# Arquitectura y Stack Tecnológico: Red-Nómada MVP

## Visión General
App mobile-first para profesionales itinerantes que evalúan espacios de trabajo según 3 pilares técnicos (Conectividad, Energía, Ambiente). Diseñada para funcionar con conectividad intermitente y bajo consumo de batería.

---

## Frontend (Mobile-First PWA)

- **Framework:** React 18+ con Vite
- **Styling:** TailwindCSS 3.x (utility-first, rápido para prototipos y consistente con diseño mobile)
- **State Management:** Zustand (ligero, sin boilerplate)
- **Routing:** React Router v6
- **PWA:** Workbox (Service Workers para cache offline)
- **Mapas:** Leaflet + OpenStreetMap (gratuito, sin dependencia de API keys de Google)
- **Geolocalización:** API nativa del navegador (`navigator.geolocation`) con permisos granulares

### Justificación PWA
- Instalable en dispositivos sin pasar por App Store
- Cache offline nativo vía Service Workers
- Menor consumo de batería vs. apps nativas con GPS en background
- Un solo codebase para web + mobile

---

## Backend (API REST)

- **Runtime:** Node.js 20 LTS
- **Framework:** Express.js 4.x
- **ORM:** Prisma (type-safe, migraciones automáticas)
- **Validación:** Zod (schemas compartidos con frontend)
- **Autenticación:** JWT (access + refresh tokens)
- **Rate Limiting:** express-rate-limit

### Justificación Node.js
- Ecosistema JavaScript unificado (frontend + backend)
- Excelente para I/O asíncrono (reportes en tiempo real)
- Comunidad amplia, iteración rápida para MVP

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
  - Frontend: Vercel (CDN global, gratis para PWA)
  - Backend: Railway o Render (free tier para MVP)
  - PostgreSQL: Neon (serverless PostgreSQL, free tier)
  - Redis: Upstash (serverless Redis, free tier)

---

## Estructura de Carpetas (Monorepo)

```
red-nomada/
├── Docs/
│   └── specs/              # Documentación SDD
├── apps/
│   ├── web/                # Frontend React PWA
│   │   ├── src/
│   │   │   ├── components/ # Componentes UI reutilizables
│   │   │   ├── features/   # Módulos por feature (snapshot, verification, gamification)
│   │   │   ├── hooks/      # Custom hooks (geolocation, permissions)
│   │   │   ├── services/   # API client, cache, offline
│   │   │   ├── stores/     # Zustand stores
│   │   │   └── utils/      # Helpers, constantes de pilares
│   │   └── public/
│   └── api/                # Backend Express
│       ├── src/
│       │   ├── routes/     # Endpoints REST
│       │   ├── controllers/
│       │   ├── services/   # Lógica de negocio
│       │   ├── middleware/ # Auth, rate-limit, validation
│       │   ├── prisma/     # Schema y migraciones
│       │   └── utils/      # Helpers, constantes
│       └── tests/
└── packages/
    └── shared/             # Tipos y schemas Zod compartidos
```

---

## Decisiones Clave

| Decisión | Elección | Alternativa descartada | Razón |
|---|---|---|---|
| App type | PWA | React Native | Un codebase, offline nativo, sin App Store |
| Backend | Node.js/Express | Python/FastAPI | JS unificado, velocidad de iteración |
| DB | PostgreSQL | MongoDB | Datos relacionales, PostGIS, integridad |
| Cache/TTL | Redis | In-memory | TTL nativo para expiración de 2h |
| Mapas | Leaflet/OSM | Google Maps | Gratuito, sin API key, offline tiles |
| State | Zustand | Redux | Menos boilerplate, más simple para MVP |
