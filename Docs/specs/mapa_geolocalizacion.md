# Mapa y Geolocalización: Red-Nómada MVP

## Referencia a HU
- **HU1:** "UI Eficiente: El diseño de la interfaz debe garantizar que el usuario pueda identificar el estado de estos 3 pilares y tomar una decisión en menos de 5 segundos"
- **HU2:** "Gestión de Permisos: 'siempre', 'una vez', 'solo cuando está en uso la app'"
- **Restricción:** "Prohibición de rastreo GPS en segundo plano"

---

## 1. Configuración del Mapa

### Provider
- **Android:** Google Maps (requiere API key en `app.json`)
- **iOS:** Apple Maps (nativo, sin API key)
- Configuración vía `react-native-maps` con `provider={PROVIDER_DEFAULT}`

### Estado inicial
- Región inicial: ubicación del usuario si tiene permisos
- Si no hay permisos: Buenos Aires centro (default: lat -34.6037, lng -58.3816)
- Zoom inicial: `latitudeDelta: 0.02, longitudeDelta: 0.02` (~2km radio visible)
- **Modo claro por defecto** — no implementar modo oscuro en MVP

### Controles visuales
- Botón "Mi ubicación" (esquina inferior derecha) → centra en usuario
- Zoom nativo (pinch + botones del provider)
- **Sin** compás ni rotación en MVP

---

## 2. Marcadores de Lugares

### Iconos por categoría
| Categoría | Icono | Color |
|---|---|---|
| CAFE | Taza de café | Marrón |
| COWORKING | Laptop/escritorio | Azul |
| BIBLIOTECA | Libro | Verde |
| OTRO | Pin genérico | Gris |

### Indicador de frescura
- **Fresco (< 2h):** borde del marcador sólido, icono completo
- **Necesita verificación (>= 2h):** borde punteado/naranja, badge de alerta ⚠️ sobre el marcador
- **Sin datos:** icono gris/desaturado

### Clustering
- **Threshold:** agrupar cuando > 15 marcadores visibles en pantalla
- Diseño del cluster: círculo con número de lugares
- Color del cluster: promedio de frescura de los lugares agrupados (si alguno necesita verificación → naranja)
- Tap en cluster → zoom in automático para expandir

### Animaciones
- Marcadores aparecen con fade-in al cargar nuevos lugares
- El marcador seleccionado tiene bounce/scale animation

---

## 3. Interacciones del Mapa

### Tap en marcador
- **Comportamiento:** abre bottom sheet con preview del lugar (no navega a otra pantalla)
- **Preview incluye:** nombre, dirección, 3 iconos de pilares con valores, estado de frescura, botón "Ver detalle"
- Tap en "Ver detalle" → navega a `/lugar/[id]`

### Pan y zoom
- Al terminar el movimiento (`onRegionChangeComplete`) → fetch nuevos lugares para la región visible
- **Debounce:** 500ms después de que el usuario deja de mover el mapa
- Radio de búsqueda = diagonal visible del mapa (cálculo automático)
- **Límite de request:** no más de 1 request cada 2 segundos

### Mi ubicación
- **One-shot:** al tocar el botón "Mi ubicación" → obtener posición actual y centrar mapa
- **No tracking continuo** (ahorra batería, restricción del proyecto)
- Si no hay permisos → mostrar toast: "Activá la ubicación en Configuración"

### Long press (post-MVP)
- Placeholder para futura feature de "sugerir nuevo lugar"
- En MVP: no implementar

---

## 4. Filtros del Mapa

### Filtros disponibles
| Filtro | Opciones | Default |
|---|---|---|
| Categoría | Todas, Café, Coworking, Biblioteca, Otro | Todas |
| Conectividad | Todas, Alta, Media, Baja | Todas |
| Energía | Todas, Muchos, Suficientes, Pocos | Todas |
| Ambiente | Todos, Silencioso, Moderado, Ruidoso | Todos |
| Solo frescos | Toggle on/off | Off |

### UI de filtros
- Barra de filtros horizontal debajo del header (chips scrollables)
- Tap en chip → abre popover/dropdown con opciones
- Filtros activos se muestran con color destacado
- Botón "Limpiar filtros" cuando hay al menos 1 activo

### Comportamiento
- Al aplicar filtro → se refrescan los marcadores del mapa
- Los filtros son **AND** entre sí (ej: CAFÉ + Conectividad ALTA + Solo frescos)
- Los filtros se envían como query params al endpoint `GET /lugares`

---

## 5. Flujo de Geolocalización

### Solicitud de permisos
- **Primer uso:** modal explicativo antes del dialog nativo
  - Mensaje: "Nómada necesita tu ubicación para mostrarte espacios de trabajo cercanos"
  - Opciones: "Ahora no" | "Permitir"
- **Si acepta:** solicitar con expo-location → sistema nativo pregunta "siempre/una vez/solo en uso"
- **Si rechaza:** la app funciona normal, muestra mapa centrado en default (Buenos Aires)
- **Re-solicitud:** si el usuario rechazó, mostrar banner persistente con link a Configuración

### Estrategia de obtención de posición
- `Location.getCurrentPositionAsync({ accuracy: Location.Accuracy.Balanced })`
- **Accuracy Balanced** (no High) para ahorrar batería
- **No usar** `watchPositionAsync` en MVP
- Se obtiene la posición al:
  1. Abrir la app (si tiene permisos)
  2. Tocar "Mi ubicación"
  3. Iniciar un reporte

### Fallback sin permisos
- Usar ubicación guardada en cache (última conocida)
- Si no hay cache → Buenos Aires centro
- La app es 100% funcional sin ubicación (solo debe escribir la dirección/lugar manualmente)

---

## 6. Performance

### Carga de lugares
- **Viewport-based:** solo se piden lugares visibles en el mapa actual
- **Paginación:** máximo 50 lugares por request
- **Cache local:** los lugares del viewport actual se cachean por 30 min (ver experiencia_offline.md)
- **No precargar** todos los lugares al inicio

### Marcadores
- **Límite visible:** máximo 100 marcadores sin clustering
- Con clustering activado: siempre manejable visualmente
- **Re-render:** solo cuando cambia la región O los filtros (no en cada render del componente)

### Evitar re-renders
- `React.memo` en componente de marcador individual
- `useMemo` para calcular marcadores filtrados
- Separar mapa y controles en componentes independientes

---

## 7. Snapshot en el Mapa

### Preview en marcador
- **NO** mostrar los 3 pilares directamente en el marcador (demasiada info, rompe los < 5 segundos)
- Solo icono de categoría + indicador de frescura

### Bottom sheet al tap
- Nombre del lugar (1 línea, ellipsis)
- Dirección (1 línea, ellipsis)
- **3 iconos de pilares** con sus valores (Baja/Media/Alta, etc.)
- Estado de frescura: "Actualizado hace X min" o "Necesita verificación AHORA"
- Botones: "Ver detalle" | "Reportar estado"

### Navegación
- "Ver detalle" → `/lugar/[id]` con toda la info + historial
- "Reportar estado" → flujo de reporte (ver flujo_reporte.md)

---

## 8. Offline en el Mapa

- Si está offline → muestra solo marcadores cacheados
- Banner: "Mapa con datos guardados — pueden no estar actualizados"
- No se puede hacer pan/zoom para traer nuevos lugares
- Los marcadores cacheados muestran su antigüedad real
