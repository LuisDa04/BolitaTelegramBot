# Plan: Ver apuestas de la sesión (HTML en navegador)

## Objetivo

Al cerrar una sesión de lotería:

1. **Bot (subadmins con privilegio `session_exporter`)**: enviar un mensaje con un botón **"👁️ Ver apuestas de la sesión"**. Al tocarlo se **abre el HTML en el navegador** (Telegram data browser) con los datos de apuestas de la sesión. Dentro del HTML, al final, hay un botón **"📥 Descargar archivo"**.
2. **Web panel (superadmin)**: en el botón **"Sesiones"**, debajo de cada lotería aparece un botón **"📊 Ver apuestas"** que abre el HTML con los datos de la **última sesión que cerró** de esa lotería. El HTML tiene el mismo botón de descarga dentro.

El archivo es un **documento HTML autónomo** (CSS inline, sin dependencias externas) servido por el backend y abierto en el navegador. **No requiere instalar ninguna librería ni crear `package.json`.**

---

## Cómo se abre y se descarga el HTML

No se envía un archivo adjunto. Se usa un **botón de tipo URL** (inline keyboard `url`) que apunta a un endpoint del backend que **sirve el HTML como página**:

| URL | Comportamiento |
|---|---|
| `GET /export-session/:sessionId?token=…` | Abre el HTML en el navegador (inline) |
| `GET /export-session/:sessionId?token=…&download=1` | Descarga el archivo (`Content-Disposition: attachment`) — lo usa el botón interno |

El acceso se protege con un **token firmado (HMAC SHA-256)** del `session_id`, usando como secreto `BOT_TOKEN` (ya disponible en ambos procesos). Solo quien tenga el enlace firmado puede abrirlo, y el enlace solo llega a los admins notificados.

No se necesita ninguna librería: `crypto` es **módulo integrado de Node.js**. Solo hay que agregar `const crypto = require('crypto');` en `bot.js` (en `backend.js` ya está importado).

---

## Cambios en archivos del proyecto

### Archivo: `bot.js`

#### A. Importar `crypto` (módulo integrado, NO instala nada)

Agregar después del `require('dotenv')` / junto a los demás require (~línea 7):

```js
const crypto = require('crypto');
```

> `crypto` es parte de Node.js. No agrega ninguna dependencia al proyecto.

#### B. Agregar `sessionExporters` al cache de roles

**Línea ~132** — Objeto `botRolesCache`:

```js
// ANTES:
let botRolesCache = { withdrawApprovers: [], depositApprovers: [], scheduleManagers: [], userManagers: [], userDeleters: [], activitySelf: [], lastFetch: 0 };

// DESPUÉS:
let botRolesCache = { withdrawApprovers: [], depositApprovers: [], scheduleManagers: [], userManagers: [], userDeleters: [], activitySelf: [], sessionExporters: [], lastFetch: 0 };
```

**Dentro de `refreshBotRolesCache()` (~línea 143)** — agregar línea:

```js
botRolesCache.sessionExporters = data?.filter(r => r.role === 'session_exporter').map(r => Number(r.telegram_id)) || [];
```

**Dentro de `hasRole()` (~línea 154)** — agregar case:

```js
case 'session_exporter': return botRolesCache.sessionExporters.includes(id);
```

**Dentro de `hasAnyRole()` (~línea 169)** — agregar al return:

```js
return ... || botRolesCache.sessionExporters.includes(id);
```

#### C. Helpers para generar el enlace firmado

Agregar cerca de las funciones auxiliares (p. ej. junto a `escapeHTML`, ~línea 206):

```js
function getSessionExportToken(sessionId) {
    return crypto.createHmac('sha256', BOT_TOKEN).update(String(sessionId)).digest('hex');
}

function buildSessionExportUrl(sessionId, download = false) {
    const token = getSessionExportToken(sessionId);
    return `${WEBAPP_URL}/export-session/${sessionId}?token=${token}${download ? '&download=1' : ''}`;
}
```

