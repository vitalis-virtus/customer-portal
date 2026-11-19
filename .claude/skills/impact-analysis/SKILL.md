---
name: impact-analysis
description: Stage Skill стадії IMPACT_ANALYSIS. Оцінює застосовність стадії за тригерами IA-01…IA-05 для погодженої story-spec, записує applicability-decision і, якщо стадія застосовна, impact-analysis (зачеплені області, контракти, дані, спільні компоненти, інші Stories, ризики регресії) та повертає result envelope. Інтерв'ю не веде, рішення не проєктує. Запускається лише вручну з готовими шляхами, версією, record_id і номером спроби.
argument-hint: "story=<ID> spec=<шлях> output=<шлях до impact-analysis v1.md> version=1 record_output=<шлях> record_id=applicability-decision/<NNNN> attempt=<N> [mode=pilot] [reason=\"<рішення людини після RESUME>\"]"
disable-model-invocation: true
user-invocable: true
---

# IMPACT_ANALYSIS — Stage Skill

## 1. Purpose

Визначити вплив Story на наявні поведінку, контракти, дані, модулі та інші Stories у роботі. Skill виконує стадію IMPACT_ANALYSIS за `docs/workflow/stages.md` (розділ 8) і `docs/workflow/stage-map.yaml` (`APPLICABILITY_IMPACT_ANALYSIS`). Ці файли й `docs/contracts/` авторитетні. Якщо вони розходяться з цим Skill, діють вони.

Стадія необов'язкова (`optional: true`), тому спочатку Skill оцінює її застосовність і завжди фіксує цю оцінку в `applicability-decision` (`stage-io-map.yaml`, R4). `impact-analysis` створюється лише тоді, коли стадія застосовна.

**Межі стадії:**
- не проєктує рішення (SOLUTION_DESIGN), API (API_DESIGN) чи дані (DB_DESIGN) і не оцінює застосовність цих стадій;
- не змінює вимог: продуктові рішення живуть лише в погодженій `story-spec` і `clarification-record`;
- не змінює код, `story-spec` чи інші артефакти;
- не є Change Impact Assessment, тобто оцінкою впливу зміни upstream-артефакта (`workflow-state-model.md` §8).

Остаточно результат перевіряє DESIGN_REVIEW: і `impact-analysis`, і обґрунтування `NOT_APPLICABLE` (`reviewed_by`, `na_checked_by`).

## 2. Inputs

Аргументи (`$ARGUMENTS`):

| Аргумент | Обов'язковий | Значення |
|---|---|---|
| `story` | так | ID Story, наприклад `US-003` |
| `spec` | так | Готовий шлях поточної погодженої версії `story-spec`. Її номер версії бери з front matter |
| `output` | так | Готовий шлях файлу `impact-analysis`. Skill не обчислює шлях сам |
| `version` | так | Версія `impact-analysis`. Збігається з `v<N>` в `output`. У цьому інкременті — лише `1` |
| `record_output` | так | Готовий шлях файлу `applicability-decision` |
| `record_id` | так | `applicability-decision/<NNNN>`. Номер береться з аргументу |
| `attempt` | так | Номер спроби стадії |
| `mode=pilot` | ні | Пілот поза зареєстрованим Workflow |
| `reason` | ні | Дослівна причина рішення людини `RESUME` після `CHANGES_REQUIRED` цієї стадії |

Людини в контексті немає: Skill нічого не питає. Якщо бракує обов'язкового аргументу або вхід не читається, verdict — `BLOCKED` (розділ 6).

`reason` — це рішення людини щодо попереднього Loopback, а не джерело вимог. Воно може пояснити, чому людина відхилила повернення (наприклад, що виявлена суперечність не є реальною). Нових вимог воно не додає.

