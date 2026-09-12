# PLAN · Jugadas del día (reporte diario a admins)

> Reporte automático que llega a las **12:00 AM (hora de Cuba)** a los superadmins y a los
> subadmins con privilegio de **"ver apuestas"** (`session_exporter` / "Ver Jugadas"), con un
> botón que abre un **HTML** con el resumen de jugadas del día.

---

## 1. Objetivo

Que cada medianoche (hora de Cuba) el bot envíe un mensaje a:

- **Superadmins** → `ADMIN_IDS` (`.env`).
- **Subadmins con rol `session_exporter`** → tabla `admin_roles` (permiso "Ver Jugadas").

El mensaje tendrá el texto **"Jugadas del día"**, debajo la **fecha**, y debajo un **botón**
que abre un HTML (estilo idéntico al actual "ver apuestas de la sesión") con una tabla de
resumen del día.

---

## 2. Mensaje de Telegram

Ejemplo del mensaje enviado a cada admin:

```
📊 Jugadas del día

📅 10/09/2026

Pulsa el botón para ver el resumen del día.
```

Botón:

```
[ 👁️ Ver jugadas del día ]
```

- `parse_mode: 'HTML'`
- El botón es `Markup.button.url(...)` apuntando al endpoint del reporte (con token aleatorio).

---

## 3. HTML del reporte (archivo generado por el endpoint)

Visualmente **igual al HTML de "ver apuestas de la sesión"** (`generateSessionHtml`), pero con
una tabla de 3 columnas:

| Sesiones | Total CUP | Total USD |
|---|---|---|
| 🦩 Florida · Mañana 🌅 | `12 / $1.240,00` | `4 / $35,00` |
| 🍑 Georgia · Tarde ☀️ | `8 / $980,50` | `2 / $14,20` |
| 🗽 Nueva York · Noche 🌙 | `15 / $2.150,75` | `6 / $60,00` |
| **Total** | **`35 / $4.371,25`** | **`12 / $109,20`** |

### Detalles

- **Sesiones**: una fila por cada `lottery_sessions` del día reportado. Etiqueta:
  `{emoji región} {lotería} · {emoji turno} {turno}`.
- **Formato de celda** (aplica a CUP y USD): `cantidad / monto`
  - `cantidad` → nº de jugadas (fila en `bets`) con `cost_cup > 0` (col. CUP) o `cost_usd > 0` (col. USD).
  - `monto` → sumatoria de `cost_cup` o `cost_usd` de esas jugadas.
- **Última fila**: la columna *Sesiones* dice **"Total"** y en las otras dos la sumatoria de
  las cantidades y de los montos de todas las filas.
- **Botón de descarga**: debajo de la tabla, igual que el HTML de las jugadas al cerrar sesión:
  ```
  [ 📥 Descargar archivo ]
  ```
  Es un enlace al mismo endpoint con `&download=1`, con clase `.download` (estilo idéntico:
  `background: #2563eb`, texto blanco, `display: block`, centrado) y `target="_parent"` para
  que la descarga funcione dentro de la WebView (mismo arreglo que en `generateSessionHtml`).
- **Sin jugadas**: si no hubo apuestas el día reportado, se muestra el estado vacío
  ("ℹ️ No hubo jugadas en este día") igual que en la vista actual, y el botón se envía igual.
- Estilos: reutilizar el `<style>` de `generateSessionHtml` (fondo `#f3f4f6`, cabecera de tabla
  `#111827`, filas alternadas, fila total en verde `#16a34a`).

---

## 4. Cambios por archivo

### 4.1 `backend.js` — nuevo endpoint + render

1. **Nueva función `generateDailyBetsHtml(date, sessions)`** (junto a `generateSessionHtml`):
   recibe la fecha y el agregado por sesión y devuelve el HTML del punto 3.

