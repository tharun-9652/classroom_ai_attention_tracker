# ATTEN — AI Classroom Attention Monitoring

ATTEN is a Django web application for running a classroom observation session from a teacher dashboard. A phone can be paired as the room camera, anonymous face tracks are shown live, and each session produces a reviewable report and CSV export.

> **Live deployment:** [ai-classroom-attention-monitor.vercel.app](https://ai-classroom-attention-monitor.vercel.app/)

## What it does

- Creates teacher-owned monitoring sessions with a clear draft → live → ended lifecycle.
- Pairs an Android or iPhone browser camera through a QR code or secure link.
- Detects real faces in submitted camera frames and displays temporary, anonymous session tracks such as `Track-001`.
- Shows a live room preview, track count, alignment signal, sustained off-axis time, and event count.
- Stores session reports and lets teachers export a CSV summary.
- Provides a protected Django administration area for the administrator only.

The app does **not** perform facial recognition. It has no face gallery, name lookup, biometric embedding store, or cross-session identity matching.

## How the live signal works

The production deployment uses OpenCV's **headless** frontal-face detector, which works in Vercel's serverless Linux environment. For every active session it:

1. detects faces in the current frame;
2. assigns short-lived, anonymous tracks based only on nearby positions in consecutive frames;
3. estimates an observation state from where the detected face is positioned in the frame; and
4. expires preview frames and thumbnails from cache after a short time.

This is a classroom observation aid, not a measure of learning, engagement, intent, or student performance. Do not use its output as the only basis for disciplinary, grading, accessibility, or other high-impact decisions.

## Stack

| Area | Technology |
| --- | --- |
| Application | Django 5, templates, Django REST Framework |
| Live delivery | HTTP polling on Vercel; Django Channels WebSockets for compatible non-Vercel hosts |
| Camera input | Browser `getUserMedia`, QR pairing, HTTPS frame uploads |
| Vision | OpenCV headless frontal-face detection |
| Data | PostgreSQL or SQLite for local development |
| Short-lived live data | Redis recommended; local in-memory cache as a fallback |
| Deployment | Vercel Python runtime, Python 3.12 |

## Quick start

### Prerequisites

- Python **3.12**
- A modern browser
- Optional: PostgreSQL and Redis for a production-like local setup

### Run locally

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install .

python manage.py migrate
python manage.py create_demo_teacher
python manage.py runserver 0.0.0.0:8000
```

Open [http://localhost:8000](http://localhost:8000) and sign in.

The provisioning command creates or updates these **development defaults** when no environment variables are set:

| Account | Username | Password | Access |
| --- | --- | --- | --- |
| Administrator | `admin` | `admin1234` | Dashboard and `/admin/` |
| Teacher | `teacher` | `teacher12345` | Teacher dashboard and that teacher's sessions only |

Change both passwords before using the application beyond local testing.

## Run a monitoring session

1. Sign in as a teacher or administrator.
2. Create a session from **Sessions**.
3. Open the session and select **Begin live run**.
4. Select **Pair device** and scan the QR code from the room phone, or open the generated camera link on that phone.
5. Allow camera access and press **Start camera**.
6. Keep the phone steady with a reasonably frontal view of the room.
7. Review the live dashboard, then select **End run** when finished.
8. Open **View report** to inspect session results or download the CSV export.

For a phone camera, use HTTPS whenever possible. The hosted Vercel URL is already HTTPS. A phone cannot use `localhost` on the teacher's computer; it treats `localhost` as the phone itself.

## Access control

Every ordinary app user can manage only the sessions they own. Django's system administration area is server-side protected, not just hidden in the interface.

| Role | Can do |
| --- | --- |
| Teacher | Create, start, stop, pair, calibrate, report on, and export their own sessions |
| Administrator | Everything a teacher can do, plus access to `/admin/` and system records |

The build process provisions one administrator and one non-admin teacher. The two account names must be different.

## Configuration

The app reads configuration from environment variables. See [`.env.example`](.env.example) for the local shape; do not commit a real `.env` file.

### Core variables

| Variable | Required for production | Purpose |
| --- | --- | --- |
| `DJANGO_SECRET_KEY` | Yes | Long, unique Django signing secret |
| `DJANGO_DEBUG` | Yes | Set to `False` in production |
| `DJANGO_ALLOWED_HOSTS` | Yes | Comma-separated public host names |
| `DJANGO_PUBLIC_URL` | Yes for hosted camera pairing | Public HTTPS origin, such as `https://your-app.vercel.app` |
| `DJANGO_CSRF_TRUSTED_ORIGINS` | Yes for hosted POST forms | Trusted HTTPS origins |
| `DATABASE_URL` | Recommended | PostgreSQL connection string; SQLite is the local fallback |
| `REDIS_URL` | Recommended | Shared transient cache for camera previews and live tracks |

### Account-provisioning variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `DJANGO_ADMIN_USERNAME` | `admin` | Administrator username |
| `DJANGO_ADMIN_PASSWORD` | `admin1234` | Administrator password — replace in production |
| `DJANGO_ADMIN_EMAIL` | `admin@example.com` | Administrator email |
| `DJANGO_DEMO_TEACHER_USERNAME` | `teacher` | Non-admin teacher username |
| `DJANGO_DEMO_TEACHER_PASSWORD` | `teacher12345` | Non-admin teacher password — replace in production |
| `DJANGO_DEMO_TEACHER_EMAIL` | `teacher@example.com` | Non-admin teacher email |

### Live-vision variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `DJANGO_VISION_MAX_FRAME_BYTES` | `2500000` | Largest accepted encoded camera-frame payload |
| `DJANGO_VISION_MAX_ANALYSIS_EDGE` | `960` | Maximum image edge used during detection |
| `DJANGO_VISION_FRAME_TTL_SECONDS` | `75` | Cached room-preview lifetime |
| `DJANGO_VISION_TRACK_TTL_SECONDS` | `25` | Anonymous positional-track lifetime |
| `DJANGO_LAN_HOST` | unset | Optional local LAN address used in local QR links |

## Deploy to Vercel

This repository is already configured for Vercel through [`pyproject.toml`](pyproject.toml), [`vercel.json`](vercel.json), and [`app.py`](app.py).

1. Link the repository or working directory to the intended Vercel project.
2. Add the production environment variables below in the Vercel project settings.
3. Deploy to that same project:

```powershell
vercel login
vercel --prod
```

Example production configuration:

```env
DJANGO_DEBUG=False
DJANGO_SECRET_KEY=<long-random-secret>
DJANGO_ALLOWED_HOSTS=your-app.vercel.app
DJANGO_PUBLIC_URL=https://your-app.vercel.app
DJANGO_CSRF_TRUSTED_ORIGINS=https://your-app.vercel.app

DJANGO_ADMIN_USERNAME=admin
DJANGO_ADMIN_PASSWORD=<use-a-strong-unique-password>
DJANGO_ADMIN_EMAIL=admin@example.com
DJANGO_DEMO_TEACHER_USERNAME=teacher
DJANGO_DEMO_TEACHER_PASSWORD=<use-a-strong-unique-password>
DJANGO_DEMO_TEACHER_EMAIL=teacher@example.com

DATABASE_URL=<postgresql-connection-string>
REDIS_URL=<redis-connection-string>
```

During every Vercel deployment, the build runs:

```text
collectstatic → migrate → create_demo_teacher
```

`create_demo_teacher` is idempotent: it creates missing accounts and updates the configured administrator and teacher on later deployments. Keep these environment variables stable, or update them deliberately, because their password values are applied during each build.

### Vercel live transport

Vercel Functions do not host the Django Channels WebSocket endpoint used by a traditional ASGI server. The application therefore uses supported HTTPS transports there:

- the teacher dashboard polls the live snapshot endpoint once per second;
- the phone uploads compressed frames through HTTPS; and
- a non-Vercel ASGI deployment can still use the Channels WebSocket routes.

## Local phone pairing

For local development, start Django on all network interfaces:

```powershell
python manage.py runserver 0.0.0.0:8000
```

When the dashboard is open at `localhost` or `127.0.0.1`, ATTEN attempts to put the teacher computer's LAN address in the QR link. If it picks the wrong adapter, specify it explicitly:

```powershell
$env:DJANGO_LAN_HOST="192.168.1.44"
python manage.py runserver 0.0.0.0:8000
```

For an iPhone, HTTPS is generally required for camera access. Prefer the deployed URL for a classroom demonstration.

## Data handling and privacy

- Student tracks are anonymous and are scoped to one live session.
- Full source frames and temporary face thumbnails are stored only in cache, not the relational database.
- Cached previews and track associations expire according to the vision TTL configuration.
- Session analytics store aggregate state, timestamps, and anonymous track identifiers.
- The operator is responsible for obtaining appropriate consent, complying with institutional policy, and following applicable privacy law before using a camera in a classroom.

## Testing

Run the focused application test suite and Django's system checks:

```powershell
python manage.py test monitoring
python manage.py check
```

The test suite covers session ownership, admin-only access, account provisioning, pairing, the HTTP camera upload path, analytics persistence, health checks, and initialization of the headless OpenCV detector.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| **Live detector unavailable** | Confirm the deployment uses the current `opencv-python-headless` dependency and redeploy. Review Vercel function logs for the import failure. |
| **No tracks appear** | Start the session first, allow camera access, make sure the phone is uploading frames, and use a well-lit, reasonably frontal camera view. |
| **Phone camera page cannot access the camera** | Open the QR link over HTTPS, grant browser camera permission, and avoid a phone-local `localhost` URL. |
| **QR link uses the wrong local address** | Set `DJANGO_LAN_HOST` to the teacher computer's Wi-Fi/LAN IP address. |
| **Dashboard appears stale across instances** | Configure `REDIS_URL`; in-memory cache is only suitable for local development or a single process. |
| **Cannot enter `/admin/`** | Sign in with the configured administrator account. Teacher accounts are intentionally not staff or superusers. |

## Project layout

```text
classfocus/                              Django settings and ASGI/WSGI entry points
monitoring/                              Sessions, reports, camera endpoints, and vision service
monitoring/services/attention_engine.py  Anonymous OpenCV tracking pipeline
monitoring/management/commands/          Account provisioning command
templates/                               Server-rendered dashboard, session, camera, and report views
static/monitoring/                       Dashboard and mobile-camera JavaScript and styles
pyproject.toml                           Python dependencies and Vercel build command
vercel.json                              Vercel configuration
```

## Health endpoint

Use `GET /healthz/` for an unauthenticated deployment health check. A healthy app returns:

```json
{"ok": true}
```
