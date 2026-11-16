# Stages — Workflow Definition

**Статус документа:** `APPROVED` (revision 3). Погоджено людиною 2026-11-16 після фінального Review.

## 1. Призначення

Документ пояснює кожну стадію Story Lifecycle: навіщо вона існує, що потребує на вході, що створює і коли вважається завершеною.

Правила маршрутизації тут не повторюються. Порядок стадій, `next`, Loopbacks, рішення Human Gates, Optional Stages і термінальні стани визначає [stage-map.yaml](stage-map.yaml). Якщо цей документ і `stage-map.yaml` розходяться, канонічним є `stage-map.yaml`, а розбіжність треба повідомити.

Пов'язані документи:

- [workflow-state-model.md](workflow-state-model.md) — Workflow State, Execution State, Stage Verdict, Artifact Status, Recovery, Retry, Staleness, History.
- [workflow-policies.md](workflow-policies.md) — Human Gates, Reviews, Controlled Autonomy, Optional Stages, Findings, Definition of Done, Git.

### Що не входить

- Схеми артефактів, front matter, структура Markdown і шляхи до файлів. Їх визначає етап Artifact Contracts. Тут артефакти названо лише логічними ідентифікаторами.
- Інструкції виконання стадій. Їх визначають майбутні Stage Skills.
- Технологічні правила (мова, фреймворки, сховища). Вони належать Project Foundation.

### Маркери походження

