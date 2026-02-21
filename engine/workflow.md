# Engine workflow

End-to-end flow from chapter content to the quiz the user sees.

## 1. Prerequisites

- **Chapter content:** `engine/data/medical/cvs.json` must exist and contain a `full_text` string. This is the only source for question generation (no Markdown file).
- **LLM:** Either Ollama is running (e.g. `ollama run llama3.2`) or the embedded model is installed and used (`requirements-llm.txt` + `USE_EMBEDDED_LLM=1` when needed).

## 2. Server startup

1. User runs `python quiz_server.py` from the `engine` directory.
2. Flask app starts; serves static files from `engine/public` and registers:
   - `GET /` → `index.html`
   - `GET /api/questions` → returns JSON with 5 questions or an error.

## 3. User opens the quiz

1. User opens http://127.0.0.1:5000 in the browser.
2. Browser loads `public/index.html`.
3. The page immediately sends `GET /api/questions` to the server.

## 4. API: GET /api/questions

Flow inside the server:

1. **Load chapter**  
   - Read `engine/data/medical/cvs.json`.  
   - If missing or no `full_text`: respond with **404** and message to add `cvs.json` with `full_text`.

2. **Get question pool**  
   - Try to load cached pool from `engine/.quiz_pool.json`.  
   - If the pool has fewer than 5 questions (or file missing):
     - Call **Ollama** (unless `USE_EMBEDDED_LLM=1`) to generate up to 20 questions from `full_text`.  
     - If that fails or returns &lt; 5 questions, call **embedded LLM** (quiz_llm) to generate up to 20 questions.  
     - If the resulting pool has ≥ 5 questions, save it to `.quiz_pool.json`.  
   - If after all attempts the pool still has &lt; 5 questions: respond with **503** and a message that the LLM could not generate enough questions.

3. **Pick and shuffle**  
   - Take a **random 5** questions from the pool (`random.sample`).  
   - For each question, **shuffle the four options** and update the `correct` index so the right answer still matches.  
   - Return JSON: `{ "questions": [ { "question", "options", "correct" }, ... ] }`.

## 5. Browser: render and submit

1. The frontend receives the 5 questions and renders them as radio groups.
2. User selects one option per question and clicks “Submit & check answers”.
3. The frontend compares selected indices to `correct`, marks correct/wrong, and shows the score (e.g. 4/5).
4. “Try again (new questions)” reloads the page, which triggers a new `GET /api/questions` and a new random set of 5 from the pool.

## 6. Diagram (high level)

```
cvs.json (full_text)
        │
        ▼
┌───────────────────┐     miss / < 5      ┌─────────────────────┐
│ load .quiz_pool    │ ──────────────────►│ generate via Ollama │
│ .json              │                     │ or quiz_llm         │
└─────────┬─────────┘                     └──────────┬──────────┘
          │ pool ≥ 5                                 │ success
          │                                          ▼
          │                                save pool to .quiz_pool.json
          │                                          │
          └──────────────────┬───────────────────────┘
                              ▼
                   random.sample(5) + shuffle options
                              │
                              ▼
                   JSON response → browser → quiz UI
```

## 7. Error responses

| HTTP | When | Body |
|------|------|------|
| **404** | `cvs.json` missing or no `full_text` | `error`, `detail` (path to cvs.json). |
| **503** | Pool has &lt; 5 questions after trying Ollama and/or embedded LLM | `error` (e.g. “Is Ollama running?”), `detail` (e.g. “Need at least 5 questions from the LLM; got 0”). |

The frontend shows the `error` message in the red error box when the request fails.

## 8. Optional: pre-download embedded model

To populate the embedded model cache without triggering it from a quiz request:

- From `engine`: `python download_model.py`  
- Or: `python -c "from quiz_llm import ensure_model_downloaded; ensure_model_downloaded(); print('OK')"`

See [Question generation](question-generation.md) for prompt format, JSON contract, and pool behavior.
