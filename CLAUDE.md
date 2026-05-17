# La Esquinita — Restaurant Tracker

## Overview

Espacio personal de Isabela y Mateo para trackear restaurantes en Cali y alrededores. 55+ restaurantes activos. Single HTML file con React + Firebase Realtime Database + Leaflet.js para mapa interactivo. Diseño "Earth & Terracotta" con dark mode. PWA instalable.

- **Página:** https://mateob6.github.io/cali-gastro/
- **Firebase:** https://cali-gastro-default-rtdb.firebaseio.com/
- **Repo:** https://github.com/Mateob6/cali-gastro
- **Stitch Project:** `projects/8507783255875084601`

## Stack

- React 18 via CDN (babel standalone, JSX in-browser)
- Firebase Realtime Database (compat SDK via CDN)
- Firebase Authentication (Google Sign-in, restringido a 3 emails)
- Firebase Storage (fotos de restaurantes, comprimidas client-side)
- Leaflet.js 1.9.4 + OpenStreetMap (mapa interactivo, sin API key)
- Material Symbols Outlined (iconos)
- GitHub Pages para hosting
- PWA (manifest.json + iconos para instalación en home screen)
- Un solo archivo principal: `index.html`

## Autenticación

Google Sign-in restringido a tres cuentas:
- `mbcvarios019@gmail.com` (Mateo)
- `mateo.belalcazar6@gmail.com` (Mateo)
- `cnscisabelacas@gmail.com` (Isabela)

La app muestra una pantalla de login antes de cargar datos. El avatar del usuario aparece como pill fija en la esquina superior derecha (click → sign out). Botón de dark mode al lado del avatar. Emails no autorizados son rechazados automáticamente.

## Fotos

Galería de fotos por restaurante, compartida entre ambos usuarios.

### Storage: `photos/{restaurantId}/{timestamp}_{uid_short}.jpg`
### Realtime DB: `cali-gastro/photos/{restaurantId}/{photoId}`

```javascript
{
  url: "https://firebasestorage.googleapis.com/...",
  storagePath: "photos/odiseo-bistro/1718300000000_abc123.jpg",
  uploadedBy: "Mateo",
  uploadedByUid: "abc123",
  uploadedAt: 1718300000000,
  caption: "El mejor ceviche"  // editable desde lightbox
}
```

Las fotos se comprimen a JPEG (max 800px, quality 0.65) antes de subir usando `createImageBitmap` (no bloquea el hilo principal). Se suben en paralelo cuando se seleccionan múltiples fotos.

### Captions

Las captions se editan directamente en el lightbox (input debajo de la imagen). Se auto-guardan a Firebase on blur. Se muestran en el estilo polaroid del feed y en el overlay de thumbnails.

### HEIC/HEIF (fotos de iPhone)

- **Desde iPhone Safari**: HEIC sube directo (Safari decodifica nativamente)
- **Desde Chrome desktop**: Chrome no soporta HEIC. La app intenta convertir con `heic2any`, pero si falla muestra error pidiendo convertir a JPEG
- **Conversión manual en Mac**: `sips -s format jpeg -s formatOptions 80 foto.HEIC --out foto.jpg`
- **Conversión batch**: `for f in *.HEIC; do sips -s format jpeg -s formatOptions 80 "$f" --out "${f%.HEIC}.jpg"; done`

## Páginas

| Página | Componente | Descripción |
|--------|-----------|-------------|
| **Inicio** | `HomePage` | Saludo dinámico (hora del día), stats (total, por visitar, visitados, gasto total, visitas, promedio), destacados, nav cards incluyendo randomizer |
| **Restaurantes** | `ListPage` | Grid/tabla con filtros (zona, categoría, precio, status, abierto hoy), búsqueda, ordenamiento, foto de portada en cards |
| **Fotos** | `FeedPage` | Dos vistas: "Por restaurante" (polaroid-style) y "Por fecha" (calendario mensual interactivo) |
| **Mapa** | `MapPage` | Mapa interactivo de Cali con marcadores color-coded por status, popups, filtro de status |

