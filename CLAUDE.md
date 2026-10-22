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
| `bloqueContexto(ctx)` | — | Dónde y con qué entrena (`.ctxLugar` / `.ctxMat`). **Limita todo lo prescribible** |
| `REGLA_COMPLETITUD` | 1374 | Todas las semanas con el detalle de la primera |

### Llamada al modelo

- `groqChat()`: una llamada con continuación automática si corta por
  `finish_reason === "length"`. `GROQ_MAX_TOKENS = 3000`.
- `generarPorTramos()`: parte el plan en bloques, cada uno con **contexto fresco**
  (solo la estrategia y la última semana escrita). Existe porque encadenar
  continuaciones hacía crecer la entrada al cuadrado: ~284.000 tokens en un plan
  de 12 semanas.
- `planificarTramos(semN,dias,tokEntrada)`: **el tamaño del tramo no es fijo.** Se
  calcula para meter la mayor cantidad de semanas que entren en una llamada sin
  pasar `MARGEN_TPM`, estimando ~90 tokens por sesión (una en formato ficha ocupa ~60).
  **Ese número y el tope de semanas por tramo deciden cuántas llamadas se hacen,
  y cada llamada extra cuesta una espera de ~45s.** Estuvo en 200 con tope de 6
  semanas: un plan de 8×4 que entra holgado en una llamada se partía en dos. `salidaParaTramo()` fija
  el `max_tokens` de esa llamada en lugar de reservar un número fijo. Un plan de
  4 semanas × 3 días sale en **una** llamada sin esperas; uno de 12 × 6 en cuatro.
- `pedirAGroq()`: ante un 429 lee del mensaje el estado real
  (`Limit X, Used Y, Requested Z`) y espera lo que el propio error indica, hasta
  `GROQ_MAX_429 = 4` reintentos **o** `ESPERA_MAX_MS` (2 min 30) de espera
  acumulada, lo que pase primero. Ese tope existe para que siempre le devuelva el
  control al profesor.

  **No intentes llevar un contador propio del cupo.** Se probó: la app registraba
  lo que iba a consumir antes de cada llamada y esperaba si no entraba. Cada
  reintento volvía a sumar al registro, el registro crecía solo y la espera se
  alargaba sola: quedaba dando vueltas varios minutos mostrando un número
  inventado ("usaste 5981 de 8000") que no era el de Groq. El único dato
  confiable es el que viene en el 429.

**El límite que manda es TPM, no el total.** El plan gratuito de Groq
(`on_demand`) da **8000 tokens por minuto**, y cuenta entrada + `max_tokens`
reservados. Un pedido de 2600 de entrada con `max_tokens` 8000 no entra nunca,
por más que el plan completo sea chico. De ahí el cálculo adaptativo: el pico por pedido
queda en ~6600 de los 8000. **Partir en más tramos NO acelera: cada llamada extra
consume del mismo minuto y obliga a esperar.** Por eso conviene la llamada más
grande que entre, no la más chica. Si tocás `MARGEN_TPM` o `TOPE_SALIDA`,
verificá con un test que entrada + max_tokens siga debajo de 8000.

**Techo del plan gratuito:** un plan de 8 semanas cuesta ~6500 tokens (3800 de
entrada + 2750 reservados de salida) de los 8000 por minuto. O sea **un plan por
minuto**: generar dos seguidos, o reintentar enseguida, espera sí o sí. No es un
bug, es el tamaño del pedido contra la cuota. El Dev Tier de Groq lo saca.

El prompt ya pasó por una compresión (4694 → 3778 de entrada) sin sacar reglas:
se acortaron `ADHERENCIA_LIB`, `REGLA_DINAMISMO`, `REGLA_COMPLETITUD` y
`FORMATO_FICHA`. Lo que queda grande es `RUN_LIB`/`CIC_LIB` (~560) y
`FORMATO_FICHA` (~420). Ojo: **recortar la entrada no baja sola el total**, porque
`salidaQueEntra()` expande la reserva para ocupar el hueco; hay que bajar también
`porSem` y el multiplicador de `salidaParaTramo()`.