Позначки `APPROVED_DECISION` і `DERIVED` мають те саме значення, що й у `stage-map.yaml`: `APPROVED_DECISION` прямо випливає з Q1–Q28, OD-01…OD-17, погоджених Review Findings або рішень фінального Review FR-01…FR-03; `DERIVED` випливає з погоджених рішень, але не погоджувався окремо. У погодженій версії `DERIVED` елементів немає (FR-01). Рішення OD описано в [workflow-policies.md](workflow-policies.md#14-рішення-od).

## 2. Джерела контексту

Workflow Core посилається на логічні джерела. Для Customer Portal вони прив'язані так:

| Логічне джерело | Customer Portal |
|---|---|
| Product Foundation | [docs/product/](../product/) — Product Vision, Personas, Business Glossary, Business Rules, Non-Functional Requirements, Epic Map |
| Architecture Foundation | [docs/architecture/](../architecture/) — Architecture, Package Map, API, Persistence і Security Conventions |
| Workflow Definition | [docs/workflow/](./) |

Кожна стадія має **мінімальний набір джерел** (поле «Входи»). Інші документи завантажуються лише тоді, коли без них стадія не може виконати свою відповідальність (Progressive Context Loading). Історія чату не є входом жодної стадії: стадії отримують контекст лише з канонічних артефактів і Workflow State.

## 3. Загальні правила для всіх стадій

Ці правила діють для кожної стадії, тому в описах стадій вони не повторюються.

**Загальні передумови:**

- Story перебуває на цій стадії згідно з Workflow State.
- Усі вхідні контрольовані артефакти існують, мають Currency `CURRENT` і не є `SUPERSEDED` ([workflow-state-model.md](workflow-state-model.md#5-artifact-status)). Виняток — процедура Revalidation: вона може читати `STALE` версії, щоб оцінити вплив змін, але її результатом є лише `revalidation-record` або нова версія.
- Approvals, на які спирається стадія, мають стан `VALID`.
- Для стадій після CODE_REVIEW виконується Review Binding Guard (`stage-map.yaml` RR-10).
- Немає відкритих питань Foundation, позначених як блокувальні для цієї Story.

**Загальна обробка збоїв:**

- Бракує інформації, яку може дати лише людина, або потрібне рішення з переліку ескалацій → `BLOCKED` з причиною.
- Story потребує зміни Product або Architecture Foundation → `BLOCKED` з причиною Foundation Change Required. Далі діє Foundation Change Request ([workflow-policies.md](workflow-policies.md)).
- Проблема в upstream-артефакті → `CHANGES_REQUIRED` з пропозицією Loopback на одну з дозволених цілей. Стадія не змінює upstream-артефакт сама.
- Перервана сесія чи технічний збій не є Stage Verdict. Їх обробляє Execution State.

**Загальні умови завершення:**

- Стадія повернула один Stage Verdict з обґрунтуванням.
- Для `PASS` створено всі обов'язкові виходи стадії. Нові версії контрольованих артефактів мають зафіксовані залежності від версій upstream-артефактів.
- Для `NOT_APPLICABLE` створено `applicability-decision` за умовою застосовності зі `stage-map.yaml`.
- Під час повторного входу після повернення стадія виконує Revalidation: створює `revalidation-record` для незмінної версії або нову версію (Rework Protocol, [workflow-state-model.md](workflow-state-model.md#14-rework-protocol)).

**Відповідальність Review:** producer-стадія не перевіряє сама себе як остаточну перевірку. Її результат перевіряє стадія, указана в `reviewed_by` у `stage-map.yaml`. Правила незалежності — у [workflow-policies.md](workflow-policies.md).

## 4. Огляд стадій

| # | Stage ID | Фаза | Тип | Optional | Походження |
|---|---|---|---|---|---|
| 1 | CLARIFICATION | CLARIFICATION | producer | ні | Q1 |
| 2 | SPECIFICATION | SPECIFICATION | producer | ні | Q1 |
| 3 | SPEC_REVIEW | SPECIFICATION | review | ні | Q6, Q28, OD-03 |
| 4 | HUMAN_SPEC_APPROVAL | SPECIFICATION | human_gate | — | Q5, OD-06 |
| 5 | IMPACT_ANALYSIS | TECHNICAL_DESIGN | producer | так | Q26, OD-07 |
| 6 | SOLUTION_DESIGN | TECHNICAL_DESIGN | producer | ні | OD-01 |
| 7 | API_DESIGN | TECHNICAL_DESIGN | producer | так | Q2, OD-07 |
| 8 | DB_DESIGN | TECHNICAL_DESIGN | producer | так | Q2, OD-07 |
| 9 | DESIGN_REVIEW | TECHNICAL_DESIGN | review | ні | Q2, OD-05 |
| 10 | IMPLEMENTATION_PLANNING | IMPLEMENTATION_PLANNING | producer | ні | Q1, OD-15 |
| 11 | PLAN_REVIEW | IMPLEMENTATION_PLANNING | review | ні | Q6, Q28, OD-04 |
| 12 | HUMAN_TECHNICAL_APPROVAL | IMPLEMENTATION_PLANNING | human_gate | — | Q5, OD-06 |
| 13 | IMPLEMENTATION | IMPLEMENTATION | producer | ні | Q1, Q3 |
| 14 | CODE_REVIEW | IMPLEMENTATION | review | ні | Q6, Q28, FR-01 |
| 15 | SECURITY_REVIEW | VERIFICATION | review | так | Q22, OD-05 |
| 16 | RECONCILIATION | VERIFICATION | verification | так | Q27 |
| 17 | VERIFICATION | VERIFICATION | verification | ні | Q1, Q23, OD-15 |
| 18 | PR_PREPARATION | DELIVERY | producer | ні | Q8, FR-01 |
| 19 | HUMAN_PR_APPROVAL | DELIVERY | human_gate | — | Q5, OD-06 |
| 20 | MERGE_READINESS_CHECK | DELIVERY | check | ні | OD-12, OD-16, FR-01, FR-02 |
| 21 | HUMAN_MERGE | DELIVERY | human_action | — | Q5, Q8, OD-12 |
| 22 | INTEGRATION_CONFIRMATION | DELIVERY | check | ні | Q4, Q8, OD-11, OD-12, OD-16, OD-17 |

Фаза `DELIVERY` погоджена (OD-02): вона відокремлює технічну готовність Story від інтеграції змін.

## 5. Логічні артефакти

Артефакти названо логічними ідентифікаторами. Їхню структуру визначає етап Artifact Contracts. Класи артефактів, Approved Content Identity, Version Status і Currency пояснено в [workflow-state-model.md](workflow-state-model.md#5-artifact-status). Supporting-артефакти — незмінні записи: виправлення створює новий запис.

| Артефакт | Клас | Створює | Призначення |
|---|---|---|---|
| `clarification-record` | controlled | CLARIFICATION | Питання, відповіді й рішення людини щодо Story |
| `story-spec` | controlled | SPECIFICATION | Специфікація Story: Story Type, Acceptance Criteria і Verification Method |
| `impact-analysis` | controlled | IMPACT_ANALYSIS | Вплив Story на наявні поведінку, контракти, дані й інші Stories |
| `solution-design` | controlled | SOLUTION_DESIGN | Загальна технічна структура рішення: Domain, Application, Integration, Infrastructure Design у потрібному обсязі |
| `api-design` | controlled | API_DESIGN | Зміни зовнішнього інтерфейсу |
| `db-design` | controlled | DB_DESIGN | Зміни схеми та структури збережених даних |
| `implementation-plan` | controlled | IMPLEMENTATION_PLANNING | Кроки реалізації, Story Risk Assessment, Testing Approach, заплановані Optional Stages |
| `implementation-changeset` | controlled | лише IMPLEMENTATION | Зміни коду й тестів у feature branch. Ідентифікується Changeset Content Identity |
| `foundation-change-request` | controlled | будь-яка стадія через BLOCKED | Запит на зміну Product або Architecture Foundation |
| `review-report` | supporting | review-стадії | Findings і verdict щодо конкретних версій артефактів і upstream-версій |
| `revalidation-record` | supporting | стадія-власник артефакта | Підтвердження, що незмінна версія актуальна щодо нових upstream-версій |
| `applicability-decision` | supporting | Optional Stage | Оцінка тригерів застосовності, зокрема обґрунтування `NOT_APPLICABLE` |
| `reconciliation-report` | supporting | RECONCILIATION | Traceability Acceptance Criteria → дизайн → changeset → тести |
| `verification-report` | supporting | VERIFICATION | Фактичні результати перевірок, Findings Disposition, стан DoD, Target Base Identity |
| `pull-request` | supporting | PR_PREPARATION | PR із feature branch до target branch. Identity без метаданих |
| `integration-record` | supporting | INTEGRATION_CONFIRMATION | Підтвердження фактичної інтеграції погоджених змін |
| `post-merge-validation-record` | supporting | INTEGRATION_CONFIRMATION | Результат Post-Merge Validation: перевірений стан target branch, виконані перевірки, діагностика у разі невдачі (OD-16). Не входить до Approval Scope |
| `risk-acceptance-record` | supporting | рішення людини за ескалацією | Прийнятий людиною ризик без порушення обов'язкових правил Foundation (WF-V2-01) |
| `foundation-exception-record` | supporting | рішення людини за процедурою Foundation | Контрольований виняток, лише якщо правило Foundation прямо його допускає (WF-V2-01) |
| `cancellation-record` | supporting | Orchestrator під час CANCEL | Причина й стан Story на момент скасування, рішення щодо ресурсів |
| `backlog-entry` | project-level | INTEGRATION_CONFIRMATION або Cancellation Protocol | Запис канонічного Backlog для погодженого FOLLOW_UP з посиланням на Story і Finding |

## 6. Стадії фази CLARIFICATION

### CLARIFICATION

**Purpose.** Усунути неоднозначності Story до специфікації: зрозуміти намір, межі, правила й обмеження, які на неї діють.

**Preconditions.** Story зареєстрована в джерелі, яке відповідає її типу ([workflow-policies.md](workflow-policies.md#16-story-types-od-14)): FEATURE — у Product Foundation (для Customer Portal — Candidate Story в Epic Map) або в Backlog; інші типи — у Backlog. Продуктові залежності, які Product Foundation позначає як такі, що мають бути вирішені до Clarification цієї Story, вирішені. Передумови з [workflow-policies.md](workflow-policies.md#15-prerequisites-і-foundation-gaps) виконані.

**Inputs.**
- Мінімальні: опис Story з її джерела; Business Glossary; Business Rules і NFR, пов'язані зі Story. Для TECHNICAL та INFRASTRUCTURE — відповідні рішення й обмеження Architecture Foundation.
- За потреби: Product Vision, Personas; Architecture Foundation — щоб виявити технічні обмеження, які впливають на вимоги.
- Під час повторного входу: попередня версія `clarification-record` і причина повернення.

**Responsibilities.**
- Запропонувати Story Type і уточнити зміст, потрібний для цього типу (OD-14). Для технічних типів не створювати штучну User Persona.
- Сформулювати питання до людини. Не вигадувати відповіді й бізнес-вимоги.
- Зафіксувати відповіді й рішення людини в канонічному артефакті, а не лише в History.
- Виявити суперечності з Foundation і відкриті питання, які блокують Story.

**Expected Outputs.** `clarification-record`.

**Completion Conditions.** Усі питання, без яких неможлива специфікація, мають відповідь людини або явно позначені як рішення людини відкласти їх із визначеним впливом на scope.

**Review Responsibilities.** Окремої review-стадії немає (OD-03). `clarification-record` перевіряє SPEC_REVIEW разом зі специфікацією.

**Optionality.** Обов'язкова.

**Failure Handling.** Очікування відповідей людини — це стан очікування, а не `BLOCKED` ([workflow-state-model.md](workflow-state-model.md#3-execution-state)). Суперечність із Foundation → `BLOCKED` (Foundation Change Required або уточнення Foundation людиною).

## 7. Стадії фази SPECIFICATION

### SPECIFICATION

**Purpose.** Сформувати перевірювану специфікацію Story.

**Preconditions.** `clarification-record` має актуальну версію.

**Inputs.**
- Мінімальні: `clarification-record`; опис Story; Business Glossary; пов'язані Business Rules і NFR.
- За потреби: інші документи Product Foundation.

**Responsibilities.**
- Зафіксувати Story Type і зміст, який для нього обов'язковий ([workflow-policies.md](workflow-policies.md#16-story-types-od-14)). Для технічних типів — Technical Goal, Scope і Constraints.
- Описати поведінку або технічний результат, scope і поза-scope Story.
- Сформулювати Acceptance Criteria так, щоб кожен критерій можна було перевірити, і для кожного вказати Verification Method.
- Посилатися на правила Foundation, а не переписувати їх.
- Не ухвалювати технічних рішень, які належать Technical Design.

**Expected Outputs.** `story-spec` зі Story Type, Acceptance Criteria і Verification Method.

**Completion Conditions.** Кожне рішення з `clarification-record` відображене в специфікації або явно визнане таким, що не впливає на неї. Кожен Acceptance Criterion перевірюваний і має Verification Method.

**Review Responsibilities.** SPEC_REVIEW.

**Optionality.** Обов'язкова.

**Failure Handling.** Нестача або суперечність вимог → `CHANGES_REQUIRED` з Loopback на CLARIFICATION.

### SPEC_REVIEW

**Purpose.** Незалежно перевірити `clarification-record` і `story-spec` до погодження людиною (OD-03). Ізоляція контексту — за risk triggers (OD-10).

**Preconditions.** Існують поточні версії `clarification-record` і `story-spec`.

**Inputs.**
- Мінімальні: `story-spec`; `clarification-record`; опис Story; пов'язані Business Rules і NFR; Business Glossary.
- За потреби: інші документи Product Foundation.
- Не входить: робочий контекст producer-стадії.

**Responsibilities.**
- Перевірити повноту, однозначність, перевірюваність Acceptance Criteria і наявність Verification Method, узгодженість із Product Foundation і з рішеннями `clarification-record`.
- Перевірити, що Story Type обґрунтований, а зміст специфікації відповідає цьому типу.
- Перевірити, що специфікація не містить вигаданих вимог і не ухвалює технічних рішень.
- Класифікувати findings ([workflow-policies.md](workflow-policies.md#7-findings-disposition)). Не виправляти артефакти.

**Expected Outputs.** `review-report`, прив'язаний до перевірених версій.

**Completion Conditions.** Verdict ухвалено. `PASS` можливий лише без BLOCKING findings.

**Review Responsibilities.** Сама є Review. Результат бачить людина на HUMAN_SPEC_APPROVAL.

**Optionality.** Обов'язкова.

**Failure Handling.** BLOCKING findings → `CHANGES_REQUIRED` з Loopback на SPECIFICATION або CLARIFICATION.

### HUMAN_SPEC_APPROVAL

**Purpose.** Людина погоджує вимоги Story.

**Preconditions.** SPEC_REVIEW має `PASS` для поточних версій артефактів Approval Scope.

**Inputs (пакет рішення).** `story-spec`, `clarification-record`, `review-report` SPEC_REVIEW. Якщо gate проходиться повторно — опис змін відносно погодженої identity, результати Change Impact Assessment і `revalidation-record`.

**Responsibilities.** Людина ухвалює одне з рішень, оголошених у `stage-map.yaml`: APPROVE, RECONFIRM (лише за умов OD-06), REQUEST_CHANGES або CANCEL. Рішення та його обґрунтування записуються в Workflow State і History і прив'язуються до Approved Content Identity.

**Expected Outputs.** Approval, прив'язаний до Approved Content Identity `story-spec` і `clarification-record`, або RECONFIRM попереднього approval.

**Completion Conditions.** Людина ухвалила рішення. Агент не може ухвалити його замість людини.

**Optionality.** Обов'язкова. Правила gates — [workflow-policies.md](workflow-policies.md#3-human-gates).

**Failure Handling.** Поки рішення немає, Story чекає на людину. Таймаутів і автоматичних рішень немає.

## 8. Стадії фази TECHNICAL_DESIGN

### IMPACT_ANALYSIS

**Purpose.** Визначити вплив Story на наявні поведінку, контракти, дані, модулі та інші Stories у роботі.

**Preconditions.** `story-spec` погоджено (HUMAN_SPEC_APPROVAL).

**Inputs.**
- Мінімальні: `story-spec`; наявний код і артефакти, яких Story торкається.
- За потреби: Architecture Foundation; Workflow State інших активних Stories (лише для читання).

**Responsibilities.** Визначити зачеплені компоненти й контракти, ризики регресії, залежності від інших Stories, потребу міграції наявних даних.

**Expected Outputs.** `impact-analysis`.

**Completion Conditions.** Кожна зачеплена область має оцінку впливу або явне обґрунтування відсутності впливу.

**Review Responsibilities.** DESIGN_REVIEW.

**Optionality.** Optional. Умова — `APPLICABILITY_IMPACT_ANALYSIS` у `stage-map.yaml`.

**Failure Handling.** Специфікація нездійсненна чи суперечить наявній системі → `CHANGES_REQUIRED` з Loopback на SPECIFICATION або CLARIFICATION (перетинає HUMAN_SPEC_APPROVAL).

**Не плутати** зі Change Impact Assessment — процедурою оцінки впливу зміни upstream-артефакта на залежні артефакти ([workflow-state-model.md](workflow-state-model.md#8-staleness-і-revalidation)).

### SOLUTION_DESIGN

Універсальна обов'язкова Design Stage (OD-01).

**Purpose.** Визначити загальну технічну структуру рішення Story в тому обсязі, який їй потрібен:
- **Domain Design** — сутності, інваріанти, розміщення бізнес-правил;
- **Application Design** — use cases, порти, межі транзакцій, узгодженість даних між сховищами, категорії помилок;
- **Integration Design** — взаємодія модулів, зовнішніх систем і контрактів між ними;
- **Infrastructure Design** — конфігурація, середовища, runtime-компоненти, запуск і відкат, якщо Story їх змінює.

Розділи, які Story не потрібні, позначаються як незачеплені з коротким обґрунтуванням. Стадія не обмежується рівнями Domain і Application.

**Preconditions.** `story-spec` погоджено. `impact-analysis` актуальний, якщо IMPACT_ANALYSIS застосовна.

**Inputs.**
- Мінімальні: `story-spec`; `impact-analysis` (якщо є); Architecture Foundation в обсязі архітектурного стилю, модулів і правил залежностей.
- За потреби: інші конвенції Architecture Foundation; наявний код зачеплених модулів.

**Responsibilities.** Ухвалювати технічні рішення в межах Foundation. Посилатися на правила Foundation, а не повторювати їх. Указати аналізи, яких Foundation вимагає від Technical Design (для Customer Portal, наприклад, Consistency Analysis із Persistence Conventions). Описати security-рішення на рівні архітектури. Указати нові залежності з класифікацією за Risk-Based Dependency Approval (OD-09).

**Expected Outputs.** `solution-design`.

**Completion Conditions.** Кожен Acceptance Criterion має місце в дизайні, яке його реалізує. Рішення не порушують Architecture Rules Foundation.

**Review Responsibilities.** DESIGN_REVIEW.

**Optionality.** Обов'язкова (OD-01). Обсяг дизайну може бути мінімальним, але стадія не повертає `NOT_APPLICABLE`.

**Failure Handling.** Загальна.

### API_DESIGN

**Purpose.** Спроєктувати зміни зовнішнього інтерфейсу системи.

**Preconditions.** `solution-design` актуальний.

**Inputs.**
- Мінімальні: `story-spec`; `solution-design`; конвенції зовнішнього інтерфейсу з Architecture Foundation (для Customer Portal — API Conventions).
- За потреби: Security-конвенції Foundation; поточний контракт інтерфейсу.

**Responsibilities.** Визначити контракт змін інтерфейсу за конвенціями Foundation. Не вигадувати загальних правил, які Foundation залишає відкритими.

**Expected Outputs.** `api-design`.

**Completion Conditions.** Кожна зміна інтерфейсу описана за конвенціями Foundation. Кожна зміна пов'язана з Acceptance Criteria.

**Review Responsibilities.** DESIGN_REVIEW.

**Optionality.** Optional. Умова — `APPLICABILITY_API_DESIGN`.

**Failure Handling.** Загальна.

### DB_DESIGN

**Purpose.** Спроєктувати зміни схеми та структури збережених даних.

**Preconditions.** `solution-design` актуальний. `api-design` актуальний, якщо API_DESIGN застосовна.

**Inputs.**
- Мінімальні: `story-spec`; `solution-design`; конвенції збереження даних з Architecture Foundation (для Customer Portal — Persistence Conventions).
- За потреби: `api-design`; `impact-analysis`; поточна схема даних.

**Responsibilities.** Визначити зміни схеми, обмежень, індексів і міграцій за конвенціями Foundation, зокрема для наявних даних.

**Expected Outputs.** `db-design`.

**Completion Conditions.** Кожна зміна схеми обґрунтована вимогою або дизайном. Міграції описані відповідно до конвенцій Foundation.

**Review Responsibilities.** DESIGN_REVIEW.

**Optionality.** Optional. Умова — `APPLICABILITY_DB_DESIGN`.

**Failure Handling.** Загальна.

### DESIGN_REVIEW

**Purpose.** Незалежно перевірити весь технічний дизайн Story як ціле. Ізоляція контексту обов'язкова (OD-10). Перший рівень security-перевірки — архітектура (OD-05).

**Preconditions.** Усі застосовні стадії фази TECHNICAL_DESIGN мають `PASS`, незастосовні — `NOT_APPLICABLE` з обґрунтуванням.

**Inputs.**
- Мінімальні: `story-spec`; усі застосовні артефакти дизайну; обґрунтування `NOT_APPLICABLE`; Architecture Rules Foundation.
- За потреби: конвенції Foundation, яких стосується дизайн; наявний код.
- Не входить: робочий контекст producer-стадій.

**Responsibilities.**
- Перевірити відповідність дизайну специфікації, Architecture Foundation і Architecture Rules.
- Перевірити узгодженість артефактів дизайну між собою.
- Перевірити `applicability-decision` для кожного `NOT_APPLICABLE` у фазі TECHNICAL_DESIGN.
- Перевірити security-рішення на рівні архітектури для кожної Story, незалежно від застосовності SECURITY_REVIEW.
- Перевірити класифікацію нових залежностей (OD-09). Перевірити, що дизайн не порушує й не послаблює обов'язкових правил Foundation, зокрема Security Rules. Таке порушення — BLOCKING, і Risk Acceptance його не усуває (RR-14).
- Перевірити `revalidation-record` для незмінних версій артефактів дизайну під час повторного проходження.
- Класифікувати findings. Не виправляти артефакти.

**Expected Outputs.** `review-report`.

**Completion Conditions.** Verdict ухвалено. `PASS` — без BLOCKING findings.

**Review Responsibilities.** Сама є Review.

**Optionality.** Обов'язкова. Не повертає `NOT_APPLICABLE`, бо SOLUTION_DESIGN обов'язкова (OD-01).

**Failure Handling.** BLOCKING findings → `CHANGES_REQUIRED` з Loopback на відповідну Design Stage або вище.

## 9. Стадії фази IMPLEMENTATION_PLANNING

### IMPLEMENTATION_PLANNING

**Purpose.** Спланувати реалізацію погодженого дизайну.

**Preconditions.** DESIGN_REVIEW має `PASS` для поточних версій артефактів дизайну.

**Inputs.**
- Мінімальні: `story-spec`; застосовні артефакти дизайну; `review-report` DESIGN_REVIEW.
- За потреби: структура проєкту з Architecture Foundation; наявний код.

**Responsibilities.**
- Розбити реалізацію на кроки, пов'язані з Acceptance Criteria і дизайном.
- Скласти **Story Risk Assessment**: категорії ризику, яких торкається Story (OD-15).
- Визначити **Testing Approach** за Testing Strategy проєкту і Story Risk Assessment, зокрема де доречний Test-First (Q3) і які додаткові перевірки потрібні для ризикових змін.
- Задекларувати застосовність SECURITY_REVIEW і RECONCILIATION у вигляді `applicability-decision`.
- Визначити ризики й зовнішні дії, які потребуватимуть рішення людини.

**Expected Outputs.** `implementation-plan` (зі Story Risk Assessment і Testing Approach); `applicability-decision` для запланованих Optional Stages фази VERIFICATION.

**Completion Conditions.** Кожен Acceptance Criterion покритий кроком реалізації і перевіркою. Кожна категорія ризику має відповідну перевірку.

**Review Responsibilities.** PLAN_REVIEW.

**Optionality.** Обов'язкова.

**Failure Handling.** Дизайн неповний для планування → `CHANGES_REQUIRED` з Loopback на відповідну Design Stage.

### PLAN_REVIEW

**Purpose.** Незалежно перевірити `implementation-plan` (OD-04). Ізоляція контексту — за risk triggers (OD-10).

**Preconditions.** Існує поточна версія `implementation-plan`.

**Inputs.**
- Мінімальні: `implementation-plan`; `story-spec`; застосовні артефакти дизайну.
- Не входить: робочий контекст producer-стадії.

**Responsibilities.** Перевірити покриття Acceptance Criteria і дизайну, повноту Story Risk Assessment, відповідність Testing Approach Testing Strategy проєкту і категоріям ризику, `applicability-decision` для запланованих Optional Stages, виявлені ризики. Не виправляти план.

**Expected Outputs.** `review-report`.

**Completion Conditions.** Verdict ухвалено. `PASS` — без BLOCKING findings.

**Optionality.** Обов'язкова.

**Failure Handling.** BLOCKING findings → `CHANGES_REQUIRED` з Loopback на IMPLEMENTATION_PLANNING або вище.

### HUMAN_TECHNICAL_APPROVAL

**Purpose.** Людина погоджує технічний дизайн і Implementation Plan (Q5).

**Preconditions.** DESIGN_REVIEW і PLAN_REVIEW мають `PASS` для поточних версій артефактів Approval Scope.

**Inputs (пакет рішення).** Артефакти з `approval_scope` у `stage-map.yaml`, `review-report` DESIGN_REVIEW і PLAN_REVIEW, `applicability-decision`, заявлені ризики та ескалації. Під час повторного проходження — зміни відносно погодженої identity, результати Change Impact Assessment і `revalidation-record`.

**Responsibilities.** Людина ухвалює рішення: APPROVE, RECONFIRM (лише за умов OD-06), REQUEST_CHANGES або CANCEL. Людина може вимагати застосування Optional Stage, яку план позначив як незастосовну.

**Expected Outputs.** Approval, прив'язаний до Approved Content Identity артефактів Approval Scope, або RECONFIRM попереднього approval.

**Completion Conditions.** Людина ухвалила рішення.

**Optionality.** Обов'язкова.

**Failure Handling.** Як для HUMAN_SPEC_APPROVAL.

## 10. Стадії фази IMPLEMENTATION

### IMPLEMENTATION

**Purpose.** Реалізувати погоджений план: код і тести (Q3).

**Preconditions.** HUMAN_TECHNICAL_APPROVAL дійсний для поточних версій дизайну й плану. Робота ведеться у feature branch Story.

**Inputs.**
- Мінімальні: `implementation-plan`; застосовні артефакти дизайну; `story-spec` (Acceptance Criteria); правила Architecture Foundation, яких стосуються зміни.
- За потреби: інші конвенції Foundation; наявний код.
- Під час повторного входу: findings, які спричинили Loopback.

**Responsibilities.**
- Реалізувати зміни в межах погодженого дизайну. Технічні рішення в цих межах агент ухвалює сам (Controlled Autonomy).
- Писати тести як частину реалізації. Застосовувати Test-First там, де план визначив це доречним.
- Не розширювати scope. Відхилення від погодженого дизайну — Loopback або ескалація, а не тиха зміна.
- IMPLEMENTATION — єдина стадія, яка створює нову Changeset Content Identity (`stage-map.yaml` RR-10). Зокрема, тут розв'язуються конфлікти з target branch, якщо для інтеграції треба змінити вміст змін.
- Незаплановану залежність фіксувати в changeset. Суттєву (OD-09) — ескалювати.

**Expected Outputs.** `implementation-changeset` (код і тести у feature branch) з новою Changeset Content Identity.

**Completion Conditions.** Кроки плану виконано. Тести, які стосуються змін, написано і вони проходять локально.

**Review Responsibilities.** CODE_REVIEW.

**Optionality.** Обов'язкова.

**Failure Handling.** Погоджений дизайн чи план не можна реалізувати → `CHANGES_REQUIRED` з Loopback (перетинає HUMAN_TECHNICAL_APPROVAL) або `BLOCKED` для ескалації.

### CODE_REVIEW

Q6, Q28, FR-01.

**Purpose.** Незалежно перевірити реалізацію. Ізоляція контексту обов'язкова (OD-10).

**Preconditions.** Існує поточна Changeset Content Identity. Кожна нова identity потребує нового CODE_REVIEW.

**Inputs.**
- Мінімальні: `implementation-changeset`; `implementation-plan`; застосовні артефакти дизайну; Architecture Rules Foundation.
- За потреби: конвенції Foundation, яких стосуються зміни.
- Не входить: робочий контекст IMPLEMENTATION.

**Responsibilities.** Перевірити відповідність дизайну й плану, Architecture Rules, якість тестів і відповідність Testing Approach, відсутність змін поза scope, класифікацію нових залежностей (OD-09). Зафіксувати security-рішення чи findings, які потребують поглибленої перевірки (тригер SEC-10). Не виправляти код.

**Expected Outputs.** `review-report`, прив'язаний до Changeset Content Identity.

**Completion Conditions.** Verdict ухвалено. `PASS` — без BLOCKING findings.

**Optionality.** Обов'язкова.

**Failure Handling.** BLOCKING findings → `CHANGES_REQUIRED` з Loopback на IMPLEMENTATION або вище.

## 11. Стадії фази VERIFICATION

Розподіл відповідальності у фазі (WF-05):

| Стадія | Що перевіряє | Чого не робить |
|---|---|---|
| SECURITY_REVIEW | Поглиблену безпеку реалізації (другий рівень OD-05) | Не перевіряє загальну якість коду, бо це робить CODE_REVIEW |
| RECONCILIATION | Traceability: зв'язки Acceptance Criteria → дизайн → changeset → тести в обидва боки | Не запускає тести і не оцінює фактичну поведінку |
| VERIFICATION | Фактичну поведінку: результати тестів і перевірок, базові security checks, Definition of Done | Не перебудовує traceability, якщо є `reconciliation-report` |

Усі три стадії прив'язують свої результати до поточної Changeset Content Identity (Review Binding Guard, `stage-map.yaml` RR-10).

### SECURITY_REVIEW

**Purpose.** Поглиблена незалежна перевірка безпеки реалізації (Q22). Другий рівень дворівневої security-перевірки (OD-05). Перший рівень — DESIGN_REVIEW. Ізоляція контексту обов'язкова (OD-10).

**Preconditions.** CODE_REVIEW має `PASS` для поточної Changeset Content Identity.

**Inputs.**
- Мінімальні: `implementation-changeset`; `solution-design` і застосовні артефакти дизайну; `review-report` DESIGN_REVIEW і CODE_REVIEW; Security Rules і Security-конвенції Foundation.
- За потреби: `story-spec`, NFR.
- Не входить: робочий контекст producer-стадій.

**Responsibilities.**
- На вході оцінити застосовність за `APPLICABILITY_SECURITY_REVIEW` проти поточної Changeset Content Identity і всіх фактів `basis_facts`, зокрема нових findings, рішень людини, Recovery і Retry Counters (AP-08). Запланований `applicability-decision` можна лише розширити. Попередній `NOT_APPLICABLE` зі зміненим Applicability Basis не використовується.
- Перевірити реалізацію на відповідність Security Rules Foundation і security-рішенням дизайну, виявити вразливості й послаблення правил безпеки. Не виправляти артефакти.

**Expected Outputs.** `review-report`, прив'язаний до Changeset Content Identity, або `applicability-decision` з `NOT_APPLICABLE`.

**Completion Conditions.** Verdict ухвалено. `PASS` — без BLOCKING findings.

**Optionality.** Optional. Умова — `APPLICABILITY_SECURITY_REVIEW`. Для змін, які відповідають умові, обов'язкова. Після кожної нової Changeset Content Identity застосовність і результат оцінюються заново.

**Failure Handling.** BLOCKING findings → `CHANGES_REQUIRED` з Loopback на IMPLEMENTATION або вище. Ризик безпеки, що не порушує обов'язкових Security Rules, → `BLOCKED` (Escalation Required) для можливого Risk Acceptance людиною. Порушення обов'язкового Security Rule → `BLOCKED` (Foundation Rule Violation): лише виправлення, Foundation Change Request або дозволений виняток (WF-V2-01).

### RECONCILIATION

**Purpose.** Повна перевірка traceability між Acceptance Criteria, дизайном, реалізацією та тестами (Q27).

**Preconditions.** CODE_REVIEW має `PASS`, а SECURITY_REVIEW — `PASS` або `NOT_APPLICABLE` для поточної Changeset Content Identity.

**Inputs.**
- Мінімальні: `story-spec`; застосовні артефакти дизайну; `implementation-plan`; `implementation-changeset`; review-звіти.
- За потреби: `clarification-record`; History Story (лише щоб знайти джерело розриву).

**Responsibilities.** Побудувати і перевірити traceability в обидва боки:
- кожен Acceptance Criterion має місце в дизайні, зміни в changeset і тести чи інші перевірки, які його покривають;
- кожна зміна в changeset має підставу у вимогах чи дизайні.

Тести стадія не запускає. Чи відображені рішення людини в канонічних артефактах, перевіряє VERIFICATION, а не RECONCILIATION.

На вході стадія оцінює застосовність за `APPLICABILITY_RECONCILIATION` і всіма фактами `basis_facts`: поверненнями через Human Gate, Foundation Change Request, Repeated Failure, розширенням Retry Budget, рішеннями людини (AP-08). Попередній `NOT_APPLICABLE` зі зміненим Applicability Basis не використовується.

**Expected Outputs.** `reconciliation-report`, прив'язаний до Changeset Content Identity і версій артефактів, або `applicability-decision` з `NOT_APPLICABLE`.

**Completion Conditions.** Кожен Acceptance Criterion має ланцюг traceability або BLOCKING finding. Кожна зміна має підставу або finding.

**Optionality.** Optional. Умова — `APPLICABILITY_RECONCILIATION`.

**Failure Handling.** Розрив traceability → `CHANGES_REQUIRED` з Loopback на стадію, де виник розрив.

### VERIFICATION

**Purpose.** Фактична перевірка результату Story: поведінка, результати тестів, базові security checks і Definition of Done.

**Preconditions.** CODE_REVIEW має `PASS`, а SECURITY_REVIEW і RECONCILIATION — `PASS` або `NOT_APPLICABLE` для поточної Changeset Content Identity.

**Inputs.**
- Мінімальні: `story-spec`; `implementation-changeset`; `implementation-plan` (Story Risk Assessment і Testing Approach); Testing Strategy проєкту; review-звіти; `reconciliation-report` (якщо є); `applicability-decision`; Currency контрольованих артефактів; Target Base Identity.
- За потреби: Architecture Rules Foundation; артефакти дизайну.

**Responsibilities.**
- Виконати перевірки з трьох джерел — Testing Strategy проєкту, Story Risk Assessment і Testing Approach (OD-15) — і зафіксувати результати. Ризикові зміни отримують додаткові перевірки ([workflow-policies.md](workflow-policies.md#18-testing-requirements-od-15)).
- Перевірити інтеграційний результат щодо поточної Target Base Identity (OD-12).
- Виконати **базову перевірку відповідності вимогам**: кожен Acceptance Criterion має фактичний доказ виконання за своїм Verification Method. Перевірка обов'язкова завжди. Якщо є `reconciliation-report`, VERIFICATION використовує його traceability і не будує її заново. Якщо RECONCILIATION має `NOT_APPLICABLE`, достатньо прямого зв'язку «критерій → доказ» (Q27).
- Виконати базові security checks (Q22) навіть тоді, коли SECURITY_REVIEW незастосовна.
- Перевірити `applicability-decision` для SECURITY_REVIEW і RECONCILIATION, зокрема те, що Applicability Basis кожного `NOT_APPLICABLE` не змінився (AP-08).
- Перевірити, що порушення обов'язкових правил Foundation вирішені не через Risk Acceptance (WF-V2-01).
- Перевірити Currency усіх контрольованих артефактів і прив'язку review-звітів до поточних версій.
- Перевірити, що рішення людини з History відображені в канонічних артефактах (DoD 7).
- Сформувати Findings Disposition і стан Definition of Done ([workflow-policies.md](workflow-policies.md#8-definition-of-done)).

**Expected Outputs.** `verification-report`, прив'язаний до Changeset Content Identity і Target Base Identity.

**Completion Conditions.** `PASS` лише тоді, коли всі агентні умови DoD виконані, а нерозв'язаних BLOCKING findings немає.

**Optionality.** Обов'язкова.

**Failure Handling.** Тести не проходять або Acceptance Criterion не має доказу → `CHANGES_REQUIRED` з Loopback на IMPLEMENTATION або вище. Порушення Acceptance Criteria не можна перенести у FOLLOW_UP.

## 12. Стадії фази DELIVERY

Фаза відокремлює технічну готовність Story від інтеграції змін (OD-02). Перевірка перед merge (MERGE_READINESS_CHECK) і підтвердження після merge (INTEGRATION_CONFIRMATION) — різні стадії (OD-12).

### PR_PREPARATION

Q8, FR-01.

**Purpose.** Підготувати зміни Story до погодження людиною: commits у feature branch і PR до target branch.

**Preconditions.** VERIFICATION має `PASS` для поточних Changeset Content Identity і Target Base Identity.

**Inputs.** `verification-report`; review-звіти; `story-spec`; стан feature branch і target branch.

**Responsibilities.**
- Підготувати commits і PR з описом, посиланнями на Story, результатами перевірок і Findings Disposition.
- Змінювати лише організацію commits і метадані PR, але не вміст змін. Перебудова commits не повинна змінювати Changeset Content Identity (RR-12).
- Не виконувати merge.

**Expected Outputs.** `pull-request`, прив'язаний до Changeset Content Identity і target branch.

**Completion Conditions.** PR відкрито, і він містить саме ту Changeset Content Identity, яку перевірила VERIFICATION.

**Review Responsibilities.** HUMAN_PR_APPROVAL.

**Optionality.** Обов'язкова.

**Failure Handling.**
- Потрібна зміна вмісту змін (зокрема конфлікт із target branch) → `CHANGES_REQUIRED` з Loopback на IMPLEMENTATION. Далі CODE_REVIEW і решта перевірок.
- Змінилася лише target branch, вміст змін той самий → Loopback на VERIFICATION з condition `CHANGESET_CONTENT_UNCHANGED`.
- Немає дозволу або доступу до віддаленого репозиторію → `BLOCKED`.

### HUMAN_PR_APPROVAL

**Purpose.** Людина погоджує конкретну версію змін Story до інтеграції (Q5).

**Preconditions.** `pull-request` відкрито для Changeset Content Identity, яку перевірила VERIFICATION.

**Inputs (пакет рішення).** Артефакти `approval_scope` у `stage-map.yaml`: `implementation-changeset`, `review-report` CODE_REVIEW і SECURITY_REVIEW (якщо застосовна), `reconciliation-report` (якщо є), `verification-report` з Findings Disposition і FOLLOW_UP, `applicability-decision`, `pull-request`. Під час повторного проходження — що змінилося відносно погодженої identity.

**Responsibilities.** Людина ухвалює рішення: APPROVE, RECONFIRM (лише за умов OD-06), REQUEST_CHANGES або CANCEL. Людина остаточно погоджує класифікацію NON_BLOCKING і FOLLOW_UP findings.

**Expected Outputs.** Approval, прив'язаний до Approved Content Identity артефактів Scope.

**Completion Conditions.** Людина ухвалила рішення.

**Optionality.** Обов'язкова.

**Failure Handling.** Як для HUMAN_SPEC_APPROVAL.

### MERGE_READINESS_CHECK

OD-12 (кроки 2–5), OD-16, FR-01, FR-02.

**Purpose.** Безпосередньо перед merge перевірити, що погоджені зміни все ще можна інтегрувати без нової перевірки.

**Preconditions.** HUMAN_PR_APPROVAL має стан `VALID`.

**Inputs.** Approval HUMAN_PR_APPROVAL; `pull-request`; поточний стан feature branch і target branch; `verification-report` (Target Base Identity).

**Responsibilities.**
- Перевірити, що feature branch містить погоджену Changeset Content Identity.
- Перевірити, що PR відповідає погодженій identity: та сама target branch і той самий вміст змін.
- Перевірити, що target branch не змінилася відносно Target Base Identity з `verification-report`.
- Зафіксувати Merge Protection Status target branch — `ACTIVE`, `NOT_SUPPORTED`, `UNAVAILABLE` чи `DISABLED` — і діяти за `merge_protection_statuses` у `stage-map.yaml` (RR-13, FR-02). Налаштування захисту не змінювати. Обмеження для `NOT_SUPPORTED` позначити в пакеті HUMAN_MERGE.

**Expected Outputs.** Результат перевірки і Merge Protection Status у History.

**Completion Conditions.** `PASS` лише тоді, коли виконано три перші умови, а Merge Protection Status — `ACTIVE`, `NOT_SUPPORTED` (за умов FR-02) або `UNAVAILABLE` з погодженим людиною порядком інтеграції. `UNAVAILABLE` без такого рішення → `BLOCKED` (Merge Protection Unavailable). `DISABLED` за обов'язкової Merge Protection Policy → `BLOCKED` (Merge Protection Disabled), доки захист не ввімкнено або немає чинного `foundation-exception-record`.

**Optionality.** Обов'язкова.

**Failure Handling.** Усі повернення перетинають HUMAN_PR_APPROVAL і потребують рішення людини:
- target branch змінилася, вміст змін той самий → Loopback на VERIFICATION з condition `CHANGESET_CONTENT_UNCHANGED`. Новий `verification-report` інвалідує HUMAN_PR_APPROVAL, тож потрібне повне APPROVE;
- PR не відповідає погодженій identity → Loopback на PR_PREPARATION;
- для інтеграції потрібно змінити вміст змін → Loopback на IMPLEMENTATION.

### HUMAN_MERGE

**Purpose.** Людина інтегрує погоджені зміни в target branch (Q5, Q8). Це дія доставки, а не погодження. Merge виконує лише людина (OD-12).

**Preconditions.** MERGE_READINESS_CHECK має `PASS`, а HUMAN_PR_APPROVAL — стан `VALID`.

**Responsibilities.** Людина виконує merge за обраною стратегією і підтверджує це, або повертає Story, якщо merge неможливий. Якщо діє контрольована ручна процедура (`NOT_SUPPORTED` або погоджений порядок для `UNAVAILABLE`), людина виконує merge одразу після MERGE_READINESS_CHECK і разом із CONFIRM_MERGED підтверджує, що target branch не змінилася після перевірки. Якщо змінилася — REQUEST_CHANGES на MERGE_READINESS_CHECK. Ручна процедура не обходить обов'язкових правил проєкту.

**Expected Outputs.** Підтвердження людини.

**Completion Conditions.** Людина ухвалила рішення з переліку в `stage-map.yaml`.

**Failure Handling.** Merge неможливий (конфлікт, зміни в target branch) → REQUEST_CHANGES на MERGE_READINESS_CHECK, PR_PREPARATION або IMPLEMENTATION. Далі діє Rework Protocol.

### INTEGRATION_CONFIRMATION

**Purpose.** Підтвердити фактом, що в target branch інтегровано саме погоджені зміни (OD-12, кроки 7–8). Якщо інтеграцію виконано на неперевіреній базі, виконати Post-Merge Validation (OD-16). Створити `backlog-entry` для погоджених FOLLOW_UP (OD-11).

Post-Merge Validation — частина цієї стадії, а не окрема стадія: вона не потребує нових маршрутів, gates чи Loopbacks ([workflow-policies.md](workflow-policies.md#9-git-integration)).

**Preconditions.** Людина підтвердила merge. HUMAN_PR_APPROVAL чинний для Changeset Content Identity. Зміни target branch після merge на нього не впливають ([workflow-state-model.md](workflow-state-model.md#7-approval-validity)).

**Inputs.** Approval HUMAN_PR_APPROVAL; `pull-request`; `verification-report` (Target Base Identity); результат merge; поточний стан target branch; Testing Strategy, Story Risk Assessment і Testing Approach (для Post-Merge Validation).

**Responsibilities.** Перевірки виконуються в обов'язковому порядку (`post_merge_validation.check_order` у `stage-map.yaml`):
1. **Integration Match.** Результат merge Story відповідає погодженій Changeset Content Identity незалежно від стратегії merge, не покладаючись лише на Git SHA. Непогодженого вмісту немає. Оцінюється результат merge, а не пізніший стан target branch.
2. **Base Comparison.** Target Base Identity на момент merge порівнюється з перевіреною. Якщо збігається, Post-Merge Validation не потрібна.
3. **Post-Merge Validation** — лише якщо база змінилася. Перевірити фактичний інтегрований результат на поточному стані target branch, виконати релевантні інтеграційні перевірки й тести, зафіксувати результат і перевірений стан у `post-merge-validation-record`.
4. **FOLLOW_UP Backlog.** Створити `backlog-entry` для кожного погодженого FOLLOW_UP з посиланням на Story і Finding Identity.

Агент не змінює target branch: не виконує merge, revert чи push і не виправляє інтегрований код.

**Expected Outputs.** `integration-record`; `post-merge-validation-record` (якщо виконувалась); `backlog-entry` для погоджених FOLLOW_UP.

**Completion Conditions.** `PASS` лише тоді, коли Integration Match пройдено, Post-Merge Validation пройдено або не потрібна, а всі `backlog-entry` створено. Тільки тоді Story переходить у DONE.

**Optionality.** Обов'язкова. Post-Merge Validation — умовна частина стадії.

**Failure Handling.**
- Merge ще не виконано → `CHANGES_REQUIRED` з Loopback на HUMAN_MERGE.
- Merge вніс не ті зміни або непогоджений вміст → `BLOCKED` (Integration Mismatch). Post-Merge Validation не виконується.
- Post-Merge Validation не пройдена → `BLOCKED` (Post-Merge Validation Failed) з діагностикою в `post-merge-validation-record`.
- Технічний збій запуску перевірок → Execution State `FAILED` у межах Technical Attempts.
- Автоматичного виправлення вже виконаного merge немає. Рішення ухвалює людина через Recovery Protocol ([workflow-state-model.md](workflow-state-model.md#9-blocked-і-recovery-protocol)).
- Після зовнішнього виправлення і `RESUME` (OD-17): для Post-Merge Validation Failed — `PASS` лише за первісного Integration Match `PASS` і повторної Post-Merge Validation `PASS`; для Integration Mismatch — `PASS` лише з доказом належної інтеграції погодженої Changeset Content Identity цієї Story. Таким доказом є новий merge після корекції (`ROUTE_BACK` → HUMAN_MERGE → новий CONFIRM_MERGED). Integration Match завжди оцінює merge, зафіксований останнім CONFIRM_MERGED. Успіх іншої Story сам по собі не дає `PASS`.
