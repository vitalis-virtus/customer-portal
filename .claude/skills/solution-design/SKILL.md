---
name: solution-design
description: Stage Skill стадії SOLUTION_DESIGN. Перетворює погоджену story-spec (і impact-analysis, якщо IMPACT_ANALYSIS застосовна) на solution-design — Domain, Application, Integration і Infrastructure Design у межах Architecture Foundation, з трасуванням AC, security-рішеннями, класифікацією залежностей і наміром змін інтерфейсу та даних — і повертає result envelope. Інтерв'ю не веде, API і схему даних не проєктує. Запускається лише вручну з готовими шляхами, версією і номером спроби.
argument-hint: "story=<ID> spec=<шлях> [impact=<шлях>] output=<шлях до solution-design v1.md> version=1 attempt=<N> [mode=pilot] [reason=\"<рішення людини після RESUME>\"]"
disable-model-invocation: true
user-invocable: true
---

# SOLUTION_DESIGN — Stage Skill

## 1. Purpose

Визначити загальну технічну структуру рішення Story в тому обсязі, який їй потрібен. Skill виконує стадію SOLUTION_DESIGN за `docs/workflow/stages.md` (розділ 8) і `docs/workflow/stage-map.yaml`. Ці файли й `docs/contracts/` авторитетні. Якщо вони розходяться з цим Skill, діють вони.

Стадія обов'язкова (`optional: false`, OD-01). Обсяг дизайну може бути мінімальним, але `NOT_APPLICABLE` вона не повертає.

**Межі стадії:**
- **API_DESIGN:** конкретні маршрути, методи, формати запиту й відповіді, коди статусів і помилок, заголовки. SOLUTION_DESIGN фіксує лише намір змін зовнішнього інтерфейсу (розділ 5, п. 8).
- **DB_DESIGN:** таблиці, колонки, типи, обмеження, індекси, ключі Redis, міграції. SOLUTION_DESIGN фіксує лише намір змін збережених даних (розділ 5, п. 9).
- Застосовність API_DESIGN і DB_DESIGN оцінюють самі ці стадії, а не SOLUTION_DESIGN.
- Код не пишеться. Імена типів і файлів — лише ілюстративні, з позначкою «наприклад».
- Вимоги не змінюються: продуктові рішення живуть у погодженій `story-spec`.

Результат перевіряє DESIGN_REVIEW (`reviewed_by`).

## 2. Inputs

Аргументи (`$ARGUMENTS`):

| Аргумент | Обов'язковий | Значення |
|---|---|---|
| `story` | так | ID Story, наприклад `US-003` |
| `spec` | так | Готовий шлях поточної погодженої версії `story-spec`. Її номер версії бери з front matter |
| `impact` | ні | Готовий шлях поточної версії `impact-analysis`. Його передає Orchestrator лише тоді, коли IMPACT_ANALYSIS застосовна. Якщо аргументу немає, стадія незастосовна: `impact-analysis` не шукай, не вимагай і не використовуй, навіть якщо в директорії Story є його файли (наприклад, історична версія після повторної оцінки `NOT_APPLICABLE`) |
| `output` | так | Готовий шлях файлу `solution-design`. Skill не обчислює шлях сам |
| `version` | так | Версія `solution-design`, збігається з `v<N>` в `output`. У цьому інкременті — лише `1` |
| `attempt` | так | Номер спроби стадії |
| `mode=pilot` | ні | Пілот поза зареєстрованим Workflow |
| `reason` | ні | Дослівна причина рішення людини `RESUME` після `CHANGES_REQUIRED` цієї стадії |

Людини в контексті немає: Skill нічого не питає. Якщо бракує обов'язкового аргументу або вхід не читається, verdict — `BLOCKED` (розділ 6).

`reason` — це рішення людини щодо попереднього Loopback, а не джерело вимог. Воно може пояснити, чому людина відхилила повернення. Нових вимог воно не додає.

Логічні входи — `docs/contracts/stage-io-map.yaml` → `SOLUTION_DESIGN`, шляхи — `docs/contracts/artifact-registry.yaml`:
- `story-spec` — `REQ`, `AC` з Verification Method, scope, Story Type, «Відкрито для Technical Design»;
- `impact-analysis` (лише якщо передано) — зачеплені області, вплив, ризики регресії, «Відкрито для Solution Design»;
- `architecture-foundation`:
  - мінімум `docs/architecture/architecture.md` (стиль, модулі, наскрізні аспекти, інваріанти AR-) і `docs/architecture/package-map.md` (модулі, шари, порти, імпорти);
  - за потреби — `api-conventions.md`, `persistence-conventions.md` (зокрема §10 Consistency Analysis), `security-conventions.md`, лише розділи, яких стосується Story.

