# Local AI Story Generator

A full-stack application that generates interactive branching stories using a locally-hosted LLM via LM Studio — no OpenAI API key required.

---

## Architecture

```
Browser (localhost:5173)
    └── Vite proxy /api
            └── FastAPI (localhost:8000)
                    └── LM Studio (127.0.0.1:1234/v1)
```

The backend delegates all text generation to a locally running `qwen/qwen3-4b-2507` model served by LM Studio. The frontend proxies `/api` requests through Vite to FastAPI, which in turn calls the LM Studio OpenAI-compatible endpoint.

---

## Prerequisites

Install the following **once** before proceeding:

| Tool | Version | Notes |
|------|---------|-------|
| [Python](https://www.python.org) | 3.11+ | Tick **"Add python.exe to PATH"** during installation |
| [Node.js](https://nodejs.org) | LTS | — |
| [LM Studio](https://lmstudio.ai) | Latest | With `qwen/qwen3-4b-2507` loaded |

After installation, restart VS Code, open a terminal (`` Ctrl+` ``), and verify:

```powershell
python --version
node --version
```

---

## 1. Configure LM Studio

The LM Studio server should already be running. One setting is critical: **the entire story is generated in a single reply**, so the context window must be large enough.

1. In the right panel, open the **Load** tab.
2. Set **Context Length** to **8192** or higher (**16384** is safer).
3. Reload the model.
4. Verify it responds:

   ```powershell
   curl.exe http://127.0.0.1:1234/v1/models
   ```

---

## 2. Project Configuration

Four files need to be edited.

### 2a. `backend/.env`

Replace the entire contents. **Do not add extra keys** — the settings class rejects unknown ones.

```env
DEBUG=True
DATABASE_URL=sqlite:///./database.db
API_PREFIX=/api
ALLOWED_ORIGINS=http://localhost:5173
OPENAI_API_KEY=lm-studio
```

> `OPENAI_API_KEY` is a dummy value only. The code requires the field to exist, but it is never used against OpenAI.

### 2b. `backend/core/story_generator.py`

Replace the entire `_get_llm` method:

```python
@classmethod
def _get_llm(cls):
    return ChatOpenAI(
        model="qwen/qwen3-4b-2507",
        api_key="lm-studio",
        base_url="http://127.0.0.1:1234/v1",
        temperature=0.7,
        timeout=600,
    )
```

### 2c. `frontend/src/util.js`

```js
export const API_BASE_URL = "/api"
```

### 2d. `frontend/vite.config.js`

Replace the entire file. This enables the proxy unconditionally, so no frontend `.env` is needed.

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': { target: 'http://localhost:8000', changeOrigin: true }
    }
  }
})
```

### Recommended Bug Fixes

| File | Change | Reason |
|------|--------|--------|
| `frontend/src/components/StoryGame.jsx` | `is_winning_endig` → `is_winning_ending` | Typo fix |
| `frontend/src/components/StoryGenerator.jsx` | `jobStatus === "processing"` → `(jobStatus === "processing" \|\| jobStatus === "pending")` | Prevents the loading screen from hanging indefinitely when the first status poll returns `pending` |

---

## 3. Run the Backend

In **terminal 1**:

```powershell
cd backend
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python main.py
```

**Notes:**

- If activation is blocked (*"running scripts is disabled"*), run `Set-ExecutionPolicy -Scope Process Bypass` first, then activate again.
- If `psycopg2-binary` fails to install, remove that line from `requirements.txt` and rerun. Postgres is not needed locally.
- Success looks like: `Uvicorn running on http://0.0.0.0:8000`. The server auto-reloads on file edits.

---

## 4. Run the Frontend

Open **terminal 2** (`` Ctrl+Shift+` ``):

```powershell
cd frontend
npm install
npm run dev
```

The app runs at **http://localhost:5173**.

---

## 5. Test the Backend API

Open **http://localhost:8000/docs** — an interactive Swagger page.

1. **`POST /api/stories/create`** — click *Try it out*, send `{"theme": "pirates"}`, and press **Execute**. Copy the `job_id` from the response.
2. **`GET /api/jobs/{job_id}`** — paste the id and execute repeatedly. The status transitions: `pending` → `processing` → `completed`. Copy the `story_id`. If it reports `failed`, the `error` field explains why.
3. **`GET /api/stories/{story_id}/complete`** — returns the full story tree as JSON.

While running, the **Developer Logs** panel in LM Studio shows the incoming request.

---

## 6. Test the Frontend

1. Open **http://localhost:5173**.
2. Enter a theme (e.g. *"space pirates"*) and click **Generate Story**.
3. Wait — a 4B model may take **30 seconds to a few minutes**.
4. You are redirected to `/story/<id>` and can click through the choices.
5. Press **F12** and open the **Network** tab to observe the `/api/...` calls.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| Job `failed` with a JSON or validation error | The 4B model returned malformed JSON | Click **Try Again** — generation is stochastic |
| Too slow / unreliable | Model generates too much branching | In `backend/core/prompts.py`, change *"2-3 options"* → *"exactly 2 options"* and *"3-4 levels deep"* → *"3 levels deep"* |
| Connection errors | LM Studio server off, or model not loaded | Confirm the server is running and the model is loaded |
| `ModuleNotFoundError` | Virtualenv not active | The terminal prompt should begin with `(.venv)` |

If a problem persists, capture the terminal error or the job's `error` text for further diagnosis.