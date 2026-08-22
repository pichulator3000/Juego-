# Copetapp — contexto técnico

Este documento explica cómo está construida la app y cómo mantenerla. Está pensado para que cualquiera (tú, otra sesión de Claude, otra persona) pueda retomar el proyecto sin tener que releer todo el historial de chat.

## Qué es

Copetapp es una web app de juegos para previas/fiestas, pensada para iPhone. Flujo: se agregan jugadores, se activan modos (Fútbol / Disco) y se indica si hay cartas, y luego se juega por turnos tocando una carta que se voltea y revela un reto, pregunta o juego.

## Estructura de archivos

```
Web app copete/
├── index.html              ← la app completa (HTML + CSS + JS, un solo archivo)
├── CONTEXTO.md              ← este archivo
└── contenido/                ← bancos de contenido en texto plano, editables a mano
    ├── nunca-nunca.txt              (349 frases, sincronizado con el código)
    ├── retos.txt                    (sincronizado)
    ├── cultura-general.txt          (sincronizado, con formato de respuesta)
    ├── trago-por-cada.txt           (sincronizado)
    ├── nombra-regala.txt            (parcialmente editado a mano, ver estado abajo)
    ├── espia-palabras.txt           (sincronizado con ESPIA_WORDS)
    ├── mentiroso-temas.txt          (NO conectado al código todavía)
    ├── cultura-chupistica-temas.txt (NO conectado al código todavía)
    ├── cacho-humano-acciones.txt    (NO conectado al código todavía)
    ├── juegos-peso-y-castigo.txt    (DESCARTADO — no se usa ni se va a conectar, ver "Cómo funciona el motor de cartas")
    ├── eventos-aleatorios.txt       (sincronizado con EVENTOS_ALEATORIOS)
    └── fotos-famosos/               (fotos para el mini-juego "¿Quién es?", ver su LEEME.md)
```

**No hay build step.** `index.html` se abre directo en un navegador o se sube tal cual a cualquier hosting estático (GitHub Pages, Netlify, etc). No usa frameworks, no usa `npm install`, no tiene dependencias externas — todo el CSS y JS están inline en el mismo archivo, con fuentes del sistema (`ui-rounded`, `-apple-system`, `ui-monospace`) para que se vea nativo en iOS sin cargar fuentes externas.

## Por qué los `.txt` no se leen solos

Los archivos de `contenido/` **no se cargan automáticamente en la app.** Cuando la app se previsualiza como Artifact (la vista que usamos para probarla en el chat), corre en un sandbox con política de seguridad (CSP) que bloquea cualquier `fetch()` a archivos externos — no puede leer nada fuera de sí misma. Por eso, aunque en producción (subida a un hosting real) sí se podría hacer un `fetch('contenido/retos.txt')`, hoy el contenido vive **hardcodeado dentro del `<script>` de `index.html`**, en el objeto `DECK`.

Esto significa que el flujo de trabajo real es:
1. Editas un `.txt` en `contenido/` a mano (agregas frases, cambias cosas).
2. Le pides a Claude que sincronice ese archivo al código (o lo haces tú mismo siguiendo la guía de abajo).
3. Claude/tú actualiza el array correspondiente dentro de `DECK` en `index.html`.

### Reglas al sincronizar (pedido explícito, importante)

- **El texto se copia tal cual está escrito, sin "corregir" nada** — ni ortografía, ni acentos, ni mayúsculas/minúsculas, ni singular/plural. Si el `.txt` dice "trabajos", va "trabajos", no "trabajo".
- **Las cartas mostradas no llevan punto final.** Se le aplicó una limpieza global a todo el contenido existente (`DECK` + `EVENTOS_ALEATORIOS`) que saca cualquier `.` que quede pegado justo antes del cierre de comillas — hazlo también a mano si agregas contenido nuevo tú mismo directo en el código.
- **Los paréntesis en los `.txt` son comandos, no texto para mostrar literal.** Indican una instrucción sobre esa línea (ej. `(toma doble)` = marcar esa palabra como premio doble, no escribir "(toma doble)" en la carta; `(N)` al final de una trivia = esa es la respuesta; `(jugador aleatorio)` = reemplazar por un nombre real al momento de jugar). Si un paréntesis es ambiguo entre "comando" y "parte real de la frase" (pasó con "Después de terminar (menos de una semana)"), se usa criterio: si describe una mecánica de juego es comando: si es parte del significado de la frase, se deja.