За потреби (`stages.md`, SOLUTION_DESIGN → «За потреби»):
- наявний код зачеплених модулів — конкретні шляхи;
- документи Product Foundation, на правила яких (BR-, NFR-) прямо посилається `story-spec`, — лише щоб зрозуміти зміст цих правил і розмістити їх у дизайні. Це не нове джерело вимог.

Історія чату і `clarification-record` не є входами. Читай лише потрібні документи (Progressive Context Loading).

## 3. Procedure

1. **Перевір аргументи.**
   - У `mode=pilot` `output` не може лежати в `docs/stories/`.
   - `version` має бути `1`, інакше розділ 6.
   - Якщо файл `output` уже існує, verdict — `BLOCKED`. Файл не перезаписуй.
2. **Прочитай `story-spec`.** Виділи `REQ`, `AC`, scope, Story Type, розділ «Відкрито для Technical Design» і правила Foundation, на які він посилається.
3. **Якщо передано `impact`, прочитай `impact-analysis`.** Кожна зачеплена область, ризик регресії і пункт «Відкрито для Solution Design» має отримати місце в дизайні або явне обґрунтування, чому дизайн його не зачіпає.
4. **Прочитай Foundation** (розділ 2): мінімум, далі — лише потрібні розділи конвенцій.
5. **Спроєктуй рішення** за розділом 3.1.
6. **Перевір межі** (розділ 3.2) і критерії якості (розділ 4).
7. **Визнач verdict** (розділ 6). Поки verdict не визначено, тримай дизайн у пам'яті, а не у файлі.
8. **Лише для `PASS` запиши `solution-design`** в `output` (розділ 5). Для інших verdicts файл не створюй навіть частково: незареєстрований Working Draft зупинить наступну спробу.
9. **Поверни envelope** (розділ 6).

### 3.1. Рівні дизайну

Для кожного рівня — рішення в обсязі Story або позначка «не зачеплено» з коротким обґрунтуванням (`stages.md`, OD-01):

- **Domain** — сутності, value objects, інваріанти; у якому модулі й шарі живе кожне бізнес-правило (`architecture.md` §4, «Розміщення бізнес-правил»).
- **Application** — use cases з входом, результатом і категоріями помилок; порти; межі транзакцій; узгодженість даних між сховищами; хто діє і як перевіряється право на дію.
- **Integration** — взаємодія модулів через публічні контракти (`package-map.md` §6), зовнішні системи, напрямки імпортів (§7).
- **Infrastructure** — конфігурація, середовища, runtime-компоненти, composition root, запуск і відкат, якщо Story їх змінює.

Посилайся на правила Foundation за ID чи розділом. Не переписуй їх.

### 3.2. Межі з API_DESIGN і DB_DESIGN

| SOLUTION_DESIGN | Не SOLUTION_DESIGN |
|---|---|
| які use cases змінюють видиму клієнту поведінку; категорії помилок use case | маршрути, методи, DTO, коди статусів, Problem Details, заголовки → API_DESIGN |
| які дані зберігаються, змінюються чи читаються і в якому сховищі; межі транзакцій; узгодженість | таблиці, колонки, типи, обмеження, індекси, ключі Redis, TTL, міграції → DB_DESIGN |
| розміщення в модулях, шарах і портах | код, імена файлів і типів, крім ілюстративних |

Якщо рішення неможливо ухвалити без деталей API чи схеми, винеси питання в «Відкрито для API_DESIGN і DB_DESIGN». Не вирішуй його тут.

### 3.3. Security і залежності

- **Security на рівні архітектури** — для кожної Story, незалежно від того, чи буде застосовна SECURITY_REVIEW (OD-05). Описуються автентифікація, авторизація й «хто діє», межі довіри, чутливі дані в логах і відповідях, сесії й облікові дані, якщо Story їх зачіпає. Підстава — посилання на `security-conventions.md` і AR-. Обов'язкових правил не послаблюй (RR-14).
- **Залежності** (`workflow-policies.md` §2, Risk-Based Dependency Approval, OD-09). Кожна нова чи змінена залежність отримує класифікацію за критеріями OD-09:
  - проста допоміжна залежність, що не відповідає жодному критерію, — пропонується в дизайні з обґрунтуванням;
  - суттєва залежність, а також будь-яка, класифікація якої неоднозначна, — `BLOCKED` (розділ 6);
  - для Customer Portal діє AR-012: нова технологія чи заміна позиції `architecture.md` §6 потребує оновлення Foundation — `BLOCKED` (Foundation Change Required);
  - якщо нових залежностей немає, так і напиши.
