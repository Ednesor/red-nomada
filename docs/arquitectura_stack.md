# Arquitectura y Stack Tecnológico: Red-Nómada

## 🎯 Principios de Selección

Todas las decisiones tecnológicas se basan en:
1. **The Snapshot (<5s):** Renderizado rápido desde cache local
2. **Autonomía Local:** Funciona offline con SQLite + sincronización eventual
3. **Conectividad Intermitente:** No asume internet constante
4. **Batería Limitada:** Sin GPS background, validación solo en foreground
5. **MVP Rápido:** Stack unificado (JavaScript/TypeScript)

---

## 📱 FRONTEND: Mobile-First (React Native)

### Justificación
- **Nómadas digitales:** Uso mayoritario desde móviles en cafés/coworking
- **Offline-first:** SQLite local para cache, funciona sin conexión
- **Validación pasiva:** WebSocket para alertas cuando app está abierta
- **Bajo consumo:** No requiere background services

### Tecnologías Exactas

| Componente | Tecnología | Versión | Justificación |
|---|---|---|---|
| **Lenguaje** | TypeScript | 5.3+ | Tipado estricto, error prevention |
| **Framework Mobile** | React Native | 0.73+ | iOS/Android unificado, comunidad activa |
| **Bundler** | Expo** | 50.0+ | Simplifica build & deployment (aún permite eject si crece) |
| **State Manager** | Redux Toolkit | 1.9+ | Cache local persistente, offline sync |
| **Local DB** | SQLite (expo-sqlite) | 12.0+ | Relativamente persistencia local, queries rápidas |
| **HTTP Client** | Axios + Retry | 1.6+ | Request queue offline, reintento automático |
| **WebSocket** | Socket.io-client | 4.5+ | Real-time validations pasivas, fallback HTTP |
| **Location** | Expo Location | 16.0+ | Gestión granular de permisos (always/while/never) |
| **UI Library** | React Native Paper | 5.11+ | Material Design, componentes accesibles |
| **Sync Engine** | WatermelonDB | 0.28+ | Local-first sync, offline + online reconciliation |

### Estructura de Directorios

```
red-nomada-mobile/
├── app/
│   ├── screens/
│   │   ├── PlacesScreen.tsx       (Historia 1: Snapshot)
│   │   ├── PlaceDetailModal.tsx   (The 3 pillars)
│   │   ├── VerificationModal.tsx  (Historia 2: 1-tap validation)
│   │   ├── MissionsScreen.tsx     (Historia 3: Gamification)
│   │   └── PermissionsScreen.tsx  (Location perms: always/while/never)
│   ├── components/
│   │   ├── PillarIcon.tsx         (Connectivity/Energy/Environment)
│   │   ├── UrgentBanner.tsx       ("Este lugar necesita verificación AHORA")
│   │   ├── ImpactMessage.tsx      ("Gracias, 12 personas evitaron...")
│   │   └── LocationPermissionModal.tsx
│   ├── store/
│   │   ├── slices/
│   │   │   ├── placesSlice.ts
│   │   │   ├── validationsSlice.ts
│   │   │   ├── missionsSlice.ts
│   │   │   └── userSlice.ts
│   │   └── store.ts
│   ├── db/
│   │   ├── schema.ts              (WatermelonDB schema)
│   │   └── sync.ts                (Sync logic)
│   ├── services/
│   │   ├── api.ts                 (Axios + queue)
│   │   ├── location.ts            (Expo Location manager)
│   │   ├── socket.ts              (Socket.io-client instance)
│   │   └── permissions.ts
│   └── hooks/
│       ├── useNearbyPlaces.ts
│       ├── usePlaceSnapshot.ts
│       ├── useValidationTrigger.ts
│       └── useImpactMessage.ts
├── app.json                        (Expo config)
├── package.json
└── tsconfig.json
```

---

## 🖥️ BACKEND: Node.js + Express

### Justificación
- **Stack unificado:** JavaScript/TypeScript frontend + backend
- **Escalabilidad:** Express + Bull para cron jobs (distribuir validaciones)
- **Real-time:** Socket.io para WebSocket (validaciones pasivas)
- **Rápido MVP:** Excelente ecosistema npm