## Cómo aplicar un cambio de un `.txt` al `index.html`

### Formato de los `.txt` de categorías (nunca, retos, cultura, trago, regala)

Todos siguen la misma estructura de 3 secciones:

```
## NORMAL
frase 1
frase 2

## MODO FUTBOL
frase futbolera 1

## MODO DISCO
frase disco 1
```

`cultura-general.txt` además tiene una línea `R: respuesta` inmediatamente después de cada pregunta que tiene respuesta revelable.

### Pasos para sincronizar a mano

1. Abre `index.html` y busca el objeto `var DECK = {` (cerca de la línea 215). Cada categoría (`nunca`, `retos`, `cultura`, `trago`, `regala`, `juegos`) tiene esta forma:
   ```js
   nombreCategoria: {
     label: 'Nombre visible', icon: '🙊', cls: 'cat-clase-css',
     items: [ /* array NORMAL */ ],
     futbol: [ /* array MODO FUTBOL */ ],
     disco: [ /* array MODO DISCO */ ]
   }
   ```
2. Reemplaza el contenido del array (`items`, `futbol` o `disco`) por las frases del `.txt`, como strings entre comillas simples separados por coma. Si la frase tiene comillas simples adentro, hay que escaparlas o usar comillas dobles para esa línea.
3. Para categorías con respuesta (como `cultura`), cada entrada es un objeto en vez de un string:
   ```js
   { text: 'Pregunta...', answer: 'Respuesta...' }
   ```
4. Guarda y prueba: abre el archivo en el navegador o pídele a Claude que lo publique como Artifact.

### Pasos para sincronizar con ayuda de Claude (recomendado si son muchas líneas)

Decirle algo como *"actualiza el código con los cambios del txt de retos"* es suficiente. Para archivos grandes (decenas o cientos de líneas, como pasó con `nunca-nunca.txt`), Claude usa Node.js para parsear el `.txt` y regenerar el array en el `index.html` automáticamente, en vez de transcribir línea por línea a mano (eso evita errores de tipeo y de comillas). El comando base es:

```js
// lee el .txt, separa por secciones ## NORMAL / ## MODO FUTBOL / ## MODO DISCO,
// ignora líneas vacías o que empiecen con #, y genera un array JS con JSON.stringify
// (que escapa comillas y caracteres especiales automáticamente)
```

Después de cualquier sincronización grande, Claude valida que el JavaScript siga siendo válido con:
```bash
node --check archivo_extraido_del_script.js
```

## Cómo funciona el motor de cartas (`DECK`)

`DECK` tiene 7 categorías: `nunca`, `retos`, `cultura`, `trago`, `regala`, `juegos`, `evento`. Cuando se toca la carta (`drawChallenge()`):

1. Se descarta la última categoría que salió (`state.lastCategory`) para que no se repita dos veces seguidas, y se elige una categoría con `weightedChoice(candidateKeys, k => CATEGORY_WEIGHT[k] || 1)` entre las que quedan. `CATEGORY_WEIGHT` hoy es `{ evento: 6, nunca: 1, retos: 1, cultura: 1, trago: 1, regala: 1, juegos: 1 }` — Evento aleatorio sale la mitad de las veces, y el resto se reparte parejo. Esto fue un pedido explícito: "lo que más debe aparecer es evento aleatorio, después de eso variar parejo". `juegos-peso-y-castigo.txt` sigue sin usarse (ver pendientes) — se descartó ese enfoque a favor de este, más simple.
2. Si la categoría es `evento`, se elige una frase al azar (sin repetir hasta agotar la lista, ver más abajo) de `EVENTOS_ALEATORIOS` y `resolveEvent()` reemplaza `{jugador}` (jugador de turno) / `{jugador1}` / `{jugador2}` (al azar, distintos entre sí) por nombres reales, y el resultado se muestra directo como si fuera un reto.
3. Si la categoría es `juegos`, se filtra la lista según si hay cartas (`state.cards === 'si'`, campo `needsCards`) y se elige uno sin repetir hasta agotar los que califican (todos pesan igual entre sí).
4. Si es cualquier otra categoría, se arma un pool ponderado con `weightedPick(catKey, cat)`:
   - Las frases `items` (normales) siempre entran, peso 1 cada una.
   - Las frases `futbol` **solo entran si `state.futbol === true`**, peso 1 cada una (todo o nada).
   - Las frases `disco` **siempre entran**, pero con peso distinto: `0.2` si `state.disco === false` (aparecen poco, es un "leak" intencional) y `3` si `state.disco === true` (dominan la mezcla, pero sin tapar del todo a las normales).
   - Se hace una selección aleatoria ponderada sobre ese pool, sin repetir hasta agotarlo.

