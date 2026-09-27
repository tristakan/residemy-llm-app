# Residemy 韌學堂: an AI Learning Platform with RAG, Video Understanding & Personal Agents

**Residemy (韌學堂)** is a full-stack AI learning platform built at the **NTHU Institute of Service Science** (Nov 2023 – Oct 2024). Students sign in with **LINE**, browse course materials from NTHU faculty, upload their own documents and lecture videos, and learn with **GPT-4o-powered assistants** that answer questions **grounded in the course materials** (retrieval-augmented generation).

| Repository | Role |
|---|---|
| **This repo:** `residemy-llm-app` | Web front end: Next.js · React · Material UI |
| [`rxthlessbeats/residemy-backend`](https://github.com/rxthlessbeats/residemy-backend) | REST API and AI pipeline: Django · LanceDB · OpenAI (GPT-4o, Whisper, embeddings) |

![Residemy screenshot](screenshot-dashboard.png)
<!-- Add 2–3 screenshots: login page with faculty, dashboard, document library / chat. -->

---

## What it does

### 1. Ask questions about course materials (RAG)
- Documents are split into chunks and embedded with OpenAI **`text-embedding-3-small`**. Embeddings are computed **in parallel** (`ThreadPoolExecutor`) to speed up ingestion.
- Vectors are stored in **LanceDB**, with a **shared knowledge base** for official course content and a **separate vector store per user** for personal uploads.
- At question time, the query is embedded, the **top-k most similar chunks** are retrieved, and they're passed to **GPT-4o** as context. Answers therefore cite the actual course materials instead of relying only on the model's memory.
- Two collections are supported: a **knowledge-object table** and a **research-paper table**.

### 2. Understand lecture videos
Uploading a lecture video triggers a multimodal pipeline:
1. **Audio extraction** with MoviePy / FFmpeg
2. **Speech-to-text** with OpenAI **Whisper**. Audio is split into ~0.5 MB chunks to stay within API limits, then the transcripts are stitched back together.
3. **Frame sampling** with OpenCV at key timestamps
4. **GPT-4o vision** reads the frames together with the transcript and writes a **Markdown summary**. If a frame is a slide, it also extracts the slide text.

### 3. Personal AI agents with memory
- **Personas:** users create and switch between custom assistant personas, stored per user.
- **Conversation memory:** chat sessions are saved and the last *N* conversations can be retrieved, so the assistant keeps context across visits.
- **Learning activities:** a per-user to-do list the agent can create, update, and mark finished.

### 4. Other AI utilities
- **Auto-generated document descriptions**, with automatic **language detection** (answers in the document's language) and **token-aware trimming** with `tiktoken` so long inputs fit each model's context window
- **Image Q&A** with GPT-4o vision
- **JSON repair**: GPT-4o returns well-formed JSON for structured outputs

### 5. Platform
- **LINE Login (OAuth 2.0 / OpenID Connect):** the front end exchanges the auth code, verifies the ID token, and registers the user with the backend
- **Personal dashboard** (韌學堂總覽) and **document library** (文件庫)
- **Faculty showcase:** 10 NTHU professors whose materials are on the platform
- **Multi-tenant data:** each user gets their **own database and vector store**, isolating personal documents and chats
- **Admin and content management** with Django CMS, with sharing / audit flags and expiry dates on documents

---

## Architecture

```
                       ┌───────────────┐
                       │  LINE Login   │  OAuth 2.0 / OIDC
                       └──────┬────────┘
                              │
┌─────────────────────────────▼──────────────────────────────┐
│  Front end: Next.js + React + MUI           (this repo)    │
│  login · dashboard · document library · assistant          │
└─────────────────────────────┬──────────────────────────────┘
                              │ REST (JSON)
┌─────────────────────────────▼──────────────────────────────┐
│  Backend: Django + Django REST Framework                   │
│                                                            │
│  Users & documents ── per-user DB ── Django CMS admin      │
│                                                            │
│  RAG           : embed (text-embedding-3-small)            │
│                  → LanceDB top-k search → GPT-4o answer    │
│  Video         : MoviePy audio → Whisper transcript        │
│                  + OpenCV frames → GPT-4o summary          │
│  Agents        : personas · conversation memory · tasks    │
└────────────────────────────────────────────────────────────┘
```

### Main API endpoints (backend)

| Endpoint | Purpose |
|---|---|
| `POST /api/store_knowledge_in_db/` · `/api/store_research_in_db/` | Chunk, embed and index documents into LanceDB |
| `POST /api/simple_rag/` · `/api/generate_response_rag/` | Retrieve relevant chunks and answer with GPT-4o |
| `POST /api/extract_text_from_video/` | Whisper transcription of a lecture video |
| `POST /api/process_meta_item/` | Frames + transcript → GPT-4o video summary |
| `POST /api/generate_description/` | Language-aware, token-trimmed document descriptions |
| `POST /api/ask_question_about_image/` | GPT-4o vision Q&A |
| `/api/save-conversation/` · `/api/retrieve-conversations/…` | Conversation memory |
| `/api/insert-persona/` · `/api/get-personas/…` | Custom assistant personas |
| `/api/insert-activity/` · `/api/get-activities/…` | Learning activity list |

---

## Tech stack

| Layer | Tools |
|---|---|
| **Front end** | Next.js 14 · React 18 · Material UI · Tailwind CSS · Redux Toolkit · Formik + Yup · ApexCharts |
| **Backend** | Python · Django 4.2 · Django REST Framework · Django CMS · django-allauth · SQLite (per-user) |
| **AI / LLM** | OpenAI GPT-4o · Whisper · `text-embedding-3-small` · LanceDB (vector DB) · tiktoken · langid |
| **Media** | MoviePy · OpenCV · pydub · PyMuPDF |
| **Auth** | LINE Login (OAuth 2.0 / OIDC) |

---

## Run locally

**Front end (this repo):**
```bash
git clone https://github.com/ganbeik/residemy-llm-app.git
cd residemy-llm-app
npm install
```
Create `.env.local` (never commit it):
```bash
NEXT_PUBLIC_LINE_CLIENT_ID=your_line_channel_id
LINE_CHANNEL_SECRET=your_line_channel_secret
NEXT_PUBLIC_BASE_URL=http://localhost:5003
NEXT_PUBLIC_API_URL=http://localhost:5005
```
```bash
npm run dev          # http://localhost:5003
```

**Backend:** see [`residemy-backend`](https://github.com/rxthlessbeats/residemy-backend). It needs `OPENAI_API_KEY` and a Django `SECRET_KEY` in a `.env` file, and runs on port 5005.

---

## Team and credits

Built by a team at the NTHU Institute of Service Science: [@rxthlessbeats](https://github.com/rxthlessbeats), [your name], and [other members].

**My contributions:** [be specific, e.g. "designed the RAG pipeline and LanceDB schemas", "built the video-summary pipeline (Whisper + GPT-4o)", "prompt engineering for description generation", "front-end pages X and Y", "testing and evaluation"]

The front-end UI is based on the open-source [Devias Material Kit](https://github.com/devias-io/material-kit-react) (MIT License).

## What I'd improve next

- **Evaluate retrieval quality** (e.g. hit rate / answer accuracy on a set of course questions) and try hybrid keyword + vector search
- **Stream** GPT-4o responses to the UI for faster perceived speed
- Replace remaining template sample data in the front end with live API data
- Move the hard-coded backend URL (`127.0.0.1:5005` in `line-callback.js`) into an environment variable
