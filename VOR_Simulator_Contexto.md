# Proyecto: Simulador VOR para iPhone

## Descripción general
App web (PWA / HTML) sin costo, para uso personal en iPhone, que simula la lectura de un VOR usando el GPS real del teléfono.

## Funcionalidades acordadas

- **GPS real del teléfono**: usa la Web Geolocation API (compatible con Safari/iPhone, solicita permiso al usuario)
- **VORs cargados manualmente**: usuario ingresa coordenadas geográficas (lat/lon) de uno o varios VORs
- **Declinación magnética**: parámetro ajustable manualmente (valor casi fijo por zona)
- **Cálculo de radial**: trigonometría de bearing entre posición GPS y coordenadas del VOR
- **Instrumento HSI**: Horizontal Situation Indicator con rosa de compass que rota con el track GPS
- **Aguja RMI**: bearing pointer que señala al VOR en todo momento (verde `#28d858`)
- **CDI needle**: deflexión perpendicular al course bar (amarillo `#f0c040`), escala ±10° = tope
- **Flag TO/FROM**: según posición relativa al VOR y curso seleccionado
- **Pseudo DME**: distancia calculada GPS→VOR en nm (sin hardware)
- **Modo HOLD**: circuito de espera dibujado sobre el HSI, orientado al rumbo de acercamiento (v11)
- **Dos pantallas**: pantalla principal HSI limpia / pantalla de configuración para VOR y declinación

## Stack técnico
- HTML + CSS + JavaScript puro (sin frameworks)
- Web Geolocation API (`watchPosition`, `coords.heading` para track GPS)
- Sin backend, sin costo, sin App Store
- Se puede usar como PWA (agregar a pantalla de inicio en iPhone)

## Flujo de desarrollo
- Desarrollo y previsualización en PC
- Prueba final en iPhone vía Safari

## NAV AIDs cargados (fuente: ANAC / lista NAV.md)

| ID  | Nombre  | Lat (°)   | Lon (°)    | Frec. / tipo | magVar default |
|-----|---------|-----------|------------|--------------|----------------|
| PAR | Paraná  | -31.80833 | -60.48472  | 116.8        | -9.55          |
| SVO | Sauce V | -31.71144 | -60.80639  | NDB          | -9.55          |
| ROS | Rosario | -32.90500 | -60.78138  | 117.3        | -9.55          |
| CBA | Córdoba | -31.31333 | -64.20362  | 114.5        | -7.40          |

Fuente coordenadas: verificadas contra OurAirports.com. Declinaciones: cartas ANAC (ver lista NAV.md).

La estructura NAVAIDS está preparada para agregar otros tipos (NDB, etc.) sin cambios adicionales.

## Declinación magnética
- **Por NAV AID**: cada navaid tiene su `magVarDefault` según carta ANAC
- **localStorage por navaid**: clave `magVar_PAR`, `magVar_ROS`, `magVar_CBA` (override individual)
- Al seleccionar un navaid en configuración → se carga su magVar en el input editable
- Fórmula: `magBearing = trueBearing - magVar` (con magVar negativo para variación oeste)

## Lógica CDI / HSI

### Bearing
```
trueRadial = bearing(vor → aircraft)   // rumbo verdadero VOR→avión
magRadial  = norm360(trueRadial - magVar)  // radial magnético que ocupa el avión
brgToVOR   = norm360(magRadial + 180)      // rumbo magnético del avión al VOR
```

### TO/FROM y aguja CDI
```
toDev   = normPM180(brgToVOR - OBS)    // desviación respecto al curso TO
fromDev = normPM180(magRadial - OBS)   // desviación respecto al curso FROM

|toDev| < 87°  → flag TO,   aguja = +toDev / 10  (±1 = full scale)
|toDev| > 93°  → flag FROM, aguja = -fromDev / 10
87°–93°        → flag OFF (zona ambigua)
```

### HSI rotaciones SVG
Todos los grupos giran alrededor de (150,150). Se dibujan **sin rotar apuntando al norte**
y el `transform` los orienta; no hay recálculo de geometría en JS.
```
Compass rose:  rotate(-magHeading, 150, 150)
Course group:  rotate(obs        - magHeading, 150, 150)
RMI needle:    rotate(brgToVOR   - magHeading, 150, 150)
Heading bug:   rotate(hdgBug     - magHeading, 150, 150)
Circuito HOLD: rotate(holdCourse - magHeading, 150, 150)
```

Cuando el track GPS no está disponible (avión quieto): `magHeading = 0` → modo North-Up.
Condición exacta: `useTrack = acLat !== null && gpsTrack !== null && gpsSpeed > 0.5`.

