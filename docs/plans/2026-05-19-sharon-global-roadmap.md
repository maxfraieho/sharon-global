# SharonGlobal — Implementation Roadmap

> **For Claude:** Use superpowers:executing-plans to implement this plan phase-by-phase.

**Goal:** Transform Sharon (single-family, Ukraine-only) into a worldwide multi-tenant crisis monitoring platform.

**Architecture:** Multi-tenant Telegram bot. Each user self-registers with their city/country. Alert channels are organized in a registry by country/conflict zone. Users receive only alerts relevant to their location. Sharon AI consultant answers in user's language.

**Tech stack:** Python 3.11+, Telethon (userbot monitoring), python-telegram-bot or aiogram (bot), PostgreSQL (multi-tenant data), FastAPI (consultant), LangGraph (AI pipeline), Docker Compose (deployment).

---

## Core Principle: Everything Per-User

Sharon Local had one `config.json`. SharonGlobal has zero global config — every setting belongs to a user:

```
UserProfile(user_id) → city, country, lat/lon, language, radius_km, notifications_on
FamilyGroup(group_id) → creator_user_id, members[]
ChannelRegistry(channel_id) → country_code, conflict_zone, language, keywords[]
AlertLog(user_id, channel_id, timestamp) → what was sent to whom
```

---

## Phase 0 — Project Bootstrap

**Goal:** Empty repo → runnable skeleton with CI.

### Task 0.1: Project structure
```
sharon-global/
├── bot/                    # Telegram bot handlers
│   ├── handlers/           # /start, /lang, /city, /family, /shelter, /threats
│   ├── i18n.py             # (copied + extended from Sharon)
│   ├── keyboards.py        # Per-user keyboards
│   └── user_profile.py     # UserProfile CRUD
├── monitor/                # Channel monitoring (Telethon userbot)
│   ├── channel_registry.py # ChannelRegistry DB queries
│   ├── classifier.py       # AI + keyword classify (from uav_watcher.py)
│   ├── dispatcher.py       # Route alert → relevant users
│   └── dedup.py            # Per-user dedup (90s window)
├── consultant/             # Sharon AI (copied, extended)
│   ├── main.py             # FastAPI :8770
│   ├── pipeline/           # LangGraph nodes
│   └── knowledge/          # Global conflict KB
├── family/                 # Family system (made multi-tenant)
│   ├── models.py
│   └── handlers.py
├── rescue/                 # Shelter search (already global via OSM)
│   └── shelter_search.py
├── db/
│   ├── models.py           # SQLAlchemy models
│   ├── migrations/         # Alembic
│   └── seed/               # Initial channel registry data
├── docker-compose.yml
├── Dockerfile
└── config.example.env      # Only secrets (tokens), no city/user data
```

### Task 0.2: Dependencies
```
telethon>=1.36
aiogram>=3.7          # modern Telegram Bot API
sqlalchemy>=2.0
alembic
asyncpg               # PostgreSQL async driver
fastapi uvicorn
langgraph langchain-core
httpx
python-dotenv
```

### Task 0.3: Docker Compose skeleton
```yaml
services:
  bot:        # Telegram bot (aiogram)
  monitor:    # Telethon userbot channel monitoring
  consultant: # FastAPI Sharon AI
  db:         # PostgreSQL
  redis:      # Dedup state + rate limiting
```

---

## Phase 1 — Multi-Tenant Core

**Goal:** User can `/start`, set city, get their profile. Zero hardcoded cities.

### Task 1.1: Database models
```python
class UserProfile(Base):
    user_id: int (PK)           # Telegram user_id
    telegram_username: str
    city: str                   # "Kyiv", "Tehran", "Tel Aviv"
    country_code: str           # "UA", "IR", "IL"
    lat: float
    lon: float
    radius_km: int = 50
    language: str = "uk"        # from i18n.py LANGS
    notifications_on: bool = True
    created_at: datetime
    last_active: datetime

class UserSession(Base):
    user_id: int (FK UserProfile)
    session_state: str          # "onboarding", "active", "setup_city"
    data: JSON                  # temp onboarding state
```

