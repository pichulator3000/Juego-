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
    └── juegos-peso-y-castigo.txt    (NO conectado al código todavía)
```

**No hay build step.** `index.html` se abre directo en un navegador o se sube tal cual a cualquier hosting estático (GitHub Pages, Netlify, etc). No usa frameworks, no usa `npm install`, no tiene dependencias externas — todo el CSS y JS están inline en el mismo archivo, con fuentes del sistema (`ui-rounded`, `-apple-system`, `ui-monospace`) para que se vea nativo en iOS sin cargar fuentes externas.

## Por qué los `.txt` no se leen solos

Los archivos de `contenido/` **no se cargan automáticamente en la app.** Cuando la app se previsualiza como Artifact (la vista que usamos para probarla en el chat), corre en un sandbox con política de seguridad (CSP) que bloquea cualquier `fetch()` a archivos externos — no puede leer nada fuera de sí misma. Por eso, aunque en producción (subida a un hosting real) sí se podría hacer un `fetch('contenido/retos.txt')`, hoy el contenido vive **hardcodeado dentro del `<script>` de `index.html`**, en el objeto `DECK`.

Esto significa que el flujo de trabajo real es:
1. Editas un `.txt` en `contenido/` a mano (agregas frases, cambias cosas).
2. Le pides a Claude que sincronice ese archivo al código (o lo haces tú mismo siguiendo la guía de abajo).
3. Claude/tú actualiza el array correspondiente dentro de `DECK` en `index.html`.

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

`DECK` tiene 6 categorías. Cuando se toca la carta (`drawChallenge()`):

1. Se elige una categoría al azar entre las 6 (peso uniforme — **esto también aplica a `juegos`**, que hoy no usa los pesos definidos en `juegos-peso-y-castigo.txt`, ver sección de pendientes).
2. Si la categoría es `juegos`, se filtra la lista de juegos según si el jugador dijo que tenían cartas (`state.cards === 'si'`, campo `needsCards` en cada juego) y se elige uno al azar de los que califican.
3. Si es cualquier otra categoría, se arma un pool ponderado con `weightedPick(cat)`:
   - Las frases `items` (normales) siempre entran, peso 1 cada una.
   - Las frases `futbol` **solo entran si `state.futbol === true`**, peso 1 cada una (todo o nada).
   - Las frases `disco` **siempre entran**, pero con peso distinto: `0.2` si `state.disco === false` (aparecen poco, es un "leak" intencional) y `3` si `state.disco === true` (dominan la mezcla, pero sin tapar del todo a las normales).
   - Se hace una selección aleatoria ponderada sobre ese pool.

Este comportamiento asimétrico entre fútbol y disco fue un pedido explícito: fútbol es estrictamente opt-in, disco nunca desaparece del todo.

## Ronda interactiva de Espía

Cuando el juego elegido es "Espía" (`state.card.isSpy === true`), el botón de la carta cambia a "Empezar ronda de Espía" (`start-espia`) en vez de avanzar el turno directamente. Esto dispara `startEspia()`:

- Elige una palabra al azar de `ESPIA_WORDS`. Si sale la entrada especial `'(Uno de los jugadores)'`, la reemplaza por el nombre de un jugador real elegido al azar (así el "secreto" es una persona presente, no una palabra literal).
- Baraja el orden de los jugadores (`shuffle`) y elige uno al azar como espía (`spyIndex`).
- Guarda todo en `state.espia = { word, spyIndex, order, step, revealed }`.

`renderEspiaStage()` muestra, jugador por jugador (según `order`), una carta que se voltea: todos ven la palabra secreta, excepto quien coincide con `spyIndex`, que ve "Eres el espía". Al terminar de recorrer a todos (`step >= order.length`), se muestra un resumen y un botón para volver al flujo normal de turnos (`finish-espia`).

## Estado de la app (`state`)

```js
{
  players: [{ name, color }],   // color es un var(--pink) etc, rota entre 4 colores
  futbol: bool, disco: bool, cards: 'si'|'no'|null,
  screen: 'setup' | 'play',
  turn: number,        // índice del jugador actual
  round: number,        // contador de turnos jugados (solo visual)
  flipped: bool,        // si la carta actual está volteada
  card: {...} | null,    // la carta actual (cat, icon, text, sub, answer, isSpy)
  answerShown: bool,      // si se reveló la respuesta de la pregunta actual
  espia: {...} | null      // estado de la sub-ronda de espía, null si no está activa
}
```

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

- **`juegos-peso-y-castigo.txt`**: define pesos de aparición y tragos de castigo por juego, pero el código todavía elige entre los 9 juegos con probabilidad uniforme, y el texto de "castigo" que se ve en la carta es el original (no los tragos que se definieron en este archivo). Falta: (1) usar los pesos en la selección aleatoria de `juegos`, (2) meter los números de castigo dentro de `rules` de cada juego en `DECK.juegos.items`.
- **`mentiroso-temas.txt`, `cultura-chupistica-temas.txt`, `cacho-humano-acciones.txt`**: siguen con el contenido de ejemplo original, nadie los ha llenado en serio todavía, y aunque se llenen, hoy esos 3 juegos solo muestran un texto de reglas fijo — no hay ninguna mecánica en la app que saque un tema/palabra/acción al azar de estos bancos para mostrarla en pantalla (a diferencia de Espía, que si tiene ese mecanismo con `ESPIA_WORDS`). Si se quiere que, por ejemplo, Cultura Chupística muestre un tema al azar en pantalla, hay que construirle un flujo similar al de `startEspia()`.
- **`nombra-regala.txt`**: fue editado a mano con contenido nuevo (menciona Colo-Colo, la U, Premier League, Mundial 2010) pero **todavía no está sincronizado** al `DECK.regala` del código — el código sigue con las frases genéricas originales.
