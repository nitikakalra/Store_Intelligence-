# Store Intelligence System

Convert raw CCTV footage into retail analytics: visitor counts, dwell time,
zone activity, and POS-based conversion rate.

End-to-end pipeline:

```
CCTV video ──► YOLOv8 + tracker ──► events.jsonl ──► FastAPI (/events/ingest)
                                                       │
                                  pos_transactions.csv─┘
                                                       │
                                                       ▼
                                              SQLite + analytics
                                  (/metrics, /funnel, /heatmap, /anomalies)
```

---

## 1. Quickstart (Docker)

```bash
# 1. Build and start the API (auto-downloads yolov8n.pt on first detection run)
docker-compose up --build

# API is now live on http://localhost:8000
# Open http://localhost:8000/docs for the interactive Swagger UI.
```

In a second terminal, run the detection pipeline against the bundled sample:

```bash
docker-compose exec api python -m pipeline.detect \
  --video data/sample.mp4 \
  --store-id store_001 \
  --output data/events.jsonl

# Ingest events + POS transactions
docker-compose exec api python -m pipeline.ingest \
  --events data/events.jsonl \
  --pos data/pos_transactions.csv \
  --store-id store_001 \
  --api http://localhost:8000
```

Then hit:

- `GET http://localhost:8000/stores/store_001/metrics`
- `GET http://localhost:8000/stores/store_001/funnel`
- `GET http://localhost:8000/stores/store_001/heatmap`
- `GET http://localhost:8000/stores/store_001/anomalies`
- `GET http://localhost:8000/health`

---

## 2. Quickstart (Local Python)

Requires Python 3.10+.

```bash
# 1. Install
pip install -r requirements.txt

# 2. Download YOLOv8 nano weights (~6 MB).
#    Ultralytics will auto-download on first use, but you can pre-fetch:
python -c "from ultralytics import YOLO; YOLO('yolov8n.pt')"

# 3. Run the API
uvicorn app.main:app --reload --port 8000

# 4. In another shell — run detection on the bundled sample video
python -m pipeline.detect --video data/sample.mp4 --store-id store_001 --output data/events.jsonl

# 5. Ingest events and POS data
python -m pipeline.ingest --events data/events.jsonl --pos data/pos_transactions.csv --store-id store_001 --api http://localhost:8000
```

---

## 3. Sample Data

A tiny synthetic CCTV clip ships in `data/sample.mp4` (a few seconds of
moving silhouettes — enough to exercise the detection + tracking + ingest
path without any external download). `data/pos_transactions.csv` contains
matching purchase records.

Replace these with real footage to run against your own store.

`pos_transactions.csv` schema:

```
transaction_id,store_id,timestamp,amount
T0001,store_001,2025-01-01T10:00:30,42.50
```

---

## 4. Tests

```bash
pytest -q
```

Covers: empty input, duplicate events, missing transactions, ingest happy path.

---

## 5. Project Layout

```
store-intelligence/
├── app/                  FastAPI service
│   ├── main.py           routes
│   ├── models.py         Pydantic schemas
│   ├── db.py             SQLite + table init
│   └── analytics.py      metrics / funnel / heatmap / anomalies
├── pipeline/
│   ├── detect.py         YOLOv8 + centroid tracker → events.jsonl
│   └── ingest.py         batch POST events + POS to API
├── data/
│   ├── sample.mp4        tiny synthetic CCTV clip
│   └── pos_transactions.csv
├── tests/                pytest suite
├── docs/
│   ├── DESIGN.md
│   └── CHOICES.md
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md
```

See `docs/DESIGN.md` and `docs/CHOICES.md` for architecture notes.