### Task 1.2: /start onboarding flow
```
/start →
  "Привіт! Я Sharon — кризовий монітор. Де ти зараз?"
  [Надіслати геолокацію] [Ввести місто вручну]

  If GPS → reverse geocode → "Знайшов: Київ, Україна. Підтвердити?"
  If text → geocode via Nominatim → confirm

  "Яка мова?" → [UK] [EN] [DE] [FR] [PL] [Інша]

  → Profile created → Main keyboard shown
```

### Task 1.3: /mycity command
Change city at any time. Re-geocode. Update UserProfile.

### Task 1.4: Geocoding service
```python
async def geocode(query: str) -> GeoResult:
    # Nominatim (OSM) — free, no key needed
    # Returns: city, country_code, lat, lon
```

---

## Phase 2 — Channel Registry

**Goal:** Channels organized by country/conflict zone. Auto-assign when user registers.

### Task 2.1: ChannelRegistry model
```python
class MonitoredChannel(Base):
    channel_id: int (PK)        # Telegram channel ID
    title: str
    country_code: str           # "UA", "IR", "IL", "SY", etc.
    conflict_zone: str          # "russia-ukraine", "iran-israel", "sudan", etc.
    language: str               # primary language of channel
    keywords: JSON              # ["тривога", "ракет", "БПЛА"] or ["siren", "strike"]
    alert_patterns: JSON        # regex patterns for threat detection
    allclear_patterns: JSON     # regex for all-clear
    active: bool = True
    region_filter: str          # e.g. "Кіровоградська" (narrow scope) or null (national)
    added_by: str               # "admin" | "auto-discovery"
    last_message_at: datetime
```

### Task 2.2: Initial channel seed data
```
db/seed/channels_ua.json  — Ukrainian channels (from current Sharon config)
db/seed/channels_ir.json  — Iran channels (TeleAlert, etc.)
db/seed/channels_il.json  — Israel channels
db/seed/channels_global.json — ACLED, UN OCHA, etc.
```

### Task 2.3: Channel auto-assignment
When user registers with country_code="UA" → auto-subscribe to all `active` channels where `country_code="UA"`.
User can manually add/remove channels via `/channels`.

### Task 2.4: Admin commands
```
/admin add_channel <id> <country> <zone> <lang>
/admin list_channels [country]
/admin deactivate_channel <id>
/admin stats
```
Protected by `ADMIN_USER_IDS` env var.

---

## Phase 3 — Multi-User Alert Dispatch

**Goal:** Incoming channel message → notify only users in affected area.

### Task 3.1: Monitor architecture
```
Telethon userbot watches ALL registered channels
        ↓
classifier.py:
    keyword_classify(text, channel.keywords) → threat/allclear/irrelevant
    ai_classify(text, cfg) → if keyword unclear
        ↓
dispatcher.py:
    extract_region(text) → "Кіровоградська", "Київ", "Tehran"
    find_affected_users(region, channel.country_code) →
        UserProfile.query where:
            country_code = channel.country_code
            AND distance(user.lat/lon, region_center) <= user.radius_km
            AND notifications_on = True
        ↓
    For each user → send_alert(user_id, formatted_message, lang=user.language)
```

### Task 3.2: Region extraction
```python
async def extract_region(text: str, channel: MonitoredChannel) -> RegionHint:
    # 1. Regex against channel.alert_patterns
    # 2. If channel has region_filter → return that (narrow scope channel)
    # 3. AI extraction if unclear
    # Returns: city_name, country_code, confidence
```

### Task 3.3: Per-user dedup
Redis key: `dedup:{user_id}:{threat_hash}` TTL=90s
Same threat → don't send twice even if from multiple channels.

### Task 3.4: Alert formatting
```python
def format_alert(threat: ThreatEvent, user: UserProfile) -> str:
    # Use i18n.py templates per user.language
    # Include: level emoji, city, reason, timestamp
```

---

## Phase 4 — Family System (Multi-Tenant)

**Goal:** Any user can create a family group. Members share location + get rollcall.

> **Foundation:** Copy `family/bot_handlers.py` from Sharon. Replace `config.json` family list with DB.