### Tecnologías Exactas

| Componente | Tecnología | Versión | Justificación |
|---|---|---|---|
| **Lenguaje** | TypeScript | 5.3+ | Type safety |
| **Runtime** | Node.js | 20 LTS | Soporte largo plazo |
| **Framework** | Express | 4.18+ | Minimalista, rápido |
| **Base de Datos** | PostgreSQL | 15+ | Relaciones complejas (Users, Validations, Missions, UserImpact) |
| **ORM** | Prisma | 5.7+ | Type-safe queries, migrations automáticas |
| **Real-time** | Socket.io | 4.5+ | WebSocket + fallback, broadcast validations |
| **Job Queue** | Bull | 4.11+ | Cron: detectar >2h obsoletos, distribuir validaciones |
| **Authentication** | JWT (jsonwebtoken) | 9.1+ | Stateless, scalable |
| **Validation** | Zod | 3.22+ | Runtime schema validation |
| **Logging** | Pino | 8.16+ | Structured logs, performance |
| **Testing** | Jest | 29.7+ | Unit + integration tests |

### Estructura de Directorios

```
red-nomada-api/
├── src/
│   ├── controllers/
│   │   ├── placesController.ts    (GET /places/:id, POST /places, GET /places/nearby)
│   │   ├── validationsController.ts (POST /validations, GET /places/:id/status)
│   │   ├── permissionsController.ts (POST/GET /permissions)
│   │   ├── missionsController.ts  (GET, POST)
│   │   └── usersController.ts
│   ├── services/
│   │   ├── placeService.ts
│   │   ├── validationService.ts   (Calcular people_saved)
│   │   ├── missionService.ts
│   │   ├── impactService.ts
│   │   └── locationService.ts
│   ├── routes/
│   │   ├── places.ts
│   │   ├── validations.ts
│   │   ├── permissions.ts
│   │   ├── missions.ts
│   │   └── users.ts
│   ├── middleware/
│   │   ├── auth.ts                (JWT verification)
│   │   ├── errorHandler.ts
│   │   └── validateInput.ts       (Zod schemas)
│   ├── db/
│   │   ├── prisma.client.ts       (Singleton instance)
│   │   └── migrations/            (Prisma migrations)
│   ├── jobs/
│   │   ├── checkObsoleteData.ts   (Cron: >2h without validation)
│   │   ├── distributionValidations.ts (Round-robin: max 3-5/user/day)
│   │   └── queue.ts               (Bull instance)
│   ├── socket/
│   │   ├── handlers.ts            (Socket.io event handlers)
│   │   └── events.ts              (Event definitions)
│   ├── schemas/
│   │   ├── places.ts              (Zod validation schemas)
│   │   ├── validations.ts
│   │   ├── permissions.ts
│   │   └── missions.ts
│   ├── types/
│   │   └── index.ts               (TypeScript types)
│   ├── utils/
│   │   ├── jwt.ts
│   │   ├── geolocation.ts         (Distance calculation)
│   │   └── cache.ts               (In-memory cache layer)
│   ├── app.ts                     (Express setup)
│   └── index.ts                   (Server entry point)
├── prisma/
│   ├── schema.prisma              (Data model)
│   └── migrations/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── .env.example
├── docker-compose.yml             (PostgreSQL local dev)
├── package.json
├── tsconfig.json
└── jest.config.js
```

---

## 🗄️ BASE DE DATOS: PostgreSQL

### Justificación
- **Relaciones complejas:** Foreign keys (Users → Validations, Places → UserImpact)
- **Escalabilidad:** Indices GIS para búsqueda geo (nearby places)
- **ACID:** Transacciones para inconsistencias (ej: impact calculation)
- **Replicación:** Backup automático (importante para datos comunitarios)

### Esquema (Prisma)