> `session_id` se trata como **string opaco** (no `parseInt`) porque la tabla `lottery_sessions` puede usar UUID o entero según el schema.

#### D. Notificación al cerrar sesión — mensaje con botón URL

El mensaje se envía **solo** a subadmins que tengan el rol `session_exporter` (no a superadmin; él usa el panel web). El botón es tipo URL, así que **no hace falta ningún callback**: al tocarlo se abre el HTML directamente.

**Punto 1: `toggle_session` action (~línea 3545)** — dentro del bloque `else if (newStatus === 'closed')`, después de `broadcastToAllUsers(...)`:

```js
// Notificación con botón de ver apuestas (solo subadmins con privilegio)
try {
    await ensureBotRolesCache();
    for (const adminId of botRolesCache.sessionExporters) {
        try {
            await bot.telegram.sendMessage(adminId,
                `📊 <b>Jugadas</b>\n\n` +
                `🎰 ${region?.emoji || '🎰'} <b>${escapeHTML(session.lottery)}</b> · <b>${escapeHTML(session.time_slot)}</b>\n` +
                `📅 ${session.date}\n\n` +
                `Pulsa el botón para ver las apuestas de la sesión.`,
                {
                    parse_mode: 'HTML',
                    reply_markup: Markup.inlineKeyboard([
                        [Markup.button.url('👁️ Ver apuestas de la sesión', buildSessionExportUrl(session.id))]
                    ]).reply_markup
                }
            );
        } catch (e) {}
    }
} catch (e) {}
```

La estructura del mensaje es:

```
📊 Jugadas                        ← encabezado

🎰 Florida · Mañana               ← lotería + turno (al lado)

📅 2026-08-27                     ← fecha debajo

Pulsa el botón para ver las apuestas de la sesión.

[👁️ Ver apuestas de la sesión]   ← botón al final (abre el HTML)
```

**Punto 2: `closeExpiredSessions()` (~línea 7028) — SIN notificación**

Cuando la sesión cierra **automáticamente** por el cron (`closeExpiredSessions`), **NO se envía ningún mensaje** con botón. Solo se mantiene lo que ya existe (el broadcast de la foto de cierre). La notificación de "Ver apuestas" queda **exclusivamente para los cierres manuales** (bot toggle y web toggle).

```js
// NO agregar nada aquí. El cierre automático NO notifica.
// Se mantiene el código actual tal cual:
await broadcastPhotoToAllUsers(photoPath);
```

> Nota: el superadmin igual podrá ver las apuestas de cualquier sesión cerrada (incluso las cerradas por cron) desde el panel web en "Sesiones" → "📊 Ver apuestas". Solo cambia que el **subadmin no recibe mensaje** en el cierre automático.

#### E. Eliminar el callback `export_session_(\d+)`

Ya **no existe** ningún callback `bot.action(/export_session_…/)`. El botón es de tipo URL y abre el navegador directamente; el HTML lo genera el backend.

---

### Archivo: `backend.js`

#### A. Agregar `sessionExporters` al cache de roles

**Línea ~68** — Objeto `rolesCache`:

```js
// ANTES:
let rolesCache = { depositApprovers: [], withdrawApprovers: [], scheduleManagers: [], userManagers: [], userDeleters: [], activitySelf: [], lastFetch: 0 };

// DESPUÉS:
let rolesCache = { depositApprovers: [], withdrawApprovers: [], scheduleManagers: [], userManagers: [], userDeleters: [], activitySelf: [], sessionExporters: [], lastFetch: 0 };
```

**Dentro de `refreshRolesCache()` (~línea 71)** — agregar línea:

```js
rolesCache.sessionExporters = data?.filter(r => r.role === 'session_exporter').map(r => Number(r.telegram_id)) || [];
```