- **Аналізи, яких вимагає Foundation.** Наприклад, Consistency Analysis для кожного use case, що змінює і PostgreSQL, і Redis (`persistence-conventions.md` §10), за всіма пунктами його таблиці. Якщо таких use cases немає, так і напиши.

## 4. Quality Criteria

- Кожен `AC` має місце в дизайні: use case, правило чи компонент, що його реалізує (`stages.md`, Completion Conditions).
- Якщо передано `impact-analysis`, кожну його зачеплену область і кожен пункт «Відкрито для Solution Design» розглянуто.
- Кожен рівень (Domain, Application, Integration, Infrastructure) має рішення або обґрунтоване «не зачеплено».
- Рішення не порушують Architecture Rules і інваріантів AR-, напрямків імпортів і правил розміщення бізнес-правил.
- Security-рішення описано. Кожну залежність класифіковано. Аналізи, яких вимагає Foundation, виконано.
- Намір змін зовнішнього інтерфейсу і збережених даних сформульовано явно: «так» або «ні», з посиланням на `REQ`/`AC`.
- Деталей API_DESIGN і DB_DESIGN немає.
- Немає вимог без джерела в `story-spec`. Відкрите продуктове питання не вирішено мовчки.
- Кожне джерело в `inputs` — конкретний шлях з актуальним `commit`.

## 5. Output

Один файл `solution-design` за шляхом `output`, мова — українська.

Front matter — поля `docs/contracts/artifact-rules.md` §1, і лише вони:
- `artifact_type: solution-design`, `story_id`, `version`;
- `produced_by: { stage: SOLUTION_DESIGN, attempt }`;
- `created_at` — фактичний UTC з `date -u +%Y-%m-%dT%H:%M:%SZ`;
- `inputs`:
  - `{ type: story-spec, version }`;
  - `{ type: impact-analysis, version }` — лише якщо передано `impact`;
  - кожен прочитаний документ Foundation і шлях коду як `{ source: architecture-foundation | product-foundation | codebase, path, commit }`, де `commit` — `git log -1 --format=%h -- <path>`.

**Правила посилань на джерела (D-3):**
- лише конкретні шляхи, ніколи корінь репозиторію `.`;
- шлях без історії в Git до `inputs` не потрапляє, бо `commit` немає. Його фіксують у «Обмеженнях»;
- для прочитаного шляху з незакоміченими змінами (`git status --porcelain -- <path>`) дизайн спирається на фактичний вміст робочого дерева, а обмеження описується в тілі.

Статусів, Currency, approval і позначки пілоту у front matter немає.

Розділи тіла, коротко і пропорційно Story:

1. **Предмет** — версії `story-spec` і `impact-analysis` (якщо є), прочитані джерела.
2. **Domain Design** — розділ 3.1.
3. **Application Design** — use cases, порти, межі транзакцій, узгодженість, категорії помилок.
4. **Integration Design** — модулі, публічні контракти, імпорти, зовнішні системи.
5. **Infrastructure Design** — або «не зачеплено» з обґрунтуванням.
6. **Трасування AC** — таблиця `AC-NN` → `REQ` → елемент дизайну.
7. **Security** — розділ 3.3.
8. **Зміни зовнішнього інтерфейсу** — так чи ні; які use cases додають чи змінюють операції або видиму поведінку, з `REQ`/`AC`. Без маршрутів і форматів.
9. **Зміни збережених даних** — так чи ні; які дані, в якому сховищі, створення, зміна чи лише читання, з `REQ`/`AC`. Без схеми і міграцій.
10. **Залежності** — класифікація за OD-09 або «нових залежностей немає».
11. **Аналізи, яких вимагає Foundation** — наприклад Consistency Analysis, або «не потрібні» з причиною.
12. **Опрацювання `impact-analysis`** — лише якщо його передано: як враховано кожну область і кожен пункт «Відкрито для Solution Design».
13. **Відкрито для API_DESIGN і DB_DESIGN** — питання, які стадія свідомо не вирішує.
14. **Обмеження** — незакомічені чи невідстежувані шляхи, межі видимості.
15. **Рішення людини** — лише якщо передано `reason`: дослівна цитата і як вона вплинула на дизайн. Рішення людини фіксується в канонічному артефакті, а не лише в History (`workflow-policies.md` §2).