Navegación: `BottomNav` persistente con pill indicator (Inicio · Restaurantes · Fotos · Mapa). Routing por estado (`page`), sin React Router.

## Arquitectura de componentes

```
App (estado global, Firebase listener, auth, dark mode, handlers)
├── Login screen (si no hay user autenticado)
├── Dark mode toggle + User pill (fijo top-right)
├── BottomNav (fijo, 4 items, pill indicator activo)
├── HomePage (saludo dinámico, stats 2 filas, destacados, nav cards)
│   ├── StatCard (6: restaurantes, por visitar, visitados, gasto, visitas, promedio)
│   ├── MiniCard
│   └── NavCard (Explorar, ¿A dónde vamos?, Mapa)
├── ListPage (toda la funcionalidad de lista)
│   ├── RestaurantCard (cover photo, tri-state, rating, abierto/cerrado, última visita, animación ya-fui)
│   ├── RestaurantTable (vista tabla)
│   └── FAB (+)
├── FeedPage
│   ├── Vista "Por restaurante" (polaroid-style, rotación aleatoria, captions)
│   └── Vista "Por fecha" (calendario mensual, navegación meses, grid de fotos por día)
├── MapPage (Leaflet.js)
│   ├── Marcadores SVG color-coded
│   └── Popups con "Ver detalle"
├── DetailPanel (modal compartido)
│   ├── Info pública (dirección, Google Maps link, rating, precio, platos, vibe, horario, reserva, Instagram link, tags)
│   ├── PhotoGallery (grid + upload + lightbox con captions editables)
│   └── Historial de visitas (múltiples visitas, agregar/eliminar)
├── AddModal (formulario completo)
├── RandomizerModal (date night picker con animación)
├── StarRating (estrellas clickeables)
└── StatusBadge (chip de estado)
```

## Modelo de datos

### Restaurante (Firebase: `cali-gastro/restaurants/{id}`)

```javascript
{
  id: "odiseo-bistro",         // kebab-case único
  name: "Odiseo Bistro",       // nombre oficial
  zone: "Granada",             // barrio/zona en Cali
  address: "Av. 9 Norte #10-107, Granada, Cali",
  cuisine: "Mediterránea / Autor",  // descripción libre, máx 4 palabras
  tags: ["mediterránea", "de-autor", "fine-dining", "romántico"],
  priceTier: "$$$",            // $, $$, $$$, $$$$
  priceRange: "$80K–90K",      // rango display
  priceMin: 80,                // en miles COP
  priceMax: 90,
  rating: 4.7,                 // Google Maps, 1-5 o null
  reviews: 1671,               // número de reseñas
  highlight: "Coliflor horneado, arroz caldoso de mariscos",
  vibe: "Noir, atmosférico",   // máx 8 palabras
  reserva: true,
  closedDay: "Dom (solo almuerzo)",
  instagram: null,             // handle sin @ o null
  lat: 3.4573,                 // coordenadas para el mapa
  lng: -76.5360
}
```

### Personal (Firebase: `cali-gastro/personal/{id}`)

```javascript
{
  status: "ya-fui",            // quiero-ir | ya-fui | no-interesa | sin-decidir
  myRating: 4,                 // 0-5
  // Campos legacy (se mantienen pero UI usa visits)
  dateVisited: "2025-12-14",
  whatIOrdered: "Ceviche + Lomo",
  totalSpent: "185000",
  notes: "...",
  wouldReturn: true,
  // Historial de visitas (nuevo)
  visits: {
    "-abc123": { date: "2026-05-10", ordered: "Ceviche + Lomo", spent: "185000", notes: "Primer date aquí", wouldReturn: true },
    "-def456": { date: "2026-03-15", ordered: "Solo cócteles", spent: "90000", notes: "Cumpleaños", wouldReturn: true }
  }
}
```

**Auto-migración:** Si un restaurante tiene campos legacy sin `visits`, el DetailPanel los migra automáticamente a un primer entry en `visits` al abrir.

## Features principales

### Historial de visitas
- Múltiples visitas por restaurante (fecha, pedido, gasto, notas, ¿volvería?)
- Se agregan desde DetailPanel con botón "Agregar visita"
- Se pueden eliminar individualmente
- Stats en HomePage suman gastos y cuentan visitas