**Dentro de `hasRole()` (~línea 103)** — agregar case:

```js
case 'session_exporter': return rolesCache.sessionExporters.includes(id);
```

**Dentro de `hasAnyRole()` (~línea 118)** — agregar al return:

```js
return ... || rolesCache.sessionExporters.includes(id);
```

#### B. Agregar `session_exporter` a `validRoles` en PUT

**Línea ~5111** — dentro del endpoint `PUT /api/admin/admin-roles/:telegramId`:

```js
// ANTES:
const validRoles = ['deposit_approver', 'withdraw_approver', 'schedule_manager', 'user_manager', 'user_deleter', 'activity_self'];

// DESPUÉS:
const validRoles = ['deposit_approver', 'withdraw_approver', 'schedule_manager', 'user_manager', 'user_deleter', 'activity_self', 'session_exporter'];
```

#### C. Helpers + generador del HTML

`crypto` ya está importado en backend.js (~línea 7). Agregar junto a `escapeHTML` (~línea 236):

```js
function getSessionExportToken(sessionId) {
    return crypto.createHmac('sha256', BOT_TOKEN).update(String(sessionId)).digest('hex');
}

function buildSessionExportUrl(sessionId, download = false) {
    const token = getSessionExportToken(sessionId);
    return `${WEBAPP_URL}/export-session/${sessionId}?token=${token}${download ? '&download=1' : ''}`;
}

function formatReferrer(bet) {
    const ref = bet.referrers || {};
    if (ref.username) return '@' + ref.username;
    return bet.referrer_id ? String(bet.referrer_id) : '';
}

function generateSessionHtml(session, bets, downloadUrl) {
    // Filas de la tabla (todo el texto del usuario se escapa)
    const rows = (bets || []).map(bet => {
        const user = bet.users || {};
        // Hora en 12h (AM/PM), hora de Cuba.
        // La fecha ya aparece en el encabezado (📅), por eso solo se muestra la hora.
        const hora = bet.placed_at
            ? new Date(bet.placed_at).toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit', second: '2-digit', hour12: true, timeZone: 'America/Havana' })
            : '';
        return `<tr>
            <td>${escapeHTML(user.first_name || user.username || String(bet.user_id))}</td>
            <td>${escapeHTML(bet.bet_type || '')}</td>
            <td>${escapeHTML(bet.raw_text || '')}</td>
            <td class="num">${(parseFloat(bet.cost_cup) || 0).toFixed(2)}</td>
            <td class="num">${(parseFloat(bet.cost_usd) || 0).toFixed(2)}</td>
            <td class="num">${(parseFloat(bet.bonus_used_cup) || 0).toFixed(2)}</td>
            <td>${escapeHTML(formatReferrer(bet))}</td>
            <td>${escapeHTML(hora)}</td>
        </tr>`;
    }).join('');

    const totalCup = (bets || []).reduce((s, b) => s + (parseFloat(b.cost_cup) || 0), 0);
    const totalUsd = (bets || []).reduce((s, b) => s + (parseFloat(b.cost_usd) || 0), 0);
    const vacio = !bets || bets.length === 0;

    return `<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>${escapeHTML(session.lottery)} - ${escapeHTML(session.time_slot)} - ${escapeHTML(session.date)}</title>
