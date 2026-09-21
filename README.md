# ReQAgnIze Edge — Face-Attendance Appliance

A **self-contained edge build** of ReQAgnIze: the recognition + enrollment **backend, AI models,
database and post-quantum crypto**, driven by a **fullscreen OpenCV kiosk UI** — no web frontend,
no web server, no browser. Copy this folder onto a mini-PC / laptop / Jetson, plug in a camera,
and run.

**Flow:** `START` → models load → **live face scan** (SFace + FSRCNN super-resolution + FAISS
voting) → **"ACCESS ALLOWED — <name>"** and attendance is marked in SQLite. `ENROLL` captures 10
frames and adds the person (PII is **NTRU** post-quantum encrypted at rest).

## What's inside (backend + models + database + PQC only)
```
ReQAgnIze_Edge/
├── app/
│   ├── ai/                 # SFace/YuNet engine, FSRCNN super-res, FAISS index, preprocessing
│   ├── quantum_security/   # NTRU post-quantum encryption of PII (keygen, ring, store)
│   ├── services/           # recognition, enrollment (+ attendance, periods, visitors)
│   ├── models/ schemas/    # SQLAlchemy tables + Pydantic types
│   ├── db/                 # SQLAlchemy engine/session (SQLite)
│   ├── storage/            # local snapshot store
│   ├── core/config.py      # all tunables (thresholds, preprocessing, paths)
│   ├── ui/kiosk.py         # the OpenCV fullscreen display
│   └── main.py             # kiosk state machine (IDLE → SCAN → RESULT / ENROLL)
├── data/                   # SQLite DB, snapshots, NTRU keys, FAISS index  (gitignored)
├── ml_models/              # downloaded model weights (SFace/YuNet/FSRCNN)  (gitignored)
├── requirements.txt        # OpenCV(GUI)+contrib, numpy, sqlalchemy, pydantic + torch/pennylane (VQC)
├── cli.py (app/cli.py)     # DB management CLI (python -m app.cli)
└── .env                    # SQLite + local storage config
```
No web frontend, no FastAPI/uvicorn/Redis/Postgres — it's the pipeline core + a local UI. Matching
uses the dependency-free NumPy cosine index (no faiss wheel). The quantum VQC face/not-face gate is
on by default (`VQC_ENABLED`), which pulls in torch/pennylane; set it `false` to drop that weight.
Same recognition pipeline as the server (SFace + adaptive gamma/CLAHE + FSRCNN + low-res
prototypes + review-learning), so embeddings are compatible.

## Quick start
```powershell
cd ReQAgnIze_Edge
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt        # opencv-contrib (GUI) enables imshow + FSRCNN

copy .env.example .env                 # already provided; edit CAMERA_SOURCE if needed
python -m app.main                     # opens the kiosk window
```
First run downloads the SFace/YuNet (+ FSRCNN) weights into `ml_models/` and creates the SQLite
DB + NTRU keys under `data/`.

**Controls:** on-screen **START / ENROLL / HOME / CANCEL** buttons (mouse), keyboard for typing
during enrollment, **Esc** to quit. Set `CAMERA_SOURCE` to a webcam index (`0`) or an `rtsp://` URL.

## Watching the backend live
The kiosk is a GUI, but the pipeline now logs to the terminal it was launched from — one line per
recognition, plus enroll/power/load events:
```
12:25:30 INFO  reqagnize.edge   | READY in 3.4s - camera=0  index vectors=40  enrolled=2
12:25:36 INFO  reqagnize.edge   | MATCH  B.Roshan Babu   [10227]  conf=93% votes=4  present/marked  142ms
12:25:42 INFO  reqagnize.edge   | REJECT not-a-face (VQC gate, face_prob=3.0%)  88ms
12:25:48 INFO  reqagnize.edge   | MISS   needs-review best-conf=31%  95ms
```
Turn the detail up or down with env vars (in `.env` or the shell):
- `LOG_LEVEL=DEBUG` — also logs every no-face scan frame; `WARNING` = errors only.
- `SQL_ECHO=true` — print every SQL statement the pipeline runs.

## Managing the database
`python -m app.cli <command>` inspects and manages `data/reqagnize_edge.db` **and keeps the vector
index + NTRU metadata in sync** (prefer it over editing the DB file directly):

| command | what it does |
|---|---|
| `info` | DB path/size, per-table row counts, index vector total, storage usage |
| `stats` | enrolled / attendance today / pending reviews / events by result |
| `employees` | list everyone enrolled (id, code, name, status, vector count) |
| `show <code> [--pii]` | one person + recent attendance; `--pii` decrypts NTRU contact data |
| `attendance [--today\|--date YYYY-MM-DD]` | attendance rows |
| `events [--limit N] [--result not_a_face]` | recent recognition events |
| `reviews [--all]` | manual-review queue (pending by default) |
| `vectors` | per-person embedding counts in the index |
| `delete <code> [--yes]` | **purge** a person: index + NTRU metadata + DB row |
| `reset-attendance [--today\|--date …] [--yes]` | delete attendance rows |
| `sql "SELECT …"` | run a read-only query (SELECT only) |

`delete` and `reset-attendance` ask for confirmation unless you pass `--yes`. For read-only browsing,
a GUI such as **[DB Browser for SQLite](https://sqlitebrowser.org)** or the VS Code SQLite extension
opens `data/reqagnize_edge.db` directly — just don't hand-edit rows the index depends on.

## Notes
- **GUI OpenCV required** (`opencv-contrib-python`, *not* headless) — the kiosk uses `cv2.imshow`.
  The contrib build also enables `cv2.dnn_superres` (FSRCNN); without it the pipeline falls back to
  bicubic automatically.
- **Self-contained:** SQLite (`data/reqagnize_edge.db`), local snapshots, on-device NTRU keys.
  **Back up `data/`** — losing the NTRU keys makes encrypted PII unrecoverable.
- To run on a Jetson/Pi, install a platform OpenCV/faiss wheel; the app code is unchanged.
