# EntrenadorMOVE — The Move Club

App web de planificación deportiva para The Move Club (Argentina). Un profesor
carga atletas y genera planes de entrenamiento con IA, fundamentados en
literatura de entrenamiento.

**Hablale al usuario en español rioplatense (voseo).** Es el dueño del club, no
es programador: explicale el *qué* y el *por qué*, no el detalle del código.

---

## 1. Lo esencial

| | |
|---|---|
| Repositorio | `emanuellezcanolic-oss/EntrenadorMOVE` (**público**) |
| Publicado en | https://emanuellezcanolic-oss.github.io/EntrenadorMOVE/ |
| Todo el código | `index.html` — un solo archivo, ~2100 líneas de JS inline |
| Build | No hay. Sin npm, sin bundler, sin dependencias instalables |
| Única dependencia externa | Chart.js por CDN |
| Deploy | GitHub Pages sirve la rama `main`. Mergear a `main` **es** publicar |
| Persistencia | `localStorage` del navegador. **No hay servidor ni base de datos** |
| IA | Groq, API compatible con OpenAI, llamada desde el navegador |

### Reglas duras

1. **El repo es público: nunca commitear una API key.** La de Groq vive solo en
   el `localStorage` del usuario (`tmc_groq_key`). Si pide "dejar la key en el
   código", explicale que GitHub la detecta y Groq la revoca automáticamente.
2. **Hay datos de salud de personas reales** en las fichas (cirugías, lesiones).
   No los subas a ningún lado ni los pegues en informes.
3. **Los datos viven en el navegador del usuario.** Una sesión de Claude Code en
   la nube **no puede leerlos ni escribirlos**. Para mover datos: el usuario usa
   `⬇️ Exportar DB`, te pasa el JSON, vos devolvés otro JSON y él usa
   `⬆️ Importar DB`.
4. **Confirmá antes de mergear a `main`**, porque eso publica en vivo.

---

## 2. Cómo trabajar acá

### Editar

El archivo es grande y tiene imágenes base64 enormes: `grep`/`sed` con rangos de
líneas, nunca leerlo entero. Para cambios con contexto, scripts de Python con
`str.replace` y un `assert s.count(old)==1` antes de cada reemplazo.

### Verificar sintaxis

```bash
S=$(grep -n "^<script>$" index.html|cut -d: -f1); E=$(grep -n "^</script>$" index.html|cut -d: -f1)
sed -n "$((S+1)),$((E-1))p" index.html > /tmp/app.js && node --check /tmp/app.js
```

### Probar de verdad

Playwright está instalado globalmente en `/opt/node22/lib/node_modules`. El
patrón que funciona: abrir el `file://`, interceptar `**/api.groq.com/**` con
`page.route()` para simular respuestas, y llamar a las funciones con
`page.evaluate()`.

```js
import { chromium } from '/opt/node22/lib/node_modules/playwright/index.mjs';
const p = await (await chromium.launch()).newPage();
await p.route('**/api.groq.com/**', r => r.fulfill({
  status: 200, contentType: 'application/json',
  body: JSON.stringify({ choices:[{ message:{content:'...'}, finish_reason:'stop' }] })
}));
await p.goto('file:///home/user/EntrenadorMOVE/index.html');
await p.evaluate(() => { window.CU = {dni:'1',name:'T'}; saveAtls([/* atleta */]); });
```

Para ejercitar `generarPlan()` hace falta: `window.CU`, un atleta en `saveAtls()`,
y poblar `#planAtl` con la opción seleccionada. **Ojo:** los `<option>` de
`#planDep` no llevan emoji (`'Ciclismo MTB'`, no `'🚵 Ciclismo MTB'`); asignar un
valor inexistente deja el `<select>` vacío y el test miente.

**El contenedor se recicla entre sesiones** y el scratchpad se borra. Los tests
hay que rehacerlos; no cuentes con que sigan ahí.

### Publicar

```bash
git add index.html && git commit   # mensaje en español, explicando el porqué
git push -u origin <rama>
git checkout main && git merge --ff-only <rama> && git push -u origin main
```

GitHub Pages tarda 1-2 min. **Decile siempre que recargue con Ctrl+Shift+R**, o
ve el HTML viejo cacheado y cree que no funcionó.

