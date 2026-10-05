# AutoClip — local Docker setup

## This is a fork

`davidsparrow/autoclip` is a fork of
[zhouxiaoka/autoclip](https://github.com/zhouxiaoka/autoclip), which remains the
upstream and the place real development happens. We forked it to hold our Docker
serving fixes, not to diverge.

```bash
git remote -v
# origin    davidsparrow/autoclip   (ours)
# upstream  zhouxiaoka/autoclip     (theirs)

git fetch upstream && git merge upstream/main   # pull their fixes
```

**Nothing in `backend/`, `frontend/` or `prompt/` is modified.** Our additions
are four new files (below), so merges stay clean. Behaviour changes we needed
live in the adapter instead, as runtime patches — see
[clip-assist/UPSTREAM_NOTES.md](https://github.com/davidsparrow/clip-assist/blob/main/UPSTREAM_NOTES.md),
which documents seven defects worth sending upstream as PRs.

## Related repositories

| repo | role |
| --- | --- |
| [clipferno](https://github.com/davidsparrow/clipferno) | The marketplace. Creators post source videos, clippers make clips. |
| [clip-assist](https://github.com/davidsparrow/clip-assist) | Authenticated adapter between the two. **The only thing that should talk to this service.** |
| this fork | The AI clipping pipeline. No auth of any kind — keep it on a private network. |

---

Notes for this machine. The four files we add are listed below; upstream files
are untouched.

## What was added and why

| File | Why |
| --- | --- |
| `nginx.conf` | Serves the built frontend and proxies `/api/` to the backend. |
| `Dockerfile.web` | Builds the frontend and bakes it into an nginx image. |
| `docker-compose.override.yml` | Adds the `web` service; Compose loads it automatically. |
| `.env` | Gemini credentials and runtime config. Gitignored. |

**The gap this fills:** upstream's `docker-compose.yml` publishes port 3000 on the
`autoclip` container, but nothing inside that container listens there. The
backend only runs uvicorn on 8000 and has no `StaticFiles` mount, so the built
`frontend/dist` is never served. Out of the box you get a working API on 8000 and
a dead page on 3000. The `web` service is the missing half.

## Daily commands

Run all of these from this directory.

```bash
docker compose up -d        # start everything (builds on first run)
docker compose ps           # what is running
docker compose logs -f      # follow all logs, Ctrl-C to stop watching
docker compose down         # stop everything, keeps your data
docker compose up -d --build   # rebuild after changing code
```

`docker compose down` stops the containers but leaves `data/`, `logs/` and
`uploads/` on disk, so projects survive a restart.

## URLs

| | |
| --- | --- |
| Web UI | http://localhost:3100 |
| API docs (English, interactive) | http://localhost:8000/docs |
| Celery task monitor (Flower) | http://localhost:5555 |

## Services

Five containers. `docker compose ps` lists them.

- **redis** — message broker between the API and the workers.
- **autoclip** — the FastAPI backend, port 8000.
- **celery-worker** — does the actual downloading, transcribing and cutting.
  Long jobs run here, so this is the log to watch when a project seems stuck.
- **celery-beat** — scheduler for periodic tasks.
- **flower** — web monitor for the task queue, port 5555.
- **web** — nginx serving the UI, port 3000. Added locally.

## Gotchas

- **The UI is in Chinese.** There is no i18n layer and no language toggle;
  the strings are hardcoded across ~52 components. `http://localhost:8000/docs`
  is in English if you would rather drive it from there.
- **`settings.json` beats `.env`.** If you save an LLM provider in the UI
  settings page, `backend/core/llm_manager.py` prefers that over the environment
  variables and your `.env` provider choice stops taking effect. It lands in
  `data/`. Delete it to fall back to `.env`.
- **Speech recognition is installed** (see below), so videos without subtitles
  work too.
- **First build is slow.** It compiles the frontend twice (once for the app
  image, once for nginx) and pulls several base images.
- **ProtonVPN breaks Docker.** While connected, it routes Docker's internal VM
  subnet (192.168.65.0/24) into the tunnel, and the daemon can no longer reach
  its own containers: `docker run` hangs forever with no error, though builds
  and `docker ps` keep working. Diagnose with
  `route -n get 192.168.65.7` — if the interface is a `utun*` rather than `en0`,
  that is the cause. Disconnect the VPN and restart Docker Desktop, or exclude
  that subnet via split tunneling.
- **The UI is on 3100, not 3000.** Port 3000 is taken by an unrelated Next.js
  dev server on this machine.
- **"未找到 API 密钥配置" at startup is harmless.** It is a DashScope-only legacy
  check in `backend/app_factory.py` that looks for a key to copy into
  `DASHSCOPE_API_KEY`. Gemini is initialised separately and works; the warning
  appears regardless.

## Speech recognition (Whisper)

Installed and verified. Videos with no subtitle track are transcribed locally —
nothing is sent to a third party for this step.

- `faster-whisper 1.2.1` + `ctranslate2 4.8.2` in `data/whisper-runtime` (335MB)
- The `base` model cached in `data/whisper-models` (147MB)

Both live under the bind-mounted `data/`, so they survive `docker compose down`
and image rebuilds, and no image change was needed. This matches exactly where
`backend/services/whisper_runtime.py` looks, so the app finds it by itself.

Verified end to end: a test clip transcribed through the app's own
`generate_subtitle_for_video()` produced a correctly timestamped SRT.

It runs on CPU with `int8` quantisation. Expect transcription to take roughly
10-20% of the video's duration. To use a more accurate (slower, larger) model,
change `model="base"` in `backend/services/simple_pipeline_adapter.py` to
`small`, `medium` or `large-v3`; it downloads on first use.

**Note:** if Whisper ever fails, the pipeline does not error — it writes an
empty outline and produces zero clips. So "finished but no clips" usually means
transcription failed. Check `docker compose logs celery-worker`.

## Upstream bugs patched locally

Three real defects in the repo, all fixed in `docker-compose.override.yml`:

1. **Nothing served the frontend.** Port 3000 was published on a container with
   no server listening on it. Fixed with the nginx `web` service.
2. **Flower could never start.** `docker-compose.yml` runs `celery ... flower`
   inside the app image, but `flower` is absent from `requirements.txt`, so the
   subcommand does not exist and the container restart-looped. Swapped for the
   official `mher/flower` image.
3. **Celery containers always reported unhealthy.** Both inherit the
   Dockerfile's `HEALTHCHECK`, which curls the API on port 8000 — but they run
   Celery, not a web server, so the check could never pass. Replaced with an
   `inspect ping` for the worker and a process check for beat.
