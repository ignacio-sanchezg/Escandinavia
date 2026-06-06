# CLAUDE.md — Guía de viaje interactiva: estructura, features y lecciones aprendidas

> Este documento describe el sistema completo de planificación de viajes desarrollado para el viaje Escandinavia Jun 2026. Sirve como plantilla y memoria para replicar el mismo sistema en futuros viajes.

---

## 1. VISIÓN GENERAL DEL SISTEMA

El sistema genera **tres artefactos** a partir de la información del viaje:

| Archivo | Uso | Herramienta |
|---|---|---|
| `Viaje_DESTINO.docx` | Documento de referencia completo para imprimir/archivar | Python + docx npm |
| `Ruta_DESTINO.xlsx` | Hoja de cálculo con ruta, presupuesto y estado reservas | Python + openpyxl |
| `DESTINO.html` | Guía interactiva mobile-first para usar en el viaje | HTML/CSS/JS puro |

El HTML se sube a un **repositorio GitHub público** y se sirve via **GitHub Pages**, accesible desde el móvil sin instalar nada.

---

## 2. ESTRUCTURA DE ARCHIVOS

```
/mnt/user-data/outputs/
├── Viaje_DESTINO_MesAno.docx     # Plan completo Word
├── Ruta_DESTINO_FECHA.xlsx       # Excel ruta y presupuesto
└── DESTINO_MesAno.html           # Guía interactiva

/home/claude/
├── build_word_vN.js              # Script generador del Word
└── [repo clonado]/               # Git repo

GitHub:
├── github.com/USER/REPO          # Repositorio
└── USER.github.io/REPO/FILE.html # GitHub Pages (URL pública)
```

---

## 3. ARQUITECTURA DEL HTML

### 3.1 Estructura general

```html
<head>
  <!-- Google Fonts: Playfair Display + DM Sans + JetBrains Mono -->
  <!-- Leaflet CSS (cdnjs) -->
  <!-- CSS inline: ~300 líneas de variables y componentes -->
</head>
<body>
  <!-- HERO: nombre del viaje, fechas, píldoras resumen -->
  <!-- NAV: 5 pestañas sticky -->
  <!-- #section-reservas (activa por defecto) -->
  <!-- #section-dias -->
  <!-- #section-pagos -->
  <!-- #section-pendientes -->
  <!-- #section-guias -->

  <!-- Script 1: navegación + checklist (inline) -->
  <!-- Leaflet JS (cdnjs CDN) -->
  <!-- Script 2: mapas Leaflet + fotos Wikipedia (inline) -->
  <!-- Script 3: meteorología Open-Meteo (inline) -->
</body>
```

### 3.2 Secciones del HTML

#### RESERVAS
- Cards tipo boarding pass para vuelos (código IATA, hora, aeropuerto, localizador monoespaciado)
- Cards para hoteles (nombre, fechas, metro, score, servicios, notas)
- Cards para coche alquiler y actividades especiales
- Badges de confirmación verde

#### DÍAS
- Selector de pestañas horizontal scrollable (días 1-N)
- Por cada día:
  1. **Header** (fondo navy): fecha, título, badge "dormir en X"
  2. **Widget meteorológico** (Open-Meteo, ver §6)
  3. **Alerta/tip** opcional (guide-alert en rojo, guide-tip en verde)
  4. **Mapa Leaflet** 220px de altura (ver §5)
  5. **Leyenda numerada** bajo el mapa (píldoras con color + número + nombre)
  6. **Tira de fotos** Wikipedia (scroll horizontal)
  7. **Timeline** con marcadores de tipo (confirmed/critical/warning/default)
  8. **Plan B** colapsable con alternativas (ver §4.6)

#### PAGOS
- Tarjeta Revolut (verde): cuándo y cómo usar
- Tarjeta Visa Platinum (azul): solo para EUR
- Tarjeta DCC (roja): nunca — con tabla comparativa de pérdidas

