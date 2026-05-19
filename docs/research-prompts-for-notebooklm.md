# SharonGlobal — Research Prompts for NotebookLM / Gemini

> **Як використовувати:**
> 1. Завантаж у NotebookLM: код Sharon (uav-watcher), всі `*.md` з `consultant/knowledge/`, план `2026-05-19-sharon-global-roadmap.md`
> 2. Запускай кожен промт окремо — Gemini аналізує з контекстом твоїх матеріалів
> 3. Відповіді зберігай у `docs/research/Q<N>-<topic>.md`

---

## Q1 — Multi-Tenant Architecture

```
Я розробляю SharonGlobal — багатокористувацьку версію кризового Telegram-бота Sharon.
Поточна Sharon (код в джерелах) використовує один config.json для однієї сім'ї.

Проаналізуй поточну архітектуру Sharon і запропонуй детальну схему переходу на
multi-tenant:

1. Перелічи всі місця в коді де config.json читається або використовується
   (особливо city, notify_chat_id, bot_token, llm_proxy_*).
2. Запропонуй структуру таблиці UserProfile в PostgreSQL — які поля потрібні,
   типи, індекси, constraints.
3. Як змінюється точка входу кожного handler — зараз cfg передається глобально,
   як правильно передавати user_profile в aiogram 3?
4. Що з session state для onboarding (користувач тільки почав, не ввів місто)?
   Запропонуй FSM схему для aiogram 3.
5. Які race conditions виникнуть якщо 1000 юзерів одночасно пишуть боту?

Результат: детальний архітектурний документ з кодовими прикладами.
```

---

## Q2 — PostgreSQL Schema (Full)

```
На основі коду Sharon і плану SharonGlobal розроби повну схему PostgreSQL бази даних.

Потрібні таблиці:
- UserProfile (користувач, його місто/координати/мова/налаштування)
- FamilyGroup + FamilyMember (сім'я, багатокористувацька версія family/bot_handlers.py)
- MonitoredChannel (реєстр каналів по країнах/зонах конфліктів)
- AlertLog (хто що отримав і коли — для dedup і аналітики)
- UserChannelSubscription (які канали слідкує конкретний юзер)
- PendingChannel (черга каналів для review адміном)

Для кожної таблиці:
1. SQL CREATE TABLE з типами, PK, FK, індексами
2. Обґрунтування кожного поля (навіщо він)
3. Які запити будуть найчастіші → оптимізація індексів
4. Alembic migration скрипт

Врахуй: система повинна витримати 10 000 активних юзерів і 100 каналів.
```

---

## Q3 — Global Channel Registry Strategy

```
Sharon зараз моніторить 8 фіксованих Telegram-каналів для Кіровоградської області.
SharonGlobal повинен покривати конфлікти по всьому світу.

Досліди і дай відповідь:

1. Які Telegram-канали існують для моніторингу загроз в цих зонах конфліктів:
   - Іран (ракетні атаки, БПЛА)
   - Ізраїль/Газа
   - Судан
   - М'янма
   - Ліван
   Дай конкретні назви каналів з їх Telegram username (@...).

2. Як класифікувати канали (channel_type):
   - офіційні (уряд, армія)
   - журналістські
   - регіональні цивільні
   - агрегатори
   Яка пріоритетність при конфлікті сигналів?

3. Keyword lists для кожної зони конфліктів — слова-тригери для threat/allclear
   на мовах: арабська, перська, іврит, бірманська, англійська.

4. Як автоматично оновлювати реєстр? Стратегія curation vs auto-discovery.

Результат: готовий JSON файл-seed для MonitoredChannel таблиці.
```

---

## Q4 — Alert Dispatch Algorithm

