# 🚀 Deploy HR Assistant API on Render — Full Step-by-Step Guide

---

## What You're Deploying

A **FastAPI** backend that exposes your RAG chatbot as a REST API.
Your friend's frontend will call `POST /ask` with a question and get back an answer.

**Endpoints:**
| Method | URL | What it does |
|--------|-----|--------------|
| GET | `/` | Health check |
| GET | `/health` | Health check (for Render) |
| POST | `/ask` | Send a question, get an answer |

---

## 📁 Files in This Package

```
backend/
├── main.py                  ← FastAPI app
├── rag_engine.py            ← RAG logic (your notebook code, cleaned up)
├── requirements.txt         ← Python dependencies
├── render.yaml              ← Render config
├── .gitignore
└── merged_embeddings.json   ← ⚠️  YOU must add this (see Step 1)
```

---

## STEP 1 — Add Your Embeddings File

Your `merged_embeddings.json` file is **not included** (it was on your local machine).

Copy it into the `backend/` folder:
```bash
cp /path/to/your/merged_embeddings.json backend/
```

> ⚠️ **Important:** This file can be large. If it is over ~50 MB, see
> the "Large File" section at the bottom of this guide.

---

## STEP 2 — Create a GitHub Repository

1. Go to **https://github.com/new**
2. Create a new **private** repo, e.g. `hr-assistant-api`
3. Do NOT initialize with README (you'll push from your machine)

Then in your terminal, inside the `backend/` folder:

```bash
git init
git add .
git commit -m "Initial commit — HR Assistant API"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/hr-assistant-api.git
git push -u origin main
```

---

## STEP 3 — Create a Render Account

1. Go to **https://render.com** and sign up (free)
2. Click **"New +"** → **"Web Service"**
3. Connect your GitHub account when prompted
4. Select your `hr-assistant-api` repository

---

## STEP 4 — Configure the Web Service

Fill in these fields on Render:

| Field | Value |
|-------|-------|
| **Name** | `hr-assistant-api` (or anything you like) |
| **Region** | Frankfurt or closest to you |
| **Branch** | `main` |
| **Runtime** | `Python 3` |
| **Build Command** | `pip install -r requirements.txt` |
| **Start Command** | `uvicorn main:app --host 0.0.0.0 --port $PORT` |
| **Instance Type** | **Standard** (Free tier will be too slow for sentence-transformers) |

---

## STEP 5 — Set Environment Variables

In the Render dashboard → your service → **"Environment"** tab,
add these variables one by one:

| Key | Value |
|-----|-------|
| `GROQ_KEY_1` | `gsk_a78NXmE9XITaCcyF73T...` (your account 1 key) |
| `GROQ_KEY_2` | `gsk_gFfu5CRSzBUSinJrYx3t...` |
| `GROQ_KEY_3` | `gsk_JgQfgl9jbyQrFCZSQED0...` |
| `GROQ_KEY_4` | `gsk_gjbZ282IKtpmhHcy4Sqz...` |
| `GROQ_KEY_5` | `gsk_hDafuV0rcBZ0zBXkEQcy...` |
| `GROQ_KEY_6` | `gsk_lm798JhW7CJ1IslRdweL...` |
| `EMBEDDINGS_FILE` | `merged_embeddings.json` |
| `EMBED_MODEL` | `intfloat/multilingual-e5-large` |
| `LLM_MODEL` | `llama-3.3-70b-versatile` |
| `TOP_K` | `3` |
| `CANDIDATES` | `20` |

> 🔒 **Never** put API keys directly in code or commit them to GitHub.
> Render stores them encrypted and injects them at runtime.

---

## STEP 6 — Deploy!

Click **"Create Web Service"**. Render will:

1. Clone your GitHub repo
2. Run `pip install -r requirements.txt`
3. Download the `intfloat/multilingual-e5-large` model (~1.1 GB) on first boot
4. Start the FastAPI server

**First boot takes ~5-10 minutes** because of the model download.
Subsequent restarts are faster (model is cached).

Watch the logs in the Render dashboard. When you see:
```
✅ Loaded N chunks
✅ Model ready
INFO:     Application startup complete.
```
...your API is live! 🎉

---

## STEP 7 — Test Your API

Your API URL will be something like:
```
https://hr-assistant-api.onrender.com
```

**Test in browser:**
```
https://hr-assistant-api.onrender.com/health
```

**Test with curl:**
```bash
curl -X POST https://hr-assistant-api.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "ما هي إجراءات التصريح بحادث العمل؟"}'
```

**Expected response:**
```json
{
  "question": "ما هي إجراءات التصريح بحادث العمل؟",
  "answer": "..."
}
```

**Interactive docs (Swagger UI):**
```
https://hr-assistant-api.onrender.com/docs
```

---

## STEP 8 — Share the URL with Your Friend

Your friend's frontend just needs to call:

```
POST https://hr-assistant-api.onrender.com/ask
Content-Type: application/json

{ "question": "their question here" }
```

CORS is already enabled for all origins (`*`), so any frontend
(React, Vue, plain HTML, etc.) can call it directly from the browser.

**Example frontend fetch:**
```javascript
const response = await fetch("https://hr-assistant-api.onrender.com/ask", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ question: userInput })
});
const data = await response.json();
console.log(data.answer);
```

---

## ⚠️ Large `merged_embeddings.json` File?

If your file is larger than **50 MB**, GitHub will reject it.

**Option A — Git LFS (recommended):**
```bash
brew install git-lfs          # or: apt install git-lfs
git lfs install
git lfs track "*.json"
git add .gitattributes
git add merged_embeddings.json
git commit -m "Add embeddings via LFS"
git push
```

**Option B — Store on Hugging Face Hub:**
1. Upload the file to a Hugging Face dataset repo
2. In `rag_engine.py`, add a download step in `load_data()`:
```python
from huggingface_hub import hf_hub_download
path = hf_hub_download(repo_id="your-user/your-dataset",
                       filename="merged_embeddings.json",
                       repo_type="dataset")
```

---

## 🛠️ Troubleshooting

| Problem | Fix |
|---------|-----|
| Build fails | Check `requirements.txt` versions match Python 3.11 |
| `GROQ_KEY` error on startup | Make sure all env vars are set in Render dashboard |
| `merged_embeddings.json not found` | Make sure the file is committed to GitHub |
| Timeout on first request | Model is still loading — wait 2 min and retry |
| Free tier sleeps after 15 min | Upgrade to Starter ($7/mo) or ping `/health` periodically |

---

## 📦 Project Structure Summary

```
main.py          — FastAPI: defines /ask, /health endpoints
rag_engine.py    — RAG logic: search, rerank, LLM call
requirements.txt — pip dependencies
render.yaml      — Render deployment config
.gitignore       — excludes .env, __pycache__, etc.
merged_embeddings.json  — your vector data (add manually)
```