#### PENDIENTES
- Checklist interactiva con grupos por urgencia (🔴🟠🟡✅)
- Estado guardado en localStorage (persiste entre sesiones)
- Items ya pagados/confirmados pre-marcados en verde

#### GUÍAS
- Guía apps de transporte (paso a paso por ciudad)
- Guía de restaurantes (web exacta, cuándo reservar, qué pedir)
- Guía del coche de alquiler
- Plantillas de email para hoteles (en inglés, listas para copiar)

---

## 4. FEATURES DETALLADAS

### 4.1 Navegación — CRÍTICO: sin window.event

```javascript
// ✅ CORRECTO: pasar 'this' desde el botón
onclick="showSection('dias', this)"
onclick="showDay(2, this)"

function showSection(id, btn) {
  document.querySelectorAll('.section').forEach(s => s.classList.remove('active'));
  document.querySelectorAll('.nav-btn').forEach(b => b.classList.remove('active'));
  document.getElementById('section-' + id).classList.add('active');
  if (btn) btn.classList.add('active');   // ← btn, NO event.target
}
```

**⚠️ LECCIÓN APRENDIDA**: `event.target` usa `window.event` que Firefox eliminó en v99. Rompe silenciosamente toda la navegación. **Siempre pasar `this` explícitamente.**

### 4.2 Mapas Leaflet

**Dependencias CDN** (en `<head>` y antes de los scripts):
```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.css"/>
<script src="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.js"></script>
```

**Tiles**: OpenStreetMap gratuito, sin API key:
```javascript
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
```

**Inicialización lazy** — CRÍTICO:
```javascript
// ✅ NO inicializar en DOMContentLoaded (sección puede estar display:none → 0px)
// ✅ Inicializar al abrir la pestaña Días o cambiar de día

const _maps = {};
function ensureMap(d) {
  const el = document.getElementById('map-day-' + d.day);
  if (!el) return;
  if (_maps[d.day]) { _maps[d.day].invalidateSize(); return; }  // Ya existe: refrescar
  const map = L.map(el, {zoomControl: false, attributionControl: false})
    .setView(d.center, d.zoom);
  // ... añadir tiles, polyline, markers
  _maps[d.day] = map;
  setTimeout(() => map.invalidateSize(), 60);  // Refrescar tras render
}
```

**⚠️ LECCIÓN APRENDIDA**: Leaflet crea el mapa con dimensiones 0×0 si el contenedor está oculto. Guarda instancias en `_maps{}` y llama `invalidateSize()` cuando el contenedor se hace visible.

**Marcadores**: divIcon personalizado con número y color:
- 🟢 Verde (`#2d7a4f`) → primera parada
- 🔵 Azul navy (`#2d5282`) → paradas intermedias
- 🔴 Coral (`#e05a3a`) → última parada

**Ruta**: `L.polyline` con línea dorada punteada (`#e8a020`, dashArray: `'6 4'`)

### 4.3 Leyenda del mapa

Píldoras con fondo blanco bajo cada mapa mostrando: número coloreado + nombre de la parada.

**⚠️ LECCIÓN APRENDIDA**: Usar concatenación de strings, NO template literals, en `renderLegend` para evitar problemas de escaping:

```javascript
// ✅ CORRECTO — sin template literals
el.innerHTML = d.stops.map(function(s, i) {
  var color = i === 0 ? '#2d7a4f' : (i === d.stops.length-1 ? '#e05a3a' : '#2d5282');
  var name = s.n.replace(/ [🚗🏨🍽✈🌅🎫🔙⚓]/gu, '').trim();
  return '<div class="map-legend-item">'
    + '<div class="map-legend-dot" style="background:' + color + '">' + (i+1) + '</div>'
    + '<span>' + name + '</span>'
    + '</div>';
}).join('');

// ❌ PROBLEMÁTICO — los \${ en capas de escaping Python rompen la interpolación
return `<div><span>\${name}</span></div>`;  // Muestra literal ${name}
```

### 4.4 Fotos Wikipedia

