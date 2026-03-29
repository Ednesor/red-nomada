# Pantalla de Perfil y Estadísticas: Red-Nómada MVP

## Referencia a HU
- **HU3:** "Visualizar a cuántas personas he ayudado con mi actualización, para sentir que mi aporte tiene un valor tangible"
- **HU3:** "Métrica de Impacto Social: 'Gracias a tu reporte, 12 personas evitaron ir a un lugar lleno'"

---

## 1. Layout de la Pantalla

### Estructura (scroll vertical)
```
┌─────────────────────────────────────┐
│ [Avatar] Nombre del Usuario         │
│          email@ejemplo.com          │
│          Explorador                 │
├─────────────────────────────────────┤
│         IMPACTO SOCIAL              │
│                                     │
│            156                      │
│     personas ayudadas               │
│                                     │
│  📋 42 reportes  ✓ 18 validaciones  │
├─────────────────────────────────────┤
│         MISIONES DE HOY             │
│  ┌─────────────────────────────┐    │
│  │ Reportá 3 cafés hoy    2/3  │    │
│  │ ████████████░░░░  66%       │    │
│  │ Recompensa: 50 puntos       │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │ Validá 3 reportes      0/3  │    │
│  │ ░░░░░░░░░░░░░░░░   0%      │    │
│  │ Recompensa: 60 puntos       │    │
│  └─────────────────────────────┘    │
├─────────────────────────────────────┤
│         INSIGNIAS (4)               │
│  [🥇] [🔍] [📋] [⭐]               │
│  Nueva: "Verificador" desbloqueada! │
├─────────────────────────────────────┤
│         CONFIGURACIÓN               │
│  📍 Permisos de ubicación: En uso   │
│  🔔 Notificaciones: Activadas       │
│  🚪 Cerrar sesión                   │
└─────────────────────────────────────┘
```

### Orden de secciones (de arriba a abajo)
1. **Header usuario** (nombre, email, rol) — fijo
2. **Impacto social** (número grande + desglose) — hero section
3. **Misiones del día** (lista con progreso)
4. **Insignias** (horizontal scroll o grid)
5. **Configuración** (permisos, notificaciones, logout)

---

## 2. Sección de Impacto Social

### Elemento principal
- **Número grande centrado:** `personas_ayudadas_total`
- Tipografía: 48-64px, bold, color primario
- Debajo: "personas ayudadas" en texto secundario
- **Animación al cargar:** conteo animado de 0 al número real (1 segundo)

### Desglose secundario
- En la misma sección, debajo del número principal:
  - "42 reportes" con ícono de reporte
  - "18 validaciones" con ícono de check
- Formato horizontal, centrados

### Actualización
- Se actualiza al volver a la pantalla (cada vez que el usuario navega al tab de perfil)
- NO se actualiza en tiempo real (se refresca al consultar la API)

---

## 3. Sección de Misiones del día

### Card de misión
```
┌─────────────────────────────────────┐
│ Título de la misión           2/3   │
│ ████████████░░░░░░░░  66%           │
│ Recompensa: 50 puntos 🏆            │
└─────────────────────────────────────┘
```

### Elementos por card
- **Título:** texto de la misión (ej: "Reportá 3 cafés hoy")
- **Progreso:** texto "X/Y" alineado a la derecha
- **Barra de progreso:** barra horizontal con porcentaje visual
- **Recompensa:** "X puntos" + ícono de insignia si la misión otorga una

### Estados visuales
| Estado | Estilo |
|---|---|
| Pendiente (0%) | Barra vacía, color gris |
| En progreso (>0%) | Barra parcial, color primario |
| Completada (100%) | Barra completa, color verde, checkmark |

### Completada con éxito
- Card cambia a fondo verde suave
- Checkmark animado al completar
- Texto: "¡Completada! +X puntos" durante 3 segundos, luego vuelve a estado normal

### Sin misiones
- Mensaje: "No hay misiones disponibles hoy. ¡Volvé mañana!"
- Ilustración simple (emoji o ícono)

