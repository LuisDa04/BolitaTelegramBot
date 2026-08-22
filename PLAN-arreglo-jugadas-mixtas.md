# Plan rápido: rechazar jugadas que mezclen "departamentos" (tipos de apuesta)

## Problema
Al jugar un tipo de apuesta (fijo / corridos / centena / parlet), si el texto incluye **al menos un número o línea de otro tipo** (ej: `320` o `17x32` metidos en una jugada FIJO), hoy ese token se **descarta en silencio** y el resto de la jugada se acepta.

**Comportamiento deseado:** si hay aunque sea 1 número que no corresponde al tipo elegido, se rechaza la jugada COMPLETA con el mismo error de cuando todas son incorrectas:

> ❌ Formato inválido. Por favor, formula correctamente tu apuesta.

## Causa técnica
`parseBetLine()` hace `continue` con los tokens que no cumplen el formato del tipo seleccionado, y `parseBetMessage()` marca `ok` solo con `items.length > 0`. Existen 3 copias idénticas en concepto: `backend.js`, `bot.js` y `app.html`.

## Cambios
| # | Archivo | Función | Cambio |
|---|---------|---------|--------|
| 1 | backend.js (~1122) | `parseBetLine` | Devuelve `{items, ok}`; `ok=false` si algún token no corresponde al tipo. Verificación de contenido extra en parlets aplica también con 1 solo par (`17x32 45` se rechaza). |
| 2 | backend.js (~1192) | `parseBetMessage` | `ok = todas las líneas válidas AND >=1 item`. Sigue devolviendo items parciales. |
| 3 | backend.js (~2114) | `/api/bets` | Mensaje unificado: `❌ Formato inválido. Por favor, formula correctamente tu apuesta.` |
| 4 | bot.js (~1982, ~2036) | mismas funciones | Mismos cambios (forma de item `{numero, usd, cup}`). |
| 5 | app.html (~1276, ~1344) | mismas funciones | Mismos cambios (preview y submit ya muestran el error correcto). |

**Llamadores que NO requieren cambios:** bot (`if (!parsed.ok)` ya responde el mensaje), web submit/preview (ya muestran el toast), `buildEditPrefill` (ya maneja `ok=false`).

## Casos de prueba
| Entrada | Tipo | Resultado esperado |
|---|---|---|
| `17 320 con 5 cup` | fijo | ❌ rechazada (antes aceptaba el 17) |
| `17 con 5 cup` | fijo | ✅ válida |
| `517 32 con 10 cup` | centena | ❌ rechazada |
| `17x32 45 con 5 cup` | parlet | ❌ rechazada |
| `17x32 con 5 cup` | parlet | ✅ válida |
| `17x32 18x45 con 5 cup` | parlet | ✅ válida |
| `17 con 5 cup` + línea `320*3 cup` | fijo | ❌ rechazada completa |
| `D5 T7 15 con 1 usd` | fijo/corridos | ✅ válida (D/T legítimos) |
| `D5 con 1 cup` | centena | ❌ rechazada (D/T no aplica) |

## Verificación
1. `node --check backend.js && node --check bot.js`
2. Harness en `%TEMP%` que extrae las funciones reales de cada archivo y ejecuta la tabla de casos.
