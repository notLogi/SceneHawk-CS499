# SceneHawk — a mood-aware movie recommender

SceneHawk recommends films by *feeling*, not just genre. Instead of matching
on keywords like "action" or "comedy", it understands requests like
*"something cozy for a rainy Sunday"* or *"a tense slow-burn that builds dread"*
and returns films whose **atmosphere, tone, and pacing** actually match.

It's a RAG (retrieval-augmented generation) system built on ~995 films pulled
from TMDB, each enriched with LLM-inferred mood metadata and embedded into a
vector database for semantic search.

## How it works

1. **Fetch** — pull movie data (title, genres, synopsis, ratings) from TMDB.
2. **Enrich** — an LLM infers each film's `pacing`, `atmosphere`, `tone`,
   `emotional_register`, and a one-line "feels like…" summary from its synopsis.
3. **Embed** — each film's plot + mood text is turned into a vector so films
   can be compared by meaning.
4. **Retrieve** — a user's request is embedded the same way, and the closest
   films are pulled from the vector store.
5. **Generate** *(optional)* — an LLM writes a natural recommendation from the
   retrieved films, explaining why each fits.

## Project layout

```
recommend.py            Embeds the films; exports the RAG data (metadata/movies_rag.jsonl)
load_chroma.py          Builds the vector index and searches (OpenAI embeddings)
chat.py                 Full RAG chat: retrieval + an LLM-written answer
enrich_films.py         Adds the mood metadata to the source films (already done)
grab_movie_api/
  tmdb_to_rag_metadata.py   Fetches raw film data from TMDB
metadata/
  movies_metadata.json      500 popular films (enriched)
  movies_underrated.json    500 underrated films (enriched)
  movies_rag.jsonl          The built dataset the recommender queries
config/
  .env                      Your API keys (gitignored — never committed)
.env.example                Template: copy to config/.env and add your keys
```

## Setup

Requires Python 3.10+.

```bash
pip install chromadb python-dotenv numpy requests
```

### Add your API keys

SceneHawk uses two keys — one to embed queries, one to write the answer:

- `EMBED_API_KEY` — OpenAI key, embeds each query for the vector search.
- `REQUESTY_API_KEY` — Requesty key, generates the written recommendation.

Copy the template and fill both in:

```bash
cp .env.example config/.env
```

Then edit `config/.env`:

```
EMBED_API_KEY=sk-proj-your-openai-key
REQUESTY_API_KEY=rqsty-sk-your-requesty-key
```

## Running it

### 1. Build the vector index (one time)

```bash
python load_chroma.py
```

### 2. Search — ranked films only

```bash
python load_chroma.py --query "a slow sad film about grief"
python load_chroma.py --query "a heist movie"
```

### 3. Full chat — an LLM-written recommendation

```bash
python chat.py "something cozy for a rainy sunday"
python chat.py                          # interactive loop; blank line to quit
```

## Regenerating the data (optional)

The dataset is already built and committed, so you don't need to do this. But
to refresh it:

```bash
# 1. fetch films (needs a TMDB_API_KEY)
python grab_movie_api/tmdb_to_rag_metadata.py
python grab_movie_api/tmdb_to_rag_metadata.py --underrated

# 2. enrich with mood metadata (needs REQUESTY_API_KEY)
python enrich_films.py metadata/movies_metadata.json metadata/movies_metadata.json

# 3. re-embed into the RAG dataset (needs EMBED_API_KEY)
python recommend.py export-rag

# 4. rebuild the vector index
python load_chroma.py
```

## Notes

- **Never commit your keys.** `config/.env` is gitignored; only `.env.example`
  (with placeholders) is shared.
- The mood fields are *inferred from synopses*, not ground truth — they reflect
  what a plot summary implies a film feels like.
- Query embedding costs a fraction of a cent per search; the films are embedded
  once (already done and committed in `metadata/movies_rag.jsonl`).