```prisma
// User Location Permission Enum
enum LocationPermission {
  ALWAYS
  WHILE_USING
  NEVER
}

// User
model User {
  id          String   @id @default(cuid())
  username    String   @unique
  email       String   @unique
  
  locationPermission LocationPermission @default(WHILE_USING)
  points      Int      @default(0)
  
  // Relations
  validations Validation[]
  missions    Mission[]
  userImpacts UserImpact[]
  createdPlaces Place[] @relation("CreatedBy")
  
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

// Place
model Place {
  id          String   @id @default(cuid())
  name        String
  
  // Geolocation
  latitude    Decimal  @db.Decimal(10, 8)
  longitude   Decimal  @db.Decimal(10, 8)
  
  // Los 3 pilares técnicos (Story 1)
  connectivity String  // 'baja' | 'media' | 'alta'
  energy       String  // 'pocos' | 'suficientes' | 'muchos'
  environment  String  // 'silencioso' | 'moderado' | 'ruidoso'
  
  // Story 2: Urgencia (>2 horas sin validación)
  lastVerified DateTime?
  verificationCount Int @default(0)
  
  createdBy   String
  createdByUser User @relation("CreatedBy", fields: [createdBy], references: [id])
  
  // Relations
  validations Validation[]
  userImpacts UserImpact[]
  
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@index([latitude, longitude])  // GIS: búsqueda nearby
  @@index([lastVerified])         // Detectar >2h obsoletos
}

// Validation (Story 2 & 3)
model Validation {
  id          String   @id @default(cuid())
  
  placeId     String
  place       Place    @relation(fields: [placeId], references: [id], onDelete: Cascade)
  
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  // Valores confirmados
  connectivity String
  energy       String
  environment  String
  
  // Relation: Impact
  userImpact  UserImpact?
  
  createdAt   DateTime @default(now())
  
  @@index([placeId, createdAt])
  @@index([userId, createdAt])
}

// Mission (Story 3)
model Mission {
  id          String   @id @default(cuid())
  
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  title       String   // "Reportá 3 cafés hoy"
  description String?
  target      Int      // Meta numérica
  current     Int      @default(0)
  completed   Boolean  @default(false)
  
  rewardPoints Int
  
  createdAt   DateTime @default(now())
  expiresAt   DateTime
  
  @@index([userId, expiresAt])
  @@index([completed])
}

// UserImpact (Story 3: "12 personas evitaron...")
model UserImpact {
  id          String   @id @default(cuid())
  
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  validationId String @unique
  validation  Validation @relation(fields: [validationId], references: [id], onDelete: Cascade)
  
  // Snapshot: cuántos consultaron este lugar en 24h
  peopleSaved Int
  
  createdAt   DateTime @default(now())
  
  @@index([userId, createdAt])
  @@index([validationId])
}
```

---

## 🚀 DEPLOYMENTS

### Development

```bash
# Backend local
docker-compose up  # PostgreSQL + Redis (para Bull job queue)
npm run dev       # Express server http://localhost:3000

# Mobile local
expo start        # QR para escanear con Expo Go
```

### Production

| Componente | Plataforma | Justificación |
|---|---|---|
| **Backend API** | Render / Railway | Node.js + PostgreSQL manejado, auto-scaling |
| **Base de Datos** | PostgreSQL (Render/Railway) | Backups automáticos, replicación |
| **Socket.io Scaling** | Adapter Redis (ioredis) | Broadcast entre múltiples servidores |
| **Job Queue** | Bull + Node (en mismo server) | Cron jobs para obsolescence + distribution |
| **Mobile App** | Expo EAS Build | CI/CD para iOS/Android, signed builds |
| **App Distribution** | App Store / Google Play | Distribución oficial |

### Configuración de Deployments (Resumen)

```yaml
# Backend (Render)
- Runtime: Node.js 20
- Build: npm install && npm run build
- Start: npm run start
- Env Vars: DATABASE_URL, JWT_SECRET, SOCKET_IO_ORIGIN

# Mobile (Expo EAS)
- Preview: eas build --platform all --profile preview
- Production: eas build --platform all --profile production
- Submission: eas submit --platform all
```

---

## 🔌 Integraciones Clave

### Socket.io (Validación Pasiva - Story 2)

```typescript
// Backend: emit cuando app está en foreground + >2h sin validación
io.to(`user:${userId}`).emit('validation:needed', {
  placeId: uuid,
  urgent: true,
  message: 'Este lugar necesita verificación AHORA'
});

// Mobile: escucha solo cuando app está abierta (AppState listener)
```