Este comportamiento asimétrico entre fútbol y disco fue un pedido explícito: fútbol es estrictamente opt-in, disco nunca desaparece del todo.

### Anti-repetición

Pedido explícito porque se estaban repitiendo mucho los juegos y las frases. Dos mecanismos combinados:
- **Categoría**: `state.lastCategory` guarda la última categoría elegida y se excluye del sorteo siguiente (no más "evento aleatorio" dos veces seguidas).
- **Frases/juegos**: `pickWithHistory(historyKey, weightedEntries)` guarda en `state.history[historyKey]` la identidad (`identityOf`, que usa `.text` o `.name` según corresponda) de cada cosa que ya salió en esa categoría. Mientras queden opciones sin usar, solo elige entre esas; cuando se agotan todas, reinicia el historial de esa categoría y vuelve a barajar desde cero. Esto aplica por separado a cada categoría de `DECK` y a la lista de `juegos`. `state.history` se reinicia completo al apretar "Empezar" en el armado.

Una entrada de `items`/`futbol`/`disco` puede ser un string simple o un objeto (`toEntry()` normaliza ambos). Los objetos permiten campos extra que viajan hasta `state.card`:
- `{ text, answer }` → categoría con pregunta y respuesta revelable (`cultura`, y buena parte de `trago.futbol`), ver "Mostrar respuesta" más abajo. Si `answer` no viene o es `null`, no aparece el botón.
- `{ text, special: 'word-drop', rounds }` → activa el mini-juego de palabras (la carta de besos en `trago.disco`), ver siguiente sección.
- `{ text, special: 'naming-challenge', topics }` → activa el desafío de nombrar contra el tiempo (la carta "..." de `trago.futbol`), ver más abajo.
- `{ text, image, answer }` → mini-juego "¿Quién es?" (varias cartas en `cultura.items`), ver más abajo.

## Mini-juego "1 trago por cada" (word-drop)

La carta de besos en `trago.disco` no es texto plano: es `{ text, special: 'word-drop', rounds: [{ label, words[] }, ...] }`. Cuando sale esa carta, el botón cambia a "Comenzar" (`start-word-drop`) en vez de avanzar el turno. `startWordDrop()`:

1. Por cada ronda en `rounds` (hoy "Lugares" y "Personas"), baraja sus palabras y toma 5.
2. Concatena todo en `state.wordDrop.words` como `{ round, word }`, en el orden de las rondas (no se mezclan entre sí).
3. Corre una cuenta regresiva de 3s (`tickCountdown`) y después va revelando cada palabra cada 5s (`scheduleNextWord`), ambas con cadenas de `setTimeout` que se auto-cancelan si `state.wordDrop` cambia (por ejemplo si el jugador vuelve a `setup` a mitad de la secuencia).

El badge de la carta muestra la ronda actual (`Ronda: Lugares` / `Ronda: Personas`) para que se sepa qué se está preguntando. Al terminar las 10 palabras, aparece "Listo, siguiente turno".

Algunas palabras vienen como `{ text, double: true }` en vez de string simple (las que en el `.txt` traían `(toma doble)` — ese paréntesis es un comando, no texto literal, así que no se muestra tal cual). Cuando le toca el turno a una palabra `double`, aparece un tag "¡DOBLE!" arriba de la palabra.

## Desafío de nombrar contra el tiempo (naming-challenge)

Una carta especial dentro de `trago.futbol` (`{ text, special: 'naming-challenge', topics: [...] }`, ~100 temas de fútbol) reemplaza el botón por "Comenzar" (`start-naming-challenge`). `startNamingChallenge()` elige un tema al azar de `topics` y arranca `state.naming = { topic, seconds: 30, phase: 'running' }`; `tickNaming()` descuenta 1 segundo por `setTimeout` hasta llegar a 0, mostrando el tema y el número grande de segundos restantes. Al llegar a 0, `phase` pasa a `'done'` y aparece "Listo, siguiente turno". Mismo patrón de cadena de `setTimeout` auto-cancelable que `word-drop`.