```javascript
async function loadPhotos(d) {
  const strip = document.getElementById('photos-day-' + d.day);
  if (!strip || strip.dataset.loaded) return;
  strip.dataset.loaded = '1';
  for (const p of d.photos) {
    // Crear tarjeta con emoji placeholder mientras carga
    const card = document.createElement('div');
    card.className = 'photo-card';
    card.innerHTML = '<div class="photo-placeholder">' + p.emoji + '</div>';
    strip.appendChild(card);
    // Fetch thumbnail de Wikipedia
    try {
      const slug = encodeURIComponent(p.title.replace(/ /g,'_'));
      const res = await fetch('https://en.wikipedia.org/api/rest_v1/page/summary/' + slug);
      if (res.ok) {
        const data = await res.json();
        if (data.thumbnail?.source) {
          card.innerHTML = '<img src="' + data.thumbnail.source + '" loading="lazy">'
            + '<div class="photo-card-label">' + p.title + '</div>';
        }
      }
    } catch(e) {}
  }
}
```

Fotos cargadas lazily al seleccionar cada día. Sin API key. Imágenes CC-BY-SA.

### 4.5 Links Google Maps

Formato en cada elemento `tl-title`:
```html
<a href="https://www.google.com/maps/search/?api=1&query=Nyhavn%20Copenhagen"
   target="_blank" class="map-link" title="Ver en Google Maps">📍</a>
```

CSS: `.map-link` pequeño, inline-flex, azul con hover, no intrusivo.

### 4.6 Alternativas / Plan B

`<details>/<summary>` HTML puro (sin JavaScript), colapsable:

```html
<details class="day-alternatives">
  <summary>Plan B · 3 alternativas · 1 plan lluvia</summary>
  <div class="alt-list">
    <div class="alt-item plan-b">...</div>
    <div class="alt-item rain">...</div>      <!-- fondo azul -->
    <div class="alt-item emergency">...</div> <!-- fondo coral, solo emergencia -->
  </div>
</details>
```

Tipos de alternativa:
- `plan-b` → borde/badge ámbar — alternativa normal
- `rain` → borde/fondo azul — plan específico para lluvia 🌧
- `emergency` → borde/fondo coral — solo en casos extremos ⚠️

---

## 5. DATOS DE MAPAS — Schema

```javascript
const DAYS_DATA = [
  {
    day: 1,                          // número de día
    emoji: "🇩🇰",                    // emoji del país/tema
    title: "Copenhague · Indre By", // título descriptivo
    zoom: 14,                        // zoom Leaflet (9 para ruta larga, 12-14 para ciudad)
    center: [55.682, 12.589],        // [lat, lng] centro del mapa
    stops: [
      { n: "Nyhavn", lat: 55.6799, lng: 12.5929 },
      // Emojis en el nombre se limpian automáticamente en la leyenda
      { n: "Gorilla 🍽", lat: 55.6617, lng: 12.555 }
    ],
    photos: [
      { title: "Nyhavn", emoji: "🏠" },           // título = artículo Wikipedia
      { title: "Frederiks Kirke", emoji: "⛪" }   // emoji = fallback si falla la foto
    ]
  }
];
```

Notas:
- Para rutas largas (ej. Riviera danesa: CPH → Hillerød → Helsingør → Humlebæk → CPH), zoom 9 y center = punto medio geográfico
- El último stop puede ser igual al primero si es ruta circular (vuelta al punto de partida)
- Las fotos de Wikipedia funcionan mejor con nombres en inglés y exactos

---

## 6. METEOROLOGÍA — Schema y funcionamiento

### 6.1 Schema de datos

```javascript
const WX_DATA = [
  {
    day: 1,
    date: "2026-06-09",            // YYYY-MM-DD
    lat: 55.6761,
    lon: 12.5683,
    city: "Copenhague",
    tz: "Europe/Copenhagen",        // Timezone IANA para Open-Meteo
    stops: [
      { time: "15:35", name: "Llegada CPH" },   // HH:MM de la ruta del día
      { time: "20:30", name: "Cena Gorilla" }
    ]
  }
];
```