У `mode=pilot` перший рядок тіла: «Пілотний артефакт поза офіційним зареєстрованим Workflow. Не є офіційно погодженим і не є входом стадій зареєстрованих Stories; може бути входом наступних стадій того самого пілоту».

## 6. Verdict and Return

SOLUTION_DESIGN має `optional: false` і такі loopbacks:
- `IMPACT_ANALYSIS` — без `crosses_gates`;
- `SPECIFICATION` і `CLARIFICATION` — з `crosses_gates: [HUMAN_SPEC_APPROVAL]`.

Тому допустимі **`PASS`**, **`CHANGES_REQUIRED`** і **`BLOCKED`**. `NOT_APPLICABLE` недопустимий.

| Ситуація | Verdict | `outputs` |
|---|---|---|
| Бракує аргументу, `version` ≠ `1`, вхід не читається або `output` уже існує | `BLOCKED` до проєктування. Файлу немає | `[]` |
| Дизайн завершено, критерії розділу 4 виконано | `PASS` | `[{ type: solution-design, version }]` |
| `impact-analysis` неповний чи хибний (пропущено зачеплену область, ризик або міграцію), або `impact` не передано, а дизайн показує, що зачіпає наявну поведінку, контракт, дані чи спільні компоненти (тригер IA-01…IA-04) | `CHANGES_REQUIRED`, `loopback_target: IMPACT_ANALYSIS` | `[]` |
| `story-spec` нездійсненна, суперечлива чи неповна для дизайну, і це усувається в `story-spec` без нового рішення людини | `CHANGES_REQUIRED`, `loopback_target: SPECIFICATION` | `[]` |
| Для дизайну потрібне нове чи інше продуктове рішення людини | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` | `[]` |
| Story потребує зміни Architecture чи Product Foundation, зокрема нової технології (AR-012) | `BLOCKED` (Foundation Change Required) | `[]` |
| Суттєва чи неоднозначна нова залежність (OD-09) або інше рішення з переліку ескалацій (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) | `[]` |
| Рішення неможливе без послаблення обов'язкового правила Foundation, зокрема Security Rules | `BLOCKED` (Foundation Rule Violation) | `[]` |

Правила:

- Якщо підходить кілька цілей, обери найвищу за течією: CLARIFICATION, далі SPECIFICATION, далі IMPACT_ANALYSIS. Після повернення Story однаково пройде наступні стадії (RR-06). У `summary` перелічи всі виявлені проблеми.
- `loopback_target` — лише пропозиція. Маршрут, підтвердження людини для цілей з `crosses_gates` (RR-04) і повторний вхід цілі належать Orchestrator. Skill їх не виконує й не імітує.
- Для `CHANGES_REQUIRED` findings передаються в `summary`: коротко кожна проблема з посиланням на `REQ`/`AC`, область `impact-analysis` чи правило Foundation. ID `F-NNN` не присвоюй, це робить review.
- Для `BLOCKED` потрібне опиши в `blocker.needed`. `foundation-change-request` Skill не створює.
- У `mode=pilot` додай у `summary` позначку «pilot».

Envelope — формат `docs/contracts/workflow-state.md` §6, без змін і без додаткових полів:

```yaml
result:
  story_id: US-003
  stage: SOLUTION_DESIGN
  attempt: 1
  verdict: PASS                      # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { type: solution-design, version: 1 }
  loopback_target: null              # IMPACT_ANALYSIS | SPECIFICATION | CLARIFICATION лише для CHANGES_REQUIRED
  blocker: null                      # для BLOCKED: { reason, details, needed }
  summary: "2 use cases, 13/13 AC; інтерфейс: так; дані: лише читання; нових залежностей немає; pilot"
```

## 7. Constraints

- Пиши лише файл `output` і лише для `PASS`.
- Не змінюй `story-spec`, `impact-analysis`, `clarification-record`, код, Foundation, Workflow Definition, `docs/contracts/` та інші артефакти.
- Не змінюй `state.yaml` і `history.jsonl` будь-якої Story, а також статуси, Currency чи approvals.
- Не обчислюй шлях чи версію: бери їх з аргументів.
- Bash — лише `git log`, `git status`, `git ls-files` і `date`.
- Не веди інтерв'ю, не питай людину і не ухвалюй продуктових рішень.
- Не проєктуй API і схему даних, не пиши код.
- Не виконуй Loopback, не проходь Human Gate, не імітуй Orchestrator, DESIGN_REVIEW чи рішень людини. Не запускай інші стадії, Skills чи subagents.
- Не роби `git commit`, `git push` чи деструктивних операцій.