## "¿Quién es?" (dentro de Cultura general)

Cartas como `{ text: '¿Quién es?', special: 'who-is-it', image: 'https://upload.wikimedia.org/...', answer: 'Leonardo da Vinci' }` no disparan un flujo aparte — se renderizan dentro del mismo branch normal de carta (no como `word-drop`/`naming-challenge`), solo que en vez del emoji de categoría muestran `<img src="{image}">` en un círculo. Usan el mismo botón "Mostrar respuesta" que cualquier `{text, answer}`.

Si la imagen no carga (`onerror` del `<img>`), se oculta y aparece una silueta 👤 en su lugar — no rompe nada, solo queda sin foto.

Las 20 fotos de hoy están enlazadas directo a Wikimedia Commons (no son archivos locales ni copias en el repo) — cada URL se verificó a mano (licencia de dominio público o CC BY/CC BY-SA) antes de usarla, justamente para no redistribuir fotos de personas reales sin derechos claros. Los créditos completos están en `contenido/fotos-famosos/LEEME.md`. Esa carpeta quedó vacía de archivos a propósito; solo tiene el `LEEME.md` con la tabla de créditos y las instrucciones para agregar más personas o cambiar el mecanismo a archivos locales si se prefiere.

## Ronda interactiva de Impostor

Se llamaba "Espía" y se renombró a "Impostor" (pedido explícito) — el nombre interno de funciones/CSS/estado (`startEspia`, `state.espia`, `ESPIA_WORDS`, `.cat-espia-spy`) se dejó igual a propósito (son detalles internos, no texto visible), pero todo lo que ve el jugador dice "Impostor".

Cuando el juego elegido es "Impostor" (`state.card.isSpy === true`), el botón de la carta cambia a "Empezar ronda de Impostor" (`start-espia`) en vez de avanzar el turno directamente. Esto dispara `startEspia()`:

- Elige una palabra al azar de `ESPIA_WORDS.normal` (+ `ESPIA_WORDS.futbol` si el modo fútbol está activo) y la resuelve con `resolveEspiaWord()`. Los paréntesis ahí también son comandos, no texto literal: `'(Uno de los jugadores al azar)'` reemplaza toda la palabra por el nombre de un jugador real; `'(jugador aleatorio)'` dentro de una frase (ej. `"Ex de (jugador aleatorio)"`) se reemplaza solo esa parte.
- Baraja el orden de los jugadores (`shuffle`).
- **5% de las veces, todos son el Impostor** (`allImpostors`) — ronda especial sin palabra real, pensada para variar. El otro 95%, hay 1 Impostor normalmente, o **2 si hay más de 6 jugadores** (`state.players.length > 6`).
- Guarda todo en `state.espia = { word, spyIndexes, allImpostors, order, step, revealed }` — `spyIndexes` es un array (antes era un solo índice) para soportar más de un Impostor.

`renderEspiaStage()` muestra, jugador por jugador (según `order`), una carta que se voltea: todos ven la palabra secreta, excepto quien esté en `spyIndexes`, que ve "Eres el Impostor". Al terminar de recorrer a todos (`step >= order.length`), se muestra "Ahora todos dicen una palabra uno por uno" y un botón para volver al flujo normal de turnos (`finish-espia`).

La sección `## Modo cabros` de `espia-palabras.txt` (nombres propios del grupo de amigos) **no está conectada** — no hay un tercer toggle de modo para eso hoy, solo Fútbol y Disco. Pendiente de definir qué hacer con esa lista.

## Estado de la app (`state`)

```js
{
  players: [{ name, color }],   // color es un var(--pink) etc, rota entre 4 colores
  futbol: bool, disco: bool, cards: 'si'|'no'|null,
  screen: 'setup' | 'play',
  turn: number,        // índice del jugador actual
  round: number,        // contador de turnos jugados (solo visual)
  flipped: bool,        // si la carta actual está volteada
  card: {...} | null,    // la carta actual (cat, icon, text, sub, answer, isSpy, special, rounds, topics, image)
  answerShown: bool,      // si se reveló la respuesta de la pregunta actual
  espia: {...} | null,     // estado de la sub-ronda de Impostor, null si no está activa
  lastCategory: string|null, // última categoría elegida, para no repetirla
  history: {}               // { [categoria o 'juegos']: [identidades ya salidas] }, ver Anti-repetición
}
```