```
Коли Sharon (поточна) отримує повідомлення з каналу — вона надсилає ОДНОМУ чату.
SharonGlobal повинна надіслати ТИСЯЧАМ юзерів в потрібному місті.

Розроби детальний алгоритм dispatch:

1. Як витягти географічний регіон з тексту повідомлення?
   Наприклад: "Повітряна тривога в Олександрійському районі" → lat/lon bbox.
   Які NLP підходи? Regex + geo dictionary vs LLM extraction?

2. Як ефективно знайти всіх юзерів в affected zone?
   SQL запит з PostGIS (haversine) vs Redis geo index?
   Порівняй продуктивність для 10k юзерів.

3. Telegram Bot API rate limit: 30 msg/sec для одного бота.
   Як надіслати 1000 повідомлень за розумний час?
   Asyncio queue + rate limiter implementation.

4. Priority queuing: загроза рівня CRITICAL повинна йти першою.
   Як організувати чергу?

5. Retry logic для failed deliveries (юзер заблокував бота, мережа впала).

Результат: псевдокод + Python реалізація dispatcher.py.
```

---

## Q5 — Telethon Userbot at Scale

```
Sharon використовує Telethon userbot для моніторингу каналів як звичайний юзер.
Поточна реалізація (uav_watcher.py) моніторить 8 каналів для однієї сесії.

SharonGlobal потребує моніторингу 100+ каналів з різних країн.

Дослідь:

1. Обмеження Telegram на одну userbot-сесію:
   - Скільки каналів можна моніторити одночасно?
   - FloodWait ризики при великій кількості каналів
   - Best practices для production userbot

2. Архітектура multi-session моніторингу:
   - Один акаунт / кілька акаунтів?
   - Як розподілити канали між сесіями?
   - Session management і відновлення після disconnect

3. Telethon event handlers для 100+ каналів — чи є performance issues?
   Порівняй: один великий handler vs per-channel handlers.

4. Як обробляти media messages (фото, відео з текстом)?
   Caption extraction для класифікації.

5. Flood wait handling — як не отримати бан при ребуті після краша?

Результат: production-ready monitor/session_manager.py архітектура.
```

---

## Q6 — LangGraph Pipeline for Multi-User AI

```
Поточний Sharon consultant (consultant/pipeline/) використовує LangGraph з
MemorySaver — in-memory, для одного юзера.

SharonGlobal потребує:
- Тисячі паралельних сесій
- Per-user контекст (місто, країна, поточна загрозна ситуація)
- Глобальні знання (не тільки Україна)

Дослідь і запропонуй:

1. LangGraph checkpointer для multi-user:
   - AsyncSqliteSaver vs PostgresSaver vs RedisSaver
   - Trade-offs пам'ять / персистентність / швидкість
   - Як обмежити розмір history per user (max N messages)?

2. Per-user context injection в system prompt:
   Як передати city, country, current_threat_level в generate() node?
   Де зберігати "поточна загроза в місті юзера" — в state чи окремо?

3. Global knowledge base структура:
   - Поточний Sharon KB (Ukrainian threats, shelter info)
   - Що додати для глобального покриття?
   - Як організувати retrieval щоб не перемішувати контексти різних країн?

4. Session cleanup strategy: коли видаляти стару history?
   TTL-based vs explicit clear (як ми вже зробили при /lang зміні)?

Результат: оновлена архітектура consultant/ для multi-tenant.
```

---

## Q7 — Family System Multi-Tenant Design

```
Поточна family система Sharon (family/bot_handlers.py) зберігає членів сім'ї
в config.json. Один deployment = одна сім'я.

SharonGlobal: будь-який юзер може створити свою сім'ю. Тисячі сімей паралельно.

Проаналізуй поточний family/bot_handlers.py і:

1. Перелічи всі функції які використовують config.json для family-даних.
   Як замінити кожну на DB-запит.

2. Invite system дизайн:
   - Генерація unguessable invite token (не sequential ID)
   - Token expiry (24h? або без обмежень?)
   - Як захистити від spam joins?

3. Cross-border families:
   Якщо мама в Києві, дочка в Берліні — як rollcall враховує timezone?
   Як показати "last seen" в локальному часі кожного члена?

4. SOS routing в multi-tenant:
   Поточний SOS надсилає в notify_chat_id. Тепер треба надіслати
   всім членам FamilyGroup. Як зробити це надійно (retry, confirmation)?

5. Family privacy: чи можуть члени сім'ї бачити точне місцезнаходження один одного?
   Запропонуй privacy settings.

Результат: повний API spec для family/ модуля + DB migrations.
```