**Subí `APP_VERSION`** (cerca de `RUN_LIB`) en cada publicación. Se muestra al pie
del menú lateral: si el usuario reporta que un cambio "no aparece", pedile esa
versión antes de buscar el bug en el código. Ya pasó más de una vez que el
arreglo estaba publicado y el navegador servía HTML viejo.

---

## 3. Cómo se genera un plan

`generarPlan(modo)` (línea ~1490) con cuatro modos: `nuevo`, `evolucionar`,
`lesion`, `cambio`. Solo `nuevo` está bien trabajado.

### Armado del prompt

| Pieza | Línea | Qué hace |
|---|---|---|
| `bloqueCargaActual(a)` | 1214 | Volumen real del atleta desde `a.metricas` + lectura del ACWR |
| `bloqueNivel(nivel)` | 1262 | Restricciones según nivel. Principiante es mucho más estricto |
| `RUN_LIB` / `CIC_LIB` | 1190 / 1202 | Bibliografía. Hay versión `_BASICO` para principiantes |
| `reglaEspanol(isCicl,isField,dep)` | 1335 | Glosario **específico del deporte** + coherencia de vocabulario |
| `REGLA_COHERENCIA` | 1354 | Los totales semanales deben cerrar |
| `MTB_BLOCK` | 1362 | MTB vs ruta, desnivel como carga |
| `bloqueMix(dep1,dep2)` | — | Plan combinado cuando se elige un segundo deporte (`#planDep2`) |
| `REGLA_COMPLETITUD` | 1374 | Todas las semanas con el detalle de la primera |

### Llamada al modelo

- `groqChat()`: una llamada con continuación automática si corta por
  `finish_reason === "length"`. `GROQ_MAX_TOKENS = 3000`.
- `generarPorTramos()`: para planes de más de `SEMANAS_POR_TRAMO = 2` semanas.
  Cada bloque arranca con **contexto fresco** (solo la estrategia y la última
  semana escrita). Existe porque encadenar continuaciones hacía crecer la entrada
  al cuadrado: ~284.000 tokens en un plan de 12 semanas.
- `pedirAGroq()`: reintenta ante un 429 esperando lo que el propio error indica,
  con la cuenta regresiva a la vista. Hasta `GROQ_MAX_429 = 4` veces.

**El límite que manda es TPM, no el total.** El plan gratuito de Groq
(`on_demand`) da **8000 tokens por minuto**, y cuenta entrada + `max_tokens`
reservados. Un pedido de 2600 de entrada con `max_tokens` 8000 no entra nunca,
por más que el plan completo sea chico. De ahí los números de arriba: el pico por
pedido queda en ~5600. **Si tocás `GROQ_MAX_TOKENS` o `SEMANAS_POR_TRAMO`,
recalculá que entrada + max_tokens siga debajo de 8000.** Un plan de 12 semanas
sale en ~6 llamadas y tarda varios minutos, con esperas de 30-60s entre medio:
es el precio del plan gratuito, no un bug.

### Revisión automática de la salida (línea ~1973)

Lo verificable se chequea en JS, no se le pide al modelo por texto:

- `corregirAnglicismos()` traduce términos en inglés. El separador tolera guion
  normal, tipográfico (`‑ – —`) y espacio.
- `auditarSumas()` compara el volumen declarado de cada semana contra la suma de
  las cargas de sus sesiones.
- `auditarPlan()` suma: semanas faltantes, jerga de potencia en nivel inicial,
  sesiones que se remiten a sí mismas.
- `panelRevision()` lo muestra arriba del plan, con `no-print`.

---

## 4. Lo que aprendimos sobre este prompt

Esto sale de una auditoría de tres frentes (fisiología, voz, ingeniería) contra
dos planes reales. **No deshagas estas decisiones sin entender por qué están.**

1. **Prohibir induce.** El prompt decía *"PROHIBIDO escribir una sesión en una
   línea tipo `LUNES – Rodaje fácil 50 min, 16 km, RPE 2`"* y el plan salió con
   `LUNES – RUTA – RODAJE FÁCIL: 50 min, 16 km, RPE 2`. Mismos números. Las listas
   de términos prohibidos funcionan como menú: los 30 anglicismos que aparecieron
   estaban todos en la lista que los prohibía. **Escribí las reglas en positivo o
   como criterio general**, y si algo se puede verificar con una regex, verificalo
   en código en vez de pedirlo.