### Dinamismo (planificación 2022 del club) — con su evidencia

`ADHERENCIA_LIB` es el fundamento: monotonía y sobreentrenamiento (Foster 1998),
carga ondulante (Rhea 2002), carga concentrada y descarga (Verkhoshansky,
Issurin), saltos de carga como causa de lesión (Gabbett 2016, Nielsen 2014),
abandono ~50% a seis meses (Dishman), autonomía-competencia-vínculo (Deci y
Ryan, Teixeira 2012), grupo (Burke 2006), disfrute durante la sesión como
predictor de adherencia (Williams 2008, Rhodes y Kates 2015) e intenciones de
implementación (Gollwitzer y Sheeran 2006), que es lo que respalda la línea
`ALT`. Como las demás bibliotecas, **los autores no se nombran dentro del plan**.

`REGLA_DINAMISMO` enuncia cada regla con su porqué, no como capricho de estilo.

`REPERTORIO` y `REGLA_DINAMISMO` salen de la planilla real de 2022 del club
(22 microciclos, niveles A/B/C en paralelo). Lo que la hacía viva y el prompt no
tenía:

- **Cada día una sesión con NOMBRE y un estímulo distinto**: Fraccionado,
  Fraccionado en escaleras, Fraccionado alternando sentido, Pasadas, Cuestas,
  Continuo uniforme / variable / progresivo / intensivo / en circuito, Fondo,
  circuito de fuerza, zona media, estabilidad, pliometría, saltabilidad, técnica,
  Test, Doble turno. No "trote 40 min" doce veces.
- **Seis tipos de microciclo, en orden variable**: Ajuste, Control, Carga,
  **Choque** (la semana pico), Descarga, Mantenimiento. En la planilla la
  secuencia cambia entre mesociclos; no es siempre Ajuste→Control→Carga→Descarga.
- **Volumen no lineal**: en la planilla va 38/38/40/31, 32/36/40/32, 36/0/34/31.
  Sube, se sostiene, baja y vuelve a subir.
- **Adherencia**: la planilla tiene salida grupal, fondo grupal, cena grupal,
  activación con el equipo y carreras (Turmalina, Valholl) metidas en el
  calendario. Cada 3-4 semanas algo que dé ganas de ir.
- **Versión alternativa por sesión**: en la planilla era "Parte Principal
  (Presencial)" vs "(Online)", por la pandemia. Generalizado a la línea `ALT`:
  qué hacer si llueve, si no llega con el tiempo o si el cuerpo no da. **No hay
  que escribir "online"**: eso era del contexto 2020-2022.

### Escala por nivel (los A/B/C de la planilla)

`escalaNivel(nivel)`. En la planilla, A/B/C son **el mismo plan en tres dosis**,
no tres planes. Medido sobre las 23 semanas: el nivel B hace **1,29×** el volumen
semanal de A y el C **1,48×** (A promedia 36 km/sem, B 46, C 53).

Lo que **no** cambia entre niveles: el tipo de sesión, la estructura de la
semana, la entrada en calor y el afloje (los tres hacen 2 km de afloje).
Lo que cambia, en este orden: repeticiones (6×500 / 8×500 / 10×500; 3×3
escaleras / 3×4 / 3×5), pausa (1'30 → 1', o sea densidad), volumen de las
continuas (4/5/6 km, o 20'/30'/40'), rótulo de intensidad ("algo duro" / "duro"
/ "muy duro") y un tramo más al final de las series escalonadas.

El repertorio y la escala no se mandan a principiantes (`esPrinc`): para alguien que empieza,
la variedad de estímulos es ruido. La biblioteca de adherencia sí va siempre.