Логічні входи — `docs/contracts/stage-io-map.yaml` → `IMPACT_ANALYSIS`, шляхи — `docs/contracts/artifact-registry.yaml`:
- `story-spec` — вимоги, AC, scope, Story Type і розділ «Відкрито для Technical Design»;
- `codebase` — лише ті шляхи, яких Story може торкатися (розділ 3.2);
- `architecture-foundation` — мінімум `docs/architecture/package-map.md` і `architecture.md`, щоб зіставити Story з модулями й спільною інфраструктурою; інші конвенції — лише за потреби;
- Workflow State інших нетермінальних Stories — лише для читання і лише в межах розділу 3.3 (`stages.md`, IMPACT_ANALYSIS → «За потреби»).

`clarification-record`, `story-source` і Product Foundation не є входами. Історія чату — теж. Читай лише потрібні документи (Progressive Context Loading).

## 3. Procedure

1. **Перевір аргументи.**
   - У `mode=pilot` `output` і `record_output` не можуть лежати в `docs/stories/`.
   - `version` має бути `1`, інакше розділ 6.
   - Якщо файл `output` або `record_output` уже існує, verdict — `BLOCKED`. Файли не перезаписуй.
2. **Прочитай `story-spec`.** Виділи `story_type`, `REQ`, `AC`, scope і поза scope, а також «Відкрито для Technical Design». Виділи також, що Story додає, змінює чи вилучає.
3. **Визнач зачеплені області** (розділ 3.1) і зафіксуй стан коду (розділ 3.2).
4. **Перевір інші Stories** (розділ 3.3).
5. **Оціни тригери IA-01…IA-05** (розділ 3.4) і визнач `result`.
6. **Якщо `result: APPLICABLE`**, проведи аналіз впливу за розділом 5.2. Поки verdict не визначено, тримай його в пам'яті, а не у файлі.
7. **Визнач verdict** (розділ 6).
8. **Запиши файли** в такому порядку:
   - `applicability-decision` в `record_output` (розділ 5.1), якщо оцінку застосовності завершено;
   - `impact-analysis` в `output` (розділ 5.2) — **лише** для `PASS`.

   Для інших verdicts файл `impact-analysis` не створюй навіть частково. Незареєстрований Working Draft зупинить наступну спробу.
9. **Поверни envelope** (розділ 6).

### 3.1. Зачеплені області

- Зіставляй `REQ` і `AC` з модулями, Platform, composition root, міграціями та публічними контрактами за `package-map.md` і `architecture.md`. Також з наявними зовнішніми інтерфейсами й структурами даних.
- Область — це конкретний шлях (файл чи каталог) або логічний контракт: операція API, структура даних, спільний компонент.
- Для кожної області зафіксуй:
  - чи існує вона в коді;
  - чи Story її додає, змінює чи вилучає;
  - посилання на `REQ`/`AC`.
- Неіснуюча область, яку Story лише додає, не є зміною наявного. Її однаково перелічи.

### 3.2. Стан коду (D-3)

- Посилайся лише на конкретні прочитані шляхи, ніколи на корінь репозиторію `.`.
- Версія джерела: `git log -1 --format=%h -- <path>` (`artifact-rules.md` §3, п. 4).
- Незакомічені зміни перевіряй командою `git status --porcelain -- <path>` для кожного прочитаного шляху. Якщо зміни є:
  - аналізуй фактичний вміст робочого дерева;
  - познач шлях `uncommitted: true` у `basis`;
  - опиши обмеження в `impact-analysis`.

  Записаний `commit` не ідентифікує такий вміст.
- Шлях без історії в Git (новий чи невідстежуваний) не може бути `inputs`, бо `commit` немає. Зафіксуй його лише в `basis` з `commit: null`.

### 3.3. Інші Stories (D-2)