### Bull Job Queue (Detectar Obsoletos + Distribuir)

```typescript
// Cada 30 minutos: detectar places >2 horas
const checkObsoleteJob = new CronExpr('0 */30 * * * *', async () => {
  const obsolete = await prisma.place.findMany({
    where: {
      lastVerified: {
        lt: new Date(Date.now() - 2 * 60 * 60 * 1000) // >2h
      }
    }
  });
  
  // Distribuir entre usuarios con permission=ALWAYS (equidad)
  const usersToValidate = selectRoundRobin(obsolete.length);
  
  for (const { place, user } of usersToValidate) {
    io.to(`user:${user.id}`).emit('validation:needed', {...});
  }
});
```

### Cálculo de Impacto (peopleSaved)

```typescript
// Inmediatamente después de POST /validations/:placeId
const peopleSaved = await prisma.validation.count({
  where: {
    placeId: placeId,
    createdAt: {
      gte: new Date(Date.now() - 24 * 60 * 60 * 1000)
    }
  }
});

// Retornar al frontend + guardar en user_impact
return {
  success: true,
  peopleSaved: peopleSaved,
  message: `Gracias a tu reporte, ${peopleSaved} personas evitaron ir a un lugar lleno`
};
```

---

## ✅ Validación contra Restricciones

| Restricción | Solución | Tecnología |
|---|---|---|
| **Snapshot <5s** | Cache local SQLite + API ultra-rápida | React Native + Expo SQLite + Node.js |
| **3 pilares únicamente** | DB schema (connectivity, energy, environment) | Prisma enum validation |
| **Datos >2h marcan "AHORA"** | `lastVerified` field + Bull cron + Socket.io | PostgreSQL + Bull + Socket.io |
| **Validación pasiva (foreground)** | AppState listener (iOS/Android) + WebSocket | React Native AppState + Socket.io-client |
| **Permisos explícitos** | Expo Location (always/while/never) + DB field | Expo + LocationPermission enum |
| **Sin GPS background** | Validación solo cuando app abierta | AppState listener, no background tasks |
| **Impacto visible** | Cálculo COUNT en validación + UI message | Prisma query + ImpactMessage component |
| **Funciona offline** | SQLite local + sync eventual | WatermelonDB |
| **Distribución equitativa** | Round-robin algorithm en Bull queue | Bull job + algoritmo |

---

## 📦 Dependencias Críticas del MVP

### Backend (package.json)
```json
{
  "dependencies": {
    "express": "^4.18.2",
    "prisma": "^5.7.0",
    "@prisma/client": "^5.7.0",
    "socket.io": "^4.5.4",
    "bull": "^4.11.0",
    "ioredis": "^5.3.2",
    "jsonwebtoken": "^9.1.0",
    "zod": "^3.22.4",
    "pino": "^8.16.0",
    "axios": "^1.6.2"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "jest": "^29.7.0",
    "@types/node": "^20.8.0"
  }
}
```

### Mobile (package.json)
```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-native": "^0.73.0",
    "expo": "^50.0.0",
    "@react-navigation/native": "^6.1.8",
    "redux": "^4.2.1",
    "@reduxjs/toolkit": "^1.9.6",
    "watermelondb": "^0.28.0",
    "socket.io-client": "^4.5.4",
    "axios": "^1.6.2",
    "expo-location": "^16.0.0",
    "zod": "^3.22.4",
    "react-native-paper": "^5.11.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/react": "^18.2.0"
  }
}
```

---

## 🎯 Próximos Pasos Recomendados

1. **Crear repositorio backend:** `red-nomada-api`
2. **Crear repositorio mobile:** `red-nomada-mobile`
3. **Setup PostgreSQL local:** Docker Compose
4. **Implementar autenticación JWT:** Base para Story 1-3
5. **Implementar endpoints de Places:** Story 1 (Snapshot)
6. **Implementar Socket.io + Bull:** Story 2 (Validación pasiva)
7. **Implementar Gamificación:** Story 3 (Misiones + Impacto)