### Task 4.1: FamilyGroup model
```python
class FamilyGroup(Base):
    group_id: UUID (PK)
    name: str
    creator_user_id: int (FK UserProfile)
    created_at: datetime
    invite_token: str           # /join <token>

class FamilyMember(Base):
    group_id: UUID (FK)
    user_id: int (FK UserProfile)
    nickname: str
    role: str                   # "admin" | "member"
    last_seen: datetime
    last_location: JSON         # {lat, lon, timestamp}
```

### Task 4.2: Commands
```
/family create <name>    → creates group, returns invite link
/family join <token>     → joins group
/family status           → shows all members + last seen
/family rollcall         → asks all members to confirm they're safe
/family sos              → sends SOS to all family members
/family leave
```

### Task 4.3: Cross-border families
Family members can be in different countries. Rollcall works regardless. Each member still gets their OWN city alerts.

---

## Phase 5 — Global Sharon Consultant

**Goal:** Sharon AI answers questions about threats in user's city/language, worldwide context.

> **Foundation:** Copy `consultant/` from Sharon. Extend knowledge base.

### Task 5.1: Per-user context injection
```python
# In generate() node:
user_profile = get_user_profile(state["session_id"])
city_context = f"User is in {user_profile.city}, {user_profile.country_code}."
system_prompt = lang_instruction + city_context + GLOBAL_SYSTEM_PROMPT
```

### Task 5.2: Global knowledge base
```
consultant/knowledge/
├── ukraine_threats.md      (existing)
├── iran_threats.md         (new)
├── israel_threats.md       (new)
├── global_emergency.md     (new — 101/112 equivalents worldwide)
├── shelter_types.md        (new — what counts as shelter in different contexts)
└── psychological_support.md (existing, language-extended)
```

### Task 5.3: Country-specific emergency numbers
```python
EMERGENCY_NUMBERS = {
    "UA": {"fire": 101, "police": 102, "ambulance": 103, "unified": 112},
    "IR": {"fire": 125, "police": 110, "ambulance": 115, "unified": 115},
    "IL": {"fire": 102, "police": 100, "ambulance": 101, "unified": 101},
    # ...
}
```
Used in i18n templates instead of hardcoded Ukrainian numbers.

---

## Phase 6 — Auto Channel Discovery

**Goal:** When user registers in a new country with no channels → bot searches Telegram.

### Task 6.1: Telegram channel search
```python
async def discover_channels(country_code: str, keywords: list[str]) -> list[ChannelCandidate]:
    # Use Telethon SearchRequest
    # Filter by language, subscriber count, recent activity
    # Return candidates for admin review
```

### Task 6.2: Admin approval flow
Discovered channels → `pending_channels` table → admin `/admin review_channels` → approve/reject → added to registry.

### Task 6.3: Community submissions
Users can suggest channels: `/suggest_channel @channelname` → goes to pending queue.

---

## Phase 7 — Production Infrastructure

**Goal:** Deployable on a $5-10/month VPS. Docker Compose single-file deploy.

### Task 7.1: Docker Compose (production)
```yaml
services:
  bot:
    image: sharon-global:latest
    command: python -m bot
    env_file: .env
    restart: unless-stopped

  monitor:
    image: sharon-global:latest
    command: python -m monitor
    volumes:
      - ./sessions:/app/sessions  # Telethon session files
    restart: unless-stopped

  consultant:
    image: sharon-global:latest
    command: uvicorn consultant.main:app --host 0.0.0.0 --port 8770
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: sharon
      POSTGRES_USER: sharon
      POSTGRES_PASSWORD: ${DB_PASSWORD}

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data
```

### Task 7.2: config.example.env
```env
# Required
BOT_TOKEN=          # @SharonGlobal_bot token from @BotFather
TELETHON_API_ID=    # my.telegram.org
TELETHON_API_HASH=
TELETHON_PHONE=     # phone for userbot (channel monitoring)

# LLM
LLM_URL=https://integrate.api.nvidia.com/v1
LLM_TOKEN=
LLM_MODEL=meta/llama-4-maverick-17b-128e-instruct

# DB
DB_URL=postgresql+asyncpg://sharon:${DB_PASSWORD}@db:5432/sharon
DB_PASSWORD=

# Admin
ADMIN_USER_IDS=123456789,987654321

# Optional
ALERTS_IN_UA_TOKEN=
SENTRY_DSN=
```