### Scroll
- Si hay > 3 misiones → scroll vertical dentro de la sección
- Máximo visible sin scroll: 3 cards

---

## 4. Sección de Insignias

### Layout
- **Grid de 2-3 columnas** (dependiendo del ancho de pantalla)
- Opcional: horizontal scroll si hay muchas

### Card de insignia
```
┌──────────────┐
│     🥇       │
│  Primer      │
│  Reporte     │
│ 28 Mar 2026  │
└──────────────┘
```

### Estados
- **Obtenida:** icono a color, nombre visible, fecha de obtención
- **No obtenida (locked):** icono en escala de grises, nombre visible, candado 🔒
- ¿Se muestran las locked? **Sí** — para motivar al usuario a conseguirlas

### Animación al obtener nueva insignia
- Al entrar a la pantalla y detectar una nueva insignia → pulse/glow animation durante 3 segundos
- Badge "¡Nueva!" en la card durante esa sesión

### Scroll
- Si hay > 6 insignias → scroll horizontal
- Sección con título "Insignias (X)" donde X = total obtenidas

---

## 5. Sección de Configuración

### Permisos de ubicación
- Selector con las 3 opciones de HU2:
  - "Siempre" (no recomendado, consume batería)
  - "Una vez" (default al otorgar)
  - "Solo en uso" (recomendado)
- Estado actual visible: "Permisos de ubicación: En uso"
- Al tocar → modal con opciones + link a Configuración del sistema

### Notificaciones (si aplica)
- Toggle: "Notificaciones de verificación"
- Default: activadas
- Al desactivar → no se muestran alertas pasivas ni notificaciones locales

### Cerrar sesión
- Botón rojo/naranja: "Cerrar sesión"
- Al tocar → modal de confirmación: "¿Seguro que querés cerrar sesión?"
- Al confirmar:
  1. Limpiar tokens de `expo-secure-store`
  2. Limpiar stores de Zustand
  3. Limpiar cache de AsyncStorage (opcional)
  4. Navegar a pantalla de Login

### Eliminar cuenta (post-MVP)
- No implementar en MVP
- Placeholder: "Eliminar cuenta" deshabilitado con tooltip "Próximamente"

---

## 6. Estados de la Pantalla

### Loading inicial
- Skeleton loading: rectángulos grises animados simulando el layout
- Duración máxima: 3 segundos → si no responde, mostrar error

### Estado vacío (usuario nuevo)
- Impacto social: "0 personas ayudadas" con mensaje "¡Hacé tu primer reporte para empezar a ayudar!"
- Misiones: se muestran normalmente (pueden existir misiones sin reportes)
- Insignias: todas en estado locked
- Configuración: siempre visible

### Error al cargar datos
- Banner de error: "No se pudieron cargar tus datos"
- Botón "Reintentar"
- Si está offline → mostrar datos cacheados + banner "Mostrando datos guardados"

### Pull-to-refresh
- Refresh de toda la pantalla
- Invalida cache y re-fetch de GET /perfil/impacto + GET /misiones

---

## 7. Datos y Cache

| Dato | Fuente | Cache | Frecuencia |
|---|---|---|---|
| Perfil + impacto | GET /perfil/impacto | AsyncStorage 1 hora | Al entrar al tab |
| Misiones | GET /misiones | AsyncStorage hasta 00:00 | Al entrar al tab |
| Insignias | GET /perfil/impacto (incluidas) | AsyncStorage 1 hora | Con perfil |

### Actualización post-acción
- Después de crear un reporte o validar → invalidar cache de misiones y perfil
- La próxima visita al tab re-fetch datos frescos
- NO se hace polling ni actualización en tiempo real

---

## 8. Performance

- **Lazy loading:** las insignias se cargan después del contenido principal
- **FlatList** para misiones si hay muchas (> 10)
- **Memoización:** `React.memo` en cards de misión e insignia
- **Animaciones:** usar `useNativeDriver: true` para animaciones de conteo
- **Imágenes de insignias:** assets locales (no remotos), precargados al inicio de la app