## Layout actual — landscape

La app pasó de vertical a **landscape fijo** (`.screen` es un flex row). Tres columnas:

```
┌─────────┬──────────────────────────────┬─────────┐
│  OBS    │  top-bar: entrada HOLD │ TO/FROM        │
│  000°   │                              │  HDG    │
│ [+180]  │         HSI (SVG)            │  000°   │
│  ▲▲     │      viewBox 14 14 272 272   │ [▶ VAR] │
│  ▲      │                              │ [+180]  │
│  ▼      │  info-bar: [PAR] 10.3nm  R224│  ▲▲ ▲   │
│  ▼▼     │                              │  ▼ ▼▼   │
└─────────┴──────────────────────────────┴─────────┘
 side-obs         center-panel            side-hdg
  155px         flex:1 (crece)             155px
```

- Los paneles laterales son columnas de 155px con los botones flecha apilados verticalmente
- El HSI usa `flex:1` y llena todo el alto disponible
- `env(safe-area-inset-left/right)` en los paneles para el notch del iPhone
- El badge de la NAV AID y el encabezado HDG son **tocables** (v11) — ver más abajo

## Modo VAR — ajustar declinación en vuelo

El botón `▶ VAR` del panel derecho permite corregir la declinación magnética **sin entrar
a configuración**, útil en vuelo con guantes o turbulencia.

- Al activarlo (`toggleMagVarEdit()`), el panel derecho se pinta naranja `#ff9900`
  y el display pasa a mostrar la declinación en vez del heading bug
- Las flechas ▲▼ cambian la declinación de a **0.1°** (`delta * 0.1`)
- El botón `+180` se convierte en `✓ LISTO`; al salir, guarda en `localStorage.magVar_<ID>`
- No convive con el modo HOLD: entrar a HOLD cierra VAR primero

## Auto-actualización

La app se actualiza sola sin intervención — importante porque en el iPhone la PWA cachea
agresivamente:

- `checkForUpdate()` hace un `HEAD` sobre `location.href` y compara `etag` / `last-modified`
- Si cambió respecto de la última vez, hace `location.reload(true)`
- Corre al cargar, cada 60s, y en cada `visibilitychange` a visible

## Wake Lock

La pantalla no se apaga en vuelo:
- Re-adquisición automática en el evento `release`
- Activación en el primer `touchstart` (iOS lo exige tras interacción del usuario)
- `setInterval` cada 30s como red de seguridad
- Si aun así se apaga: Ajustes → Pantalla y brillo → Bloqueo automático → Nunca

## Archivos del proyecto

| Archivo | Descripción |
|---------|-------------|
| `vor_hsi.html` | **La app entera** — HTML+CSS+JS en un solo archivo autocontenido |
| `VOR_Simulator_Contexto.md` | Este documento |
| `lista NAV.md` | NAV AIDs con coordenadas y declinaciones de carta ANAC |
| `subir_a_github.bat` | Doble-click → sube `vor_hsi.html` a GitHub |
| `subir_a_github.ps1` | Script usado por el .bat. **Contiene el token — nunca subirlo al repo** |
| `Instructivo_Instalacion_VOR.md/.pdf` | Instructivo de instalación en el iPhone |

## Deploy / Acceso desde iPhone

- **URL pública:** `https://acassano-hub.github.io/vor-simulator/vor_hsi.html`
- **Repositorio:** `https://github.com/acassano-hub/vor-simulator` (público)
- **Subida:** doble-click en `subir_a_github.bat`, o
  `.\subir_a_github.ps1 -Message "texto del commit"` para mensaje descriptivo
- **No hay repo git local.** El script hace un PUT a la API Contents de GitHub;
  cada PUT genera un commit. No hay `gh` CLI ni credenciales git guardadas.
- **Si el deploy devuelve 401:** venció el PAT (ya pasó en ago-2026). Generar uno nuevo
  en github.com/settings/personal-access-tokens con *Contents: Read and write* sobre
  `vor-simulator` y pegarlo en la línea 3 del `.ps1`. Alternativa sin token: editar/subir
  el archivo desde la web de GitHub.
- **Caché:** la app se auto-actualiza sola (ver abajo); si igual queda vieja, agregar `?vN`
- **PWA:** en Safari → Compartir → Agregar a pantalla de inicio

## Verificar cambios visuales antes de publicar

No hay Chrome en la PC, pero sí Edge headless. Conviene usarlo: así se detectaron
sectores de entrada mal calculados en v11 que la sola revisión del código no encontró.

