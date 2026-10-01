# Cambios del 2026-10-01 — Avisos al editar jugadas con números al máximo

> **Estado: revertido.** Los dos commits descritos abajo (`72ab9bc` y `0be7f4f`)
> se revirtieron y `main` volvió a `2c8d64b`. Este archivo queda como registro
> del análisis y de lo que se probó.

## Objetivo

El usuario editaba una jugada de 4 números (3 ya en su tope y 1 con cupo) y al
subirle a uno de los que estaban al máximo recibía el aviso rojo sin pregunta.
Se pidió que en ese caso salga el modal con elección, y que el aviso sin
pregunta quede reservado para cuando **todos** los números de la jugada ya
estaban al máximo.

## Análisis de la lógica existente

Backend: `POST /api/bets` en `backend.js`. Piezas relevantes:

- `validateBetLimits` (backend.js:1172) — agrupa por número y marca `cupExceeders`
  cuando `monto_nuevo + otras_jugadas > máximo`.
- `maxedExceeders` (backend.js:1508) — devuelve qué números **de entre los
  excedentes** están además ya en su tope.
- `maxedNoticeText` / `maxedNoticeBlock` (backend.js:1581, 1590) — los dos textos.
- Gate de "nada que aplicar" en backend.js:2718.
- Cliente: `app.html:1759-1771` — `MAXED_NUMBERS_ON_EDIT` abre modal,
  `EDIT_NOTHING_TO_APPLY` muestra toast rojo.

Detalle que resultó clave: `maxedOverall` **no es "la lista de números al
máximo"**, es "la lista de números que el usuario *intentó* subir y que además
están en su tope", porque `maxedExceeders` solo itera sobre `cupExceeders` /
`usdExceeders`.

## Commit `72ab9bc` — primer intento, incorrecto

Se condicionó el aviso sin pregunta a que todos los números de la jugada
estuvieran en `maxedOverall`:

```js
const maxedSet = new Set([...(maxedOverall.cup || []), ...(maxedOverall.usd || [])]);
const allOriginalNumsMaxed = originalBetNums.every(n => maxedSet.has(String(n)));
```

**Regresión.** En el caso "A,B,C,D los cuatro al máximo, el usuario sube A":

```
A: 150 + 0 > 100 -> excede
B: 100 + 0 > 100 -> NO excede (100 no es > 100)
C, D: igual -> NO exceden
cupExceeders  = ["A"]
maxedOverall  = ["A"]          <- B, C, D nunca aparecen
```

`maxedSet` queda `{"A"}` y `.every(...)` sobre `[A,B,C,D]` da `false`. El caso
que debía dar el aviso caía en el modal. El fallo conceptual fue usar una lista
de *intentos* para responder una pregunta de *estado*.

## Commit `0be7f4f` — corrección

El cupo se evalúa sobre el estado real de cada número — lo que la jugada tiene
más lo que tienen las otras — por moneda, y solo para las monedas que la jugada
usa. Sustituye el uso de `maxedOverall` en el gate (backend.js:2730-2757):

```js
const allNumsAtMax = originalNums.length > 0
    && originalNums.every((n) => {
        const own = originalTotals[n] || { cup: 0, usd: 0 };
        const other = otherTotals[n] || { cup: 0, usd: 0 };
        const hasCup = (own.cup || 0) > 0;
        const hasUsd = (own.usd || 0) > 0;
        if (!hasCup && !hasUsd) return false;
        // Una moneda sin tope configurado no puede estar "agotada".
        if (hasCup && (exceedData.maxCup == null || (own.cup + (other.cup || 0)) < exceedData.maxCup)) return false;
        if (hasUsd && (exceedData.maxUsd == null || (own.usd + (other.usd || 0)) < exceedData.maxUsd)) return false;
        return true;
    });
```

El aviso sin pregunta queda para cuando además el recorte no produce ningún
cambio (`nothingToApply`).

## Verificación

Siete escenarios ejecutados contra las funciones **reales** de `backend.js`
(`validateBetLimits`, `clampItemsToMax`, `maxedExceeders`, `maxedNoticeBlock`
extraídas del archivo, con la cadena de Supabase stubbeada a "sin otras
jugadas"):

| Escenario | Respuesta |
|---|---|
| A,B,C al máximo + D normal, suben A | `MAXED_NUMBERS_ON_EDIT` (modal) |
| A,B,C,D todos al máximo, suben A | `EDIT_NOTHING_TO_APPLY` (aviso) |
| Todos al máximo + añade E | `MAXED_NUMBERS_ON_EDIT` |
| 3 al máximo + D normal, suben A y D | `MAXED_NUMBERS_ON_EDIT` |
| Un solo A al máximo, suben A | `EDIT_NOTHING_TO_APPLY` |
| Todo con cupo (máx. 1000) | `APLICAR` |
| USD, todos al máximo USD | `EDIT_NOTHING_TO_APPLY` |

Textos que producían las dos ramas:

```
CASO 2: ❌ El número A ya fue apostado a su máximo permitido de 100.00 CUP.
CASO 1: ⚠️ El número A ya fue apostado a su máximo permitido de 100.00 CUP.
        ¿Deseas continuar con la edición?
```

## Consecuencia conocida y aceptada

El usuario eligió explícitamente la opción **B** para el caso 1: mostrar el
modal aunque el recorte no produzca ningún cambio real. La consecuencia es que,
si se pulsa "Sí, continuar" sin haber tocado ningún otro número, el recorte
deja la jugada idéntica y `betTotalsEqual` (backend.js:2867) responde
`{ success: true, noChanges: true }`, con lo que el cliente muestra el toast
"No hubo cambios en tu apuesta.".

Es decir: el modal ofrece continuar con una edición que no va a ocurrir. Si
alguna vez molesta, la causa es que el modal se ofrece también cuando no hay
cambio real que aplicar.

## Qué NO se verificó

- **No hay suite de tests en el repo.** El harness extrae funciones por slicing
  del archivo y stubbea Supabase.
- No se ejercitó el endpoint HTTP real, ni `parseBetLine` (el texto que llega
  del input del navegador).
- No se probó contra Supabase real, ni contra el navegador más allá de leer el
  código de `app.html`.
- `bot.js` quedó fuera: no tiene flujo de edición de jugadas
  (`excludeBetId: null` siempre en bot.js:7136), así que el cambio nunca aplica
  a Telegram.
- Cobertura real requeriría un `package.json` con runner, que el repo no tiene.

## Revert

```
git revert --no-edit 0be7f4f
git revert --no-edit 72ab9bc
```

`main` queda con el árbol de código de `2c8d64b`.