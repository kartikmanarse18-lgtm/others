# ArogyaBot AI Backend (Cloudflare Worker)

Holds your Gemini / Groq / OpenRouter keys **server-side** so every visitor to
`arogyabot.html` gets live AI symptom reports with zero setup on their end —
no key is ever pasted into the page or stored in a browser.

## What it does

- Exposes one endpoint: `POST /api/symptom-report` with body `{ "text": "..." }`
- Calls whichever of the three providers you've configured, **in parallel**,
  and returns whichever answers first (`Promise.any`) — one provider being
  slow or rate-limited never blocks a response.
- For Gemini specifically: first asks Gemini to research the symptom with
  **live Google Search grounding** turned on, then makes a second fast call
  to squeeze that researched answer into the strict JSON shape the app
  needs. (Combining grounding + forced JSON schema in a single call is
  currently only supported on Gemini 3 preview models, so this two-step
  approach keeps things working reliably on the stable Gemini 2.5 line.)
- Returns a normalized report `{ title, severity, basicReason, causes[],
  whenToSeeDoctor, homeRemedies[], specialist, redFlags[], provider }`.

## One-time setup

You only need **one** of the three keys to get this working; add more for
redundancy.

1. Install Wrangler (Cloudflare's CLI), if you don't have it:
   ```
   npm install -g wrangler
   ```

2. Log in (opens a browser window):
   ```
   wrangler login
   ```

3. From inside this `arogyabot-worker` folder, add whichever keys you have.
   Each command will prompt you to paste the key — it's stored encrypted on
   Cloudflare, never written to a file:
   ```
   wrangler secret put GEMINI_API_KEY
   wrangler secret put GROQ_API_KEY
   wrangler secret put OPENROUTER_API_KEY
   ```
   Free keys, no credit card required:
   - Gemini: https://aistudio.google.com/apikey
   - Groq: https://console.groq.com/keys
   - OpenRouter: https://openrouter.ai/keys

4. Deploy:
   ```
   wrangler deploy
   ```
   Wrangler prints a URL like `https://arogyabot-ai.YOUR-SUBDOMAIN.workers.dev`.

5. Open `arogyabot.html`, find this line near the top of the AI section:
   ```js
   const AI_BACKEND_URL = 'https://PASTE-YOUR-WORKER-URL.workers.dev';
   ```
   Replace it with the URL from step 4, save, and re-share/host the file.
   That's it — every visitor who opens the file now gets live AI reports.

## Updating keys later

Re-run `wrangler secret put GEMINI_API_KEY` (etc.) to rotate a key, then
`wrangler deploy` again. No change to `arogyabot.html` is needed.

## Before a real public launch

- In `worker.js`, change `ALLOWED_ORIGIN = '*'` to your actual hosting
  domain, so only your site can call this Worker.
- Consider adding basic rate limiting (Cloudflare's free tier includes
  simple rate-limiting rules) so one visitor can't burn through your whole
  provider quota.