### Date Night Randomizer
- Modal "¿A dónde vamos?" accesible desde HomePage
- Filtra: `quiero-ir` + abiertos hoy (usa `isOpenToday` helper)
- Filtro inline por precio
- Animación shuffle → restaurante ganador con botones "Otro", "Ver ficha", "Ir" (Google Maps)

### "Abierto hoy"
- Helper `isOpenToday(closedDay)` → `true` / `false` / `'partial'`
- Badge en cada RestaurantCard (verde "Abierto", naranja "Solo almuerzo", gris "Cerrado hoy")
- Filtro "Abierto hoy" en ListPage status chips

### Dark Mode
- Toggle en esquina superior derecha (ícono sol/luna)
- Variables CSS completas para tema oscuro
- Persiste en `localStorage`
- Aplica `data-theme="dark"` en `<html>`

### Calendario de fotos
- Vista "Por fecha" en FeedPage: grilla mensual (Lu-Do)
- Navegación entre meses con flechas
- Puntos indicadores en días con fotos
- Click en día → muestra fotos de ese día debajo
- Lightbox navega solo dentro del día seleccionado

### Polish visual
- Foto de portada en RestaurantCard (primera foto subida del restaurante)
- Bottom nav con pill indicator animado
- Animación check + confetti al marcar "Ya fui"
- Fotos estilo polaroid en feed (rotación aleatoria, caption visible)
- Saludo dinámico en HomePage según hora del día
- Skeleton shimmer en loading state
- "Última visita: hace X días" en cards de restaurantes visitados

## Agregar restaurantes

Cuando el usuario pida agregar restaurantes, seguir este flujo:

### Paso 1: Búsqueda con agentes

Mandar un agente (general-purpose) POR CADA restaurante con este prompt:

```
Busca el restaurante "[NOMBRE]" en Cali, Colombia (o la ciudad indicada por el usuario). Devuelve ÚNICAMENTE un bloque JSON válido con esta estructura exacta (sin texto adicional antes ni después del JSON):

{
  "id": "[nombre-en-kebab-case]",
  "name": "[Nombre oficial]",
  "zone": "[Barrio/zona en Cali]",
  "address": "[Dirección completa con calle y número]",
  "cuisine": "[Descripción libre de la cocina, máx 4 palabras]",
  "tags": ["[selecciona 3-5 de la lista de TAGS VÁLIDOS]"],
  "priceTier": "[$ | $$ | $$$ | $$$$]",
  "priceRange": "[$XXK–$YYK]",
  "priceMin": [número en miles, sin decimales],
  "priceMax": [número en miles, sin decimales],
  "rating": [número 1-5 con un decimal, o null si no hay datos],
  "reviews": [número entero de reseñas en Google Maps],
  "highlight": "[2-3 platos recomendados separados por coma]",
  "vibe": "[Descripción del ambiente en máximo 8 palabras]",
  "reserva": [true si se recomienda reservar, false si no],
  "closedDay": "[Día(s) cerrado o 'Abierto siempre']",
  "instagram": "[handle sin @ o null si no tiene]",
  "lat": [latitud decimal, 4 decimales mínimo],
  "lng": [longitud decimal, 4 decimales mínimo]
}

TAGS VÁLIDOS (seleccionar 3-5 que apliquen):

Cocina (origen): colombiana, peruana, japonesa, italiana, mediterránea, mexicana, francesa, asiática, árabe, americana, internacional
Estilo (preparación): fusión, de-autor, tradicional, parrilla, mariscos, pizzería, pastas, vegetariana
Experiencia (por qué vas): fine-dining, casual, coctelería, bar-gastronómico, speakeasy, brunch, mirador, música-en-vivo, romántico, pet-friendly, para-grupos

PRICE TIER:
$ = plato fuerte < $40,000 COP
$$ = plato fuerte $40,000–$70,000 COP
$$$ = plato fuerte $70,000–$120,000 COP
$$$$ = plato fuerte > $120,000 COP

Busca en Google Maps, Instagram, TripAdvisor, Restaurant Guru, Rappi, Degusta.
Para lat/lng: busca las coordenadas exactas en Google Maps o Restaurant Guru.
Si no encuentras un dato, usa null para números o "" para strings.
```

