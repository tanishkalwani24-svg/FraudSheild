# FraudShield — Google Cloud Deployment Guide

Deploy the full FraudShield stack to Google Cloud in ~15 minutes:

| Component | Service | Cost |
|-----------|---------|------|
| **Frontend** (HTML/JS/CSS) | Firebase Hosting | Free (10 GB/month) |
| **Backend** (Flask API) | Cloud Run | Free (2M requests/month) |
| **Secrets** | Secret Manager | Free (6 secrets) |
| **CI/CD** | Cloud Build | Free (120 min/day) |

---

## Prerequisites

Install these tools on your machine:

```bash
# 1. Google Cloud SDK
# Download from: https://cloud.google.com/sdk/docs/install
gcloud --version   # should print 400+

# 2. Firebase CLI
npm install -g firebase-tools
firebase --version  # should print 13+

# 3. Docker Desktop (for local testing only)
# Download from: https://www.docker.com/products/docker-desktop/
```

---

## Step 1 — Create a Google Cloud Project

```bash
# Log in
gcloud auth login

# Create a new project (pick a unique ID)
gcloud projects create fraudshield-app --name="FraudShield"

# Set it as your active project
gcloud config set project fraudshield-app

# Enable billing
# Go to: https://console.cloud.google.com/billing
# Link a billing account to "fraudshield-app" (free tier covers this app)
```

---

## Step 2 — Enable Required APIs

```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  secretmanager.googleapis.com \
  firebase.googleapis.com \
  firebasehosting.googleapis.com
```

---

## Step 3 — Store Secrets in Secret Manager

Never put secrets in environment variables or code. Use Secret Manager:

```bash
# Flask secret key (generate a random one)
echo -n "$(openssl rand -hex 32)" | \
  gcloud secrets create FLASK_SECRET_KEY --data-file=-

# Optional — only needed if you have these API keys
# IBM watsonx (skip if you don't have one — app works without it)
echo -n "YOUR_WATSONX_API_KEY" | \
  gcloud secrets create WATSONX_API_KEY --data-file=-

echo -n "YOUR_WATSONX_PROJECT_ID" | \
  gcloud secrets create WATSONX_PROJECT_ID --data-file=-

# Google Safe Browsing (skip to use pattern matching only)
echo -n "YOUR_SAFE_BROWSING_KEY" | \
  gcloud secrets create GOOGLE_SAFE_BROWSING_KEY --data-file=-
```

> **Skip** any secrets you don't have. The app has rule-based fallbacks for all of them.

---

## Step 4 — Deploy the Backend to Cloud Run

### Option A — One command (no Docker needed locally)

```bash
cd scampay/backend

gcloud run deploy fraudshield-backend \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 0 \
  --max-instances 10 \
  --timeout 120 \
  --set-env-vars FLASK_ENV=production \
  --update-secrets "FLASK_SECRET_KEY=FLASK_SECRET_KEY:latest"
```

Cloud Build will build the Docker image and deploy it automatically.  
After deployment, note the **Service URL** — it looks like:
```
https://fraudshield-backend-xxxxxxxx-uc.a.run.app
```

### Option B — Build & push manually

```bash
cd scampay/backend

# Build
docker build -t gcr.io/fraudshield-app/fraudshield-backend:latest .

# Push
docker push gcr.io/fraudshield-app/fraudshield-backend:latest

# Deploy
gcloud run deploy fraudshield-backend \
  --image gcr.io/fraudshield-app/fraudshield-backend:latest \
  --region us-central1 \
  --allow-unauthenticated \
  --memory 512Mi
```

---

## Step 5 — Update Frontend with Your Backend URL

Open `scampay/scan.html` and update the config line:

```html
<!-- Before (local dev) -->
<script>window.FRAUDSHIELD_BACKEND_URL = '';</script>

<!-- After (replace with your actual Cloud Run URL) -->
<script>window.FRAUDSHIELD_BACKEND_URL = 'https://fraudshield-backend-xxxxxxxx-uc.a.run.app';</script>
```

---

## Step 6 — Deploy Frontend to Firebase Hosting

```bash
# Log in to Firebase
firebase login

# Initialize Firebase (only needed once)
cd scampay
firebase init hosting

# When prompted:
#   - Use existing project → select "fraudshield-app"
#   - Public directory → . (current directory)
#   - Single-page app → No
#   - Overwrite index.html → No

# Deploy
firebase deploy --only hosting
```

Your site will be live at:
```
https://fraudshield-app.web.app
https://fraudshield-app.firebaseapp.com
```

---

## Step 7 (Optional) — Set Up CI/CD with Cloud Build

Auto-deploy on every `git push` to `main`:

```bash
# Store Firebase token as a secret
firebase login:ci   # prints a token
echo -n "THE_TOKEN_FROM_ABOVE" | \
  gcloud secrets create FIREBASE_TOKEN --data-file=-

# Connect your GitHub repo to Cloud Build
# Go to: https://console.cloud.google.com/cloud-build/triggers
# → Create Trigger → Connect GitHub repo → use cloudbuild.yaml
```

From now on, every push to `main` will:
1. Build a new Docker image
2. Deploy to Cloud Run
3. Deploy frontend to Firebase Hosting

---

## Step 8 (Optional) — Custom Domain

```bash
# Firebase Hosting custom domain
# Go to: https://console.firebase.google.com
# → Hosting → Add custom domain → fraudshield.in (or your domain)
# Firebase gives you 2 TXT records to add to your DNS provider

# Cloud Run also supports custom domains
gcloud run domain-mappings create \
  --service fraudshield-backend \
  --domain api.fraudshield.in \
  --region us-central1
```

---

## Verify Deployment

```bash
# Check backend health
curl https://fraudshield-backend-xxxxxxxx-uc.a.run.app/api/health
# Expected: {"service": "FraudShield Backend", "status": "ok"}

# Check frontend
curl -I https://fraudshield-app.web.app
# Expected: HTTP/2 200
```

---

## Cost Estimate (Free Tier)

| Resource | Free Tier | Typical Usage |
|----------|-----------|---------------|
| Cloud Run | 2M requests/month, 360K CPU-sec | ~50K scans/month |
| Firebase Hosting | 10 GB storage, 360 MB/day transfer | Well within free |
| Secret Manager | 6 secret versions free | 4 secrets used |
| Cloud Build | 120 min/day | ~5 min/deploy |

**Expected monthly cost: $0** for typical usage.

---

## Troubleshooting

### Backend returns 500 on `/api/scan`
```bash
gcloud run logs read --service fraudshield-backend --region us-central1 --limit 50
```

### Frontend can't reach backend (CORS error)
- Make sure `window.FRAUDSHIELD_BACKEND_URL` in `scan.html` matches exactly your Cloud Run URL (no trailing slash)
- The Cloud Run service must have `--allow-unauthenticated`

### `pyzbar` fails in container
```bash
# libzbar0 is already in the Dockerfile — if still failing, check logs:
gcloud run logs read --service fraudshield-backend --region us-central1
```

### Firebase deploy says "project not found"
```bash
firebase use fraudshield-app   # explicitly set the project
firebase deploy --only hosting
```