### 6.2 API Open-Meteo

URL de la llamada:
```
https://api.open-meteo.com/v1/forecast
  ?latitude=55.6761
  &longitude=12.5683
  &hourly=temperature_2m,apparent_temperature,precipitation_probability,weathercode,windspeed_10m
  &daily=temperature_2m_max,temperature_2m_min,precipitation_sum,windspeed_10m_max,weathercode
  &timezone=Europe%2FCopenhagen     ← encodeURIComponent obligatorio
  &start_date=2026-06-09
  &end_date=2026-06-09
  &models=best_match                ← usa DMI para DK, SMHI para SE, ECMWF global
```

**Totalmente gratuito, sin API key.** Modelos: DMI (Dinamarca), SMHI (Suecia), ECMWF (resto).

Ventana de previsión: **hasta 16 días**. Para viajes próximos, los datos son reales. Para viajes lejanos, aparece "Sin conexión" hasta que entren en ventana.

### 6.3 Caché y refresco

- **Cache key**: `wx_escan_v8` (o versionar por viaje)
- **TTL**: 12 horas (43.200.000 ms)
- **Comportamiento**: carga caché inmediatamente si existe, luego refresca si ha caducado
- **Botón ↻** en cada día para forzar actualización manual

**⚠️ IMPORTANTE**: Cambiar el cache key (`wx_escan_v9`, etc.) cada vez que cambies la estructura de `WX_DATA`, para invalidar datos corruptos del localStorage del usuario.

### 6.4 Temperatura aparente (sensación térmica)

`apparent_temperature` de Open-Meteo ya incluye corrección por viento, humedad y radiación solar. Se muestra como `(s.14°)` cuando difiere de la temperatura real.

### 6.5 Recomendación de ropa

Lógica por temperatura media del día + lluvia + viento:
- ≥22°C → 🩳 Ropa ligera
- 16-22°C → 👕 Camiseta + chaqueta ligera
- 10-16°C → 🧥 Jeans y chaqueta
- <10°C → 🧣 Abrigo y capas
- +3mm precipitación → añadir "Lleva chubasquero"
- +25km/h viento → añadir "Lleva cortavientos"

---

## 7. CHECKLIST — Schema

```javascript
const CHECKS = ['gorilla', 'schonnemann', 'pelikan', /* ... items pendientes */,
                'v1', 'v2', 'h1', /* ... items ya confirmados */];

function toggleCheck(id) {
  const item = document.getElementById('ci-' + id);
  const box  = document.getElementById('box-' + id);
  const done = item.classList.toggle('done');
  // ... actualizar visuals
  localStorage.setItem('chk_' + id, done ? '1' : '0');  // ← prefijo 'chk_' para evitar colisiones
}
```

Grupos sugeridos:
- 🔴 Urgente (ya abierto para reservar)
- 🟠 1-2 semanas antes
- 🟡 Antes de salir / al llegar
- ✅ Ya confirmado (pre-marcado, no toca)

---

## 8. CSS — SISTEMA DE DISEÑO

Variables principales:
```css
:root {
  --navy: #0f1e3a;           /* fondo principal, headers */
  --navy-mid: #1a2f52;       /* gradientes */
  --blue-light: #2d5282;     /* acentos, marcadores intermedios */
  --gold: #e8a020;           /* acentos principales, timeline, fechas */
  --gold-light: #fef3d8;     /* fondos sutiles */
  --coral: #e05a3a;          /* alertas, último marcador */
  --coral-light: #fdf0ec;
  --green: #2d7a4f;          /* confirmado, primer marcador */
  --green-light: #edf7f1;
  --stone: #f4f1eb;          /* fondos neutros */
  --radius: 12px;
  --shadow-sm: 0 1px 3px rgba(0,0,0,.08);
}
```