### Paso 2: Inyección a Firebase

Con el JSON resultante del agente, ejecutar:

```bash
curl -X PUT "https://cali-gastro-default-rtdb.firebaseio.com/cali-gastro/restaurants/[ID].json?auth=$FIREBASE_DB_SECRET" \
  -H "Content-Type: application/json" \
  -d '[JSON completo del restaurante]'
```

### Paso 3: Verificación

La página se actualiza sola (Firebase realtime listener). No necesita push a GitHub ni reload manual. El restaurante aparece en lista Y mapa automáticamente.

### Notas

- Mandar agentes en PARALELO si hay múltiples restaurantes
- El ID debe ser kebab-case del nombre (ej: "La Sole" → "la-sole")
- Si el restaurante ya existe, el PUT sobreescribe — útil para actualizar datos
- Para eliminar: `curl -X DELETE "https://cali-gastro-default-rtdb.firebaseio.com/cali-gastro/restaurants/[ID].json?auth=$FIREBASE_DB_SECRET"`
- **`$FIREBASE_DB_SECRET`** está en `~/.zshrc` (Firebase legacy database secret)
- **lat/lng son necesarios** para que el restaurante aparezca en el mapa

## Taxonomía de tags

### Cocina (origen)
colombiana · peruana · japonesa · italiana · mediterránea · mexicana · francesa · asiática · árabe · americana · internacional

### Estilo (preparación)
fusión · de-autor · tradicional · parrilla · mariscos · pizzería · pastas · vegetariana

### Experiencia (por qué vas)
fine-dining · casual · coctelería · bar-gastronómico · speakeasy · brunch · mirador · música-en-vivo · romántico · pet-friendly · para-grupos

### Precio
$ (<40K) · $$ (40K–70K) · $$$ (70K–120K) · $$$$ (>120K)

## Estructura Firebase

```
cali-gastro/
├── restaurants/
│   ├── {id}/  → datos públicos + lat/lng + tags
│   └── ...
├── personal/
│   ├── {id}/  → status, myRating, visits: { visitId: { date, ordered, spent, notes, wouldReturn } }
│   └── ...
└── photos/
    ├── {restaurantId}/
    │   ├── {photoId}/  → url, storagePath, uploadedBy, uploadedByUid, uploadedAt, caption
    │   └── ...
    └── ...
```

## Design System: Earth & Terracotta

Generado en Stitch (`assets/17553987259345874469`). Incluye dark mode.

| Token | Light | Dark |
|-------|-------|------|
| Surface | `#fff8f6` | `#1a1210` |
| Primary | `#9b3f25` | `#ffb4a0` |
| Secondary | `#5c614d` | `#c8cdb1` |
| Tertiary | `#006765` | `#4ddad8` |
| On Surface | `#231917` | `#f0ddd8` |
| Outline Variant | `#ddc0b9` | `#534340` |
| Font Headline | Newsreader (serif) | — |
| Font Body | Be Vietnam Pro (sans) | — |
| Radius | 12px (0.75rem) | — |
| Icons | Material Symbols Outlined | — |

### Marcadores del mapa
- Teal (`#006765`): Quiero ir
- Terracotta (`#9b3f25`): Ya fui
- Gris (`#89726c`): Sin decidir / No me interesa

## Archivos del proyecto

```
restaurantes/
├── index.html       ← app completa (HTML + CSS + JSX)
├── manifest.json    ← PWA manifest
├── icon-192.png     ← PWA icon 192x192
├── icon-512.png     ← PWA icon 512x512
└── CLAUDE.md        ← este archivo
```

## Desarrollo

Para hacer cambios al HTML:
1. Editar `index.html`
2. Probar localmente (`open index.html`)
3. Commit + push → GitHub Pages actualiza en ~60s

Para cambiar datos sin tocar el código:
- Usar curl a Firebase directamente (ver flujo arriba)
- La página se sincroniza en tiempo real
