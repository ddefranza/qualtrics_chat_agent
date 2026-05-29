# Qualtrics LLM Chat Widget

Embed a multi-turn LLM chat interface inside a Qualtrics survey, with full conversation data recorded in embedded data fields for export and analysis.

Built for researchers who want to study human–AI interaction within a controlled survey context, with the ability to swap models (OpenAI, xAI/Grok, Anthropic, or a custom/RAG backend) across experimental conditions.

---

## How it works

A chat interface is hosted on GitHub Pages and embedded in a Qualtrics survey via an `<iframe>`. Qualtrics pipes configuration values (API key, model, system prompt, turn limit) into the iframe URL as query parameters at render time. When the participant finishes the chat, the hosted page sends the full conversation back to Qualtrics via `postMessage`, where a JavaScript listener writes it to embedded data fields that appear in the survey export.

```
Qualtrics Survey
│
├── Survey Flow: Embedded Data
│   ├── Inputs:  llm_api_key, llm_model, llm_system_prompt, llm_max_turns
│   └── Outputs: chat_conversation_json, chat_model, chat_total_turns, ...
│
├── Question: iframe → GitHub Pages/index.html
│   │         ?key=...&model=...&system=...&turns=...
│   │
│   └── postMessage (llm_chat_finished)
│           ↓
└── Question JS tab: writes data to embedded fields → advances survey
```

---

## Files

| File | Purpose |
|---|---|
| `index.html` | The chat widget — host this on GitHub Pages |
| `qualtrics_iframe_embed.html` | 3-line iframe snippet — paste into Qualtrics question body |
| `qualtrics_iframe_javascript.js` | postMessage listener — paste into Qualtrics question JS tab |

---

## Setup

### 1. Host the chat widget on GitHub Pages

1. Create a new GitHub repository (e.g. `llm-chat`)
2. Add `index.html` to the root of the repo
3. Go to **Settings → Pages** and set source to `main` branch, root folder
4. Your widget URL will be:
   ```
   https://YOUR-USERNAME.github.io/YOUR-REPO/index.html
   ```
5. Test it works by visiting the URL with parameters directly in your browser:
   ```
   https://YOUR-USERNAME.github.io/YOUR-REPO/index.html?key=sk-xxx&model=gpt-4o&system=You are a helpful assistant.&turns=6
   ```

### 2. Configure Qualtrics Survey Flow

In your survey's **Survey Flow**, add an **Embedded Data** block *before* the chat question. Define all 9 fields:

**Input fields** (set values here):

| Field | Value |
|---|---|
| `llm_api_key` | Your API key (e.g. `sk-…`) |
| `llm_model` | Model name (e.g. `gpt-4o`) |
| `llm_system_prompt` | Your system prompt text |
| `llm_max_turns` | Max exchanges (e.g. `6`; leave blank for unlimited) |
| `condition` | Condition label if using randomisation (e.g. `treatment_A`) |

**Output fields** (leave values blank — the widget writes to these):

| Field |
|---|
| `chat_conversation_json` |
| `chat_model` |
| `chat_total_turns` |
| `chat_system_prompt` |
| `chat_timestamp` |

### 3. Add the chat question

1. Add a **Text/Graphic** question where you want the chat to appear
2. Click the `</>` **HTML source** button in the question body editor
3. Paste the contents of `qualtrics_iframe_embed.html`, replacing `YOUR-USERNAME` and `YOUR-REPO` with your GitHub details:
   ```html
   <iframe
     id="llm-chat-frame"
     src="https://YOUR-USERNAME.github.io/YOUR-REPO/index.html?key=${e://Field/llm_api_key}&model=${e://Field/llm_model}&system=${e://Field/llm_system_prompt}&turns=${e://Field/llm_max_turns}&condition=${e://Field/condition}"
     style="width:100%; height:640px; border:none; border-radius:12px; display:block;"
     allow="clipboard-write">
   </iframe>
   ```