---

## Q8 — Global Crisis Knowledge Base

```
Sharon consultant зараз має knowledge base (consultant/knowledge/) орієнтований
на Україну: БПЛА, ракети, укриття, психологічна підтримка.

SharonGlobal потребує глобального KB.

На основі наданих матеріалів і власних знань:

1. Структура глобального KB:
   - Які документи потрібні для кожного конфліктного регіону?
   - Як організувати retrieval щоб запит "What to do during missile attack?"
     від юзера в Ірані не змішувався з українським контекстом?

2. Контент для нових регіонів (напиши готові .md файли):
   - iran_threats.md: типи загроз, shelter поради, екстрені номери (IR)
   - israel_threats.md: Iron Dome, Home Front Command інструкції
   - general_crisis.md: universal поради незалежно від країни

3. Multilingual KB:
   - Чи зберігати KB окремо по мовах чи перекладати на льоту через LLM?
   - Trade-offs якість/швидкість/вартість

4. KB оновлення стратегія:
   - Як підтримувати актуальність (автоматично vs ручне)?
   - Де брати достовірні джерела для кожного регіону?

Результат: структура `consultant/knowledge/` + готові md файли для 3 регіонів.
```

---

## Q9 — Telegram Rate Limits & Scaling

```
Я будую бота який при великій тривозі може надіслати 10 000 повідомлень за хвилину.
Telegram Bot API має ліміт: 30 повідомлень/сек, max 20 повідомлень/хвилину в одну групу.

Дослідь production-ready підхід:

1. Точні ліміти Telegram Bot API:
   - sendMessage rate limits (global vs per-chat)
   - Що відбувається при перевищенні (429 error, retry_after)
   - Broadcast flooding: як розробники великих ботів (Durov/Third-party) це вирішують

2. Python asyncio queue implementation:
   Напиши MessageQueue клас який:
   - Приймає (user_id, message, priority) задачі
   - Надсилає з max 25 msg/sec (з запасом від ліміту 30)
   - Priority: CRITICAL > HIGH > NORMAL
   - Retry при 429 з exponential backoff
   - Метрики: queue depth, send rate, failures

3. Batching strategy:
   Якщо 1000 юзерів в Києві — чи краще sendMessage×1000
   чи одне повідомлення в групу? Trade-offs.

4. Redis vs in-memory queue для persistence:
   Якщо сервер падає між надсиланнями — як не загубити 500 невідправлених алертів?

Результат: production MessageQueue implementation + конфігурація.
```

---

## Q10 — Auto Channel Discovery

```
Для нових країн де немає готових каналів в реєстрі SharonGlobal,
потрібна система автоматичного пошуку релевантних Telegram каналів.

Дослідь:

1. Telethon SearchRequest можливості:
   - Як шукати канали по ключових словах?
   - Які метадані доступні (назва, опис, кількість підписників, мова)?
   - Є обмеження на кількість запитів?

2. Scoring алгоритм для candidate channels:
   Як оцінити якість каналу як crisis alert source?
   Фактори: subscriber count, post frequency, keyword density,
   verified status, language match.

3. Human-in-the-loop review:
   Знайдені канали → pending queue → admin review → approve/reject.
   Як зробити review зручним через Telegram (inline buttons)?

4. Community sourcing:
   Юзери можуть пропонувати канали через /suggest_channel @name.
   Як запобігти spam/abuse?

5. Automatic quality monitoring:
   Після додавання каналу — як відстежувати що він ще активний?
   Alert якщо канал не постив > 24h або змінив тематику.

Результат: channel_discovery.py module spec + admin review flow.
```

---

## Q11 — Production Deployment Architecture

