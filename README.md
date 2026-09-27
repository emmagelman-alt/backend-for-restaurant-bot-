# backend-for-restaurant-bot-

Backend for the "need a restaurant rec?" chat on my portfolio site. Visitors ask for a
city (or country) plus a cuisine or type of place, and get back spots from my Beli
"Been" list that I rated 8.0 or higher.

## How it works

1. `POST /api/recs` receives the visitor's message and recent chat history.
2. Groq turns the free text into filters (city, country, cuisine, place type), limited to
   values that actually appear in `beli_list.json`.
3. Plain Python filters the list and sorts by score, so the bot can't invent restaurants.
4. Groq writes a short, casual intro; the frontend renders the list from the returned data.

## Files

- `app.py`: FastAPI service
- `beli_list.json`: my Beli Been list
- `render.yaml`: Render deploy config

## Run locally

```bash
pip install -r requirements.txt
cp .env.example .env   # then add your GROQ_API_KEY
uvicorn app:app --reload
```

The frontend lives in my portfolio site repo. The API allows requests from the origins in
`ALLOWED_ORIGINS` (defaults to my GitHub Pages site).

## Deploy

Create a Render Blueprint from this repo (it reads `render.yaml`) and set `GROQ_API_KEY`
in the dashboard. The free tier sleeps when idle, so the first request can take ~30-50s.
