# SharonGlobal 🌍

**Multi-tenant global crisis monitoring bot**

SharonGlobal is the evolution of [Sharon (UAV Watcher)](https://github.com/maxfraieho/uav-watcher) from a single-family local deployment into a worldwide multi-user crisis monitoring platform.

## What it does

- Monitors Telegram alert channels **per user's location** (city/country)
- Covers **all global conflict zones**: Ukraine, Iran, Israel, Sudan, Myanmar, etc.
- Each user gets alerts **only for their city and region**
- **5+ languages** with auto-detection from Telegram settings
- **Family system** — per-user family groups, rollcall, location sharing, SOS
- **Sharon AI consultant** — answers questions about current threats in user's language
- No server setup for users — just `/start` the bot

## vs Sharon Local

| Feature | Sharon Local | SharonGlobal |
|---------|-------------|--------------|
| Users | 1 family per deployment | Unlimited, worldwide |
| City config | config.json (admin sets) | Per user, self-service |
| Languages | 5 | 5+ (extensible) |
| Conflict zones | Ukraine only | Worldwide |
| Channels | Hardcoded list | Registry by country/zone |
| Deployment | Termux / home server | Cloud / VPS |
| Storage | SQLite + config.json | PostgreSQL (multi-tenant) |

## Bot

**@SharonGlobal_bot** (to register)

## Status

🚧 **Planning phase** — see [docs/plans/2026-05-19-sharon-global-roadmap.md](docs/plans/2026-05-19-sharon-global-roadmap.md)

## Foundation

Built on Sharon (uav-watcher) codebase. Reused components:
- `bot/i18n.py` — 5-language i18n
- `bot/lang_store.py` → evolved to full UserProfile
- `consultant/` — Sharon AI (extended to global context)
- `shelter_search.py` — OSM-based (already worldwide)
- `family/` — family system (made multi-tenant)
- `rescue/location_tracker.py` — GPS check-in

## License

MIT
