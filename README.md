# 🎲 Bolita Telegram Bot · 4pu3$t4$_Qva

Bot de Telegram de apuestas de lotería con **Web App integrada**, pensado para las loterías
estaduales de **Florida 🦩, Georgia 🍑 y Nueva York 🗽**. Incluye panel de administración,
sistema de bonos, referidos, recargas/retiros multi-moneda y sesiones de juego programadas.

## ✨ Características

- 🎰 Apuestas por lotería y turno (Fijo 🎯, Corridos 🏃, Centena 💯, Parlet 🔗)
- 💵 Saldos en CUP y USD con tasas de cambio configurables (+ USDT, TRX, MLC para recargas/retiros)
- 🎁 Bono de bienvenida configurable (no transferible ni retirable)
- 👥 Sistema de referidos con comisión de por vida
- 🔄 Transferencias entre usuarios (CUP/USD)
- 🛡️ Panel admin completo: depósitos, retiros, ganadores, tasas, precios, horarios, roles de sub-admins, estadísticas y gestión de usuarios
- 🕐 Sesiones de lotería automáticas (apertura/cierre) con `node-cron` y hora de Cuba
- 🖼️ Fotos de apertura/cierre de sesión, fondo de la web y bienvenida con imagen (`Assets/`)
- 🔒 Protección de contenido (`protect_content`) en fotos sensibles: resultados y bienvenida
- 💬 Sistema de soporte: los mensajes de usuarios se reenvían a admins con opción de respuesta

## 🗂️ Estructura del proyecto

| Archivo | Descripción |
|---|---|
| `bot.js` | Bot de Telegram (Telegraf): comandos, apuestas, panel, broadcasts |
| `backend.js` | API Express + servidor de la Web App (autenticación vía initData de Telegram) |
| `app.html` | Interfaz web (Telegram Web App) con Tailwind CSS |
| `Assets/` | Imágenes: sesiones por lotería/turno, fondo web (`back.jpg`) y bienvenida (`inicio.webp`) |

## ⚙️ Tecnologías

- [Node.js](https://nodejs.org)
- [Telegraf](https://telegraf.js.org) + telegraf-session-local
- [Express](https://expressjs.com) + Multer + CORS
- [Supabase](https://supabase.com) (PostgreSQL como base de datos)
- node-cron · moment-timezone · axios · dotenv

## 🚀 Puesta en marcha

1. Instalar dependencias:

   ```bash
   npm install telegraf telegraf-session-local @supabase/supabase-js express cors multer node-cron moment-timezone axios dotenv
   ```

2. Crear un archivo `.env` junto al proyecto:

   ```env
   BOT_TOKEN=token_del_bot_de_BotFather
   ADMIN_IDS=123456789,987654321
   SUPABASE_URL=https://xxxx.supabase.co
   SUPABASE_SERVICE_KEY=service_role_key
   BONUS_CUP_DEFAULT=70
   TIMEZONE=America/Havana
   WEBAPP_URL=https://tu-dominio.com
   OCR_API_KEY=opcional_para_lectura_de_comprobantes
   ```

3. Ejecutar:

   ```bash
   node bot.js      # Bot de Telegram
   node backend.js  # API + Web App
   ```

## 📝 Notas

- La carpeta `Assets/` contiene las imágenes usadas por el bot y la web; debe existir junto a
  `bot.js`/`backend.js` en producción.
- El alta de usuarios, bonos y re-ingresos tras eliminación se gestionan automáticamente:
  la bienvenida (con foto) solo se envía al crear la cuenta o al re-registrarse.