2. **El prompt enseña con el ejemplo.** Usaba "drills", "sweet spot", "core" y
   "foam roller" mientras los prohibía, y pedía "citar autores" (por eso salían
   `(Seiler 2008)` dentro del plan del atleta). Mantené el prompt limpio de
   aquello que no querés ver en la salida.
3. **Los glosarios se copian.** Mandar las traducciones de ciclismo en todos los
   planes hacía que uno de running dijera "pedaleo rápido y liviano": el modelo
   pegaba las frases del glosario como contenido. Por eso `reglaEspanol()` es una
   función del deporte y avisa que el glosario no es texto del plan.
4. **No se puede declarar un total antes de escribir lo que se suma.** La plantilla
   pide el volumen en el encabezado de semana; un modelo autorregresivo no puede
   sumar lo que todavía no escribió. Por eso la verificación es posterior.

---

## 5. Pendiente

Por orden de importancia:

1. **El modo `lesion` no tiene banderas rojas ni criterios de derivación**, y
   reutiliza el system prompt de generación de planes completos (a una consulta
   sobre dolor le exige estructura de 12 semanas). Es lo único de la lista que es
   responsabilidad y no calidad.
2. **Bloque de voz de entrenador**: que cada sesión lleve un `Para qué:`, que la
   sesión dura y la larga lleven `Si el día viene mal:`, y las de RPE ≥6 una
   `Señal:` observable. Es lo que más va a sacarle el tono robótico. Hay un
   borrador trabajado en la auditoría de voz; pedíselo al usuario si lo guardó.
3. **Modo `evolucionar`**: solo ve los primeros 600 caracteres del plan anterior
   (líneas ~1529 y ~1687) y descarta `sem`, `dias`, `notas`, `obj` y `mon`.
4. **Modo `cambio`**: dice "Redesenhá" (portugués) y no mira el plan anterior.
5. **Falta trabajo de fuerza** en los planes de resistencia, pese a estar en las
   bibliotecas: no está en el FORMATO ni en la cuenta `semanas × días`.
6. **Deportes de campo**: `FIELD_SPORTS` incluye tenis, pádel y vóley, pero todo
   el bloque está escrito para fútbol. Con el plan combinado esto se nota más:
   un mix con pádel usa la rama de campo y le salen rondos.
7. `kbShort` (~línea 1535) se calcula y nunca se usa: la Biblioteca de Conocimiento
   del profesor se arma, se recorta y se tira.
8. `mdToHtml` renderiza tablas pero no descarta la fila separadora `|---|---|`;
   hay dos instrucciones "NO uses tablas" tapando un bug de una línea.

---

## 6. Modelo de datos

Atletas en `localStorage`, clave `tmc_<dni>_atls`, vía `getAtls()`/`saveAtls()`
(dependen de la global `CU`, el usuario logueado).

```js
{ id, nombre, fnac, sexo, peso, altura, dep, nivel, email, lesiones,
  vamT, vamK, ftp, obj,
  tests:   [{ type, result, date }],
  planes:  [{ id, fecha, dep, obj, txt, html }],
  metricas:[{ semana, lun, mar, mie, jue, vie, sab, dom, fc }] }   // km por día
}
```

`nivel` ∈ Principiante | Intermedio | Avanzado | Competitivo | Élite. **Maneja
todo el comportamiento del generador**, vía `bloqueNivel()`.

`metricas` alimenta `wtotal()` (980), `calcACWR()` (981) y `bloqueCargaActual()`.
Export/import de toda la base: `exportarDB()` / `importarDB()`.

---

## 7. Configuración de la IA

Panel **⚙️ API Key IA** en el menú lateral. Guarda en `localStorage`:
`tmc_groq_key` y `tmc_groq_model`.

Groq da de baja modelos seguido, así que **el modelo no está fijo en el código**:
`cargarModelosGroq()` pide `GET /v1/models` y arma el selector con lo que la
cuenta realmente tiene. `probarGroqKey()` se autorrepara ante un 404. Si aparece
un error de modelo, la solución es "Cargar modelos", no editar el código.

`groqErrorMsg()` traduce los códigos HTTP a mensajes en castellano (401 key
revocada, 429 límite, etc.).