```
Sharon Local запускається на Termux або home server (1GB RAM).
SharonGlobal потребує cloud deployment для 24/7 роботи з тисячами юзерів.

Дослідь оптимальну production архітектуру:

1. VPS вимоги для різних масштабів:
   - 100 active users: мінімальна конфігурація
   - 1000 active users: середня
   - 10000 active users: велика
   CPU, RAM, storage, bandwidth — конкретні цифри.

2. Docker Compose vs Kubernetes:
   Для якого масштабу Docker Compose достатньо?
   Коли переходити на K8s? (Не переускладнювати передчасно)

3. Database backup strategy:
   PostgreSQL для критичних даних (UserProfile, FamilyGroup).
   Як часто бекапити? Куди (S3/Backblaze)? Як швидко відновити?

4. Monitoring і alerting для самого бота:
   - Prometheus + Grafana vs простіше рішення
   - Які метрики критичні (message queue depth, DB connections, LLM latency)
   - Telegram алерт адміну якщо щось впало

5. Zero-downtime deployment:
   Як оновити бота без пропуску алертів під час деплою?

Результат: production docker-compose.yml + deployment runbook.
```

---

## Q12 — Monetization & Sustainability

```
SharonGlobal — безкоштовний публічний сервіс в зонах конфлікту.
Але сервер коштує гроші, підтримка потребує часу.

Дослідж стійкі моделі для подібних humanitarian tech проектів:

1. Аналог проекти:
   - Як фінансуються Alert-UA, @air_alert_ua та подібні сервіси?
   - Приклади open-source crisis bots з sustainable моделлю

2. Можливі моделі для SharonGlobal:
   a) Donations (Ko-fi, Patreon, Open Collective)
   b) Grant funding (EU humanitarian tech grants, USAID, Mozilla Foundation)
   c) Freemium (базове безкоштовно, premium: більший радіус, пріоритет)
   d) B2B: white-label для NGO/муніципалітетів
   e) Advertising (неприйнятно в crisis context — discuss)

3. Grant opportunities:
   Які конкретні гранти доступні для humanitarian crisis tech в 2025-2026?
   Умови, суми, дедлайни подачі.

4. Open source community building:
   Як залучити контрибуторів для додавання нових регіонів/каналів?
   GitHub-based contributor workflow.

5. Legal/ethical considerations:
   GDPR для EU юзерів (location data є sensitive).
   Мінімально необхідна документація (Privacy Policy, Terms).

Результат: sustainability roadmap + конкретні grant посилання.
```

---

## Q13 — Загальний системний дизайн (фінальний синтез)

```
Ти проаналізував всі аспекти SharonGlobal по окремих питаннях.

Тепер зроби фінальний синтезований документ:

**Architecture Decision Record (ADR) для SharonGlobal:**

1. ADR-001: aiogram 3 vs Telethon для bot handlers
2. ADR-002: PostgreSQL vs SQLite для multi-tenant data
3. ADR-003: Redis vs in-memory для dedup і rate limiting
4. ADR-004: LangGraph MemorySaver vs PostgresSaver
5. ADR-005: Single bot token vs multiple regional bots
6. ADR-006: Monolith vs microservices для початкового deploy

Для кожного ADR:
- Контекст проблеми
- Розглянуті альтернативи
- Прийняте рішення і чому
- Наслідки (trade-offs)

Також зроби:
- Sequence diagram: повний шлях від "канал постив загрозу" до "юзер отримав алерт"
- Component diagram: всі сервіси і їх зв'язки
- Data flow diagram: звідки береться інформація, як трансформується, куди іде

Результат: повний Technical Design Document для SharonGlobal v1.0.
```

---

## Порядок запуску в NotebookLM

1. **Завантаж джерела:**
   - Весь код `uav-watcher/` (особливо `uav_watcher.py`, `consultant/`, `family/`, `bot/`)
   - `consultant/knowledge/*.md` — всі файли KB
   - `docs/plans/2026-05-19-sharon-global-roadmap.md`
   - Будь-які статті/дослідження що маєш про crisis tech, Telegram bots

2. **Запускай Q1 → Q13 послідовно**, зберігаючи відповіді

3. **Q13 запускай в самому кінці** — він синтезує все попереднє

4. Готові відповіді збирай в `docs/research/` по одному файлу на питання
