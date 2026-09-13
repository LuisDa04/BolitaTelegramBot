# 🔧 Requerimientos para montar el bot

## 1. Dependencias npm

El repo no incluye `package.json`, así que hay que instalar las dependencias directamente:

```bash
npm install telegraf telegraf-session-local @supabase/supabase-js express cors multer node-cron moment-timezone axios dotenv
```

Requiere **Node.js** (probado con Node v24). En desarrollo se necesita también que `node_modules` exista junto a `bot.js`/`backend.js`.

## 2. Servicios externos

- **Supabase** (PostgreSQL) con las tablas:
  `users`, `bets`, `lottery_sessions`, `play_prices`, `winning_numbers`, `exchange_rate`,
  `deposit_requests`, `deposit_methods`, `withdraw_requests`, `withdraw_methods`,
  `admin_roles`, `app_config`, `deleted_users`.
- **Telegram (BotFather)**: bot creado, token `BOT_TOKEN`, Web App habilitada (botón de menú)
  y nombre visible del bot (se usa para los enlaces ocultos).
- **OCR.space** (opcional): `OCR_API_KEY` solo para lecturas de tasas USDT/TRX diarias.
- **Dominio HTTPS** para `WEBAPP_URL` (Telegram Web Apps exigen HTTPS en producción;
  en desarrollo sirve `http://localhost:3000`).

## 3. Archivo `.env`

```env
BOT_TOKEN=token_del_bot_de_BotFather
ADMIN_IDS=123456789,987654321
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_SERVICE_KEY=service_role_key
BONUS_CUP_DEFAULT=70
TIMEZONE=America/Havana
WEBAPP_URL=https://tu-dominio.com
OCR_API_KEY=opcional_para_lectura_de_comprobantes
PORT=3000
```

## 4. Carpeta `Assets/`

El bot y la web dependen de imágenes/videos de `Assets/` (no está versionada en el repo):

- `Assets/Back.jpg` — fondo de la Web App.
- `Assets/Inicio.webp` — bienvenida con foto.
- `Assets/FLO.mp4`, `Assets/GA.mp4`, `Assets/NY.mp4` — videos de sesiones.
- `Assets/{loteria}/{loteria}_{turno}.*` — fotos de apertura/cierre por lotería y turno.

Sin esta carpeta el servidor arranca, pero faltan fotos y videos.

## 5. Arranque

Un solo proceso:

```bash
node backend.js
```

Levanta Express + el bot de Telegram en polling (webhook desactivado) + los crons.

> Nota: el README menciona `node bot.js` y `node backend.js`, pero `bot.launch()` solo
> ocurre en `backend.js`. Correr ambos procesos duplicaría los crons.

## 6. Archivos/generados localmente

- `session_db.json` / `sessions.json` — sesiones locales (ignorados por git).
- `.env` — ignorado por git, no se sube.