- Перевіряй лише **зареєстровані** Stories, **видимі** в поточному робочому дереві: `state.yaml` під `storage_root` реєстру (`docs/stories/*/state.yaml`). Поточну Story пропусти.
- Пілотні знімки під `docs/pilots/` не є зареєстрованими Stories, тому їх не перевіряй. Виняток — файли поточної Story, передані аргументами.
- **Релевантні** — лише Stories зі `story_status` не `DONE` і не `CANCELLED`. Для кожної такої Story порівняй зачеплені області (розділ 3.1) з тим, чого вона торкається:
  - за її поточними `impact-analysis`, `solution-design`, `implementation-plan`, `files` з `implementation-changeset`, якщо вони є;
  - інакше — за scope її `story-spec`.
- Реєстру всіх Stories чи пошуку по гілках не будуй. Зафіксуй у `basis` фактичний обсяг: перевірений шлях, ID і стан знайдених Stories, обмеження «видимі лише Stories поточної гілки». Якщо Stories немає, так і напиши.

### 3.4. Тригери застосовності

Тригери — `stage-map.yaml` → `APPLICABILITY_IMPACT_ANALYSIS`, дослівно:

| ID | Тригер | Що перевірити |
|---|---|---|
| IA-01 | Змінює або вилучає наявну поведінку, наявний зовнішній контракт або наявну структуру даних, а не лише додає нові | Області з розділу 3.1, які існують і змінюються або вилучаються |
| IA-02 | Змінює компоненти, конфігурацію або код, якими користуються кілька модулів або спільна технічна інфраструктура | Зміни в Platform, composition root, `go.mod`, спільній конфігурації |
| IA-03 | Потребує міграції чи перетворення даних, які вже існують | Наявні збережені дані й міграції, які Story зачіпає |
| IA-04 | Змінює артефакти, контракти чи файли, яких торкається інша нетермінальна Story | Розділ 3.3 |
| IA-05 | Story Type — BUG, INFRASTRUCTURE або SECURITY | `story_type` у `story-spec` |

Правила (`stage-map.yaml`, AP-01…AP-03):

- Кожен тригер отримує `met` або `not_met` з посиланням на джерело: файл і розділ чи ID.
- Стадія застосовна (`APPLICABLE`), якщо `met` хоча б один тригер (AP-01).
- Якщо застосовність тригера неоднозначна (бракує даних, область не вдалося зіставити, перетин з іншою Story неясний), тригер вважається `met` (AP-02). Причину неоднозначності запиши.
- `NOT_APPLICABLE` — лише коли всі тригери `not_met`, кожен з джерелом (AP-03).

## 4. Quality Criteria

- Кожна зачеплена область має оцінку впливу або явне обґрунтування, що впливу немає (`stages.md`, Completion Conditions).
- Кожен тригер має висновок і джерело. Висновки не суперечать переліку областей.
- Кожен `met` тригер розкрито в `impact-analysis`.
- Кожне джерело в `inputs` і `basis` — конкретний шлях з актуальним `commit`. Незакомічені зміни позначено.
- Обсяг перевірки інших Stories і його обмеження зафіксовано.
- Технічних рішень немає: структури, маршрутів, схем, алгоритмів, бібліотек. Відкриті технічні питання винесено в «Відкрито для Solution Design».
- Немає вимог без джерела в `story-spec`. Відкрите питання не інтерпретовано мовчки.

## 5. Output

Мова обох файлів — українська. У `mode=pilot` перший рядок тіла обох файлів: «Пілотний артефакт поза офіційним зареєстрованим Workflow. Не є офіційно погодженим і не є входом стадій зареєстрованих Stories; може бути входом наступних стадій того самого пілоту». Статусів, Currency, approval і позначки пілоту у front matter немає.

### 5.1. `applicability-decision`

Front matter — спільний заголовок запису і поля типу з `docs/contracts/artifact-rules.md` §2, і лише вони:

