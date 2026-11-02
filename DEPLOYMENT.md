# Production deployment on Vercel

This repository is designed to deploy directly from GitHub to Vercel. The deployed service is a Django ASGI application on Vercel Fluid Compute, exposed through the conventional `api/index.py` Python Function and a catch-all rewrite. Phone cameras communicate using secure HTTPS frame uploads, rather than depending on a process-bound WebSocket connection. That is intentional: each Vercel Function can scale, pause, and eventually recycle independently.

## Required production services

Vercel supplies the HTTPS application endpoint. Before the first deployment, provision these two durable services in a region close to the Vercel Function region:

1. Managed PostgreSQL, with a production `DATABASE_URL`.
2. Managed Redis, with a production TLS-enabled `REDIS_URL` when offered by the provider.

PostgreSQL holds users, lectures, reports, and analytics. Redis holds only short-lived camera frames, thumbnails, and anonymous track state. SQLite and process memory are deliberately rejected when `DJANGO_DEBUG=False` because Vercel Functions do not provide durable local disk or reliable shared memory.

## GitHub to Vercel release

1. Create a new GitHub repository and push this directory's contents. Do not commit `.env` files or any database.
2. In Vercel, select **Add New → Project**, import the GitHub repository, and keep the detected Django/Python settings. There is no custom build command to enter; the committed `pyproject.toml` supplies it.
3. Create or connect PostgreSQL and Redis, then set every value in [`.env.production.example`](.env.production.example) under **Project Settings → Environment Variables**. Apply them to **Production** and create separate database/Redis branches or instances for **Preview** deployments.
4. Set `DJANGO_PUBLIC_URL` to the exact HTTPS deployment/custom-domain origin, with no trailing slash.
5. Choose a strong unique `DJANGO_ADMIN_PASSWORD`. `admin1234` is rejected whenever `DJANGO_DEBUG=False`.
6. Deploy. Vercel installs `requirements.txt`, runs migrations, collects static assets, and runs the idempotent `bootstrap_admin` command. The command creates the administrator only when absent and never overwrites an existing password.
7. Verify `https://<your-domain>/healthz/`, sign in, start a lecture, pair the phone using the generated QR code, and permit browser camera access.

Each push to the production branch triggers a new production deployment through the Vercel Git integration. Pull requests receive Preview deployments only if they use an isolated Preview database/Redis configuration. Never point Preview and Production at the same mutable database.

## Command-line deployment

If Git integration is not wanted, install Node.js and use the provided scripts after setting Vercel environment variables:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\scripts\vercel-preview.ps1
.\scripts\vercel-production.ps1
```

The first command requests Vercel login/project linking through the official CLI. It produces an HTTPS Preview URL, which is the correct place to test phone camera access. The second promotes a linked project to production.

## Production behavior

- Vercel automatically provides HTTPS, so `navigator.mediaDevices.getUserMedia` is available on Android and iPhone browsers.
- The camera page uses token-scoped HTTPS uploads on Vercel, so it works on both `*.vercel.app` and a custom domain.
- Django rejects production startup without a non-default secret key, PostgreSQL, and Redis.
- HTTPS redirect, secure cookies, HSTS, and HSTS preload are enabled by default in production.
- The repository pins Python 3.12 and MediaPipe 0.10.21, which retains the FaceMesh API used by the analyzer.