Tipografía (Google Fonts):
- **Playfair Display** (serif, 400/700/900) → títulos, headers, números grandes
- **DM Sans** (400/500/600) → texto principal
- **JetBrains Mono** (400/600) → localizadores, horas, códigos, teclas

Componentes clave:
- `.card` + `.card-header` + `.card-body` + `.card-footer` → reservas
- `.timeline` + `.timeline-item` → plan del día (con pseudoelemento ::before para línea dorada)
- `.weather-widget` → meteo (border-left de 4px con color según condición)
- `.day-map` → contenedor Leaflet 220px
- `.photo-strip` → scroll horizontal de fotos 140×95px
- `.map-legend` + `.map-legend-item` → leyenda numerada
- `.guide-alert` (coral) / `.guide-tip` (verde) → avisos en la guía

---

## 9. JAVASCRIPT — ARQUITECTURA DE SCRIPTS

El HTML tiene exactamente **3 bloques `<script>` inline** + 1 externo (Leaflet):

```
<script src="leaflet.min.js">       ← CDN, antes de Script 2
<script> Script 1: Nav + Checklist  ← primero, define showSection/showDay/toggleCheck
<script> Script 2: Mapas + Fotos    ← segundo, después de Leaflet JS
<script> Script 3: Meteorología     ← tercero, independiente
```

**Script 1** (navegación y checklist):
- Define `showSection(id, btn)` y `showDay(n, btn)` — **siempre con `btn`**
- Define `toggleCheck(id)` y `loadChecks()`
- Llama `loadChecks()` al final para restaurar estado desde localStorage

**Script 2** (mapas + fotos):
- Define `DAYS_DATA` como JSON inline
- Constantes de color (`C_ROUTE`, `C_START`, `C_END`, `C_MID`)
- `makeIcon(color, num)` → divIcon Leaflet
- `_maps = {}` → almacén de instancias Leaflet
- `ensureMap(d)` → crea o invalida mapa al mostrarse
- `renderLegend(d)` → leyenda numerada
- `loadPhotos(d)` → fotos Wikipedia (async)
- Wrappers de `showSection` y `showDay` para llamar `ensureMap`/`loadPhotos`

**Script 3** (meteorología):
- Define `WX_DATA` como JSON inline
- Funciones con prefijo `wx` para evitar colisiones: `wmoI()`, `wxC()`, `wxHour()`, `wxLC()`, `wxSC()`, `wxRender()`, `wxFetch()`, `wxRefresh()`, `wxInit()`
- `window.wxRefresh` para el botón ↻
- `DOMContentLoaded` → llama `wxInit()`

---

## 10. LECCIONES APRENDIDAS — CRÍTICAS

### L1: window.event / event.target — NUNCA USAR
Firefox eliminó `window.event` en v99 (2022). Safari lo soporta pero con warnings.
```javascript
// ❌ ROMPE en Firefox
function showSection(id) { event.target.classList.add('active'); }

// ✅ SIEMPRE pasar this
onclick="showSection('dias', this)"
function showSection(id, btn) { if (btn) btn.classList.add('active'); }
```

### L2: Leaflet en contenedor oculto
Si el contenedor tiene `display:none` cuando `L.map()` se ejecuta, el mapa queda en 0×0px.
```javascript
// ❌ NO hacer en DOMContentLoaded si la sección está oculta
document.addEventListener('DOMContentLoaded', () => { initMap(); });

// ✅ Hacer cuando el contenedor SE HACE VISIBLE
// Y siempre llamar invalidateSize() después
setTimeout(() => map.invalidateSize(), 60);
```

### L3: Template literals en scripts generados por Python — TRAMPA
Al generar JS desde Python, los template literals tienen múltiples trampas:

```python
# ❌ TRAMPA 1: \` en heredoc bash → la shell interpreta ` como inicio de sustitución
script = """return `<div>${name}</div>`"""  # Puede romperse según el contexto

# ❌ TRAMPA 2: \${ en Python → el JS recibe \${ → bloquea interpolación
return `<span>\${name}</span>`;  # Muestra literal "${name}"