```yaml
---
record_type: applicability-decision
record_id: applicability-decision/0001       # з аргументу
story_id: US-003
produced_by: { stage: IMPACT_ANALYSIS, attempt: 1 }
created_at: 2026-11-19T18:00:00Z             # date -u +%Y-%m-%dT%H:%M:%SZ
subjects:
  - { type: story-spec, version: 2 }         # погоджена версія, за якою оцінено застосовність
target_stage: IMPACT_ANALYSIS
kind: ENTRY
result: NOT_APPLICABLE                       # APPLICABLE | NOT_APPLICABLE
triggers:
  IA-01: { status: not_met, ref: "story-spec v2 §Scope; internal/ — модуля ще немає" }
  IA-02: { status: not_met, ref: "..." }
  IA-03: { status: not_met, ref: "..." }
  IA-04: { status: not_met, ref: "docs/stories/ — нетермінальних Stories немає" }
  IA-05: { status: not_met, ref: "story-spec v2 front matter: story_type FEATURE" }
basis:                                       # basis_facts стадії
  - { type: story-spec, version: 2 }
  - { fact: story_type, value: FEATURE }
  - { source: codebase, path: main.go, commit: <commit>, uncommitted: false }
  - { source: architecture-foundation, path: docs/architecture/package-map.md, commit: <commit>, uncommitted: false }
  - { fact: other_stories, scope: "docs/stories/*/state.yaml у поточному робочому дереві", visible: [], limitation: "видимі лише Stories поточної гілки; пілотні знімки docs/pilots/ не перевірялися" }
---
```

`verdict` і `findings` запис не містить: це не звіт review (`artifact-rules.md` §2).

Розділи тіла:
1. **Предмет** — версія `story-spec`, прочитані шляхи коду й Foundation.
2. **Зачеплені області** — короткий перелік з розділу 3.1.
3. **Тригери** — для кожного IA-xx висновок, обґрунтування і причина неоднозначності, якщо застосовано AP-02.
4. **Інші Stories** — обсяг перевірки й обмеження (розділ 3.3).
5. **Рішення людини** — лише якщо передано `reason`: дослівна цитата і як її враховано.

### 5.2. `impact-analysis` (лише `PASS`)

Front matter — поля `docs/contracts/artifact-rules.md` §1, і лише вони:
- `artifact_type: impact-analysis`, `story_id`, `version`;
- `produced_by: { stage: IMPACT_ANALYSIS, attempt }`;
- `created_at`;
- `inputs`: `{ type: story-spec, version }` і кожен прочитаний шлях з історією в Git як `{ source: codebase | architecture-foundation, path, commit }`.

Розділи тіла:
1. **Предмет** — версія `story-spec`, `record_id` рішення про застосовність, `met` тригери.
2. **Зачеплені області** — таблиця: область, чи існує, чи Story її додає / змінює / вилучає, `REQ`/`AC`, оцінка впливу або обґрунтування, що впливу немає.
3. **Поведінка і зовнішні контракти** — що з наявного змінюється чи вилучається і для кого.
4. **Дані й міграції** — наявні дані, які зачеплено, і чи потрібна їхня міграція чи перетворення. Схему не проєктуй.
5. **Спільні компоненти** — Platform, composition root, конфігурація, залежності модулів.
6. **Інші Stories** — перетини або їх відсутність, обсяг перевірки.
7. **Ризики регресії** — що з наявного може зламатися, з посиланням на область.
8. **Обмеження аналізу** — незакомічені чи невідстежувані шляхи, неоднозначності (AP-02), межі видимості.
9. **Відкрито для Solution Design** — питання, які стадія свідомо не вирішує.
10. **Рішення людини** — лише якщо передано `reason`: дослівна цитата і як вона вплинула на аналіз. Рішення людини фіксується в канонічному артефакті, а не лише в History (`workflow-policies.md` §2).

## 6. Verdict and Return

IMPACT_ANALYSIS має `optional: true` і loopbacks `SPECIFICATION` і `CLARIFICATION`, обидва з `crosses_gates: [HUMAN_SPEC_APPROVAL]`. Тому допустимі **`PASS`**, **`NOT_APPLICABLE`**, **`CHANGES_REQUIRED`** і **`BLOCKED`**.