**Las bibliotecas pesadas viajan solo en la primera llamada.** Sumarlas llevó el
prompt de ~2800 a ~4265 tokens y duplicó las llamadas (un plan de 4 semanas
volvió a necesitar dos, y uno de 12×6 pasó a doce). `generarPorTramos` recibe un
`systemCont` sin `libBlock`, `REPERTORIO`, `escalaNivel()` ni `ADHERENCIA_LIB`
para los tramos siguientes: ~3100 en vez de ~4265. Además `salidaQueEntra()` recorta el
`max_tokens` de cada llamada contra la entrada real, de modo que entrada +
salida nunca pase `MARGEN_TPM`. Sin ese tope la primera llamada se iba a 8265.

### Para quién es el plan

`REGLA_DESTINATARIO`. **La planilla A/B/C del club es documento del profesor; el
socio nunca la ve.** Los tres niveles existen para que el profe clasifique. Lo
que la app entrega es el plan de UN socio, a SU dosis.

Por eso la nomenclatura de planificación no va adentro del plan: microciclo,
mesociclo, ajuste/control/carga/choque/descarga, ACWR, TSS, umbral, VO2,
porcentajes de distribución, autores y citas. El rótulo de cada semana dice lo
que le pasa al socio ("arrancamos", "la más fuerte del mes", "aflojamos para
asimilar"), no el nombre técnico del microciclo.

`auditarJergaDeProfe()` lo verifica sobre la salida: vocabulario técnico,
semanas rotuladas con el nombre del microciclo, y citas bibliográficas.

El material de ACWR y carga sí vive en la app, pero en el panel del profesor
(Control de Carga, pestaña Métricas), que es donde corresponde.

### Formato de ficha

El plan ya no se renderiza desde markdown libre. Al modelo se le pide un formato
estricto, línea por línea (`FORMATO_FICHA`), y la app lo dibuja:

```
# BASES
EC | paso | paso | paso          AF | paso | paso          OJO | cuidado | cuidado
# SEMANA 1 | 1 h 40 | reconocer
DIA | A | 30' | Fraccionado
EC | 10' | la de siempre
PR | 15' | 15' de trote suave continuo
NOTA | ritmo de poder hablar de corrido
ALT | si llovió, 4 series en vez de 6
AF | 5' | el de siempre
```

- `parsearFicha(txt)` devuelve la estructura, o `null` si el modelo no respetó el
  formato; ahí se cae a `mdToHtml()` como antes. **Nunca quites ese respaldo.**
- `renderFicha(obj)` arma el HTML: tres cajas arriba (entrada en calor, afloje,
  cuidados) y una tabla por semana con Día A/B/C, los tiempos a la izquierda y el
  trabajo a la derecha. Estilos `.ficha` / `.fbx` / `.fwk` / `.fet` / `.fco`, con
  su bloque `@media print` en claro.
- `cabeceraImpresion()` agrega el membrete al imprimir desde el generador: antes
  el `.print-hdr` solo existía en la pantalla del atleta y el plan salía sin logo
  ni nombre. El `@media print` también oculta el formulario de configuración.
- **Los días se llaman A, B, C**, no lunes/martes: el socio los acomoda en su
  semana.

### Revisión automática de la salida (línea ~1973)

Lo verificable se chequea en JS, no se le pide al modelo por texto:

- `corregirAnglicismos()` traduce términos en inglés. El separador tolera guion
  normal, tipográfico (`‑ – —`) y espacio.
- `auditarSumas()` compara el volumen declarado de cada semana contra la suma de
  las cargas de sus sesiones.
- `auditarRecursos()` marca ejercicios imposibles para el contexto del atleta
  (juego reducido si entrena solo, pesas sin gimnasio, pelota sin pelota).
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
   el bloque está escrito para fútbol: a un plan de pádel le salían "pases de
   fútbol". `bloqueContexto()` tapa lo peor (ya no manda juego reducido a quien
   entrena solo), pero los deportes de raqueta siguen sin bloque propio.

   **Contexto importante del club:** la mayoría de los socios entrenan **solos,
   al aire libre, sin material**. El prompt asumía infraestructura de club y
   prescribía juego reducido 3 vs 3 y sentadillas con carga. Por defecto las
   casillas vienen en "Parque o aire libre" + "Sin material".
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