### Task 7.3: One-command deploy
```bash
git clone https://github.com/maxfraieho/sharon-global
cp config.example.env .env
# Edit .env with your tokens
docker compose up -d
docker compose exec bot python -m db.migrate  # alembic upgrade head
```

---

## Phase 8 — Web Dashboard (Optional / Later)

**Goal:** Minimal admin UI. Not user-facing — users interact only via bot.

- Channel registry management (add/remove/approve)
- User stats (country distribution, active users)
- Alert log (who got what, when)
- System health

Stack: FastAPI + HTMX (no JS framework, server-rendered) or just Telegram admin commands.

**Decision:** Start with Telegram admin commands only. Add web dashboard if really needed.

---

## Sprint Plan (Suggested Order)

| Sprint | Phase | Duration | Deliverable |
|--------|-------|----------|-------------|
| 1 | 0 — Bootstrap | 1 day | Runnable bot skeleton, Docker Compose |
| 2 | 1 — Multi-tenant core | 2 days | /start, UserProfile, geocoding |
| 3 | 2 — Channel registry | 1 day | Registry model, Ukrainian seed data |
| 4 | 3 — Alert dispatch | 3 days | Full monitoring pipeline, per-user alerts |
| 5 | 4 — Family system | 2 days | /family commands, rollcall, SOS |
| 6 | 5 — Global consultant | 2 days | Per-user context, global KB |
| 7 | 6 — Auto discovery | 2 days | Telegram search, admin review queue |
| 8 | 7 — Production infra | 1 day | Docker deploy, .env, docs |

**Total estimate:** ~14 development days

---

## Key Decisions

### Why PostgreSQL instead of SQLite?
Multiple async writers (bot + monitor + consultant running in parallel). SQLite has write-lock issues with concurrent processes. PostgreSQL handles this natively.

### Why aiogram 3 instead of Telethon for bot?
Telethon is still used for **userbot** (monitoring channels as a regular user account). But for the **bot** interactions (receiving commands from users), aiogram 3 is more mature, better typed, and easier to scale.

### Why NOT copy config.json architecture?
In Sharon Local, `load_config()` is called everywhere. In SharonGlobal, the equivalent is `get_user_profile(user_id)`. Every handler needs `user_id` to fetch the user's settings. This is a clean pattern for multi-tenant.

### SharonGlobal bot name options
- `@SharonGlobal_bot` — clear, memorable
- `@SharonAlert_bot` — emphasizes alerts
- `@SafetySharon_bot` — safety-first framing

---

## What Stays From Sharon Local

| Component | Status | Notes |
|-----------|--------|-------|
| `bot/i18n.py` | ✅ Copy | Add country-specific emergency numbers |
| `consultant/pipeline/` | ✅ Copy | Add global KB, per-user city context |
| `shelter_search.py` | ✅ Copy | Already uses OSM → worldwide |
| `rescue/location_tracker.py` | ✅ Copy | Minor changes for multi-tenant |
| `family/bot_handlers.py` | ✅ Copy | Replace config family → DB FamilyGroup |
| `bot/city_switch.py` | ❌ Drop | Replace with UserProfile.city setter |
| `web_config.py` | ❌ Drop | Admin via Telegram commands only (Phase 8 optional) |
| `uav_watcher.py` | 🔄 Refactor | Split into `monitor/classifier.py` + `monitor/dispatcher.py` |
| `config.json` | ❌ Drop | Replaced by UserProfile + .env |

---

## Open Questions (For Discussion)

1. **Bot hosting**: where does SharonGlobal run? Needs 24/7 server. Suggested: DigitalOcean/Hetzner $6/mo VPS (4GB RAM, not OrangePi).

2. **Userbot account**: channel monitoring requires a real Telegram account (not bot). Use dedicated phone number or same as Sharon Local?

3. **Global channel curation**: Who adds channels for Iran, Israel, Sudan? Manual by admin or community-sourced?

4. **Rate limits**: If 1000 users in Kyiv and a threat hits, bot needs to send 1000 messages in seconds. Telegram rate limit = 30 msg/sec. Need queue + batching.

5. **Monetization / sustainability**: Free? Donations? Premium features (custom radius, more channels)?

---

*Plan created: 2026-05-19 | Base: Sharon uav-watcher | Next: Review and approve Sprint 1*