| Ситуація | Verdict | `outputs` |
|---|---|---|
| Бракує аргументу, `version` ≠ `1`, вхід не читається, `output` чи `record_output` уже існує | `BLOCKED` до оцінки. Файлів немає | `[]` |
| Усі тригери `not_met` | `NOT_APPLICABLE` | `[{ record: <record_id> }]` |
| `APPLICABLE`, аналіз завершено, критерії розділу 4 виконано | `PASS` | `[{ type: impact-analysis, version }, { record: <record_id> }]` |
| `APPLICABLE`, `story-spec` нездійсненна чи суперечить наявній системі, і це усувається в `story-spec` без нового рішення людини | `CHANGES_REQUIRED`, `loopback_target: SPECIFICATION` | `[{ record: <record_id> }]` |
| `APPLICABLE`, і усунення потребує нового чи іншого продуктового рішення людини | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` | `[{ record: <record_id> }]` |
| Суперечність із Foundation, яку не усунути зміною артефактів Story | `BLOCKED` (Foundation Change Required) | `[{ record }]`, якщо оцінку завершено, інакше `[]` |
| Потрібне рішення з переліку ескалацій (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) | те саме |
| Бракує інформації, яку може дати лише людина | `BLOCKED` (Missing Information) | те саме |

Правила:

- `CHANGES_REQUIRED` можливий лише за `result: APPLICABLE`. Якщо стадія незастосовна, verdict — `NOT_APPLICABLE`. Проблеми дизайну тоді виявляють наступні стадії SOLUTION_DESIGN, API_DESIGN, DB_DESIGN і DESIGN_REVIEW.
- `loopback_target` — лише пропозиція. Обидві цілі перетинають HUMAN_SPEC_APPROVAL. Підтвердження, маршрут і рішення людини належать Orchestrator (RR-04). Skill їх не виконує й не імітує.
- Для `CHANGES_REQUIRED` findings передаються в `summary`: коротко кожна проблема з посиланням на `REQ`/`AC` і область коду. ID `F-NNN` не присвоюй, це робить review.
- Для `BLOCKED` потрібне опиши в `blocker.needed`. `foundation-change-request` Skill не створює.
- У `mode=pilot` додай у `summary` позначку «pilot».

Envelope — формат `docs/contracts/workflow-state.md` §6, без змін і без додаткових полів:

```yaml
result:
  story_id: US-003
  stage: IMPACT_ANALYSIS
  attempt: 1
  verdict: PASS                      # PASS | NOT_APPLICABLE | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { type: impact-analysis, version: 1 }
    - { record: applicability-decision/0001 }
  loopback_target: null              # SPECIFICATION | CLARIFICATION лише для CHANGES_REQUIRED
  blocker: null                      # для BLOCKED: { reason, details, needed }
  summary: "APPLICABLE (IA-01, IA-02); 4 області, 2 ризики регресії; pilot"
```

## 7. Constraints

- Пиши лише файли `record_output` і, для `PASS`, `output`.
- Не змінюй `story-spec`, `clarification-record`, код, Foundation, Workflow Definition, `docs/contracts/` та інші артефакти.
- Не змінюй `state.yaml` і `history.jsonl` будь-якої Story, а також статуси, Currency чи approvals.
- Не обчислюй шлях, версію чи номер запису: бери їх з аргументів.
- Bash — лише `git log`, `git status`, `git ls-files` і `date`.
- Не веди інтерв'ю, не питай людину і не ухвалюй продуктових чи технічних рішень замість відповідних стадій.
- Не виконуй Loopback, не проходь Human Gate, не імітуй Orchestrator, DESIGN_REVIEW чи рішень людини. Не запускай інші стадії, Skills чи subagents.
- Не роби `git commit`, `git push` чи деструктивних операцій.