# ❌ TRAMPA 3: \\U0001F327 en Python → el JS recibe \U0001F327 → no es escape JS válido
'\\U0001f327'  # Python: "\U..." = emoji. Con doble \\: Python escribe \U, JS no entiende → U0001f327

# ✅ SOLUCIÓN: usar concatenación de strings en JS generado desde Python
'<span>' + name + '</span>'  # Nunca falla

# ✅ PARA EMOJIS: embeber el carácter directamente
'if(c<=2) return "' + '\U0001f324' + '"'  # Python escribe el emoji literal 🌤
# O usar \u{XXXX} que SÍ entiende JavaScript ES6+:
'\\u{1F324}'  # Python escribe \u{1F324} → JS emoji ✓
```

### L4: Backticks en Python + escaping acumulado
Cada capa de parche sobre el mismo archivo puede introducir caracteres mal escapados:
```
Python string → archivo HTML → JavaScript parser
\`   → ` (OK en Python, FALLA en bash heredoc sin quotes)
\\`  → \` (en archivo) → JS: \ ignorado antes de ` = inicio template literal ROTO
```

**Regla de oro**: Después de cualquier generación de JS, validar con:
```bash
node --check script.js
```

### L5: Acumulación de parches = deuda técnica catastrófica
Tras 8+ rondas de parches sobre el mismo HTML:
- Funciones duplicadas (`initMap` + `_initMapReal` + `ensureMap`)
- Wrappers de wrappers (`_origShowSection` → `_showSectionOrig`)
- Variables con el mismo nombre en scopes distintos
- Cache keys desactualizados

**Regla de oro**: Al tercer parche sobre el mismo código, **regenerar desde cero**.

### L6: Validar JS antes de subir a GitHub
```python
import subprocess, tempfile, os
def validate_js(script):
    with tempfile.NamedTemporaryFile(mode='w', suffix='.js', delete=False) as f:
        f.write("global.window=global; const document={getElementById:()=>null};\n")
        f.write(script); tmp = f.name
    r = subprocess.run(['node', '--check', tmp], capture_output=True, text=True)
    os.unlink(tmp)
    return r.returncode == 0, r.stderr
```

### L7: GitHub Pages CDN cache
Tras cada push, la URL puede tardar 1-3 minutos en actualizarse.
**Siempre hacer Ctrl+Shift+R (hard refresh)** después de un push.

### L8: localStorage — versionar el cache key
Cada vez que cambies la estructura de los datos cacheados:
```javascript
// Incrementar el número de versión en el cache key
var WX_KEY = 'wx_escan_v8';  // Antes era v7, v6, v5...
```
Esto invalida automáticamente datos corruptos del localStorage del usuario.

### L9: Open-Meteo — el timezone hay que encodear
```javascript
// ❌ FALLA con algunos proxies
'&timezone=Europe/Copenhagen'

// ✅ SIEMPRE encodear
'&timezone=' + encodeURIComponent(tz)
```

### L10: Git como red de seguridad
Antes de cualquier modificación compleja, hacer commit de la versión funcional:
```bash
git add . && git commit -m "versión funcional antes de cambios X"
```
Con el historial de git se puede recuperar cualquier versión funcional anterior.

---

## 11. WORKFLOW PARA NUEVOS VIAJES

### Fase 1 — Confirmación de reservas (semanas antes)

1. **Vuelos** → localizador + ruta + hora + terminal + franquicia equipaje
2. **Hoteles** → dirección, metro/transporte, check-in/out, servicios, score
3. **Coche/transporte especial** → compañía, horarios, seguro, condiciones
4. **Actividades prepagas** → referencia, hora, lugar de encuentro

### Fase 2 — Plan operativo

Por cada día:
- Hora de cada parada
- Coordenadas lat/lng (para el mapa)
- Nombre del artículo Wikipedia (para la foto)
- Alternativas Plan B (rain/plan-b/emergency)

### Fase 3 — Generar artefactos

```python
# 1. Generar Word con docx npm
node build_word.js

