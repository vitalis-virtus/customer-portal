# Artifact Contracts — Simplified MVP

**Статус:** `APPROVED (MVP)` — Phase 4, Simplified Artifact Contracts MVP. Людина-архітектор погодила дизайн 2026-11-17.

Пакет визначає, де лежать артефакти Story, які метадані вони мають, як зберігаються стан і рішення людини, і що кожна з 22 стадій читає та створює. Мета — практична основа для першої реальної Story. Пакет не реалізує Stage Skills, Orchestrator чи валідацію.

**Межі погодження.** Погоджено лише дизайн MVP-контрактів, тобто п'ять файлів цього пакета. Погодження не означає, що реалізовано розширені гарантії Workflow Definition. Відкладене й консервативно спрощене перелічено в розділі «Deferred Advanced Features / Workflow Compatibility». Workflow Definition лишається чинною в повному обсязі: жодна Story не може досягти `DONE`, оминувши її обов'язкові вимоги.

Пакет замінив попередню, надмірно складну редакцію Phase 4 (JSON Schema, catalog, fixtures, Durable Transition Protocol, Git identity profiles). Ця редакція ніколи не погоджувалась як Phase 4 і не є канонічною. Її повну копію збережено поза репозиторієм разом з маніфестом SHA-256.

## Файли

| Файл | Що містить |
|---|---|
| [artifact-registry.yaml](artifact-registry.yaml) | Єдине джерело шляхів: типи артефактів, категорії, стадії-власники, gate, де тип погоджується |
| [artifact-rules.md](artifact-rules.md) | Метадані документів і записів, версії, Staleness, findings, хто що пише |
| [workflow-state.md](workflow-state.md) | `state.yaml`, `history.jsonl`, approvals, result envelope стадії |
| [stage-io-map.yaml](stage-io-map.yaml) | Входи й виходи кожної з 22 стадій |

Маршрути, verdicts, loopbacks, gates і Approval Scope визначає лише погоджена Workflow Definition ([stage-map.yaml](../workflow/stage-map.yaml)). Пакет їх не повторює і не змінює.

## Як це працює

1. Стадія бере шляхи входів з реєстру, читає поточні версії, записує новий файл (`v<N>.md` або запис `<NNNN>.md`) і повертає result envelope.
2. Orchestrator перевіряє результат, реєструє файли в `state.yaml`, обирає маршрут зі `stage-map.yaml` і дописує подію в `history.jsonl`.
3. На Human Gate Orchestrator зупиняється, показує людині пакет рішення і записує її рішення: хто, що, коли, які версії.
4. Статуси (Version Status, Currency, Approval State) живуть лише в `state.yaml`. Файли артефактів після реєстрації не змінюються.

## Storage

MVP-розміщення:

```
docs/stories/<story_id>/
├── state.yaml
├── history.jsonl
├── artifacts/<artifact_type>/v<N>.md
└── records/<record_type>/<NNNN>.md
docs/backlog/<entry_id>.md          # рівень проєкту
```

Дані Story комітяться у feature branch цієї Story разом з кодом. Git дає історію змін і незмінний слід.

Це **запропоноване MVP-спрощення**, а не погоджена заміна політики ізольованого storage root (рішення D-01 старого пакета). Обмеження:

- **Перемикання гілок.** Стан Story видно лише у feature branch цієї Story. На іншій гілці файлів Story немає або вони інші. Orchestrator працює зі Story лише на її гілці.
- **Кілька Stories одночасно.** Кожна Story має свою гілку й директорію. Паралельна робота — через окремі гілки чи `git worktree`. Спільний огляд усіх Stories з однієї гілки недоступний.
- **Дані Story потрапляють у PR.** Review коду їх не враховує (`workflow-state.md`, розділ 8).
- **Стан після merge — відоме нерозв'язане обмеження.** Після HUMAN_MERGE оновлення стану Story (`state.yaml`, `history.jsonl`, `integration-record`, `post-merge-validation-record`, перехід у `DONE` чи `BLOCKED`) з'являються вже після того, як feature branch інтегровано. Поточний MVP не визначає, як ці оновлення потрапляють і зберігаються в target branch. Це завдання майбутньої реалізації Orchestrator. Структура Story від цього не змінюється.
- **Коміти.** [AGENTS.md](../../AGENTS.md) забороняє коміти без явного дозволу. Хто комітить дані Story (людина чи агент з дозволом), треба вирішити до першої Story.

## Deferred Advanced Features / Workflow Compatibility

MVP не реалізує всіх правил погодженої Workflow Definition. Жоден Human Gate не прибрано і не послаблено: кожен gate вимагає рішення людини.

### Підтримано повністю