4. Click the gear icon on the question → **Add JavaScript**
5. Paste the contents of `qualtrics_iframe_javascript.js`

---

## Supported models

| Model | Provider | Value for `llm_model` |
|---|---|---|
| GPT-4o | OpenAI | `gpt-4o` |
| GPT-4o mini | OpenAI | `gpt-4o-mini` |
| GPT-4 Turbo | OpenAI | `gpt-4-turbo` |
| Grok 3 | xAI | `grok-3` |
| Grok 3 Mini | xAI | `grok-3-mini` |
| Claude Sonnet 4 | Anthropic | `claude-sonnet-4-20250514` |
| Custom / RAG | Your backend | Set `llm_model` to `custom` and pass `&endpoint=https://your-api.com/v1/chat/completions` |

To add a new model, add an entry to the `ENDPOINTS` object in `index.html`:
```javascript
'your-model-name': 'https://api.provider.com/v1/chat/completions'
```
Any OpenAI-compatible endpoint works without further changes.

---

## Running experiments across model conditions

Use Qualtrics' **Randomiser** in the Survey Flow to split participants into conditions, then set `llm_model` (and optionally `condition`) per branch before the chat block. The iframe URL pipes the assigned model automatically — no code changes needed per study.

```
Survey Flow
├── Randomiser (evenly distribute)
│   ├── Branch A → Embedded Data: llm_model = gpt-4o, condition = openai
│   └── Branch B → Embedded Data: llm_model = grok-3, condition = grok
└── Chat question block (shared)
```

---

## Data recorded per participant

The `chat_conversation_json` embedded data field contains the full conversation as a JSON array:

```json
[
  { "turn": 1, "user": "What is X?",        "assistant": "X is…"  },
  { "turn": 2, "user": "Can you elaborate?", "assistant": "Sure…" }
]
```

Additional fields recorded: `chat_model`, `chat_total_turns`, `chat_system_prompt`, `chat_timestamp`, `condition`.

### Parsing in R

```r
library(tidyverse)
library(jsonlite)

df <- read_csv("qualtrics_export.csv") |>
  mutate(chat = map(chat_conversation_json, ~ fromJSON(.x))) |>
  unnest_wider(chat)
```

### Parsing in Python

```python
import pandas as pd
import json

df = pd.read_csv("qualtrics_export.csv")
df["chat"] = df["chat_conversation_json"].apply(json.loads)
turns = df["chat"].explode().apply(pd.Series)
```

---

## Using a custom or RAG model backend

Deploy a server that exposes an OpenAI-compatible endpoint:

```
POST https://your-backend.com/v1/chat/completions
Authorization: Bearer <your-internal-token>
Content-Type: application/json

{
  "model": "my-rag-model",
  "messages": [{"role": "user", "content": "..."}],
  "max_tokens": 1024
}
```

Then in your Survey Flow set:
- `llm_model` → any value not in the built-in list (e.g. `my-rag-model`)
- Add `&endpoint=https://your-backend.com/v1/chat/completions` to the iframe URL

A minimal FastAPI wrapper:

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import your_rag_pipeline

app = FastAPI()

@app.post("/v1/chat/completions")
async def chat(request: Request):
    body = await request.json()
    messages = body["messages"]
    user_text = messages[-1]["content"]
    history   = messages[:-1]
    answer = your_rag_pipeline.generate(user_text, history=history)
    return JSONResponse({
        "choices": [{"message": {"role": "assistant", "content": answer}}],
        "usage":   {"total_tokens": len(answer) // 4}
    })
```

---

## Security notes

- API keys are passed as URL parameters and are visible in the iframe `src` attribute in the page source. This is acceptable for a research PoC with a restricted-scope API key.
- For production deployments, use a backend proxy so keys never reach the browser.
- Create a rate-limited, model-scoped API key (OpenAI and xAI both support this) to limit exposure if a key is extracted.
- Set CORS on any custom backend to your Qualtrics domain only.