```
"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" --headless=new ^
  --disable-gpu --hide-scrollbars --force-device-scale-factor=2 ^
  --screenshot=out.png --window-size=880,410 "file:///ruta/test.html"
```

Se copia el HTML a un temporal inyectando estado simulado antes de `</script>`
(`S.acLat`, `S.acLon`, `S.gpsTrack`, `S.gpsSpeed > 0.5` para que haya track, y `update()`).
880×410 aproxima el iPhone en landscape, que es el layout real.
Una sola invocación de Edge por comando — encadenar varias en un loop falla en silencio.

## Estado

- [x] v1: CDI básico con GPS, TO/FROM, selector de VOR, OBS
- [x] v2: HSI con rosa de compass, RMI, pseudo-DME, pantalla de configuración
- [x] v3: Course bar con gap para CDI, heading bug magenta, RMI doble barra verde, declinación 0.05°, textos más grandes
- [x] v4: Compass x2 todo blanco, botón +180 en OBS, botones simplificados (◀◀◀▶▶▶), heading bug solo triángulo
- [x] v5: SVG full-width (ocupa toda la pantalla del iPhone), controles OBS/HDG en 2 líneas, botones más grandes, deploy en GitHub Pages + script de subida automática
- [x] v6: colores por instrumento (OBS amarillo, HDG cyan, DME rojo), course bar + CDI amarillos, bug cyan, Wake Lock robusto
- [x] v7: localStorage para magVar (persiste entre sesiones), heading bug con forma de herradura clásica (Π)
- [x] v8: mejoras visuales — DME/OBS/HDG +20% tamaño, TO/FROM junto a OBS (fuera del compás), VOR badge +50% verde RMI (#28d858), ticks de 10° del compás en blanco
- [x] Prueba en vuelo real: exitosa
- [x] v9: alto contraste para uso con sol — (1) textos de config 2× en general; (2) "Volver" y declinación magnética 2.5×; (3) radial blanco bold 3× (54px); (4) OBS 1.5× (54px), botón +180 blanco/negro bold 1.5×; (5) triángulos OBS/HDG 2× (40px), tamaño de botones sin cambios; settings-body scrollable
- [x] v9b: refinamientos UI — (1) coords eliminadas de cards VOR; (2) magvar label "W = (-)" + input 210px; (3) triángulos −20% (32px), 1° blandos (#aaa), 10° contorno (-webkit-text-stroke); (4) ticks CDI → 10 líneas blancas (5 por lado, ±2°/4°/6°/8°/10°, cada 2°), sin tick central, sin líneas horizontales en símbolo avión; (5) TO/FROM margin-left 28px; DME #ff5555
- [x] v10: (1) compás: labels solo cada 30°, más chicos (cardinals 26px, numéricos 20px), más alejados del bisel (tR=R-42); (2) renombrado VORS→NAVAIDS, label "NAV AID activa", badge blanco #e8ecf4 (preparado para NDB y otros tipos); (3) magVar por navaid (valores ANAC en lista NAV.md, localStorage `magVar_ID`, input editable al seleccionar en config, frecuencia reemplazada por magVar en las cards); (4) controles OBS y HDG envueltos en rectángulos redondeados (.ctrl-group), botón +180 agregado a HDG; recuadro OBS en amarillo #f0c040
- [x] v10b: SVO (Sauce V, NDB) agregado al listado. Settings: GPS fijo arriba separado por línea, listado NAV AIDs + declinación en zona scrollable independiente (preparado para crecer sin perder acceso al GPS)
- [x] v11: (1) el engranaje ⚙ se eliminó — ahora se toca el **badge de la NAV AID** (PAR/ROS/…) para entrar a configuración; (2) **modo HOLD**: tocar el encabezado HDG conmuta a circuito de espera y de vuelta
- [ ] Posibles mejoras futuras: agregar NDBs y otros NAV AIDS desde lista NAV.md, modo simulación sin GPS, heading por brújula del teléfono, tiempos de pata / cronómetro en el HOLD

## Diseño v11 — modo HOLD (circuito de espera)

### Acceso a configuración
- Botón `⚙` y su clase `.btn-settings` **eliminados**
- `#vor-badge` lleva `onclick="showSettings()"` + recuadro `1px #4a5568` como pista de que es tocable
- El hueco liberado en la top-bar lo ocupa `#hold-entry`

### Conmutador HDG ⇄ HOLD
- El `.ctrl-header` del panel derecho lleva clase `.tap` y `onclick="toggleHoldMode()"`
- En HOLD: la etiqueta pasa a `HOLD`, todo el panel se pinta violeta `#d060ff` (clase `.hdg-holdmode`)
- Las flechas ▲▼ cambian el **rumbo de acercamiento** en vez del heading bug; `+180` da el recíproco
- El heading bug cyan **sigue visible y en su valor** — se pone el bug antes de pasar a HOLD
- El botón `▶ VAR` se reconvierte en el selector de sentido `↻ DER` / `↺ IZQ` (`varBtnTap()` despacha según modo)
- `S.holdCourse` arranca tomando el valor de `S.obs` la primera vez, después conserva el suyo
- El modo VAR y el modo HOLD no conviven: entrar a HOLD cierra VAR

### Geometría del circuito (grupo `g-hold`)
Se dibuja **sin rotar** con la pata de acercamiento apuntando arriba y el fijo en el centro (150,150);
`rotate(holdCourse - magHeading, 150, 150)` lo orienta, igual que el course bar.

```
HOLD_R = 22   (radio de viraje)     HOLD_L = 58   (largo de patas)
s = derecha ? +1 : -1     sweep = derecha ? 1 : 0
M 150,150  A R,R 0 0 sweep  xo,150  L xo,yb  A R,R 0 0 sweep  150,yb  Z
   xo = 150 + s·2R        yb = 150 + L
```
- Radio máximo ocupado ≈ 84px → entra holgado dentro del anillo de rótulos (r=98)
- Flechas de sentido a media pata; anillo `r=10` marcando el fijo bajo el símbolo de avión
- `g-hold` va **después** del course bar y **antes** del avión, para quedar visible arriba de todo

### Sectores de entrada (regla 70°/110°)
`d = derecha ? norm360(rumbo − curso) : norm360(curso − rumbo)`

| d | Entrada |
|---|---------|
| 0–180° | DIRECTA (sector de 180°) |
| 180–290° | PARALELA (110°, lado de no espera) |
| 290–360° | GOTA (70°, pegado al lado de la espera) |

Se muestra en la top-bar izquierda solo en modo HOLD; sin track GPS válido muestra `ENTRADA —`.

## Diseño HSI v4 — detalles visuales

### Course bar (g-course group, unrotated = bar apunta norte/arriba)
- Flecha superior: polygon y=62→84, shaft y=84→122 (se detiene antes del gap)
- **GAP central: y=122 a y=178** (56px) — aquí flota la aguja CDI
- Cola inferior: shaft y=178→228, crossbar y=224
- CDI needle: línea vertical y=122→178, se desplaza en X (±55px = full scale 10°), color magenta #e040fb

### RMI (g-rmi group) — doble barra verde
- Cabeza: diamante sólido polygon, color #28d858
- Doble línea x=143 y x=157 de cabeza al centro
- Barra transversal en y=150
- Cola doble (opacidad 50%) con remate en y=204

### Heading bug (g-hdg-bug group) — magenta, v4
- **Solo triángulo**, sin colita: `<polygon points="150,14 143,38 157,38" fill="#e040fb" opacity="0.95"/>`
- Rota independiente: `rotate(norm360(hdgBug - magHeading), 150, 150)`
- Controles: ◀◀ ◀ ▶ ▶▶ (sin etiquetas numéricas)
- Readout pequeño en SVG (abajo izquierda): "HDG 000°"

### Rosa de compass (v4 — todo blanco, x2 tamaño)
- Labels cardinales: **34px bold** (#e8ecf4)
- Labels numéricos (03, 06, etc.): **28px** (#e8ecf4) — todos blancos (antes grises)
- Ticks mayores 15px, medios 10px, menores 5px

### Botones OBS (v4)
- ◀◀ (−10°) / ◀ (−1°) / ▶ (+1°) / ▶▶ (+10°) / **+180** (recíproco)
- Sin etiquetas numéricas; solo "+180" escrito en el botón recíproco

### Botones HDG (v4)
- ◀◀ (−10°) / ◀ (−1°) / ▶ (+1°) / ▶▶ (+10°)

### CSS tamaños labels (v4)
- `.dme-tag`: 24px, blanco
- `.dme-nm`: 28px, blanco
- `.ctrl-tag`: 22px, blanco, width 54px

## Diseño HSI v5 — cambios respecto a v4

### SVG
- `width="100%"` (antes 300px fijo) — se escala al ancho del iPhone automáticamente
- `style="display:block"` para eliminar espacio extra bajo el SVG

### Controles OBS / HDG (2 líneas cada uno)
```
OBS  000°          ← ctrl-header-row
◀◀  ◀  ▶  ▶▶  +180  ← ctrl-btns (ancho completo)
                   ← ctrl-divider (6px)
HDG  000°
◀◀  ◀  ▶  ▶▶
```
- `.ctrl-btn`: padding 13px, font-size 20px (antes 10px/12px)
- `.ctrl-val`: font-size 30px, sin width fijo
- Márgenes verticales reducidos (vor-badge, dme-row) para que todo entre en pantalla

### Script de subida a GitHub
- `subir_a_github.ps1`: usa `$PSScriptRoot` para ruta relativa (evita problema con ó en el path)
- Token almacenado en el .ps1 (no compartir el archivo)

## Diseño HSI v6 — colores por instrumento

| Elemento | Color | Notas |
|----------|-------|-------|
| Course bar (flecha, shaft, cola) | `#f0c040` amarillo | antes blanco |
| CDI needle | `#f0c040` amarillo | antes magenta |
| OBS display | `#f0c040` bold | consistente con course bar |
| RMI aguja | `#28d858` verde | sin cambios |
| HDG bug triángulo | `#00e5ff` cyan | antes magenta |
| HDG display | `#00e5ff` bold | consistente con bug |
| DME valor | `#ff4444` rojo bold | antes ámbar |
| HDG readout SVG | eliminado | queda solo el display del panel |

## Diseño HSI v8 — mejoras visuales

### Tamaños de indicadores (+20%)
| Elemento | Antes | Después |
|----------|-------|---------|
| `.dme-tag` | 24px | 29px |
| `#dme-val` | 36px | 43px |
| `.dme-nm` | 28px | 34px |
| `.ctrl-tag` (OBS/HDG label) | 22px | 26px |
| `.ctrl-val` (OBS/HDG valor) | 30px | 36px |

### TO/FROM reubicado
- **Eliminado** del interior del SVG (donde ocupaba espacio dentro del compás)
- **Agregado** como `<span id="tofrom-text">` en la fila header de OBS, al lado del valor numérico
- JS actualizado: `tf.style.color` en lugar de `tf.setAttribute('fill', ...)` (elemento HTML, no SVG)
- Estilo: 26px, bold, Courier New; amarillo `#f0c040` cuando TO/FROM activo, gris `#333` cuando OFF

### VOR badge (arriba)
- `font-size`: 18px → **27px** (+50%)
- `color`: `#5a8acc` → **`#28d858`** (mismo verde que la aguja doble del RMI)

### Ticks de 10° del compás
- Color: `#454850` (gris oscuro) → **`#e8ecf4`** (blanco)
- Solo afecta las subdivisiones de cada 10° (isMed), no los ticks mayores (cada 30°) ni los menores (cada 5°)

## Diseño HSI v7 — cambios respecto a v6

### Declinación magnética persistente (localStorage)
- `S.magVar` se inicializa leyendo `localStorage.getItem('magVar')` (default `-7.5` si no hay valor guardado)
- Al cambiar el input en configuración: `localStorage.setItem('magVar', S.magVar)`
- En `INIT`: `$('magvar-input').value = S.magVar` para que el campo refleje el valor guardado
- Funciona en Safari/iPhone (localStorage disponible en PWAs)

### Heading bug (v7) — herradura clásica (Π)
- Reemplaza el triángulo anterior por un path SVG en forma de Π (dos patas + barra superior)
- `<path d="M 140,11 L 160,11 L 160,36 L 154,36 L 154,17 L 146,17 L 146,36 L 140,36 Z" fill="#00e5ff" opacity="0.92"/>`
- Dimensiones: 20px de ancho, 25px de alto, patas de 6px de espesor, apertura hacia el centro del HSI
- Color y rotación: igual que antes (#00e5ff cyan, rota independiente del compass)

## Notas técnicas
- `coords.heading` del API de geolocation = track **verdadero** (no heading magnético).
  Se convierte: `magHeading = gpsTrack - magVar`
- El CDI needle en el grupo SVG del course bar se desplaza en X (perpendicular al bar
  rotado) → la geometría la maneja el transform SVG, no código adicional
- **Deploy desde Claude Code:** en la sesión del 13-ago-2026 se ejecutó
  `subir_a_github.ps1` directamente y funcionó — sí hay salida a internet.
  (Una nota vieja de v7 decía lo contrario; era incorrecta.)
- Este documento se desincronizó del código entre v10b y v11: no registraba el layout
  landscape ni el modo VAR. **Ante una discrepancia, el HTML manda.** Conviene
  actualizarlo en la misma sesión en que se cambia el código.
- El proyecto se retoma subiendo este archivo a una nueva sesión de Claude