- Усі 22 стадії, маршрути й gates — за незмінним `stage-map.yaml`.
- Власники артефактів, незмінні версії, нова версія для виправлення, залежності від конкретних версій.
- Staleness контрольованих артефактів, Revalidation через `revalidation-record`.
- Review PASS ≠ Human Approval. Рішення людини фіксує, хто, що, коли і які версії.
- APPROVE, REQUEST_CHANGES, CANCEL, CONFIRM_MERGED. Підтвердження Loopback через gate, дії Recovery.
- Optional Stages з `applicability-decision`, `NOT_APPLICABLE` лише для `optional: true`.
- Retry Counters (Technical Attempts, Quality Loops, Repeated Failure), Cancellation Protocol, Step Mode і Auto Mode.

### Підтримано консервативно або вручну

| Вимога | MVP | Наслідок |
|---|---|---|
| Approved Content Identity | Версія документа, `record_id` запису, `changeset_commit` для changeset | Без hash. Цілісність тримається на незмінності файлів і Git |
| Review Binding Guard (RR-10) | Review прив'язано до версії changeset. Після `changeset_commit` дозволено змінювати лише файли Story | Суворіше: будь-який rebase чи squash вимагає нового IMPLEMENTATION і review |
| `CHANGESET_CONTENT_UNCHANGED` | `changeset_commit` не змінився | Суворіше |
| Staleness Foundation | Залежність від git commit файлу чи каталогу | Грубо: будь-яка зміна файлу робить залежні версії `STALE` |
| Applicability Basis (AP-04, AP-08) | `basis` записано в рішенні, переоцінка — на кожному повторному вході | Суворіше, але більше роботи |
| Finding Identity, Repeated Failure | `F-NNN` присвоює reviewer, збіг визначає LLM | Менш детерміновано |
| Out-of-Stage Revalidation | Не підтримано окремо. Людина обирає `ROUTE_BACK` | Суворіше й дорожче |
| Session Recovery, цілісність | Розбіжність `state.yaml`, History і файлів → `BLOCKED` (Integrity Error), вирішує людина | Без автоматичного відновлення |
| MERGE_READINESS_CHECK | Порівняння commits: між `changeset_commit` і head PR змінено лише файли Story, target branch = `verified_target_base` (`workflow-state.md`, розділ 8) | Достатньо для типового випадку |
| SUSPENDED → VALID | `revalidation-record` повертає Currency `CURRENT`, але approval лишається `SUSPENDED`, доки людина не ухвалить APPROVE | Суворіше: замість RECONFIRM — повне APPROVE |

### Відкладено. Потрібне погоджене спрощення Workflow

| Можливість | Що вимагає Workflow | Що робить MVP |
|---|---|---|
| **RECONFIRM** | Gate decision RECONFIRM (OD-06) | Не пропонується. Людина обирає APPROVE. Обмеження вибору людини, тож потребує погодження |
| **Cryptographic Changeset Content Identity** | Перебудова commits без зміни змісту не змінює identity (`workflow-policies.md`, розділ 9) | Identity = commit. Розбіжність із формулюванням розділу 9 |
| **Full Git Integration Verification** | Доказ, що merge вніс саме погоджені зміни і нічого іншого (Integration Match), і Post-Merge Validation (OD-12 крок 7, OD-16, OD-17) | Artifact Contracts цю перевірку **не реалізують**. Вони задають лише мінімальні Git-метадані: `feature_branch`, `base_commit`, `changeset_commit`, `verified_target_base`, `merge_commit`, `target_before_merge`. Без погодженого способу перевірки INTEGRATION_CONFIRMATION не повертає `PASS`, а Story лишається в `BLOCKED` до рішення людини |
| **Durable Transition / Crash Recovery** | Детерміноване відновлення узгодженості State і History | Немає. Розбіжність розв'язує людина |
| **Advanced contract validation** | Детермінована перевірка контрактів, версіоновані контракти, закріплення версій | Немає валідатора і version pinning. Правила виконують Orchestrator і review за інструкціями |

**DONE.** MVP не позначає Story як `DONE`, доки не виконано вимоги інтеграції погодженого Workflow: Integration Match `PASS`, Post-Merge Validation `PASS` (якщо target branch змінилася) і всі умови DoD (`stage-map.yaml` RR-09, `terminal_states.DONE`). Наявність `integration-record` чи атестація людини сама по собі цього не доводить.

### Передумови першої Story (не залежать від цього пакета)

- **FG-01** — Testing Strategy. Без неї перша реальна Story не реєструється (`workflow-policies.md`, розділ 15).
- **PRQ-02** — Merge Protection Configuration до першого HUMAN_MERGE.
- **PRQ-01** — канонічний Backlog. Реєстр визначає шлях `docs/backlog/`, але потрібне погодження, що це і є канонічний Backlog.

## Що запозичено з реалізації ментора

- Реєстр як єдине джерело шляхів і власників.
- Stage I/O через логічні ключі, а не шляхи.
- Result envelope: Skill не змінює стан.
- Простий YAML-стан і JSONL-журнал.

Відмінності від ментора: окремий файл на кожну версію (у ментора один шлях на тип), статуси лише в стані (у ментора — у front matter), дані окремо для кожної Story, збереження того, хто ухвалив рішення.
