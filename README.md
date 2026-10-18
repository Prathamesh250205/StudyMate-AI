# StudyMate AI

RAG study assistant: upload notes → grounded chat with citations → AI quizzes → weak-topic dashboard → Stripe top-ups.

## Setup
1. **MongoDB Atlas** (free M0): create a cluster and copy the connection string. The vector index is created automatically on first start.
2. **Gemini API key**: https://aistudio.google.com/apikey. Optional key pool: put several comma-separated keys in `GEMINI_API_KEY`, each from a **different** Google project (keys in one project share a quota). Requests rotate across them, and a key that's rate-limited or overloaded hands the request to the next one.
3. `cp server/.env.example server/.env` and fill it in. For Google sign-in, create an OAuth **Web application** client in Google Cloud (Google Auth Platform → Clients) with origins `http://localhost` and `http://localhost:5173`, add your Gmail as a test user, and put the Client ID in `GOOGLE_CLIENT_ID` (server/.env) and `VITE_GOOGLE_CLIENT_ID` (client/.env).
4. **Windows shortcut:** double-click `start.bat`. It installs packages on first run, starts LiteLLM in Docker if `LITELLM_URL` is set, opens the API and website in two windows, and opens http://localhost:5173 once it's ready. Close those two windows to stop.

   Or run, in two terminals:
   ```
   cd server && npm install && npm start
   cd client && npm install && npm run dev
   ```
   Open http://localhost:5173 once the server prints `StudyMate API on :3000`.

   Use `npm start` for normal use and demos. `npm run dev` (watch mode) restarts the server when it detects file changes; on some Windows PCs it restarts mid-request, which shows up as "Can't reach the server" / `ECONNRESET`. Only use it while editing server code.

   **Troubleshooting**
   - `SSL alert number 80`: Atlas is blocking your IP. Add `0.0.0.0/0` under Atlas → Security → Network Access and wait for it to show Active.
   - `querySrv ECONNREFUSED`: your PC's DNS can't resolve `mongodb+srv://`. Use Atlas's standard `mongodb://host1,host2,host3/?ssl=true&replicaSet=...&authSource=admin` connection string instead.
   - "The AI is busy right now": Gemini's free tier is rate-limited or overloaded. Wait a minute and try again.

## LiteLLM proxy (optional, Docker)
Routes all Gemini calls through a LiteLLM container, which does the key pooling and retries.
1. Install and start Docker Desktop.
2. `cp litellm/.env.example litellm/.env` and fill in `GEMINI_KEY_1` and a `LITELLM_MASTER_KEY` (any `sk-...` string). For a pool, add `GEMINI_KEY_2`... and copy the two entries in `litellm/config.yaml`.
3. `cd litellm && docker compose up -d` (it then restarts with Docker Desktop automatically).
4. In `server/.env` add `LITELLM_URL=http://localhost:4000` and `LITELLM_KEY=` (same value as `LITELLM_MASTER_KEY`), then restart the server.

Remove those two lines from `server/.env` to call Gemini directly again (no Docker needed).

## Quick local test (no Atlas)
Uses your local MongoDB (`mongodb://127.0.0.1:27017`). Demo mode ignores `GEMINI_API_KEY` in `.env`, so pass the key in the terminal (PowerShell):
```
cd server
$env:GEMINI_API_KEY="your-key"; npm run demo
```
Then run the client as above and sign up with any email (no verification, 10,000 free tokens).
Login, notes, quizzes with Source **General knowledge (no notes)**, the test room and Progress all work.
Quizzes built from your notes and the AI tutor need Atlas Vector Search, so use the full setup for those.

## Password reset email (optional)
Set `SMTP_URL` (e.g. `smtps://user:pass@smtp.example.com:465`), `MAIL_FROM` and `APP_URL` (your site's address) in `server/.env`. Without `SMTP_URL`, reset links are printed to the server console, which is fine for local development.

## Payments (optional, test mode)
1. In the Stripe dashboard, create two Payment Links (a one-time token pack and a monthly Pro subscription). Set each one's after-payment redirect to `<your app>/wallet?paid=1`, and turn on the no-code Customer Portal.
2. Put them in `client/.env`:
   ```
   VITE_STRIPE_PACK_LINK=https://buy.stripe.com/test_...
   VITE_STRIPE_PRO_LINK=https://buy.stripe.com/test_...
   # Customer portal login link, so Pro users can cancel
   VITE_STRIPE_PORTAL_LINK=https://billing.stripe.com/p/login/test_...
   # Escape the $ (Vite expands $NAME in .env files)
   VITE_PACK_PRICE=\$5
   VITE_PRO_PRICE=\$9
   ```
3. Forward webhooks: `stripe listen --forward-to localhost:3000/webhook`, then put the `whsec_...` it prints into `server/.env`.
4. Pay with test card 4242 4242 4242 4242.

## Check
`cd server && npm run check` tests chunking (with PDF page ranges), citation parsing, grading, quiz question cleanup/shuffling and progress analytics (weak topic = last 3 attempts average under 70%).

## Production
`cd client && npm run build`, then `cd server && npm start` serves the API and the built client on :3000.

