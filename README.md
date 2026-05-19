# SnapStudy — Backend Setup Guide

## What's built

| Layer | Tech | Cost |
|---|---|---|
| **OCR (handwriting → text)** | OpenRouter free tier — Llama 4 Scout vision | **Free** — 200 req/day, no card |
| **Text AI (quiz, cleanup)** | Groq free tier — Llama 3.3 70B | **Free** — 1,000 req/day, no card |
| **Backend** | Python FastAPI | Free |
| **Database + Auth + Storage** | Supabase free tier | Free |
| **Hosting** | Render free tier | Free |

Both AI providers run fully open-source models (Meta Llama). No credit card required for either.

---

## Step 1 — Get API keys (10 min, all free, no card)

### OpenRouter (vision / OCR)
1. Go to https://openrouter.ai → sign up (email only)
2. **Keys** → **Create Key**
3. Copy it — looks like `sk-or-v1-...`
4. Free tier gives **200 requests/day** with no payment

### Groq (text generation — quiz + cleanup)
1. Go to https://console.groq.com → sign up (email or GitHub)
2. **API Keys** → **Create API Key**
3. Copy it — looks like `gsk_...`
4. Free tier gives **1,000 requests/day, 6,000 tokens/min** with no payment

### Supabase
1. Go to https://supabase.com → **New project** → pick a name and password
2. Wait ~2 min for it to provision
3. Go to **Settings → API**:
   - Copy **Project URL** → `SUPABASE_URL`
   - Copy **service_role** secret key → `SUPABASE_SERVICE_KEY` (NOT the anon key)
   - Copy **anon public** key → needed for the frontend `index.html`

---

## Step 2 — Set up Supabase database (5 min)

1. In your Supabase project → **SQL Editor** → **New query**
2. Paste the entire contents of `backend/sql/schema.sql`
3. Click **Run**

### Create the storage bucket
1. In Supabase → **Storage** → **New bucket**
2. Name: `note-images`
3. **Public bucket**: OFF (keep private)
4. Click **Create**
5. Go to **Storage → Policies** and add these policies for the `note-images` bucket:

   **INSERT policy** — "Users upload own images"
   ```sql
   (bucket_id = 'note-images' AND auth.uid()::text = (storage.foldername(name))[1])
   ```

   **SELECT policy** — "Users view own images"
   ```sql
   (bucket_id = 'note-images' AND auth.uid()::text = (storage.foldername(name))[1])
   ```

   **DELETE policy** — "Users delete own images"
   ```sql
   (bucket_id = 'note-images' AND auth.uid()::text = (storage.foldername(name))[1])
   ```

### Enable Google OAuth (for "Continue with Google")
1. Supabase → **Authentication → Providers → Google**
2. You'll need a Google OAuth client ID + secret from https://console.cloud.google.com
3. Enable and save

---

## Step 3 — Run locally (2 min)

```bash
cd backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp ../.env.example .env
# Edit .env with your keys

uvicorn main:app --reload
```

Visit http://localhost:8000/docs — you'll see the full Swagger UI with every endpoint.

---

## Step 4 — Deploy to Render (10 min)

### Push to GitHub first
```bash
cd /path/to/SnapStudy
git init
git add .
git commit -m "initial commit"
# Create a repo on github.com, then:
git remote add origin https://github.com/YOUR_USERNAME/snapstudy.git
git push -u origin main
```

### Create Render service
1. Go to https://render.com → **New → Web Service**
2. Connect your GitHub repo
3. Render will detect `render.yaml` automatically

### Set environment variables in Render
In your Render service → **Environment**:

| Key | Value |
|---|---|
| `OPENROUTER_API_KEY` | `sk-or-v1-...` |
| `GROQ_API_KEY` | `gsk_...` |
| `SUPABASE_URL` | `https://xxx.supabase.co` |
| `SUPABASE_SERVICE_KEY` | `eyJ...` (service_role key) |

4. Click **Save** — Render auto-redeploys.

Your API will be live at `https://snapstudy-api.onrender.com` (or similar).

---

## Step 5 — Connect the frontend

Open `frontend/index.html` and update the three config lines at the top:

```html
<script>
  window.__SNAPSTUDY_SUPABASE_URL  = 'https://YOUR_PROJECT.supabase.co';
  window.__SNAPSTUDY_ANON_KEY      = 'YOUR_SUPABASE_ANON_KEY';   // anon key, NOT service_role
  window.__SNAPSTUDY_API_BASE      = 'https://snapstudy-api.onrender.com';
</script>
```

