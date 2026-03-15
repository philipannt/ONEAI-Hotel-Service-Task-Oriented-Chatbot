# OneAI — Hotel Service Task-Oriented Chatbot

**OneAI** is a task-oriented AI chatbot designed to replace hotel front-desk staff. It runs as a **Multi-tenant SaaS** — hotel owners register on the Platform, configure their hotel data via the Admin Panel, and embed the chatbot into their website as a widget.

> **Stage:** Monolith MVP &nbsp;·&nbsp; **Target:** 10K+ hotels, microservices split for production

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [System Architecture](#system-architecture)
- [Getting Started](#getting-started)
  - [Dev Mode (no DB/Redis required)](#dev-mode-no-dbRedis-required)
  - [Full Mode (PostgreSQL + Redis)](#full-mode-postgresql--redis)
- [API Endpoints](#api-endpoints)
- [NLU Pipeline](#nlu-pipeline)
- [Dialogue Manager (FSM)](#dialogue-manager-fsm)
- [Project Structure](#project-structure)
- [Training ML Models](#training-ml-models)
- [Testing](#testing)
- [Production Roadmap](#production-roadmap)

---

## Features

- 🏨 **Booking & Services** — 10 order types: room booking, spa, transport, tickets, meals, laundry, vehicle rental, extra hour, room decoration, repair
- 💬 **7 Language Support** — NLU models: VI + EN; NLG templates: VI, EN, ZH, ES, FR, JA, DE
- 🧠 **2-Tier NLU** — Global NER (16 entity types) + Tenant Gazetteer (13 hotel-specific entity types)
- 🔄 **Config-driven FSM** — Add a new intent by adding a YAML file, no code changes needed
- 🗂️ **Multi-turn Dialogue** — Intent Stack (interrupt handling), Intent Queue (multi-intent), Slot Carryover
- 🔑 **Multi-tenant Isolation** — Every query is scoped to `hotel_id`
- 🚀 **Dev Mode** — Run the full pipeline without PostgreSQL, Redis, or ML model weights

---

## Tech Stack

| Component | Technology |
|---|---|
| Backend | FastAPI (Python ≥ 3.11) + uvicorn |
| Database | PostgreSQL 16 + pgvector |
| Cache / Session | Redis 7 |
| NLU Intent (VI) | PhoBERT-v2 (fine-tuned, sigmoid multi-label) |
| NLU Intent (EN) | DeBERTa-v3-base (fine-tuned, sigmoid multi-label) |
| NLU NER (VI) | GLiNER `urchade/gliner_multi-v2.1` (span-based) |
| NLU NER (EN) | DeBERTa-v3 BIO token classification |
| Date Parser | dateparser + custom regex (VI/EN/ZH/ES/FR/JA/DE) |
| NLG | YAML templates (7 languages) |
| Dialogue Manager | Custom FSM Engine (config-driven YAML) |
| ORM | SQLAlchemy 2.0 async + asyncpg |
| Config | pydantic-settings + python-dotenv |
| Containerization | Docker Compose (dev) |

---

## System Architecture

```
Client (Hotel Widget)
    │
    ▼ POST /chat/v1/message  { message, user_id, session_id }
┌──────────────────────────────────────────────────┐
│  FastAPI  (app/main.py)                          │
│                                                  │
│  1. Auth: X-API-Key → hotel_id (api/deps.py)     │
│  2. load_session()          ← Redis              │
│  3. detect_language()       ← lingua             │
│  4. nlu_engine.process()    ← PyTorch in-process │
│  5. gazetteer_match()       ← Redis catalog      │
│  6. fsm.process_turn()      ← Pure Python FSM    │
│  7. dispatch_action()       → PostgreSQL write   │
│  8. fetch_query_data()      ← PostgreSQL read    │
│  9. render_reply()          ← YAML templates     │
│ 10. save_session()          → Redis              │
└──────────────────────────────────────────────────┘
    │
    ▼ ChatResponse  { session_id, reply, intent, state, filled_slots, ... }
```

**Multi-tenant isolation:** Every request includes `hotel_id`. Tenant catalogs are cached in Redis (TTL 5 min) and auto-rebuilt whenever an admin updates rooms or services.

---

## Getting Started

### Dev Mode (no DB/Redis required)

A full pipeline runs in-memory using JSON mock data — no infrastructure needed.

```bash
# 1. Create and activate virtual environment
python -m venv .venv
.venv\Scripts\activate       # Windows
# source .venv/bin/activate  # Linux/macOS

# 2. Install core + dev dependencies (no torch required)
pip install -e ".[dev]"

# 3a. Start dev server (MockDB, port 8000)
python run_dev.py

# 3b. Interactive CLI chat
python run_dev.py --cli

# 3c. Print mock data summary
python run_dev.py --test-data
```

Windows convenience scripts:
```
start_server.bat   # Start dev server
start_cli.bat      # Interactive CLI chat
```

**What Dev Mode replaces:**

| Production | Dev Mode |
|---|---|
| PostgreSQL | JSON files (`data/mock/*.json`) |
| Redis | In-memory dict |
| ML models | Graceful fallback to `nlu_fallback` |

**Test credentials:** API Key: `test-key-sunrise-2024` · Swagger UI: `http://localhost:8000/docs`

---

### Full Mode (PostgreSQL + Redis)

```bash
# 1. Start database and Redis
docker-compose up -d

# 2. Configure environment
cp .env.example .env
# Edit DATABASE_URL, REDIS_URL as needed

# 3. Install core + ML inference dependencies
pip install -e ".[ml]"

# 4. Run SQL migrations
psql $DATABASE_URL -f migrations/001_create_tables.sql
psql $DATABASE_URL -f migrations/002_seed_test_data.sql
psql $DATABASE_URL -f migrations/003_create_users_orders.sql

# 5. Start server
python run_dev.py
# or
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

---

## API Endpoints

### Chat (Public — Hotel Widget)

```http
POST /chat/v1/message
Content-Type: application/json
X-API-Key: <hotel_api_key>

{
  "message": "I want to book a Deluxe room",
  "user_id": "uuid | null",
  "session_id": "str | null"
}
```

**Response:**
```jsonc
{
  "session_id": "...",
  "reply": "Which dates would you like to book?",
  "intent": "BookHotelRoom",
  "confidence": 0.95,
  "entities": [{"type": "room_type", "value": "Deluxe", "confidence": 0.98, "source": "gazetteer"}],
  "state": "collecting_slots",
  "filled_slots": [{"name": "room_type", "value": "Deluxe", "status": "filled"}],
  "language": "en",
  "requires_auth": true,
  "order_id": null
}
```

### Admin Panel (Hotel Owner — `X-API-Key` required)

| Method | Endpoint | Description |
|---|---|---|
| GET / PUT | `/admin/v1/hotel` | Hotel information |
| GET / POST | `/admin/v1/rooms` | Room type management |
| GET / PUT / DELETE | `/admin/v1/rooms/{room_id}` | Room type detail |
| GET / POST | `/admin/v1/services?category=` | Service management |
| GET / PUT / DELETE | `/admin/v1/services/{service_id}` | Service detail |
| GET / POST | `/admin/v1/policies?category=` | Policy management |
| PUT / DELETE | `/admin/v1/policies/{policy_id}` | Policy detail |
| GET / POST | `/admin/v1/promotions` | Promotion management |
| PUT / DELETE | `/admin/v1/promotions/{promo_id}` | Promotion detail |
| GET / POST | `/admin/v1/faq` | FAQ management |
| PUT / DELETE | `/admin/v1/faq/{faq_id}` | FAQ detail |
| GET | `/health` | Health check |

---

## NLU Pipeline

### Intent Classification

| Language | Model | Training data | Intents |
|---|---|---|---|
| Vietnamese | PhoBERT-v2 (fine-tuned) | 12,689 samples | 30 |
| English | DeBERTa-v3-base (fine-tuned) | 7,072 samples | 30 |

- **Inference:** Sigmoid multi-label — returns top-3 intents with calibrated confidence scores
- **Thresholds:** Accept ≥ 0.5 · Fallback < 0.4 · Secondary intent queue ≥ 0.3
- **Calibration:** Temperature scaling per model (`calibration.json`)

### Entity Extraction — 2 Tiers

**Tier 1: Global NER** (per-language model, 16 entity types)

| Entity Type | Description |
|---|---|
| `start_date`, `end_date` | Check-in / check-out dates (via Date Parser) |
| `guest_adult`, `guest_child` | Number of guests |
| `room_quantity`, `vehicle_quantity` | Quantity |
| `contact_info` | Email or phone number |
| `booking_id` | Booking UUID |
| `customer_name`, `description_text` | Free-text name / description |
| `urgency_level` | low / medium / high |
| `pick_up`, `drop_off`, `location` | Locations |
| `adult_ticket`, `child_ticket` | Ticket counts |

**Tier 2: Tenant Gazetteer** (per-hotel catalog from Redis, 13 entity types)

`room_type`, `bed_type`, `amenity_type`, `spa_type`, `food_name`, `vehicle_type`, `ticket_name`, `ticket_category`, `laundry_type`, `clothing_type`, `event_type`, `service_name`, `payment_method`

### Entity Disambiguation

| Confidence | Action |
|---|---|
| ≥ 0.8 | Accept immediately → FILLED |
| 0.5 – 0.8 | Ask for confirmation: *"Did you mean X?"* |
| < 0.5 | Ignore — FSM will re-ask the slot |

---

## Dialogue Manager (FSM)

Config-driven — each intent maps to one `configs/intents/*.yaml` file. **Adding a new intent requires no code changes.**

```
Lifecycle: idle → collecting_slots → confirming → completed → idle
```

### 28 Intent Configs

| Group | Intents |
|---|---|
| **Service** (13) | BookHotelRoom, CancelBooking, AskRequestStatus, CheckRoomAvailability, BookTransport, SpaMassage, BookTicket, BookMeal, RentVehicle, Laundry, ExtraHour, RoomDecoration, Repair |
| **Query** (6) | AskRoomInfo, AskPromotion, AskService, AskServiceDetail, SuggestRoomType, AskPolicy |
| **System** (4) | greet, goodbye, thanks, nlu_fallback |
| **Context** (3) | affirm, deny, stop |
| **Special** (2) | HandoverToStaff, OutOfScope |

*Modify intent is handled inline in the FSM engine — no YAML config required.*

### Example Config (`configs/intents/BookHotelRoom.yaml`)

```yaml
intent: BookHotelRoom
auth: required
intent_type: service
slots:
  - name: room_type
    type: enum
    extraction: gazetteer      # Tier 2 — per-hotel catalog
    validation: must_exist_in_db
  - name: start_date
    type: date
    extraction: global_ner     # Tier 1 + Date Parser
    validation: ">= today"
  - name: end_date
    type: date
    extraction: global_ner
    validation: "> start_date"
  - name: guest_adult
    type: integer
    extraction: global_ner
    validation: ">= 1"
  - name: contact_info
    type: contact
    extraction: global_ner
    validation: email_or_phone
    source: user_profile       # Pre-filled from user profile
    carryover: true
on_confirm: action_submit_booking
on_deny: action_ask_modify_which_slot
```

### Advanced FSM Features

| Feature | Description |
|---|---|
| **Intent Stack** | User interrupts mid-flow → push current intent, handle new one, pop and resume (max depth 3) |
| **Intent Queue** | Multiple intents in one message → queue and process sequentially (cap 5) |
| **Slot Carryover** | Reuse slots from previous intent (e.g. `contact_info`, `start_date`) |
| **Cascading Validation** | Modifying `start_date` auto-invalidates dependent `end_date` |
| **Max Retries** | After 3 failed slot attempts → handover to human staff |

---

## Project Structure

```
OneAI/
├── app/
│   ├── api/                  # FastAPI routers
│   │   ├── chat.py           # POST /chat/v1/message
│   │   ├── hotel.py, rooms.py, services.py
│   │   ├── policies.py, promotions.py, faq.py
│   │   └── deps.py           # Auth dependency (API key → hotel)
│   ├── db/
│   │   ├── mock_db.py        # MockDB — JSON files (DEV_MODE)
│   │   ├── session.py        # SQLAlchemy async session factory
│   │   └── redis.py          # Redis client
│   ├── models/               # SQLAlchemy ORM models
│   │   ├── hotel.py, room_type.py, service.py
│   │   ├── policy.py, promotion.py, faq.py
│   │   ├── user.py, order.py
│   │   └── base.py           # SoftDeleteMixin, TimestampMixin
│   ├── schemas/              # Pydantic API schemas
│   ├── services/
│   │   ├── nlu/              # NLU pipeline
│   │   │   ├── engine.py     # NLUEngine singleton + language router
│   │   │   ├── intent_vi.py  # PhoBERT intent classifier
│   │   │   ├── intent_en.py  # DeBERTa intent classifier
│   │   │   ├── ner_vi.py     # GLiNER NER (VI)
│   │   │   ├── ner_en.py     # DeBERTa BIO NER (EN)
│   │   │   ├── date_parser.py # Date normalization (7 languages)
│   │   │   ├── word_segmenter.py # VnCoreNLP segmenter (VI)
│   │   │   └── schemas.py    # NLUInput / NLUOutput types
│   │   ├── fsm/              # Dialogue Manager
│   │   │   ├── engine.py     # FSM core logic
│   │   │   ├── config_loader.py # YAML intent config loader (lru_cache)
│   │   │   ├── slot_filler.py  # Slot fill + validation + carryover
│   │   │   └── models.py     # FSMContext, FSMState, SlotStatus
│   │   ├── nlg/              # Natural Language Generation
│   │   │   ├── renderer.py   # Template renderer
│   │   │   └── templates/    # YAML reply templates (7 languages)
│   │   ├── chat_service.py   # Main request orchestrator
│   │   ├── gazetteer.py      # Per-hotel entity matching
│   │   ├── catalog.py        # Catalog build + Redis cache
│   │   ├── knowledge_base.py # Query data (rooms, services, policies, ...)
│   │   ├── action_dispatcher.py # Map FSM actions → DB writes
│   │   ├── order_service.py  # Order CRUD
│   │   ├── query.py          # Read queries (knowledge base)
│   │   └── session.py        # Session load/save (Redis / in-memory)
│   ├── config.py             # Settings (pydantic-settings + .env)
│   └── main.py               # FastAPI app + lifespan startup
│
├── configs/
│   └── intents/              # 28 intent YAML configs
│
├── data/
│   ├── mock/                 # Mock data JSON files (DEV_MODE)
│   ├── training/             # ML training data (JSONL format)
│   │   ├── intents_vi.jsonl  # 12,689 samples — VI intents
│   │   ├── intents_en.jsonl  # 7,072 samples  — EN intents
│   │   ├── ner_vi.jsonl      # 3,030 samples  — VI NER
│   │   └── ner_en.jsonl      # 3,030 samples  — EN NER
│   ├── train_intent_vi.py, train_intent_en.py
│   ├── train_ner_vi.py, train_ner_en.py
│   ├── test_intent_vi.py, test_intent_en.py, test_all_models.py
│   └── generate_training_data.py
│
├── models/                   # Trained model weights (gitignored)
│   ├── intent_vi/final/      # model.safetensors + label_mapping.json
│   ├── intent_en/final/
│   ├── ner_gliner/final/     # GLiNER weights (VI NER)
│   ├── ner_en/final/
│   └── vncorenlp/            # VnCoreNLP word segmenter
│
├── migrations/               # SQL migration scripts
│   ├── 001_create_tables.sql
│   ├── 002_seed_test_data.sql
│   └── 003_create_users_orders.sql
│
├── tests/                    # pytest test suite
│   ├── conftest.py
│   ├── test_fsm_engine.py
│   ├── test_chat_e2e.py
│   ├── test_nlg_renderer.py
│   └── test_*.py
│
├── docs/                     # Technical documentation
├── run_dev.py                # Dev runner: server / CLI / test-data
├── run_tests.py              # Quick system test runner (no pytest)
├── test_mock_api.py          # Black-box API test (requires running server)
├── pyproject.toml            # All dependencies (core / ml / dev / train)
├── docker-compose.yml        # PostgreSQL (pgvector) + Redis
└── .env.example              # Environment variable template
```

---

## Training ML Models

### Requirements

```bash
# Install training dependencies
pip install -e ".[train]"

# GPU (recommended) — install torch separately for CUDA support:
# CUDA 12.4:
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
# CUDA 12.1:
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
```

### Training Scripts

```bash
# Intent — Vietnamese (PhoBERT-v2)
python data/train_intent_vi.py --epochs 10 --batch_size 32

# Intent — English (DeBERTa-v3)
python data/train_intent_en.py --epochs 10 --batch_size 32

# NER — Vietnamese (GLiNER)
python data/train_ner_vi.py --epochs 5 --batch_size 8

# NER — English (DeBERTa BIO)
python data/train_ner_en.py

# Evaluate all models
python data/test_all_models.py
```

**Each model outputs:** `model.safetensors` + tokenizer files + `label_mapping.json` + `calibration.json`

### Known Issues

- **DeBERTa-v3:** Must use `dtype=torch.float32` in `from_pretrained()` to prevent NaN from float16 checkpoint weights
- **DeBERTa-v3:** Requires `_fix_deberta_state_dict_hook()` to fix LayerNorm `gamma/beta` naming in checkpoints
- **VnCoreNLP:** Downloads model via `urllib` (not `wget`) for Windows compatibility

---

## Testing

### Unit + Integration Tests (pytest)

```bash
# Run full test suite
pytest tests/ -v

# Run specific tests
pytest tests/test_fsm_engine.py -v
pytest tests/test_chat_e2e.py -v
```

### Quick System Test (no pytest required)

```bash
# Runs 11 scenarios: FSM flows, NLU, Gazetteer, NLG, Actions
python run_tests.py
```

### Black-box API Test

```bash
# Step 1: Start the dev server
python run_dev.py

# Step 2 (separate terminal): Run API tests
python test_mock_api.py
```

---

## Production Roadmap

| Area | Current | Target |
|---|---|---|
| **NLU Inference** | PyTorch in-process (CPU) | Triton Inference Server (GPU, gRPC) |
| **Action Dispatch** | Sync DB write in request | Pub/Sub async workers + DLQ |
| **Session** | Redis GET/SET | Lua scripts (atomic updates) |
| **FSM Config** | Filesystem YAML + lru_cache | GCS/DB versioned, hot-reload via Pub/Sub |
| **Authentication** | API Key | JWT / OAuth2 |
| **NLU Languages** | VI + EN | Add ZH, ES, FR, JA, DE |
| **Architecture** | Monolith | 3-plane: Edge / Orchestration / Inference |

---

## License

Private — All rights reserved.
