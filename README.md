# EYON – Phase 5 (live camera + real-time YOLO detection)

FastAPI + PostgreSQL/SQLite + Ultralytics YOLO. The frontend (`static/index.html`) is served by FastAPI at `/`.
The browser camera stays the only video source: frames from the existing `<video>` are sent over a WebSocket,
YOLO runs on the server, and the boxes you see are the model's actual output.

## Frontend design
`static/index.html` holds all CSS and JS. Design tokens are CSS variables at the top of its `<style>` block (`--or` #FF8A00, `--or2` #F36B00,
`--bk` #080A0D, `--dk` #171D23, `--lg` #F5F6F7). `EYON` set as text is the logo; there is no separate symbol. Motion respects
`prefers-reduced-motion`. AR glasses appear only as a labelled future vision on the landing page.

## Quick start on Windows (CMD, SQLite, no PostgreSQL needed)

```
cd backend
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
set DATABASE_URL=sqlite:///./eyon.db
set SECRET_KEY=eyon-test-secret-key-change-me-1234567890
alembic upgrade head
uvicorn app.main:app --port 8080
```
Open http://127.0.0.1:8080 (Ctrl+F5 once). `pip install` is large (PyTorch). The first start downloads the
`yolo11n.pt` weights (about 5 MB, from GitHub), so it needs internet once; the page shows "Model loading…" until ready.
Opening a new terminal later: activate the venv and re-run the two `set` lines, then `uvicorn`.
Mac/Linux: use `source .venv/bin/activate` and `export` instead of `set`.

## PostgreSQL instead of SQLite
```
createuser eyon --pwprompt && createdb eyon -O eyon
cp .env.example .env     # set DATABASE_URL and SECRET_KEY, then: alembic upgrade head
```

## Configuration (.env or environment)

| Variable | Default | Meaning |
|---|---|---|
| YOLO_ENABLED | true | `false` skips loading the model (API and tests still work) |
| YOLO_MODEL | yolo11n.pt | any Ultralytics weights name or local path |
| YOLO_DEVICE | auto | `auto` uses an NVIDIA GPU if PyTorch sees one, else CPU; or `cpu` / `cuda:0` |
| YOLO_IMGSZ | 640 | inference size; try 416 for slower CPUs |
| MAX_FRAME_BYTES | 1000000 | larger WebSocket frames are rejected |
| DEFAULT_CONFIDENCE | 0.40 | starting threshold (the slider changes it live per session) |

NVIDIA GPU: install a CUDA build of PyTorch, e.g. `pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121`
(pick the command for your CUDA version at pytorch.org), then check `/api/detection/status` shows `"device": "cuda:0"`.

## Tests
```
pytest -q            # 27 tests, SQLite. The real-model test needs tests/data/bus.jpg (see below) and is skipped without it
curl -L -o tests/data/bus.jpg https://raw.githubusercontent.com/ultralytics/ultralytics/main/ultralytics/assets/bus.jpg
```
`e2e/*.js` drive the real page in jsdom against a running server (`npm i jsdom ws`; empty users table, `http://127.0.0.1:8000`).
`e2e/yolo-flow.js` needs the server started with `YOLO_ENABLED=true` and tests/data/bus.jpg.

## API (responses: `{success, data, message}` or `{success:false, error:{code,message,details}}`)

| Method | Path | Auth |
|---|---|---|
| POST | /api/auth/signup, /api/auth/login | no |
| GET | /api/auth/me, POST /api/auth/logout | Bearer |
| GET / PUT | /api/users/settings | Bearer |
| GET | /api/detections (history, own records only) | Bearer |
| GET | /api/detection/status (model state, device, classes) | Bearer |
| WS | /api/detection/ws | first message |
| GET | /api/health (includes `model`: disabled/loading/ready/error) | no |

### WebSocket protocol
1. Client sends `{"type":"auth","token":"<JWT>"}` within 5 s (the token never appears in the URL). Bad/expired token -> close 1008.
2. Server replies `{"type":"ready","model":{...},"confidence":0.4}`.
3. Client sends one binary JPEG/PNG/WebP frame, waits for the reply, then sends the next (no queue builds up).
   Reply: `{"type":"result","width","height","inference_ms","confidence","detections":[{"label","class_id","confidence","box":[x1,y1,x2,y2]}]}`
   with boxes normalised to 0..1.
4. Optional `{"type":"config","confidence":0.05..0.95}` changes the threshold used for later frames.
5. Errors keep the connection open: `invalid_frame`, `frame_too_large`, `model_not_ready`, `model_unavailable`, `inference_failed`, `bad_config`.
   A second session for the same user closes the first (`replaced`, code 4000); an expired token closes with 4401.

## Known limits
- The model is the pretrained COCO set (80 classes: person, chair, car, bicycle, bottle, cup, cell phone, laptop, tv, dog, cat,
  backpack, handbag, suitcase, ...). It has no "table" (COCO calls it `dining table`), and it does not know doors, stairs or other
  custom obstacles; those need a custom-trained model.
- No distance, direction, voice alerts or navigation yet (Phase 6+). Detections are not saved to the history table.
- Logout does not revoke the JWT server-side; no rate limiting, email verification or password reset.
- Frontend is one HTML file with CSS animations and an IntersectionObserver (no React/Vite, no Framer Motion). The orange/black/white
  redesign kept that architecture on purpose: adding Framer Motion would have required migrating to React and a build step.

## License warning
Ultralytics YOLO is licensed under **AGPL-3.0** (or a paid Enterprise license). If you distribute EYON or offer it as a network
service, AGPL obligations can apply to your code. Check this before any commercial use, or swap in a permissively licensed detector.
"# EYON" 
