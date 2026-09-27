# backend-for-restaurant-bot-

Backend for the **"need a restaurant rec?"** chat on my portfolio site
([emmagelman-alt.github.io/15-113-portoflio](https://emmagelman-alt.github.io/15-113-portoflio/)).
A visitor names a city or country plus a cuisine or kind of place, and the bot answers with
spots from my Beli "Been" list that I rated **8.0 or higher**.

- **Live API:** `https://emma-restaurant-recs.onrender.com`
- **Stack:** Python, FastAPI, Groq API (`openai/gpt-oss-120b`), hosted on Render
- **Frontend code:** lives in the portfolio repo (`index.html`, `js/script.js`, `css/style.css`)

## What the backend does

The AI never chooses the restaurants. Groq does two small language jobs, and plain Python does
the matching against `beli_list.json`, so every result is a real place I've been to and rated.

1. **Understand the request (Groq):** turn free text like "sushi in nyc" into filters
   (`city: "New York City"`, `cuisine: "Sushi"`). Groq may only choose from the cities, countries,
   cuisines and place types that exist in my list, and the server throws out any value that isn't
   in that list.
2. **Find matches (Python):** filter my 368 places rated 8.0+ by area, cuisine and place type,
   and return **every** match, sorted by score. Matching is forgiving:
   - A place type also matches a tag with the same name, so a cafe tagged "Bakery" counts as a
     bakery.
   - If the exact request finds nothing, the search loosens step by step. "Japanese dessert"
     tries Japanese, then dessert. A tag with no local matches, like "Pastries", falls back to the
     place type it usually belongs to (bakery). The intro then says the results are broader than
     asked.
3. **Write the intro (Groq):** one or two casual sentences in my voice about the matched places
   only. The list itself is sent as data, not written by the AI.

If the location or the cuisine is missing, or nothing matches, the server replies with a fixed
follow-up question and makes no second Groq call.

### Endpoints

#### `GET /healthz`
Health check. Render uses it to confirm the service is up, and the frontend calls it to wake the
server from free-tier sleep.

```json
{ "ok": true, "places": 368, "cities": 54 }
```

#### `POST /api/recs`
Get recommendations for one chat message.

**Request body (JSON)**

| Field | Type | Required | Notes |
|---|---|---|---|
| `message` | string | yes | The visitor's latest message, up to 500 characters |
| `history` | array of `{role, content}` | no | Earlier turns (`role` is `"user"` or `"assistant"`), up to 12. Lets follow-ups like "what about dessert?" keep the city |

```json
{
  "message": "what about dessert?",
  "history": [
    { "role": "user", "content": "sushi in nyc" },
    { "role": "assistant", "content": "I've been on a sushi hunt around the city..." }
  ]
}
```

**Response (200)**

| Field | Type | Notes |
|---|---|---|
| `reply` | string | The bot's message: an intro, or a follow-up question when details are missing |
| `places` | array | All matching places, best first. Empty when `reply` is a question or nothing matched |

Each place has `name`, `city`, `country`, `category` (`restaurant`, `cafe`, `bakery`, `dessert`,
`bar` or `market`), `cuisines` (list of strings) and `score` (my Beli rating out of 10).

```json
{
  "reply": "If you're craving something sweet in the city, here are my top dessert picks!",
  "places": [
    { "name": "Lysée", "city": "New York City", "country": "United States",
      "category": "dessert", "cuisines": ["Korean", "Dessert", "Bakery"], "score": 9.7 }
  ]
}
```

**Errors** return `{ "detail": "<message safe to show a visitor>" }`:

| Status | When |
|---|---|
| 422 | Empty message, or a message or history that's too long |
| 429 | More than 30 requests in a minute, or Groq's rate limit was reached |
| 502 / 504 | Groq failed or timed out |
| 503 | `GROQ_API_KEY` isn't set on the server |

## How the frontend talks to the backend

All of this is in the "Restaurant rec chat" section at the end of `js/script.js` in the portfolio
repo.

| When | Call | What the frontend does with it |
|---|---|---|
| Visitor hovers over or opens the "need a restaurant rec?" bubble | `GET /healthz` | Nothing; the call only wakes the Render free-tier server so it's ready by the time they send |
| Visitor sends a message or taps a suggestion | `POST /api/recs` with `message` and the last 12 turns of `history` | Replaces "thinking…" with `reply` as a chat bubble and renders `places` as a list (score, name, city, cuisines). Adds both turns to its history for the next request |
| Request fails | (same call) | Shows `detail` from the error, or, if the server couldn't be reached at all, "I'm still waking up, send that again", and pings `/healthz` again |

The server stores nothing between requests. The browser keeps the conversation and sends recent
turns with each message. Everything from the server is inserted with `textContent`, never
`innerHTML`, so a response can't inject HTML or scripts into the page.

## Setup and running locally

Requires Python 3.11+ and a free Groq API key from
[console.groq.com/keys](https://console.groq.com/keys).

```bash
git clone https://github.com/emmagelman-alt/backend-for-restaurant-bot-.git
cd backend-for-restaurant-bot-
python -m venv .venv
.venv\Scripts\activate          # macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # then paste your key into .env
uvicorn app:app --reload
```

Then open http://localhost:8000/docs and try `POST /api/recs` with `{"message": "sushi in nyc"}`.

### Environment variables

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `GROQ_API_KEY` | yes | none | Groq API key |
| `GROQ_CHAT_MODEL` | no | `openai/gpt-oss-120b` | Groq model used for both calls |
| `MIN_SCORE` | no | `8.0` | Lowest Beli score to recommend |
| `ALLOWED_ORIGINS` | no | `https://emmagelman-alt.github.io` | Comma-separated sites allowed to call the API. To test a local copy of the frontend, add it, e.g. `https://emmagelman-alt.github.io,http://localhost:4173` |

### Deploying on Render

`render.yaml` defines the service. In Render choose **New → Blueprint**, pick this repo and paste
`GROQ_API_KEY` when asked. The free tier sleeps after about 15 idle minutes, so the first request
after that can take 30–50 seconds.

## Secrets and security

- **The Groq API key only exists on the server.** Locally it's in `.env`, which is gitignored and
  never committed; `.env.example` has the variable name with no value. On Render it's set in the
  dashboard (`sync: false` in `render.yaml` keeps it out of the file).
- **The frontend has no secrets.** The browser only knows the public Render URL; every Groq call
  happens on the backend.
- **CORS** allows only my GitHub Pages site, so other websites can't call the API from their own
  pages.
- **Abuse limits:** messages are capped at 500 characters and history at 12 turns, and the server
  allows 30 requests a minute.
- **Model output is checked:** filters Groq returns are checked against my real data before use,
  and the prompts tell the model to treat visitor text as data, not instructions.
- There's no user login. The API is intentionally public and read-only.

## Files

| File | What it is |
|---|---|
| `app.py` | FastAPI service (endpoints, Groq calls, matching) |
| `beli_list.json` | My Beli Been list (name, city, country, cuisine, place type, score), transcribed from a screen recording with ChatGPT |
| `requirements.txt` | Python dependencies |
| `render.yaml` | Render deploy config |
| `.env.example` | Template for local environment variables |
| `prompt_log.md` | AI tools used and the key prompts behind this project |
