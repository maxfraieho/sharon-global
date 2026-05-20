---
title: "Ознайомлення з результатами AI-DRAKON Pipeline"
type: plan
project: sharon-global
tags: [orientation, ui-guide, drakon-ir, docs, diagrams]
created: 2026-05-20
status: active
---

# Ознайомлення з результатами AI-DRAKON Pipeline

Цей документ — покрокова інструкція для огляду того, що вже згенерував pipeline: модульна документація Sharon Global, DRAKON IR схеми, граф знань.

---

## Передумови — перевірка системи

Перед тим як відкривати UI, перевір що агенти живі:

```bash
# З dev сервера (192.168.3.184)
curl -s http://localhost:8767/health   # docs-agent
curl -s http://localhost:8766/health   # architect-agent
curl -s http://localhost:8765/health   # drakon-agent
```

Очікуваний результат — `{"status":"ok", ...}` для кожного.

Перевір поточні налаштування docs-agent:
```bash
curl -s http://localhost:8767/settings | python3 -m json.tool
```
Має бути: `repo_root=/home/vokov/workspace/sharon-global`, `proxy_model=claude-haiku-4-5`, `proxy_protocol=anthropic`.

---

## Частина 1 — Читання документації модулів

### Крок 1: Відкрити UI
Перейди на [https://ai-drakon-setup.pages.dev](https://ai-drakon-setup.pages.dev) та залогінься.

### Крок 2: Перейти до Документації
Натисни **"Документація"** у боковому меню (або URL `/docs`).

### Крок 3: Вкладка «Документи»
Вибери вкладку **"Документи"** — з'явиться список документів sharon-global:

| Назва | Slug | Зміст |
|-------|------|-------|
| Sharon-Global Architecture | ARCHITECTURE | Загальна архітектура системи |
| Модуль UAV Watcher | modules/uav_watcher | Телеграм-моніторинг загроз, класифікація |
| Модуль пошуку укриттів | modules/shelter_search | OSM Overpass, геопошук укриттів |
| Модуль консультанта Sharon | modules/consultant | AI-консультант, FastAPI :8770 |
| Модуль моніторингу ситуації | modules/situation_watcher | Агрегація загроз по регіону |
| location_tracker | modules/location_tracker | Трекінг локації членів родини |
| geo_monitor | modules/geo_monitor | Динамічний конструктор патернів |
| SharonGlobal Roadmap | plans/2026-05-19-sharon-global-roadmap | Roadmap реалізації |

Натискай на документ — відкривається повний Markdown-контент українською.

### Якщо список порожній
Це означає що Worker не проксіює запит до docs-agent. Перевір:
```bash
# Тест напряму через Worker (замість JWT — свій токен з Settings сторінки)
curl -H "Authorization: Bearer <твій_jwt>" \
  https://drakon-mcp-worker.maxfraieho.workers.dev/v1/notes/list
```
Має повернути JSON з 9 документами.

---

## Частина 2 — Граф документів

### Крок 1: Вкладка «Граф»
На тій же сторінці `/docs` вибери вкладку **"Граф"**.

### Що побачиш
Зараз граф має **9 вузлів** (кожен документ — окремий вузол) та **0 ребер**.

Ребра з'являться коли документи посилаються один на одного через `[[wikilink]]` синтаксис в тілі документу. Поки що в документах `related: []` і немає явних wiki-посилань — це нормальна початкова стадія.

### Як додати зв'язки між документами
Відредагуй будь-який документ через вкладку "Документи" → додай в тіло:
```markdown
Дивись також: [[modules/shelter_search]] та [[modules/situation_watcher]]
```
Після збереження граф покаже ребра між цими вузлами.

---

## Частина 3 — DRAKON схеми в редакторі

### Поточний стан
10 DRAKON IR схем згенеровано і збережено в git-репозиторії:
```
sharon-global/docs/drakon-ir/
  ai_classify.json
  build_ai_prompt.json
  keyword_classify.json
  send_notification.json
  send_allclear_notification.json
  _haversine.json
  _query_overpass.json
  _nominatim_address.json
  _fmt_last_seen.json
  _make_alert_call.json
```

### Як відкрити схему в редакторі

**Спосіб 1 — Імпорт JSON (зараз доступно):**

1. Завантаж JSON файл схеми з GitHub:
   - Перейди на [https://github.com/maxfraieho/sharon-global/tree/main/docs/drakon-ir](https://github.com/maxfraieho/sharon-global/tree/main/docs/drakon-ir)
   - Натисни на потрібний файл (наприклад `keyword_classify.json`)
   - Натисни **"Download raw file"** → файл збережеться на диск

2. У UI перейди на сторінку **"Схеми"** (або URL `/diagrams`)

3. Якщо немає жодної схеми в поточній папці, побачиш кнопки:
   - `+ Нова схема`
   - **`Імпорт JSON`** ← натисни цю

4. Вибери завантажений JSON файл → схема з'явиться в редакторі

5. Ліворуч побачиш список схем, схема відображається в центрі як DRAKON-граф

**Спосіб 2 — Через MCP (для автоматизації):**
```
# В Claude Code з підключеним MCP drakon:
drakon.listgitdiagrams owner=maxfraieho repo=sharon-global branch=main
```
Потім `drakon.getgitdiagram` для конкретної схеми.

*(Увага: цей інструмент шукає в `drn/` папці репозиторію, а не в `docs/drakon-ir/` — потрібна міграція файлів або UI-доповнення, описане нижче.)*

### Що побачиш в редакторі

Кожна схема відображає алгоритм модуля у вигляді DRAKON-блок-схеми:
- **keyword_classify** — алгоритм класифікації за ключовими словами (2-гілкова логіка)
- **ai_classify** — виклик LLM з fallback-логікою
- **send_notification** — умови відправки сповіщення з cooldown
- і т.д.

---

## Частина 4 — Agent Studio (Pipeline граф)

Перейди на сторінку **"Agent Studio"** (`/pipeline`).

Тут відображається граф взаємодії агентів:
- Sharon (вхідні Telegram повідомлення)
- UAV Watcher → keyword_classify → ai_classify → send_notification
- Зв'язки між вузлами показують flow даних

---

## Відомі обмеження та план усунення

| Проблема | Статус | Рішення |
|----------|--------|---------|
| Граф документів без ребер | Очікувано — wikilinks відсутні | Редагувати docs, додати `[[посилання]]` |
| DRAKON IR не видно в DiagramsPage автоматично | Потребує роботи | Додати docs-agent ендпоінти `/drakon-ir/list`, `/drakon-ir/get` + Worker маршрут + UI таб |
| `drakon.listgitdiagrams` шукає в `drn/` а не `docs/drakon-ir/` | Розбіжність | Скопіювати IR файли в `drn/` або оновити логіку Worker |
| SettingsPage зберігає лише в localStorage | Агенти не перезапускаються | Додати `POST /settings` ендпоінти на агентах |

---

## Швидкий чеклист огляду результатів

- [ ] `curl http://192.168.3.184:8767/health` → `{"status":"ok"}`
- [ ] `curl http://192.168.3.184:8767/notes/list` → 9 документів
- [ ] UI `/docs` → вкладка "Документи" → 9 записів у списку
- [ ] Натиснув на `modules/uav_watcher` → прочитав документацію
- [ ] UI `/docs` → вкладка "Граф" → 9 вузлів видно
- [ ] Завантажив `keyword_classify.json` з GitHub
- [ ] UI `/diagrams` → "Імпорт JSON" → завантажив схему → схема відображається
- [ ] Порівняв DRAKON схему з вихідним кодом `uav_watcher.py`

