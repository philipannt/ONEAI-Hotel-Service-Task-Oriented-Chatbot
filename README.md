# OneAI — Hotel Service Task-Oriented Chatbot

A multi-tenant SaaS chatbot that replaces hotel front-desk staff. Hotel owners register on the Platform, create their website, and integrate the AI Chatbot module via a widget. The chatbot extracts Intents + Entities and orchestrates slot-filling dialogues to handle room bookings, services, and hotel information queries.

> **Stage:** Monolith MVP · **Target:** 10K+ hotels, split into microservices for production

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Backend | FastAPI (Python ≥ 3.11) |
| Database | PostgreSQL 16 + pgvector |
| Cache / Session | Redis 7 |
| NLU - Intent (VI) | PhoBERT-v2 (fine-tuned) |
| NLU - Intent (EN) | DeBERTa-v3-base (fine-tuned) |
| NLU - NER (VI) | GLiNER mDeBERTa-v3 |
| NLU - NER (EN) | DeBERTa-v3 BIO tagging |
| NLG | YAML templates (7 languages) |
| Dialogue | FSM Engine (config-driven YAML) |

## Architecture

```
Client → FastAPI (POST /chat/v1/message)
  → API Key Auth → Load session (Redis)
  → Language Detection → NLU (Intent + NER)
  → Gazetteer Matching (per-hotel catalog, Redis)
  → FSM Engine (slot-filling → confirm → action)
  → Action Dispatcher (sync DB write)
  → NLG Renderer (YAML templates)
  → Save session (Redis) → Response
```

**Multi-tenant isolation:** Every query includes `hotel_id` (UUID). Tenant catalogs are cached in Redis and auto-rebuilt when an admin updates data.

## Quick Start

### 1. Start DB & Redis

```bash
docker-compose up -d
```

### 2. Configure environment

```bash
cp .env.example .env
# Edit .env if needed
```

### 3. Run server

```bash
pip install -e .
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

### 4. Health check

```bash
curl http://localhost:8000/health
# {"status": "ok"}
```

## API Endpoints

### Chat (Public — Hotel Widget)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/chat/v1/message` | Send a chat message |

### Admin Panel (Hotel Owner — `X-API-Key` required)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET/PUT | `/admin/v1/hotel` | Hotel information |
| GET/POST | `/admin/v1/rooms` | Room type management |
| GET/PUT/DELETE | `/admin/v1/rooms/{room_id}` | Room type detail |
| GET/POST | `/admin/v1/services?category=` | Service management |
| GET/PUT/DELETE | `/admin/v1/services/{service_id}` | Service detail |
| GET/POST | `/admin/v1/policies?category=` | Policy management |
| PUT/DELETE | `/admin/v1/policies/{policy_id}` | Policy detail |
| GET/POST | `/admin/v1/promotions` | Promotion management |
| PUT/DELETE | `/admin/v1/promotions/{promo_id}` | Promotion detail |
| GET/POST | `/admin/v1/faq` | FAQ management |
| PUT/DELETE | `/admin/v1/faq/{faq_id}` | FAQ detail |

## Chat Request / Response

```jsonc
// Request
{ "message": "I want to book a Deluxe room", "user_id": "uuid|null", "session_id": "str|null" }

// Response
{
  "session_id": "...",
  "reply": "Which dates would you like to book?",
  "intent": "BookHotelRoom",
  "confidence": 0.95,
  "entities": [{"type": "room_type", "value": "Deluxe", "confidence": 0.98, "source": "gazetteer"}],
  "state": "collecting_slots",
  "filled_slots": [{"name": "room_type", "value": "Deluxe", "status": "filled"}],
  "language": "en",
  "requires_auth": false,
  "order_id": null
}
```

## NLU Pipeline

**2-tier entity extraction:**

| Tier | Scope | Description | Entity Types |
|------|-------|-------------|-------------|
| Global NER | Per-language model | Dates, numbers, contacts, locations, ... | 16 types |
| Tenant Gazetteer | Per-hotel catalog | Room types, services, food, spa, ... | 13 types |

**Language support:** NLU models — VI, EN · NLG templates — VI, EN, ZH, ES, FR, JA, DE

## Dialogue Manager (FSM)

Config-driven FSM — each intent = 1 YAML file. Adding a new intent only requires a new config, no code changes.

```
States: idle → collecting_slots → confirming → completed → idle
```

**28 intent configs:** 13 service · 6 query · 4 system · 3 context · 2 special

| Group | Intents | Auth |
|-------|---------|------|
| Service | BookHotelRoom, CancelBooking, BookTransport, SpaMassage, BookTicket, BookMeal, RentVehicle, Laundry, ExtraHour, RoomDecoration, Repair, CheckRoomAvailability, AskRequestStatus | Required (except CheckRoom) |
| Query | AskRoomInfo, AskPromotion, AskService, AskServiceDetail, SuggestRoomType, AskPolicy | Guest OK |
| System | greet, goodbye, thanks, nlu_fallback, affirm, deny, stop | — |
| Special | Modify, HandoverToStaff, OutOfScope | — |

**Features:** Intent Stack (interrupt handling, depth=3) · Intent Queue (multi-intent, cap=5) · Slot Carryover · Entity Disambiguation (confidence-based)

## Data Layer

| Store | Description | Technology |
|-------|-------------|-----------|
| Knowledge Base | Hotel info (rooms, services, policies, promotions, FAQ) | PostgreSQL + pgvector |
| User DB | User profiles | PostgreSQL |
| Order DB | All orders (10 types) | PostgreSQL (JSONB details) |
| Tenant Catalog | Cache for Gazetteer matching | Redis (TTL 5 min) |
| Session | Conversation state | Redis (TTL 30 min) |

## Project Structure

```
OneAI/
├── app/
│   ├── api/              # FastAPI routers (chat, admin CRUD)
│   ├── db/               # SQLAlchemy async engine, Redis client
│   ├── models/           # ORM models (hotel, room, service, order, ...)
│   ├── schemas/          # Pydantic schemas (API contracts)
│   ├── services/
│   │   ├── nlu/          # NLU pipeline (intent, NER, date parser)
│   │   ├── fsm/          # FSM dialogue manager (config-driven)
│   │   ├── nlg/          # NLG renderer + 7-lang YAML templates
│   │   ├── chat_service  # Main orchestration
│   │   ├── gazetteer     # Per-hotel entity matching
│   │   └── ...           # Session, catalog, actions, orders, query
│   ├── config.py         # Settings (pydantic-settings)
│   └── main.py           # FastAPI app + lifespan
├── configs/intents/      # 28 intent YAML configs
├── data/training/        # Training data (JSONL) + scripts
├── models/               # Trained ML model weights
├── migrations/           # SQL migrations (3 files)
├── tests/                # pytest test suite
├── docker-compose.yml    # PostgreSQL (pgvector) + Redis
└── pyproject.toml
```

## Testing

```bash
pytest tests/ -v
```

## License

Private — All rights reserved.
