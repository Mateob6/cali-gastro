# Esquina Foodie — Restaurant Tracker

## Overview

Tracker colaborativo de restaurantes en Cali. Single HTML file con React + Firebase Realtime Database + Leaflet.js para mapa interactivo.

- **Página:** https://mateob6.github.io/cali-gastro/
- **Firebase:** https://cali-gastro-default-rtdb.firebaseio.com/
- **Repo:** https://github.com/Mateob6/cali-gastro

## Stack

- React 18 via CDN (babel standalone)
- Firebase Realtime Database (compat SDK via CDN)
- Leaflet.js + OpenStreetMap (mapa interactivo, sin API key)
- GitHub Pages para hosting
- Un solo archivo: `index.html`

## Páginas

| Página | Descripción |
|--------|-------------|
| **Inicio** | Landing con stats, restaurantes destacados, accesos rápidos |
| **Restaurantes** | Lista completa con grid/tabla, filtros, búsqueda, detalle, agregar |
| **Mapa** | Mapa interactivo de Cali con marcadores por restaurante |

Navegación via BottomNav persistente (Inicio · Restaurantes · Mapa).

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

La página se actualiza sola (Firebase realtime listener). No necesita push a GitHub ni reload manual. El restaurante aparece tanto en la lista como en el mapa automáticamente.

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
│   ├── {id}/  → datos públicos + lat/lng
│   └── ...
└── personal/
    ├── {id}/  → status, myRating, notes, dateVisited, etc.
    └── ...
```

## Design System: Earth & Terracotta

- **Fonts:** Newsreader (headlines) + Be Vietnam Pro (body/labels)
- **Colors:** Terracotta (#9b3f25) + Olive (#5c614d) + Cream (#fff8f6)
- **Icons:** Material Symbols Outlined
- **Map:** Leaflet.js + OpenStreetMap tiles

## Desarrollo

Para hacer cambios al HTML:
1. Editar `index.html`
2. Probar localmente (`open index.html`)
3. Commit + push → GitHub Pages actualiza en ~60s

Para cambiar datos sin tocar el código:
- Usar curl a Firebase directamente (ver flujo arriba)
