# AI Classroom Attention Monitoring

ClassFocus is a production-oriented Django application for live classroom attention monitoring. A teacher starts a lecture, pairs a phone through a QR code, and receives anonymous face-geometry attention signals and reports. The system does **not** identify people: it has no face enrollment, recognition, name lookup, embedding store, or cross-session tracking.

## Deployment target: Vercel

This repository is built to deploy from GitHub to **Vercel**. The Vercel deployment—not an HTTP LAN server—is the supported phone-camera environment:

- Vercel gives the phone an HTTPS origin, which is required by modern mobile browsers for `getUserMedia` camera access.
- Django runs as one Python ASGI application on Vercel Fluid Compute.
- Phone frames use token-scoped HTTPS uploads on Vercel. This avoids coupling live camera operation to a long-lived, process-bound WebSocket connection.
- PostgreSQL provides durable application data and Redis provides shared, short-lived live-camera state.
- Production fails safely if it is missing PostgreSQL, Redis, or a real Django secret.

The LAN HTTP warning in the prior preview was expected browser security behavior. Mobile Chrome and Safari do not expose camera access to `http://192.168.x.x` pages. Test the camera through the Vercel Preview or Production HTTPS URL instead.

## Deploy from GitHub

1. Create a GitHub repository and push this directory. Do not add `.env`, a SQLite database, or service credentials.
2. In [Vercel](https://vercel.com/new), import the repository. Vercel detects this Django project through `manage.py`, `requirements.txt`, and `app.py`.
3. Provision managed PostgreSQL and managed Redis. Add the resulting connection strings and every other required variable from [`.env.production.example`](.env.production.example) in **Project Settings → Environment Variables**.
4. Set `DJANGO_PUBLIC_URL` to the exact Vercel/custom-domain HTTPS URL, without a trailing slash.
5. Set a strong `DJANGO_ADMIN_USERNAME`, `DJANGO_ADMIN_PASSWORD`, and `DJANGO_ADMIN_EMAIL`. The production build deliberately rejects `admin1234`.
6. Deploy. The committed Vercel build configuration installs dependencies, migrates the database, collects static files, and bootstraps the administrator if it does not already exist.
7. Open `https://<your-domain>/healthz/`, sign in, create and start a lecture, then pair a phone through the QR code.

Every push to the configured production branch redeploys automatically. Pull requests should use separate Preview PostgreSQL/Redis services (or provider-supported branches): never share mutable production data with Preview builds.

Complete step-by-step instructions and the deployment design are in [DEPLOYMENT.md](DEPLOYMENT.md).

## One-command Vercel CLI releases

Git integration is preferred. If you use the Vercel CLI instead, Node.js is the only local prerequisite; the scripts invoke the official CLI through `npx` and do not require Python or a local app server.

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\scripts\vercel-preview.ps1
.\scripts\vercel-production.ps1
```

The first command signs in/links the Vercel project interactively and produces an HTTPS Preview URL. Use that URL for real phone-camera validation. Before the production command, configure all Production environment variables in Vercel.

## Required Vercel environment variables

Use [`.env.production.example`](.env.production.example) as the source of truth. These values must be stored in Vercel, not committed to Git:

```env
DJANGO_DEBUG=False
DJANGO_SECRET_KEY=<long-random-secret>
DJANGO_ALLOWED_HOSTS=<your-project>.vercel.app,your-custom-domain.example
DJANGO_PUBLIC_URL=https://your-custom-domain.example
DJANGO_CSRF_TRUSTED_ORIGINS=https://your-custom-domain.example
DJANGO_ADMIN_USERNAME=<unique-admin-name>
DJANGO_ADMIN_PASSWORD=<strong-unique-password>
DJANGO_ADMIN_EMAIL=<administrator-email>
DATABASE_URL=<managed-postgresql-url>
REDIS_URL=<managed-redis-url>
```

Vercel automatically supplies `VERCEL` / `VERCEL_ENV`; do not set `DJANGO_DEBUG=True` in Vercel. HTTPS redirects, secure cookies, HSTS, and preload are enabled in production.

## Runtime and operational details

- Python is pinned to 3.12 for Vercel. MediaPipe `0.10.21` is pinned because it retains the FaceMesh API used by the application.
- Frame uploads are bounded in size and analyzed only after a teacher has started a lecture.
- PostgreSQL never contains source frames or face crops. Those are cached in Redis under brief TTLs.
- The administrator bootstrap is idempotent: it creates the configured administrator only on its first run and never resets an existing password.
- Vercel bundles no local virtual environments, SQLite data, media directory, or environment files. `requirements.txt` is deliberately included so the deployment installs the exact runtime dependencies.

## Verification

The repository includes unit tests and Django deployment checks for the production settings. The intended release verification is:

1. Push to GitHub and inspect the Vercel build log.
2. Confirm `/healthz/` returns `{"ok": true}` on the generated HTTPS URL.
3. Sign in and pair a mobile phone from the Vercel URL.
4. Confirm the phone page displays **HTTP upload active** and begins increasing **Frames sent** after camera permission is granted.
