# Esquina Foodie — Restaurant Tracker

## Overview

Tracker colaborativo de restaurantes en Cali. Single HTML file con React + Firebase Realtime Database + Leaflet.js para mapa interactivo. Diseño "Earth & Terracotta" generado con Stitch.

- **Página:** https://mateob6.github.io/cali-gastro/
- **Firebase:** https://cali-gastro-default-rtdb.firebaseio.com/
- **Repo:** https://github.com/Mateob6/cali-gastro
- **Stitch Project:** `projects/8507783255875084601`

## Stack

- React 18 via CDN (babel standalone, JSX in-browser)
- Firebase Realtime Database (compat SDK via CDN)
- Leaflet.js 1.9.4 + OpenStreetMap (mapa interactivo, sin API key)
- Material Symbols Outlined (iconos)
- GitHub Pages para hosting
- Un solo archivo: `index.html`

## Páginas

| Página | Componente | Descripción |
|--------|-----------|-------------|
| **Inicio** | `HomePage` | Landing con stats, restaurantes destacados, accesos rápidos a lista y mapa |
| **Restaurantes** | `ListPage` | Grid/tabla con filtros (zona, categoría, precio, status), búsqueda, ordenamiento, detalle, agregar |
| **Mapa** | `MapPage` | Mapa interactivo de Cali con marcadores color-coded por status, popups, filtro de status |

Navegación: `BottomNav` persistente (Inicio · Restaurantes · Mapa). Routing por estado (`page`), sin React Router.

## Arquitectura de componentes

```
App (estado global, Firebase listener, handlers)
├── BottomNav (fijo, 3 items)
├── HomePage (stats, destacados, nav cards)
│   ├── StatCard
│   ├── MiniCard
│   └── NavCard
├── ListPage (toda la funcionalidad de lista)
│   ├── RestaurantCard (tarjeta con tri-state buttons)
│   ├── RestaurantTable (vista tabla)
│   └── FAB (+)
├── MapPage (Leaflet.js)
│   ├── Marcadores SVG color-coded
│   └── Popups con "Ver detalle"
├── DetailPanel (modal compartido entre list y map)
│   ├── Info pública (dirección, rating, precio, platos, vibe, horario, reserva, instagram, tags)
│   └── Mi experiencia (rating personal, fecha, pedido, gasto, notas, volvería)
├── AddModal (formulario completo)
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
  status: "quiero-ir",         // quiero-ir | ya-fui | no-interesa | sin-decidir
  myRating: 4,                 // 0-5
  dateVisited: "2025-12-14",   // fecha o ""
  whatIOrdered: "Ceviche + Lomo",
  totalSpent: "185000",        // COP o ""
  notes: "Increíble ambiente nocturno...",
  wouldReturn: true            // true | false | null
}
```

## Agregar restaurantes

Cuando el usuario pida agregar restaurantes, seguir este flujo:

### Paso 1: Búsqueda con agentes

Mandar un agente (general-purpose) POR CADA restaurante con este prompt:

```
Busca el restaurante "[NOMBRE]" en Cali, Colombia. Devuelve ÚNICAMENTE un bloque JSON válido con esta estructura exacta (sin texto adicional antes ni después del JSON):

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
curl -X PUT "https://cali-gastro-default-rtdb.firebaseio.com/cali-gastro/restaurants/[ID].json" \
  -H "Content-Type: application/json" \
  -d '[JSON completo del restaurante]'
```

### Paso 3: Verificación

La página se actualiza sola (Firebase realtime listener). No necesita push a GitHub ni reload manual. El restaurante aparece en lista Y mapa automáticamente.

### Notas

- Mandar agentes en PARALELO si hay múltiples restaurantes
- El ID debe ser kebab-case del nombre (ej: "La Sole" → "la-sole")
- Si el restaurante ya existe, el PUT sobreescribe — útil para actualizar datos
- Para eliminar: `curl -X DELETE "https://cali-gastro-default-rtdb.firebaseio.com/cali-gastro/restaurants/[ID].json"`
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
└── personal/
    ├── {id}/  → status, myRating, notes, dateVisited, whatIOrdered, totalSpent, wouldReturn
    └── ...
```

## Design System: Earth & Terracotta

Generado en Stitch (`assets/17553987259345874469`).

| Token | Valor |
|-------|-------|
| Surface | `#fff8f6` |
| Primary (Terracotta) | `#9b3f25` |
| Secondary (Olive) | `#5c614d` |
| Tertiary (Teal) | `#006765` |
| On Surface | `#231917` |
| Outline Variant | `#ddc0b9` |
| Font Headline | Newsreader (serif) |
| Font Body | Be Vietnam Pro (sans) |
| Radius | 12px (0.75rem) |
| Icons | Material Symbols Outlined |

### Marcadores del mapa
- Teal (`#006765`): Quiero ir
- Terracotta (`#9b3f25`): Ya fui
- Gris (`#89726c`): Sin decidir / No me interesa

## Desarrollo

Para hacer cambios al HTML:
1. Editar `index.html`
2. Probar localmente (`open index.html`)
3. Commit + push → GitHub Pages actualiza en ~60s

Para cambiar datos sin tocar el código:
- Usar curl a Firebase directamente (ver flujo arriba)
- La página se sincroniza en tiempo real
