# Prompt log

## AI tools used

| Tool | Used for |
|---|---|
| **Claude Code** (Claude desktop app, model **Claude Opus 5.5**) | Planning the architecture, writing `app.py`, the frontend chat in my portfolio repo, `render.yaml` and this documentation; testing locally in its browser pane; committing and pushing |
| **ChatGPT** ([shared conversation](https://chatgpt.com/share/6ab9716d-3db4-83ea-b3ed-8a4b02f206d0)) | Turning a screen recording of my Beli Been list into `beli_list.json` |
| **Groq API**, model `openai/gpt-oss-120b` | Runs inside the deployed app, not a coding tool: extracts search filters from each visitor message and writes the short reply intro |

## Creating the data with ChatGPT

Beli has no export or public API, so I screen-recorded my Been list in the app and had ChatGPT
transcribe it. The restaurants and ratings are my own; ChatGPT only converted them to JSON.

1. *(Uploaded the screen recording)*: ChatGPT extracted 395 entries in rank order as JSON and
   flagged 20 names it couldn't read confidently (cut-off or non-Latin text) with
   `"needs_review": true`.
2. > Can you also include the location of each restaurant (city, country) and the type of cuisine,
   > and if it is a restaurant, bar, dessert, bakery, cafe)

   This added `city`, `country`, `cuisine` and `venue_type`. Places it couldn't identify were
   flagged with `"metadata_needs_review": true`.
3. > Instead of having "rank" can you assign each entry the score I gave it on beli?

   This replaced `rank` with my Beli `score`, the field the backend uses for the 8.0 cutoff.

Because this was transcribed from video, the backend defends against its gaps. `load_places()`
skips placeholder names like `[Korean name]` and entries with no city, and keeps only the
highest-scored copy of any duplicate.

## Key prompts to Claude Code

My prompts are quoted as I typed them, in order. Each is followed by what it led to.

**1. The original request and plan**
> plan. I would like to add an option to chat with the cartoon drawing of me on my github pages
> website that allows visitors to click on a text bubble that says "need a restaurant rec?" which
> opens up a chat window on the whiteboard where the user can chat with me to ask for restaurant
> recommendations. The user should be able to ask for recs by giving a location (city, country), a
> cuisine, and optionally a price point. I would like the website to connect to a python service on
> render.com that uses an api (i dont know which would be the best, maybe a combo of chatgpt api and
> something else?) I want the chatbot to respond based off the data from my Beli app been list. the
> chatbot should take in the visitors request and give back a list of restaurants or cafes or
> dessert places etc i have been to that I have rated with an 8.0 or higher that are in the same
> geographic location, category, and are the same cuisine type. [...] Can you show me what the
> python code looks like along with a simple html file that implements the frontend? (ask me any
> clarifying questions before acting)

Claude asked four clarifying questions first. My answers set the architecture:
- **Data:** use an exported file of my Beli list, because Beli has no public API and scraping
  would be fragile.
- **AI:** use Groq to read the request and write the reply, while plain Python does the
  filtering, so the bot can't invent restaurants.
- **Deployment:** a separate public Render service, not part of my password-protected
  Sobremesa app.
- **Frontend:** the chat takes over the whiteboard in place of the project notes.

**2. Supplying the data**
> here is the beli json file

> use this instead

These were the two ChatGPT versions above. The first only had rank order, not scores. The second had a score for every place, so the
backend was rewritten to read the JSON directly. It skips placeholder names, removes repeat
visits and keeps places scored 8.0 or higher. The file has no prices, so the price filter was
dropped, and the bot says so if someone asks about budget.

**3. Setting up the repo and keeping secrets out of it**
> i want to put the backend code into a new github repo. what kind of git ignore should i add

> should the backend repo be private or public

> move this code into the new github repo i made called backend-for-restaurant-bot-

> only commit the code for the backend to the new repo.

This produced the Python `.gitignore` (so `.env` is never committed), a `.env.example` template,
a public repo containing only backend files, and a `render.yaml` for deployment.

**4. Running and deploying**
> i installed python. what else do i have to do to make it test and use render

> i paste key into .env example right

Claude set up a virtual environment and tested the data loading and matching without a key. It
also caught that the key belongs in `.env` (gitignored), not in `.env.example`, which is
committed. After I deployed on Render, Claude confirmed the live `/healthz`, a real
`/api/recs` answer, and that CORS allows my GitHub Pages site.

**5. Building the real frontend**
> yes go ahead

This added the speech bubble and the whiteboard chat to my portfolio, plus a full-screen version
for phones. Opening the chat pings `/healthz` to wake the Render free tier. Claude tested it
against a local copy of the backend: follow-up questions, the error shown when the server is
asleep, and closing with Escape and the × button. Then it pushed the change to GitHub Pages.

**6. Understanding and documenting it**
> explain what each function and endpoint does so you can demonstrate understanding

> [assignment requirements for this README and prompt log]