**Indicador de turno:** el nombre de quien tiene el turno ya no se muestra afuera de la tarjeta — va adentro, como una píldora brillante (`.turn-glow`) arriba de todo el contenido de la carta (badge, foto, texto), tanto en la cara frontal ("?") como en la trasera. Se armó como una píldora con fondo semi-transparente en vez de solo texto con `text-shadow`, porque el color de fondo de la carta cambia según la categoría (`cat-nunca` es oscuro, `cat-cultura` es claro) y un brillo de texto fijo se veía mal en la mitad de los casos.

Todo el render es una función pura de `state`: cada acción del usuario muta `state` y llama a `render()`, que decide entre `renderSetup()` y `renderPlay()` (y dentro de esta, `renderEspiaStage()` si `state.espia` no es null). No hay ningún framework — es un patrón manual de "estado → HTML string → `innerHTML`", con **delegación de eventos**: un solo listener de `click`/`submit`/`change` en `#app` que lee el atributo `data-action` de lo que se tocó.

## Diseño / paleta

Paleta fija (no cambia con el tema claro/oscuro del sistema — es una decisión de diseño deliberada, ambiente de fiesta nocturna):

- `--ink` `#150C2B` fondo base
- `--pink` `#FF3D8F`, `--lime` `#D8FF3E`, `--cyan` `#2FE6C4`, `--violet` `#8B5CF6` — acentos
- `--cloud` `#F8F4FF` texto principal, `--muted` `#B6A6DA` texto secundario

Cada categoría tiene su color de carta (`cat-nunca`, `cat-retos`, etc., ver CSS cerca de la línea 179). Tipografías: `ui-rounded` para títulos/marca (se ve como SF Rounded en iOS/Safari), `-apple-system` para texto de UI, `ui-monospace` para números (contador de turno, etc).

La app está optimizada para safe areas reales de iPhone (`env(safe-area-inset-*)`, `viewport-fit=cover`) — no simula un iPhone visualmente, es la página real.

## Cómo probar cambios

- Abrir `index.html` directo en un navegador (doble clic, o arrastrar al navegador) — funciona sin servidor.
- O pedirle a Claude que lo publique como Artifact para verlo en el chat.

## Git / GitHub

- Repo: `https://github.com/pichulator3000/Juego-`
- Rama única: `master`
- El repo se inicializó localmente en esta carpeta (`git init` + `git remote add origin ...`) — no viene de un `git clone`.
- Flujo normal: editar → `git add` → `git commit` → `git push origin master`. La primera vez se usó `--force` porque el repo tenía un proyecto anterior distinto ("Copetito") que se reemplazó por completo, previa confirmación.

## Pendiente / no conectado todavía

- **`juegos-peso-y-castigo.txt`**: descartado por pedido explícito, no se va a usar. La forma en que se controla qué tan seguido aparece cada cosa hoy es `CATEGORY_WEIGHT` (categorías) — dentro de `juegos` los 9 juegos pesan igual entre ellos (solo hay anti-repetición, no pesos distintos).
- **`mentiroso-temas.txt`, `cultura-chupistica-temas.txt`, `cacho-humano-acciones.txt`**: siguen con el contenido de ejemplo original, nadie los ha llenado en serio todavía, y aunque se llenen, hoy esos 3 juegos solo muestran un texto de reglas fijo — no hay ninguna mecánica en la app que saque un tema/palabra/acción al azar de estos bancos para mostrarla en pantalla (a diferencia de Espía, que sí tiene ese mecanismo con `ESPIA_WORDS`). Si se quiere que, por ejemplo, Cultura Chupística muestre un tema al azar en pantalla, hay que construirle un flujo similar al de `startEspia()`.
- **`nombra-regala.txt`**: fue editado a mano con contenido nuevo (menciona Colo-Colo, la U, Premier League, Mundial 2010) pero **todavía no está sincronizado** al `DECK.regala` del código — el código sigue con las frases genéricas originales.