# 2. Generar Excel con openpyxl
python3 build_excel.py

# 3. Generar HTML desde cero con script limpio
python3 build_html.py

# 4. Validar JS
python3 validate_scripts.py

# 5. Subir a GitHub
cd repo && git add . && git commit -m "..." && git push

# 6. Activar GitHub Pages (primera vez)
curl -X POST -H "Authorization: token TOKEN" \
  https://api.github.com/repos/USER/REPO/pages \
  -d '{"source":{"branch":"main","path":"/"}}'
```

### Fase 4 — Pendientes a reservar

Checklist tipo en casi todos los viajes:
- Restaurantes (1-4 semanas antes según popularidad)
- Entradas atracciones con aforo (con antelación)
- Transporte local (apps a descargar + passes a comprar)
- Coche de alquiler (lo antes posible para precio + disponibilidad)
- Avisos a hoteles (check-in tardío, consigna equipaje)

---

## 12. CONFIGURACIÓN DE PAGOS EN VIAJES INTERNACIONALES

Reglas estándar aplicadas en Escandinavia (adaptables):

### Revolut (moneda local)
- Usar para TODO lo que no sea EUR
- Convertir en día laborable (0% comisión) en lugar de fin de semana (+1%)
- Convertir antes del viaje: cantidad estimada × 1.2 de margen

### Tarjeta Platinum/premium con seguro
- Reservar para pagos en EUR que activen el seguro de viaje (vuelos, coche)
- Verificar condiciones en app del banco antes del viaje

### DCC — Dynamic Currency Conversion
- NUNCA pagar en "su moneda" cuando el datáfono lo ofrezca
- Siempre pagar en moneda local: "DKK please" / "SEK please" / "NOK please"
- Pérdida típica: 5-10% sobre el importe

---

## 13. APIS Y SERVICIOS EXTERNOS USADOS

| Servicio | URL | Auth | Uso |
|---|---|---|---|
| Open-Meteo | api.open-meteo.com/v1/forecast | Ninguna | Previsión horaria |
| Wikipedia REST | en.wikipedia.org/api/rest_v1/page/summary/ | Ninguna | Fotos de atracciones |
| OpenStreetMap | {s}.tile.openstreetmap.org | Ninguna | Tiles del mapa |
| Leaflet.js | cdnjs.cloudflare.com | Ninguna | Librería de mapas |
| Google Maps (links) | maps.google.com/search | Ninguna | Links externos |
| GitHub Pages | pages.github.com | Token PAT | Hosting del HTML |

**Todos los servicios son gratuitos.** El HTML funciona sin servidor propio.

---

## 14. ESTRUCTURA DE CARPETAS DEL PROYECTO CLAUDE

Para cada viaje, crear un proyecto en Claude con:
- Este `claude.md` en la raíz del proyecto
- Conversaciones de planificación dentro del proyecto
- Los documentos generados subidos como referencia

El proyecto Claude permite:
- Memoria entre conversaciones (datos del viaje, reservas, decisiones)
- Contexto persistente para actualizaciones parciales
- Historial de cambios en los documentos

---

## 15. CHECKLIST DE CALIDAD DEL HTML

Antes de dar el HTML como terminado verificar:

```python
assert 'event.target' not in html           # L1: sin window.event
assert html.count('id="wx-day-') == N_DAYS  # Widget meteo por día
assert html.count('id="map-day-') == N_DAYS # Mapa por día
assert html.count('class="alt-item') == N_ALTS  # Alternativas

# Validar JS
for script in scripts:
    assert node_check(script) == OK         # L4: sin errores de sintaxis

# Sin escaping problemático
BACKSLASH = chr(92)
assert (BACKSLASH + chr(36) + '{') not in html  # L3: sin \${ en templates
assert (BACKSLASH + 'U000') not in html          # L3: sin \UXXXXX (Python escape)
```

---

*Última actualización: Junio 2026 — Escandinavia Jun 2026 (Iñaki & Krystel)*