2. **Nuevo endpoint protegido por token**, siguiendo el patrón `/export-session/:sessionId`:
   ```
   GET /reporte-dia/:date?<parametroOculto>=<token>
   ```
   - El token se genera con `generateSessionExportToken()` y se guarda en `app_config`
     con key `daily_report_token_{date}` (patrón igual a `export_token` de las sesiones).
   - Lógica:
     1. Validar token.
     2. Traer `lottery_sessions` de esa fecha (`.eq('date', date)`).
     3. Traer `bets` de esas sesiones (`.in('session_id', ids)`) para evitar N+1.
     4. Agregar por sesión: `countCup`, `sumCup`, `countUsd`, `sumUsd`.
     5. Renderizar con `generateDailyBetsHtml`.
   - **Descarga**: si llega `download=1`, responder con `Content-Disposition: attachment;
     filename="Jugadas_del_dia_{date}.html"` (like `/export-session/:sessionId`) y ocultar el
     botón de descarga dentro del archivo descargado.
   - `momento útil`: la fecha viaja en la URL (`/reporte-dia/2026-09-10?...`), así el enlace
     queda autocontenido y legible.

### 4.2 `bot.js` — cron de medianoche + envío

1. **Nueva función `buildDailyReportUrl(date)`**: genera/lee el token en `app_config`
   (`daily_report_token_{date}`) y arma `WEBAPP_URL + /reporte-dia/{date}?<param>` reutilizando
   `getBotUsernameParam()` (parámetro oculto con el `first_name` del bot, como ya se hace).

2. **Nueva función `notifyDailyBetsReport()`**:
   - `const reportDate = moment.tz(TIMEZONE).subtract(1, 'day').format('YYYY-MM-DD');`
   - Fecha legible para el mensaje: `moment.tz(TIMEZONE).subtract(1, 'day').format('DD/MM/YYYY')`.
   - Destinatarios: `ADMIN_IDS` + `botRolesCache.sessionExporters`.
   - Mensaje con inline button hacia `buildDailyReportUrl(reportDate)`.

3. **Nuevo cron** (junto a los existentes):
   ```
   cron.schedule('0 0 * * *', notifyDailyBetsReport, { timezone: TIMEZONE });
   ```

---

## 5. Permisos y seguridad

- El rol es `session_exporter` (en el panel se muestra como "Ver Jugadas" → es el privilegio de
  "ver apuestas" que pide el usuario).
- El endpoint NO debe estar abierto al público: requiere token aleatorio (como
  `/export-session/:sessionId`). El token queda persistido en `app_config` para que el botón
  siga válido.
- El envío del mensaje se hace solo a superadmins y `session_exporters`; nunca en broadcast
  a todos los usuarios.

---

## 6. Decisiones a confirmar (supuestos del plan)

1. **¿Qué día se reporta?** El cron corre a las 00:00, cuando aún no hay sesiones del día nuevo.
   El plan asume **el día que acaba de terminar** (`fecha de hoy − 1 día`). Si prefieren el día
   actual (con las sesiones todavía por abrir), cambiar sólo la línea del `reportDate`.
2. **Formato de celda**: `cantidad / monto` aplica **a ambas columnas** (CUP y USD), tal como
   aparece descrito para "Total USD".
3. **Sesiones abiertas del día anterior a la medianoche**: el plan reporta todas las sesiones con
   esa `date` (abiertas y cerradas). Si se prefiere sólo `status = 'closed'`, se añade un filtro.

---

## 7. Archivos

| Archivo | Acción |
|---|---|
| `backend.js` | Nuevo endpoint `/reporte-dia/:date` + `generateDailyBetsHtml` + helper de token diario |
| `bot.js` | Nuevas `buildDailyReportUrl`, `notifyDailyBetsReport` y cron `0 0 * * *` |
| `ejemplo-jugadas-del-dia.html` | Vista previa del HTML (ya creado en el root) |

---

## 8. Vista previa

El archivo **`ejemplo-jugadas-del-dia.html`** (root del proyecto) muestra el diseño final del
reporte con datos de ejemplo. Ábrelo en el navegador para revisar antes de implementar.