Then in your screen files, replace mock `go()` calls that hard-code data with real API calls:

```jsx
// Example: replace mock scan with real upload in ScrScan
const fire = async () => {
  setShutter(true);
  const file = /* get from camera/file input */;
  try {
    const note = await api.notes.upload([file]);
    go('notes', { noteId: note.id });
  } catch (e) {
    alert(e.message);
  }
};

// Example: load real notes on Home screen
React.useEffect(() => {
  api.notes.list().then(setNotes);
}, []);
```

---

## API reference

The full interactive docs are at `your-render-url/docs`.

| Method | Path | Description | Credits |
|---|---|---|---|
| `POST` | `/api/notes/upload` | Upload 1-5 images, run OCR+AI | -5 |
| `GET` | `/api/notes` | List all your notes | — |
| `GET` | `/api/notes/:id` | Get note + signed image URLs | — |
| `PUT` | `/api/notes/:id` | Edit title/content | — |
| `DELETE` | `/api/notes/:id` | Delete note + images | — |
| `POST` | `/api/quiz/generate` | Generate quiz from note | -5 |
| `GET` | `/api/quiz` | List quizzes | — |
| `GET` | `/api/quiz/:id` | Get quiz + questions | — |
| `POST` | `/api/quiz/:id/submit` | Record score | — |
| `GET` | `/api/library` | Subjects with note counts | — |
| `GET` | `/api/library/:subject` | Notes in one subject | — |
| `GET` | `/api/profile` | Credits, streak, stats | — |

---

## Credits system

- 10 credits per user per day (resets at midnight UTC)
- Scanning costs 5 credits (`POST /api/notes/upload`)
- Quiz generation costs 5 credits (`POST /api/quiz/generate`)
- Override defaults via `DAILY_CREDITS`, `SCAN_COST`, `QUIZ_COST` env vars

---

## AI providers — why this combination?

| Task | Provider | Model | Cost |
|---|---|---|---|
| Handwriting OCR | OpenRouter | Llama 4 Scout (vision) | Free |
| Quiz generation | Groq | Llama 3.3 70B | Free |
| Text cleanup + subject detection | Groq | Llama 3.3 70B | Free |

**Why not self-hosted OCR (Tesseract / PaddleOCR)?**
Tesseract is very poor on handwriting. PaddleOCR needs ~500 MB model download and a GPU for decent speed. Render's free tier has 512 MB RAM — too small.

**Why split two providers?**
Groq's free tier has no vision capability, so we use OpenRouter (which does) just for the image-reading step. All text tasks (which are the majority) go through Groq which has a higher request limit.

**If OpenRouter's free vision model changes**, swap the `VISION_MODEL` constant in `backend/services/ai_service.py` to any other free model on openrouter.ai (e.g. `qwen/qwen2.5-vl-72b-instruct:free`).

---

## Folder structure

```
SnapStudy/
├── backend/
│   ├── main.py              FastAPI app
│   ├── config.py            Env var settings
│   ├── database.py          Supabase client
│   ├── dependencies.py      Auth middleware
│   ├── requirements.txt
│   ├── routers/
│   │   ├── notes.py         Upload, CRUD
│   │   ├── quiz.py          Generate, submit
│   │   ├── library.py       Grouped by subject
│   │   └── profile.py       Stats + credits
│   ├── services/
│   │   ├── ai_service.py    All Claude API calls
│   │   └── storage_service.py  Supabase Storage
│   └── sql/
│       └── schema.sql       Run in Supabase SQL Editor
├── frontend/
│   ├── index.html           Entry point (fill in your credentials)
│   ├── api.js               API + auth client
│   ├── lib.jsx              Design tokens + shared components
│   ├── ios-frame.jsx        iOS device frame
│   ├── android-frame.jsx    Android device frame
│   ├── design-canvas.jsx    Figma-like canvas
│   ├── tweaks-panel.jsx     Live tweak controls
│   ├── screens-a.jsx        Onboarding, Home, Scan, Processing, Notes
│   ├── screens-b.jsx        Quiz, Results, Library, Paywall, Profile
│   └── app.jsx              App shell + routing
├── render.yaml              Render deployment config
├── .env.example             Copy to backend/.env
└── .gitignore
```