<style>
    body { font-family: -apple-system, "Segoe UI", Roboto, sans-serif; margin: 0; padding: 16px; background: #f3f4f6; color: #111; }
    h1 { font-size: 20px; margin: 0 0 4px; }
    .sub { color: #4b5563; margin: 0 0 12px; font-size: 14px; }
    .total { background: #16a34a; color: #fff; padding: 10px 12px; border-radius: 8px; margin: 12px 0; font-size: 15px; }
    .wrap { background: #fff; border-radius: 10px; overflow-x: auto; box-shadow: 0 1px 2px rgba(0,0,0,.08); }
    table { width: 100%; border-collapse: collapse; min-width: 560px; }
    th, td { padding: 8px 10px; border-bottom: 1px solid #e5e7eb; text-align: left; font-size: 13px; white-space: nowrap; }
    th { background: #111827; color: #fff; position: sticky; top: 0; }
    tr:nth-child(even) { background: #f9fafb; }
    .num { text-align: right; }
    .empty { text-align: center; padding: 24px; color: #6b7280; }
    .download { display: block; margin-top: 16px; text-align: center; background: #2563eb; color: #fff; padding: 12px 16px; border-radius: 8px; text-decoration: none; font-size: 15px; }
</style>
</head>
<body>
    <h1>🎰 ${escapeHTML(session.lottery)} — Turno ${escapeHTML(session.time_slot)}</h1>
    <p class="sub">📅 ${escapeHTML(session.date)} · ${(bets || []).length} apuestas</p>
    <div class="total">💰 Total: $${totalCup.toFixed(2)} CUP · $${totalUsd.toFixed(2)} USD</div>
    <div class="wrap">
        <table>
            <thead>
                <tr><th>Usuario</th><th>Tipo</th><th>Jugadas</th><th class="num">CUP</th><th class="num">USD</th><th class="num">Bono</th><th>ID</th><th>Hora</th></tr>
            </thead>
            <tbody>
                ${vacio ? `<tr><td colspan="8" class="empty">No hay apuestas en esta sesión</td></tr>` : rows}
            </tbody>
        </table>
    </div>
    <a class="download" href="${escapeHTML(downloadUrl)}" download="${escapeHTML(session.lottery)}_${escapeHTML(session.time_slot)}_${escapeHTML(session.date)}.html">📥 Descargar archivo</a>
</body>
</html>`;
}
```

#### D. Endpoint público que sirve el HTML (abre/descarga)

Agregar después de los endpoints de sesión existentes (~línea 3460):

```js
// --- Ver/descargar apuestas de una sesión (HTML) ---
// Acceso protegido: requiere ?token= (HMAC del session id). Sin token no se puede abrir.
app.get('/export-session/:sessionId', async (req, res) => {
    const sessionId = req.params.sessionId;
    const expected = getSessionExportToken(sessionId);
    const token = req.query.token;
    if (!token || token !== expected) {
        return res.status(403).send('<!DOCTYPE html><html lang="es"><body style="font-family:sans-serif;text-align:center;padding:40px">⛔ <b>No autorizado</b></body></html>');
    }

    const { data: session } = await supabase
        .from('lottery_sessions')
        .select('*')
        .eq('id', sessionId)
        .single();

    if (!session) return res.status(404).send('Sesión no encontrada');

    const { data: bets } = await supabase
        .from('bets')
        .select('*, users:user_id(first_name, username), referrers:referrer_id(first_name, username)')
        .eq('session_id', sessionId)
        .order('placed_at', { ascending: true });

    const download = req.query.download === '1';
    const downloadUrl = buildSessionExportUrl(sessionId, true);
    const html = generateSessionHtml(session, bets || [], downloadUrl);

    if (download) {
        res.setHeader('Content-Disposition', `attachment; filename="${session.lottery.replace(/\s+/g, '_')}_${(session.time_slot || '').replace(/[^\w]/g, '')}_${session.date}.html"`);
    }
    res.setHeader('Content-Type', 'text/html; charset=utf-8');
    res.send(html);
});
```

> Comparación del token: para rigor se puede usar `crypto.timingSafeEqual`, pero con igualdad normal basta para este caso (el token es HMAC).

#### E. Endpoint admin que devuelve el enlace de la última sesión cerrada por lotería (panel web)

```js
// --- URL de apuestas de la última sesión cerrada de una lotería (web panel) ---
app.get('/api/admin/session-export-url', requireAdmin, async (req, res) => {
    const { lottery } = req.query;
    if (!lottery) return res.status(400).json({ error: 'Falta lottery' });

    const { data } = await supabase
        .from('lottery_sessions')
        .select('id, lottery, time_slot, date')
        .eq('lottery', lottery)
        .eq('status', 'closed')
        .order('end_time', { ascending: false })
        .limit(1)
        .maybeSingle();

    if (!data) return res.status(404).json({ error: 'No hay sesiones cerradas para esta lotería' });

    res.json({ url: buildSessionExportUrl(data.id), session: data });
});
```

> `end_time` descendente garantiza que es la **última sesión que cerró**, no solo la última por fecha.

#### F. Notificación en el toggle web (~línea 3437)

En el bloque `else` del `POST /api/admin/lottery-sessions/toggle`, después de `broadcastToAllUsers(...)`, cambiar el botón por URL y enviar **solo** a subadmins con `session_exporter` (ya no se incluye `ADMIN_IDS`; el superadmin usa el panel web). Reutiliza `data` que ya devuelve el update:

```js
} else {
    await broadcastToAllUsers(
        `🔴 <b>SESIÓN CERRADA</b>\n\n` +
        `🎰 ${region?.emoji || '🎰'} <b>${data.lottery}</b> - Turno <b>${data.time_slot}</b>\n` +
        `📅 Fecha: ${data.date}\n\n` +
        `❌ Ya no se reciben más apuestas.\n` +
        `🔢 Pronto anunciaremos el número ganador. ¡Mantente atento!`
    );

    // Notificación con botón de ver apuestas (solo subadmins con privilegio)
    const { data: rolesData } = await supabase.from('admin_roles').select('telegram_id').eq('role', 'session_exporter');
    for (const r of rolesData || []) {
        try {
            await axios.post(`https://api.telegram.org/bot${BOT_TOKEN}/sendMessage`, {
                chat_id: Number(r.telegram_id),
                text: `📊 <b>Jugadas</b>\n\n🎰 ${data.lottery} · <b>${data.time_slot}</b>\n📅 ${data.date}\n\nPulsa el botón para ver las apuestas de la sesión.`,
                parse_mode: 'HTML',
                reply_markup: {
                    inline_keyboard: [[{ text: '👁️ Ver apuestas de la sesión', url: buildSessionExportUrl(data.id) }]]
                }
            });
        } catch (e) {}
    }
}
```

---

### Archivo: `app.html`

#### A. Botón "Ver apuestas" por lotería en `renderSessionsView()` (~línea 2749)

El botón **solo se muestra cuando la lotería tiene una sesión `closed`** (ni activa ni inactiva), y abre los datos de la última sesión cerrada. Para saberlo, la vista carga además la lista de sesiones cerradas:

**Inicio de la función** — agregar el fetch de sesiones cerradas junto al existente:

```js
// ANTES:
let sessions = [];
try {
    const res = await fetch(`/api/admin/lottery-sessions?date=${today}&userId=${currentUser.telegram_id}`);
    if (res.ok) sessions = await res.json();
} catch (e) { console.warn('Error fetching sessions', e); }

// DESPUÉS:
let sessions = [];
let closedSessions = [];
try {
    const [sessionsRes, closedRes] = await Promise.all([
        fetch(`/api/admin/lottery-sessions?date=${today}&userId=${currentUser.telegram_id}`),
        fetch(`/api/admin/lottery-sessions/closed?userId=${currentUser.telegram_id}`)
    ]);
    if (sessionsRes.ok) sessions = await sessionsRes.json();
    if (closedRes.ok) closedSessions = await closedRes.json();
} catch (e) { console.warn('Error fetching sessions', e); }
```

**Cabecera de cada bloque de lotería** — el botón aparece solo si existe una sesión cerrada de esa lotería (y solo para superadmin, usando `isAdmin`):

```js
// ANTES:
html += `<div class="mb-4 bg-gray-800/30 p-3 rounded-xl"><h5 class="font-bold text-lg mb-2">${lottery}</h5><div class="grid grid-cols-${slots.length} gap-2">`;

// DESPUÉS:
const hasClosed = closedSessions.some(s => s.lottery === lottery);
html += `<div class="mb-4 bg-gray-800/30 p-3 rounded-xl">
    <div class="flex items-center justify-between mb-2">
        <h5 class="font-bold text-lg">${lottery}</h5>
        ${isAdmin && hasClosed ? `<button onclick="openSessionExport('${lottery}')" class="bg-yellow-600 px-3 py-1 rounded-lg text-xs">📊 Ver apuestas</button>` : ''}
    </div>
    <div class="grid grid-cols-${slots.length} gap-2">`;
```

> El botón se oculta si no hay ninguna sesión cerrada para esa lotería (todas activas o inactivas). Al tocarlo abre el HTML de la última sesión cerrada de esa lotería (el endpoint `session-export-url` ordena por `end_time` desc).

#### B. Función `openSessionExport(lottery)`

Agregar junto a `window.toggleSession` (~línea 2811):

```js
window.openSessionExport = async (lottery) => {
    try {
        const res = await fetch(`/api/admin/session-export-url?lottery=${encodeURIComponent(lottery)}&userId=${currentUser.telegram_id}`);
        const data = await res.json();
        if (!res.ok) {
            iziToast.warning({ message: data.error || 'No hay sesiones cerradas para esta lotería' });
            return;
        }
        window.open(data.url, '_blank');
    } catch (e) {
        iziToast.error({ message: 'Error al abrir las apuestas' });
    }
};
```

> El `window.open(url, '_blank')` abre el HTML en una pestaña nueva del navegador; dentro, el botón "📥 Descargar archivo" dispara la descarga en el teléfono.

#### C. Agregar `session_exporter` al mapa de labels (~línea 3948)

En `renderFuncionesAdmin()`:

```js
const privilegeLabels = {
    deposit_approver: { icon: '✅', label: 'Aprobar o Denegar Depósitos' },
    withdraw_approver: { icon: '✅', label: 'Aprobar o Denegar Retiros' },
    schedule_manager: { icon: '🕐', label: 'Gestionar Horario de Retiros' },
    user_manager: { icon: '👥', label: 'Gestión de Usuarios' },
    session_exporter: { icon: '📊', label: 'Exportar datos de sesiones' }
};
```

#### D. Columna/checkbox del rol `session_exporter` en `renderRoleManagerView()` (~línea 4517)

**Cabecera** — antes del cierre `</tr>` del `<thead>`:

```html
<th title="Exportar datos de sesión">📊 Datos</th>
```

**Fila** — después del checkbox `activity_self` (~línea 4559), antes del cierre `</tr>`:

```js
const hasExporter = userRolesList.includes('session_exporter');
html += `<td class="text-center">
    <input type="checkbox" id="expChk_${u.telegram_id}"
        ${hasExporter ? 'checked' : ''}
        onchange="toggleUserRole(${u.telegram_id}, 'session_exporter', this.checked)"
        class="accent-yellow-500 w-4 h-4 cursor-pointer">
</td>`;
```

(Revisar el nombre de la variable lista de roles existente en esa función — en el código actual es `userRolesList`.) `session_exporter` es independiente, no requiere cascading en `toggleUserRole()`.

---

## Resumen de flujo

```
Sesión se cierra
        │
        ├─ Cierre MANUAL (bot toggle / web toggle) ──► mensaje a subadmins [session_exporter]
        │                                             botón URL ──► GET /export-session/:id?token=… ──► HTML (tabla) ──► botón "📥 Descargar"
        │
        └─ Cierre AUTOMÁTICO (cron) ──► NO se notifica a subadmins (solo broadcast foto)
        │
Web panel superadmin → botón "Sesiones" → "📊 Ver apuestas" por lotería (solo si hay sesión cerrada)
        └─ GET /api/admin/session-export-url?lottery=X → { url } → window.open(url) → HTML → botón "📥 Descargar"
```

## Resumen de archivos modificados

| Archivo | Cambios principales |
|---------|-------------------|
| `bot.js` | `require('crypto')` (builtin), rol en cache (4 lugares), helpers `getSessionExportToken`/`buildSessionExportUrl`, notificación con botón URL solo en el cierre manual (`toggle_session`), **sin callback** |
| `backend.js` | Rol en cache (3 lugares), `validRoles` actualizado, helpers + `generateSessionHtml()`, endpoint público `GET /export-session/:sessionId` (abre/descarga), endpoint admin `GET /api/admin/session-export-url`, notificación con botón URL en el toggle web |
| `app.html` | Botón "📊 Ver apuestas" por lotería en vista Sesiones (solo si hay sesión cerrada), función `openSessionExport()`, columna+checkbox del rol, label en `renderFuncionesAdmin` |
| `package.json` | **NO se crea** — no hay dependencias nuevas |

---

## Notas importantes

1. **No hay dependencias nuevas**: `crypto` es de Node.js. No se instala `xlsx`, no se crea `package.json`.

2. **El botón del bot es URL, no callback**: al tocarlo Telegram abre el navegador con el HTML. No existe `bot.action(/export_session_…/)`.

3. **Privilegio**: el mensaje del bot y el botón se otorgan **solo** a subadmins con rol `session_exporter` (se lee de `admin_roles`). El superadmin usa su botón del panel web (requiere `isAdmin`). `hasRole()` sigue retornando `true` para superadmin si se le diera un enlace manualmente.

4. **Seguridad del enlace**: el endpoint `/export-session/:id` es público pero exige `?token=` válido (HMAC del `session_id` con `BOT_TOKEN`). Sin token → 403. Solo quienes reciban el mensaje/botón tienen el enlace firmado.

5. **Botón de descarga dentro del HTML**: enlaza a la misma URL con `&download=1`; el backend responde con `Content-Disposition: attachment`, lo que dispara la descarga en el navegador del teléfono. Es el mismo HTML para subadmin y superadmin.

6. **Sesión "última que cerró"**: se obtiene ordenando por `end_time` descendente con `status='closed'`.

7. **`session_id` opaco**: no se aplica `parseInt`; se usa como string (soporta UUID o entero).

8. **Seguridad del contenido**: todo texto de usuario (nombres, jugadas) pasa por `escapeHTML()`.

9. **Columnas del HTML**: la columna "Bono" (antes "Bonus"); la columna "ID" muestra el `@username` del referido y, si no tiene username, su `telegram_id`; la columna "Hora" muestra solo la hora en formato 12h (con AM/PM) porque la fecha ya figura en el encabezado `📅`.

10. **Join de referido**: la consulta de apuestas agrega `referrers:referrer_id(first_name, username)` (además de `users:user_id(...)`) para resolver el usuario referidor y mostrar su `@username`.

11. **Mensaje al subadmin solo en cierres manuales**: layout fijo — encabezado `📊 Jugadas`, luego `🎰 Lotería · Turno` (al lado), `📅 fecha` debajo y el botón al final. Si la sesión cierra automáticamente (cron), **no se notifica** a los subadmins.

12. **Botón "Ver apuestas" del panel web**: solo aparece si la lotería tiene al menos una sesión cerrada (ni activa ni inactiva); muestra los datos de la última sesión cerrada de esa lotería.

9. **Sin apuestas**: el HTML se muestra igual pero con "No hay apuestas en esta sesión" y totales en 0.

10. **Los puntos de cierre de sesión**:
    - `bot.js toggle_session` — cierre manual desde bot
    - `bot.js closeExpiredSessions` — cierre automático por cron
    - `backend.js toggle` — cierre desde web panel