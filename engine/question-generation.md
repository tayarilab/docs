# Question generation

How the engine generates multiple-choice questions from the chapter and serves them to the quiz.

## Source: cvs.json

- **Path:** `engine/data/medical/cvs.json`
- **Required field:** `full_text` (string). This is the only text sent to the LLM. It should contain the full chapter (e.g. Cardiovascular System) in one block; the server truncates to the first **12,000 characters** before building the prompt.
- **Optional:** `title`, `sections` (array of `{ "heading", "content" }`). Not used for generation; useful for structure or future features.

## Question format

Each question is an object:

- **question** (string): The question text.
- **options** (array of 4 strings): The four choices (A–D). Order is randomized per request so the correct answer is not always in the same position.
- **correct** (integer 0–3): Index of the correct option in `options` (after shuffling, this is updated so it still points to the right answer).

The quiz always shows **5 questions** per run; each run picks a random 5 from a **pool** of up to **20** questions.

## Generation flow

1. **Pool check**  
   If `engine/.quiz_pool.json` exists and contains at least 5 questions, the server uses that pool and skips generation.

2. **Ollama (default)**  
   - Unless `USE_EMBEDDED_LLM=1`, the server calls Ollama at `OLLAMA_HOST` (default `http://localhost:11434`) with model `OLLAMA_QUIZ_MODEL` (default `llama3.2`).
   - Request: `POST /api/generate` with `stream: false`, `format: "json"`, and a **prompt** that includes the chapter text and asks for a JSON array of **20** questions in the format above.
   - The server parses the JSON response, normalizes each item (question, options, correct), and uses the result as the new pool (up to 20 questions). If there are at least 5 valid questions, the pool is saved to `.quiz_pool.json`.

3. **Embedded LLM (fallback or forced)**  
   - Used if Ollama is not used (e.g. `USE_EMBEDDED_LLM=1`) or if Ollama fails or returns too few questions.
   - Package: `engine/quiz_llm`. It uses `llama-cpp-python` and a GGUF model (default: SmolLM2-360M-Instruct from Hugging Face). The model is downloaded on first use (or via `download_model.py`).
   - Same **prompt and output contract** as Ollama: generate 20 questions, JSON array with `question`, `options`, `correct`. Result is normalized and, if ≥ 5 questions, saved as the pool.

4. **No fallback questions**  
   If after both Ollama and embedded LLM the pool still has fewer than 5 questions, the server does **not** use any built-in questions. It returns **503** and an error message.

## Prompt (summary)

The prompt instructs the model to:

- Act as a medical educator.
- Use **only** the provided chapter content.
- Generate exactly **20** multiple-choice questions (or the configured pool size).
- Vary topics across the chapter.
- Each question: exactly **4 options**, **1 correct**.
- Return **only** a valid JSON array; no extra text. Each element: `{"question": "...", "options": ["A", "B", "C", "D"], "correct": 0}` with `correct` as 0–3.

The first 12,000 characters of `full_text` are pasted into the prompt after “Chapter content:”.

## Response parsing

- The server strips optional markdown code fences (e.g. ```json … ```) from the model output.
- It finds the first `[` and last `]` and parses the substring as JSON.
- For each object in the array (up to 20): it checks for `question` and `options` (length ≥ 4), and `correct` (default 0), and normalizes to the format above. Invalid or incomplete items are skipped.

## Serving a quiz (random 5 + shuffle)

When the pool has ≥ 5 questions:

1. **Random 5:** `random.sample(pool, 5)` so each request can get a different set.
2. **Shuffle options:** For each of the 5 questions, the four options are shuffled and the `correct` index is updated so it still refers to the same answer. So the correct answer can appear as A, B, C, or D at random.
3. The API returns `{ "questions": [ ... ] }` with these 5 questions.

## Environment and configuration

| Item | Where | Effect |
|------|--------|--------|
| Chapter path | `quiz_server.py`: `CVS_JSON_PATH` | Path to `cvs.json`. |
| Pool file | `quiz_server.py`: `POOL_FILE` | Where the 20-question pool is cached. |
| Pool size | `quiz_server.py`: `POOL_SIZE_TARGET` (20); `quiz_llm/generator.py`: `POOL_SIZE` (20) | Number of questions to ask the LLM to generate. |
| Quiz size | `quiz_server.py`: `QUIZ_SIZE` (5) | Number of questions returned per API call. |
| Ollama URL/model | `OLLAMA_HOST`, `OLLAMA_QUIZ_MODEL` | Which Ollama instance and model to call. |
| Embedded only | `USE_EMBEDDED_LLM=1` | Skip Ollama; use only quiz_llm. |
| Embedded model cache | `QUIZ_LLM_CACHE` | Directory for GGUF model when using project cache. |

## Optional: pre-download embedded model

To avoid downloading the model on the first quiz request:

```bash
cd engine
python download_model.py
```

Or:

```bash
python -c "from quiz_llm import ensure_model_downloaded; ensure_model_downloaded(); print('OK')"
```

This fills the cache (e.g. `engine/.quiz_models` or `QUIZ_LLM_CACHE`) so the first real request can use the embedded LLM without a download step.
