---
name: story-orchestrator
description: Story Orchestrator MVP (Phase 6, increments 6.4.1–6.4.3; Phase 7, increments 7.4.1–7.4.2; Phase 8, increments 8.3.2, 8.4). Ініціалізує пілотну Story поза зареєстрованим Workflow, показує її статус з перевіркою State і History, виконує одну стадію за запуск (CLARIFICATION inline, SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS і SOLUTION_DESIGN у subagent) у Step Mode до API_DESIGN, підтримує повторний вхід в IMPACT_ANALYSIS після Loopback від SOLUTION_DESIGN, застосовує рішення людини на HUMAN_SPEC_APPROVAL (APPROVE, REQUEST_CHANGES, CANCEL) і на підтвердженні Loopback від IMPACT_ANALYSIS та SOLUTION_DESIGN (RESUME, CANCEL). Запускається лише вручну з mode=pilot.
argument-hint: "init|status|next story=<ID> mode=pilot | decide story=<ID> mode=pilot decision=APPROVE|REQUEST_CHANGES|CANCEL|RESUME|ROUTE_BACK by=<id> [target=<stage>] [reason=\"...\"] [comment=\"...\"]"
disable-model-invocation: true
user-invocable: true
---

# Story Orchestrator — MVP (increments 6.4.1–6.4.3, 7.4.1–7.4.2, 8.3.2, 8.4)

## 1. Purpose

Orchestrator веде Workflow State і History однієї Story: `state.yaml` і `history.jsonl` змінює **лише** він (`docs/contracts/workflow-state.md` §1). Маршрути, verdicts, loopbacks і gates визначає лише `docs/workflow/stage-map.yaml`; формат стану — `docs/contracts/workflow-state.md`; шляхи — `docs/contracts/artifact-registry.yaml`. Ці файли авторитетні; у разі розбіжності діють вони, а не цей Skill.

Skill працює в основній сесії.

**Межі.** Команди: `init`, `status`, `next`, `decide`.

- `next` виконує одну стадію за запуск (Step Mode): CLARIFICATION, SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS або SOLUTION_DESIGN.
- Після `PASS` SPEC_REVIEW Story зупиняється на HUMAN_SPEC_APPROVAL і чекає на рішення людини.
- Після `PASS` або `NOT_APPLICABLE` IMPACT_ANALYSIS Story переходить на SOLUTION_DESIGN.
- Після `PASS` SOLUTION_DESIGN Story переходить на API_DESIGN. Цю стадію `next` ще не виконує.
- `decide` застосовує рішення людини:
  - `APPROVE`, `REQUEST_CHANGES` або `CANCEL` на HUMAN_SPEC_APPROVAL (розділ 8);
  - `RESUME` або `CANCEL` на підтвердженні Loopback (`LOOPBACK_CONFIRMATION`), коли IMPACT_ANALYSIS чи SOLUTION_DESIGN повернула `CHANGES_REQUIRED` з ціллю через gate (розділ 8.7). `ROUTE_BACK` відхиляється без змін (розділ 8.7.2).
- Повторний вхід після повернення підтримується:
  - для SPECIFICATION — після loopback від SPEC_REVIEW або REQUEST_CHANGES на gate (розділ 7.3.1);
  - для IMPACT_ANALYSIS — після loopback від SOLUTION_DESIGN (розділ 7.3.2).
- `REQUEST_CHANGES` з `target=CLARIFICATION` відхиляється без змін (розділ 8.4.1).
- Не реалізовано: виконання інших стадій, повторний вхід у CLARIFICATION і SOLUTION_DESIGN з новою версією, інші Human Gates, RECONFIRM, Loopback через gate (`ROUTE_BACK`), Recovery і Auto Mode. Якщо людина просить щось інше — поясни, що цього ще немає, і зупинись.

## 2. Inputs

Аргументи (`$ARGUMENTS`):

| Аргумент | Обов'язковий | Значення |
|---|---|---|
| команда | так | `init`, `status`, `next` або `decide` |
| `story` | так | ID Story, формат `US-NNN` |
| `mode=pilot` | так | Явна позначка пілоту. Без неї Skill не працює: реєстрація реальних Stories не підтримується |
| `decision` | лише для `decide` | Рішення людини на gate або на підтвердженні Loopback (розділ 8.1) |
| `by` | лише для `decide` | Ідентифікатор людини, яка ухвалила рішення (розділ 8.1) |
| `target` | лише для `decide` з `REQUEST_CHANGES` | Стадія-ціль повернення (розділ 8.1) |
| `reason` | лише для `decide` з `REQUEST_CHANGES`, `CANCEL` або `RESUME` | Причина рішення (розділ 8.1) |
| `comment` | ні, лише для `decide` з `APPROVE` | Коментар людини до рішення (розділ 8.1) |

Якщо аргументу бракує або він інший — зупинись і поясни. Не вгадуй.

## 3. Pilot Storage

Пілот лежить **поза** `docs/stories/` і ніколи не подається як зареєстрована чи погоджена Story.

- Корінь пілоту: `docs/pilots/orchestrator/<story_id>/`.
- Структура під коренем — та сама, що в реєстрі під `storage_root`; змінюється лише префікс `docs/stories/<story_id>/` → `docs/pilots/orchestrator/<story_id>/`:

  ```
  docs/pilots/orchestrator/<story_id>/
  ├── state.yaml
  ├── history.jsonl
  ├── artifacts/<artifact_type>/v<N>.md
  └── records/<record_type>/<NNNN>.md
  ```

- **Шлях артефакта** = шаблон `path` з `artifact-registry.yaml` із заміненим префіксом. Наприклад, `clarification-record` v1 → `docs/pilots/orchestrator/US-001/artifacts/clarification-record/v1.md`.
- `artifacts/` і `records/` під час `init` не створюються.
- Будь-який шлях пілоту в `docs/stories/` — помилка: зупинись.
- Ознака пілоту — шлях. Полів, яких немає в контракті (`mode` тощо), у `state.yaml` немає.
- Файли інших пілотів (зокрема ручного `docs/pilots/US-001/`) не є входами і не змінюються.

## 4. Command `init`

1. **Перевір аргументи** (розділ 2).
2. **Перевір джерело Story.** `story` є Candidate Story у `docs/product/epic-map.md` (`story-source`). Backlog ще немає (PRQ-01), тож інших джерел немає. Не знайдено — зупинись.
3. **Перевір, що пілоту ще немає.** Якщо `docs/pilots/orchestrator/<story_id>/` **існує в будь-якому вигляді** (навіть без `state.yaml`) — зупинись, перелічи знайдені файли й попроси рішення людини. Не перезаписуй, не переміщуй і не видаляй наявні файли.
4. **Передумови.** FG-01 (Testing Strategy) блокує реєстрацію першої **реальної** Story (`workflow-policies.md` §15). Пілот не є реєстрацією, тож FG-01 його не блокує, але й не вважається виконаною. Скажи про це людині.
5. **Візьми час:** `date -u +%Y-%m-%dT%H:%M:%SZ`. Одне значення для `updated_at` і `ts`.
6. **Запиши `state.yaml`** (розділ 4.1).
7. **Запиши `history.jsonl`** з однією подією (розділ 4.2).
8. **Покажи результат** командою `status` (розділ 5), без додаткових змін.

Кроки 6–7 — один крок Orchestrator (`workflow-state.md` §7). Якщо запис другого файлу не вдався — не виправляй мовчки: повідом людині, що пілот неузгоджений.

### 4.1. `state.yaml` під час init

Значення стадії — з `stage-map.yaml` → `initial_state`. Усі ключі верхнього рівня з `workflow-state.md` §3 присутні; ще не визначене — `null` або порожнє.

```yaml
# PILOT — поза зареєстрованим Workflow. Не зареєстрована і не погоджена Story.
story_id: US-001
story_type: null                    # до реєстрації story-spec
story_status: ACTIVE                # initial_state.story_status
current_stage: CLARIFICATION        # initial_state.stage
execution_state: READY              # initial_state.execution_state
last_stage_verdict: null
execution_mode: STEP                # Auto Mode не підтримується
updated_at: <date -u>

artifacts: {}

records:
  review-report: {}
  applicability-decision: {}
  revalidation-record: []

approvals: {}

pending_human_decision: null

blocker: null

retry:
  technical_attempts: {}
  quality_loops:
    target: {}
    initiator: {}
  repeated_failure: {}
  extensions: []

git:                                # пілот не має feature branch; іменування гілок не визначене
  feature_branch: null
  target_branch: null
  changeset_commit: null
  verified_target_base: null
  merge_protection_status: null
```

### 4.2. Подія ініціалізації

Для пілоту використовується `story.registered` (`workflow-state.md` §5) з явною позначкою, що Story не зареєстрована у Workflow. Це рішення погоджене для ізольованого MVP-пілоту. Один рядок JSON:

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"orchestrator","id":"story-orchestrator"},"event":"story.registered","stage":"CLARIFICATION","refs":[],"details":{"mode":"pilot","registered_in_workflow":false,"note":"Pilot state initialized outside the registered Workflow; not a registered or approved Story","story_status":"ACTIVE","execution_state":"READY","execution_mode":"STEP"}}
```

## 5. Command `status`

Тільки читання. Жодних змін файлів — навіть якщо знайдено неузгодженість.

1. **Перевір аргументи** (розділ 2) і що `docs/pilots/orchestrator/<story_id>/state.yaml` та `history.jsonl` існують. Немає — зупинись.
2. **Прочитай** обидва файли повністю.
3. **Перевір узгодженість** (розділ 6).
4. **Покажи:**
   - Story ID і позначку «PILOT — не зареєстрована в офіційному Workflow і офіційно не погоджена. Рішення Human Gates у пілоті симульовані»;
   - `story_status`, `current_stage`, `execution_state`, `last_stage_verdict`, `execution_mode`, `story_type`, `updated_at`. Для `CANCELLED` — «скасовано в пілоті» і `records.cancellation-record`;
   - поточні версії артефактів зі `status` і `currency`, чинні записи з `records`, зокрема `applicability-decision` за стадіями з їхнім `result`. Біля `APPROVED` — «у пілоті». Якщо чинне рішення IMPACT_ANALYSIS — `NOT_APPLICABLE`, а `impact-analysis` зареєстровано, познач його «історична версія, не є входом» (розділ 7.1);
   - `approvals` під заголовком «Рішення пілоту (симульований Human Gate), не офіційне погодження Workflow»: для кожного gate — `decision`, `decided_by`, `decided_at`, `scope`, `approval_state`;
   - `pending_human_decision`, `blocker`, ненульові `retry.technical_attempts` і `retry.quality_loops`. Для `kind: LOOPBACK_CONFIRMATION` — також запропоновану ціль (`loopback_target` останньої події `stage.completed`), `crosses_gates` і доступні в MVP рішення (`RESUME`, `CANCEL`; розділ 8.7);
   - кількість подій History і останню подію (`ts`, `event`, `stage`);
   - результат перевірки: «узгоджено» або перелік розбіжностей.

Якщо є розбіжність: за `workflow-state.md` §7 і `workflow-state-model.md` §12 Story мала б перейти в `BLOCKED` (Integrity Error). Поточна версія Skill цього **не записує**. Скажи людині, що саме розійшлося, де, і що розбіжність розв'язує вона. Не виправляй.

## 6. Validation

Перевіряй лише те, що прямо визначають джерела. Кожна розбіжність — з посиланням на файл і поле.

**`state.yaml`** (`workflow-state.md` §3):

- присутні всі ключі верхнього рівня з розділу 4.1;
- `story_id` збігається з аргументом і назвою директорії;
- `story_status` ∈ `ACTIVE | WAITING_FOR_HUMAN | BLOCKED | DONE | CANCELLED`;
- `execution_state` ∈ `READY | RUNNING | AWAITING_HUMAN | COMPLETED | INTERRUPTED | FAILED | INVALID_RESULT`;
- `execution_mode` ∈ `STEP | AUTO`;
- `current_stage` — Stage ID зі `stage-map.yaml` → `stages`;
- `last_stage_verdict` — `null` або одне зі `stage_verdicts`;
- `story_type` — `null` або одне зі `story_types`; якщо `story-spec` зареєстровано — збігається з її `story_type`;
- для кожного артефакта: `status` ∈ `DRAFT | REVIEWED | APPROVED`, `currency` ∈ `CURRENT | STALE`.

**Взаємні умови** (`workflow-state-model.md` §3, §4):

- `story_status: WAITING_FOR_HUMAN` ⇔ `pending_human_decision` не `null` ⇔ `execution_state: AWAITING_HUMAN`;
- `story_status: BLOCKED` ⇔ `blocker` не `null`;
- `story_status: CANCELLED` ⇔ `records.cancellation-record` не порожній; тоді `pending_human_decision: null`, `execution_state: COMPLETED`, а остання подія History — `story.cancelled` з цим записом у `refs` (розділ 8.5.3).

**`approvals`** (`workflow-state.md` §3, §4). Якщо `approvals.HUMAN_SPEC_APPROVAL` не `null`:

- є поля `decision`, `decided_by`, `decided_at`, `scope`, `package`, `comment`, `approval_state`, `previous`; `decision: APPROVE`; `approval_state` ∈ `VALID | SUSPENDED | INVALIDATED`;
- для `approval_state: VALID` кожен елемент `scope` — поточна версія свого типу в `artifacts` зі `status: APPROVED`;
- у History є подія `human_decision.applied` стадії `HUMAN_SPEC_APPROVAL` з `details.decision: APPROVE`, `ts` якої дорівнює `decided_at`.

**`history.jsonl`** (`workflow-state.md` §5):

- кожен рядок — валідний JSON з полями `ts`, `story_id`, `actor`, `event`, `stage`, `refs`, `details`;
- `story_id` збігається зі станом;
- `event` — з таблиці подій §5;
- `actor.kind` ∈ `agent | human | orchestrator`;
- `ts` не спадає;
- перша подія — подія ініціалізації (розділ 4.2);
- `execution_state: RUNNING` ⇔ остання подія — `stage.started` або `human_decision.applied` для `current_stage`. Виняток — `human_decision.applied` з `details.decision: RESUME` (`LOOPBACK_CONFIRMATION`, розділ 8.7.3): після неї `execution_state: READY`;
- `execution_state: AWAITING_HUMAN` ⇔ остання подія — `human_decision.requested` для `current_stage`.

**Файли** (`workflow-state-model.md` §12, крок 2):

- кожна версія з `artifacts` існує за шляхом пілоту, а її front matter має той самий `artifact_type`, `version` і `story_id`;
- кожен запис з `records` існує, а його front matter має той самий `record_id` і `story_id`. Для `records.applicability-decision` це кожне значення `<STAGE>: { ENTRY: <record_id> }`; його front matter має також `target_stage: <STAGE>` і `kind: ENTRY`.

**Git Binding** не перевіряється, поки всі поля `git` — `null` (пілот без feature branch). Скажи про це в статусі.

## 7. Command `next`

Виконує одну спробу поточної стадії і зупиняється.

### 7.1. Підтримувані стадії

| Стадія | Stage Skill | Де виконується | Вихід | Входи-артефакти |
|---|---|---|---|---|
| `CLARIFICATION` | `.claude/skills/clarification/SKILL.md` | inline, основна сесія (інтерв'ю з людиною) | `clarification-record` | — |
| `SPECIFICATION` | `.claude/skills/specification/SKILL.md` | окремий subagent `general-purpose` | `story-spec` | `clarification-record` |
| `SPEC_REVIEW` | `.claude/skills/spec-review/SKILL.md` | окремий subagent `general-purpose` | запис `review-report` | `story-spec`, `clarification-record` |
| `IMPACT_ANALYSIS` | `.claude/skills/impact-analysis/SKILL.md` | окремий subagent `general-purpose` | запис `applicability-decision` (завжди, коли оцінку завершено) і `impact-analysis` (лише `PASS`) | `story-spec` |
| `SOLUTION_DESIGN` | `.claude/skills/solution-design/SKILL.md` | окремий subagent `general-purpose` | `solution-design` | `story-spec`; `impact-analysis` — лише якщо IMPACT_ANALYSIS застосовна (розділ 7.2, крок 7) |

Вихід і власник — з `artifact-registry.yaml` (`produced_by`), входи — з `stage-io-map.yaml`. SPEC_REVIEW перевіряє обидва входи (`stage-map.yaml` → `reviews: [CLARIFICATION, SPECIFICATION]`). `codebase`, `architecture-foundation` і `product-foundation` для IMPACT_ANALYSIS і SOLUTION_DESIGN — джерела, а не зареєстровані артефакти: їх читає сам Stage Skill (`impact-analysis/SKILL.md` §2, `solution-design/SKILL.md` §2). Дозволені verdicts і маршрути — лише зі `stage-map.yaml` (розділ 7.6).

**Умовний вхід `impact-analysis`.** Умову `when: applicable` (`stage-io-map.yaml`) визначає чинний `applicability-decision` стадії, тобто `records.applicability-decision.IMPACT_ANALYSIS.ENTRY` з `result: APPLICABLE`, а не наявність артефакта. Якщо чинне рішення — `NOT_APPLICABLE`, зареєстрована раніше версія `impact-analysis` лишається в `artifacts` як історична, але входом жодної стадії не є, і Orchestrator її не передає (розділ 7.2, крок 7).

### 7.2. Перевірка перед запуском

1. **Перевір аргументи** і узгодженість (розділ 6). Розбіжність — зупинись, як у `status`.
2. **`story_status` має бути `ACTIVE`.** Інакше — покажи стан і зупинись. Для `pending_human_decision.kind: GATE` на HUMAN_SPEC_APPROVAL і для `kind: LOOPBACK_CONFIRMATION` на IMPACT_ANALYSIS чи SOLUTION_DESIGN скажи, що рішення людина ухвалює командою `decide` (розділи 8, 8.7). Для `CANCELLED` — Story скасована в пілоті, стан термінальний і повторно не відкривається. Інші рішення людини і Recovery ще не реалізовано.
   Окремий випадок — `WAITING_FOR_HUMAN` з `pending_human_decision.kind: CLARIFICATION_ANSWERS`: новий запуск `next` означає, що інтерв'ю цієї спроби перервалося разом із сесією. **Обмеження MVP:** продовження інтерв'ю в новій сесії не підтримується, а контракт не визначає переходу з `AWAITING_HUMAN` у `INTERRUPTED` під час відновлення сесії (`workflow-state-model.md` §12 описує лише `RUNNING`). Нічого не змінюй: поясни це людині і зупинись.
3. **`current_stage` має бути в таблиці 7.1.** Інша стадія — «ще не підтримується», зупинись.
4. **`execution_state`:**
   - `READY`, `INTERRUPTED`, `FAILED`, `INVALID_RESULT` — можна починати нову спробу;
   - `RUNNING` — попередню спробу перервано без результату (`workflow-state-model.md` §12, крок 3). Запиши невдачу `INTERRUPTED` (розділ 7.7) і зупинись. Нову спробу людина запускає наступним `next`;
   - інше — зупинись і поясни.
5. **Входи-артефакти** (`stage-io-map.yaml`, правило R1). Для кожного входу з таблиці 7.1 (умовний `impact-analysis` — лише якщо його передають за кроком 7):
   - тип зареєстровано в `state.yaml` → `artifacts`, `currency: CURRENT`, файл поточної версії існує;
   - кожен upstream-артефакт з його `inputs` (`type`, `version`, тип відмінний від типу самого входу) — поточна версія цього типу в `artifacts` (`artifact-rules.md` §4);
   - посилання на попередню версію того самого типу — це lineage, а не upstream-залежність (`artifact-rules.md` §3): дозволене лише для `version > 1`, лише на `version - 1`, і файл цієї версії існує за шляхом пілоту. Бути поточною вона не мусить: вона `SUPERSEDED`;
   - кожне джерело з його `inputs` (`source`, `path`, `commit`) досі має той самий `git log -1 --format=%h -- <path>` (`artifact-rules.md` §4).

   Невиконано — нічого не змінюй, поясни людині і зупинись. **Обмеження MVP:** подію `staleness.detected` і перехід у `BLOCKED` (Upstream Changed) поточна версія Skill не записує.
6. **Погодження і тригер входу (IMPACT_ANALYSIS і SOLUTION_DESIGN).** Передумова обох стадій — погоджений `story-spec` (`stages.md`, IMPACT_ANALYSIS і SOLUTION_DESIGN; `workflow-state-model.md` §6, крок 4):
   - `approvals.HUMAN_SPEC_APPROVAL` не `null`, `decision: APPROVE`, `approval_state: VALID`;
   - поточна версія `story-spec` в `artifacts` має `status: APPROVED` і збігається з версією `story-spec` у `approvals.HUMAN_SPEC_APPROVAL.scope`;
   - **тригер входу** — остання подія History з `details.to_stage`, як у розділі 7.3.1. Її `to_stage` має дорівнювати `current_stage`. Дозволені тригери:

     | Стадія | Тригер |
     |---|---|
     | IMPACT_ANALYSIS | `human_decision.applied` HUMAN_SPEC_APPROVAL з `decision: APPROVE` (перший вхід); `stage.completed` SOLUTION_DESIGN з `verdict: CHANGES_REQUIRED` і `loopback_target: IMPACT_ANALYSIS` (повторний вхід, розділ 7.3.2); `human_decision.applied` IMPACT_ANALYSIS з `decision: RESUME` (розділ 8.7.3) |
     | SOLUTION_DESIGN | `stage.completed` IMPACT_ANALYSIS з `verdict: PASS` або `NOT_APPLICABLE`; `human_decision.applied` SOLUTION_DESIGN з `decision: RESUME` |

     Інший тригер означає маршрут, якого цей MVP не виконує;
   - SOLUTION_DESIGN: `artifacts.solution-design` відсутній. Повторного входу в SOLUTION_DESIGN з новою версією MVP не підтримує (`solution-design/SKILL.md` §3, крок 1).

   Невиконано — нічого не змінюй, поясни людині і зупинись.
7. **Застосовність IMPACT_ANALYSIS (лише SOLUTION_DESIGN).** Визначає, чи передається `impact-analysis` (розділ 7.1, «Умовний вхід»):
   - `records.applicability-decision.IMPACT_ANALYSIS.ENTRY` є, а файл запису існує;
   - його `subjects` — поточна версія `story-spec`;
   - остання подія `stage.completed` стадії IMPACT_ANALYSIS має `refs` з цим `<record_id>` і `verdict`, узгоджений із `result` запису: `PASS` ⇔ `APPLICABLE`, `NOT_APPLICABLE` ⇔ `NOT_APPLICABLE`;
   - **`APPLICABLE`:**
     - `artifacts.impact-analysis` є;
     - `refs` тієї самої події містять `impact-analysis@<version>` поточної версії в `artifacts`, тобто артефакт створено тим самим оцінюванням;
     - крок 5 для нього виконано (`currency: CURRENT`, файл існує);
     - `impact` = шлях цієї версії;
   - **`NOT_APPLICABLE`:**
     - `impact` не передається;
     - якщо `artifacts.impact-analysis` є (історична версія з попереднього застосовного оцінювання), він не є входом: не передавай і не перевіряй його за кроком 5.

   Будь-яка інша комбінація — розбіжність стану. Нічого не змінюй, поясни людині, що саме не збігається (файл, поле, подія), і зупинись.

### 7.3. Параметри спроби

Обчислює Orchestrator, а не Stage Skill:

| Параметр | Як визначається |
|---|---|
| `attempt` | Кількість подій `stage.started` для `current_stage` у History + 1 |
| `version` | Для документа: поточна версія вихідного типу в `artifacts` + 1; якщо його немає — `1` |
| `record_id` | Для запису: `<record_type>/<NNNN>`, де `record_type` — `review-report` (SPEC_REVIEW) або `applicability-decision` (IMPACT_ANALYSIS), а `NNNN` — наступний номер типу в Story (`artifact-rules.md` §2): найбільший номер `<record_type>/…` у `refs` подій `stage.completed` History + 1, чотири цифри; якщо таких немає — `0001` |
| `output` | Шлях вихідного типу за реєстром із префіксом пілоту (розділ 3). Для SPEC_REVIEW — запис `records/review-report/<NNNN>.md`. Для IMPACT_ANALYSIS — документ `artifacts/impact-analysis/v<version>.md`. Для SOLUTION_DESIGN — `artifacts/solution-design/v<version>.md` |
| `record_output` | Лише IMPACT_ANALYSIS: `records/applicability-decision/<NNNN>.md` під коренем пілоту |
| `clarification`, `spec` | Шляхи поточних версій `clarification-record` і `story-spec` (для SPECIFICATION — лише `clarification`; для IMPACT_ANALYSIS і SOLUTION_DESIGN — лише `spec`) |
| `impact` | Лише SOLUTION_DESIGN і лише за розділом 7.2, крок 7 (`APPLICABLE`): шлях поточної версії `impact-analysis` |
| `previous_reports` | Для SPEC_REVIEW: шляхи попередніх `review-report` цієї стадії з History (`stage.completed` стадії SPEC_REVIEW); немає — не передавай |
| `previous`, `reason` | SPECIFICATION: лише для `version > 1` — за розділом 7.3.1. IMPACT_ANALYSIS — за розділом 7.3.2 |
| `reason` (SOLUTION_DESIGN) | Лише якщо тригер входу (розділ 7.2, крок 6) — `human_decision.applied` стадії `SOLUTION_DESIGN` з `details.decision: RESUME` (розділ 8.7.3). Значення — `details.reason` цієї події дослівно. Інакше не передавай. `version` завжди `1` |

`focus` для SPEC_REVIEW не передається: Orchestrator не підказує гіпотез чи findings.

#### 7.3.1. Повторний вхід після повернення (`version > 1`)

Підтримується лише SPECIFICATION. **Тригер** — остання подія History з `details.to_stage`, тобто остання подія маршруту: `stage.completed` або `human_decision.applied` рішення gate. Події спроб (`stage.started`, `stage.failed`) тригер не змінюють. Тригер має `to_stage: SPECIFICATION` і є одним із двох видів:

| Тригер | Умови | `reason` |
|---|---|---|
| Loopback від SPEC_REVIEW | `stage.completed` стадії `SPEC_REVIEW` з `verdict: CHANGES_REQUIRED`; `refs` містять запис-тригер `review-report/<NNNN>`, зареєстрований у `records.review-report.SPEC_REVIEW`; `last_stage_verdict: CHANGES_REQUIRED` | `"SPEC_REVIEW CHANGES_REQUIRED: <record_id> (<шлях звіту>); BLOCKING: <F-NNN, …>; NON_BLOCKING: <F-NNN, …>"` — ID `OPEN` findings з front matter звіту-тригера, без переказу їхнього змісту |
| REQUEST_CHANGES на HUMAN_SPEC_APPROVAL (розділ 8.4) | `human_decision.applied` стадії `HUMAN_SPEC_APPROVAL` з `actor.kind: human`, `details.decision: REQUEST_CHANGES` і непорожнім `details.reason`; `last_stage_verdict: PASS` (Gate Decision verdict не змінює). BLOCKING findings не вимагаються | `"HUMAN_SPEC_APPROVAL REQUEST_CHANGES (<actor.id>, <ts>): <details.reason дослівно>"` |

Не виконано, або `current_stage: CLARIFICATION` з `version > 1` — нічого не змінюй, поясни і зупинись. **Обмеження MVP:** повторний вхід у CLARIFICATION (нова версія через інтерв'ю, Staleness `story-spec` і повторне проходження SPECIFICATION за Rework Protocol, `workflow-state-model.md` §14) не реалізовано. Повернення не перенаправляється на іншу стадію.

Параметри (`specification/SKILL.md` §2):

| Параметр | Значення |
|---|---|
| `previous` | Шлях поточної зареєстрованої версії `story-spec` (`v<N-1>`). Вона не змінюється |
| `reason` | За видом тригера (таблиця вище) |
| `version`, `output` | За загальним правилом: `v<N>` — новий файл |
| `clarification` | Поточна версія `clarification-record` |

Findings — діагностика reviewer, а причина людини — вказівка, що виправити. Ні те, ні інше не є джерелом продуктових рішень: вони й далі беруться лише з `clarification-record` (D-03).

#### 7.3.2. IMPACT_ANALYSIS: перший і повторний вхід (D-4)

Кожен вхід в IMPACT_ANALYSIS — нова оцінка застосовності і новий `applicability-decision` (AP-04, AP-08; `impact-analysis/SKILL.md` §3). Повторний вхід — повернення від SOLUTION_DESIGN: автоматичний Loopback без `crosses_gates` (розділ 7.8).

**`version`, `previous` і `output`** — за загальним правилом документа:

| Стан `artifacts.impact-analysis` | `version` | `previous` |
|---|---|---|
| немає (перший вхід або всі попередні оцінки — `NOT_APPLICABLE`) | `1` | не передається |
| є версія `N`, чинна чи історична (розділ 7.1, «Умовний вхід») | `N+1` | шлях `v<N>`. Вона не змінюється |

`output` — `artifacts/impact-analysis/v<version>.md`. Файл створюється лише для `PASS`. Інакше `version` наступної спроби лишається тим самим.

**`reason`** — причини поточного входу, дослівно, кожна окремим рядком у порядку History. Збираються від останньої події, яка привела Story в IMPACT_ANALYSIS з іншої стадії (тригер з розділу 7.2, крок 6, — APPROVE або loopback від SOLUTION_DESIGN), до останньої події з `to_stage: IMPACT_ANALYSIS`:

| Подія | Рядок `reason` |
|---|---|
| `stage.completed` SOLUTION_DESIGN з `verdict: CHANGES_REQUIRED`, `loopback_target: IMPACT_ANALYSIS` | `"SOLUTION_DESIGN CHANGES_REQUIRED (attempt <N>, <ts>): <details.summary дослівно>"` |
| `human_decision.applied` IMPACT_ANALYSIS з `decision: RESUME` | `"LOOPBACK_CONFIRMATION RESUME (<actor.id>, <ts>): <details.reason дослівно>"` |

Якщо рядків немає (перший вхід після APPROVE без `RESUME`), `reason` не передається. Findings SOLUTION_DESIGN — діагностика, а не джерело вимог (D-03).

Попередні версії артефакта, записи і події History не змінюються. Повторний вхід лічильників не змінює: Quality Loops враховано під час `CHANGES_REQUIRED` SOLUTION_DESIGN (розділ 7.8). Approval HUMAN_SPEC_APPROVAL не зачіпається, бо `story-spec` і `clarification-record` не змінюються.

**Незареєстрований Working Draft.** Перед стартом будь-якої спроби перевір, чи існує файл `output` (для IMPACT_ANALYSIS — також `record_output`), якого немає в `state.yaml` → `artifacts` чи `records`. Якщо існує — це Working Draft попередньої спроби (`artifact-rules.md` §3, п. 1). Не перезаписуй, не видаляй і не реєструй його. Нічого не змінюй у стані: покажи шлях, поясни, що знайдено незареєстрований Working Draft, і зупинись. Рішення ухвалює людина поза Skill.

> **Тимчасове обмеження MVP.** Контракт дозволяє повторній спробі перезаписати Working Draft. Цей Skill свідомо цього не робить, доки не реалізовано Recovery. Контракт Working Draft не змінюється.

### 7.4. Старт спроби

Один крок Orchestrator:

- `state.yaml`: `execution_state: RUNNING`, `updated_at`;
- `history.jsonl`: подія `stage.started`.

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"orchestrator","id":"story-orchestrator"},"event":"stage.started","stage":"SPECIFICATION","refs":["clarification-record@1"],"details":{"attempt":1,"version":1,"output":"docs/pilots/orchestrator/US-001/artifacts/story-spec/v1.md","mode":"pilot"}}
```

`refs` — входи-артефакти спроби (`<type>@<version>`); для CLARIFICATION — `[]`. Для SPEC_REVIEW у `details` замість `version` — `record_id`. Для IMPACT_ANALYSIS `refs: [story-spec@<N>]`, а для повторного входу з `version > 1` — ще й `impact-analysis@<N-1>`. `details` містять `version`, `output`, `record_id` і `record_output`, а також `previous` і `reason`, якщо вони є (розділ 7.3.2). Для SOLUTION_DESIGN `refs` складаються з `story-spec@<N>`, `impact-analysis@<M>` (лише якщо передано `impact`) і чинного `applicability-decision` IMPACT_ANALYSIS, який визначив вхід. `details` містять `version`, `output` і `reason`, якщо він є. Для повторного входу в SPECIFICATION (7.3.1) `refs` додатково містять попередню версію (`story-spec@1`), а для loopback від SPEC_REVIEW — і запис-тригер (`review-report/0001`). `details` містять `previous` і `reason`. Для тригера «рішення людини» запису немає: рішення ідентифікує `reason` (`actor.id`, `ts`).

### 7.5. Виконання стадії

Stage Skills мають `disable-model-invocation: true`, тому їх не викликають інструментом Skill: виконавець читає `SKILL.md` і діє за ним. Поки виконується стадія, діють лише правила Stage Skill. Orchestrator не підказує рішень і змінює стан лише там, де це сказано нижче.

Стадія завершується result envelope (`workflow-state.md` §6). Без envelope результату немає. Людина може зупинити стадію — спроба `INTERRUPTED`. Технічний збій інструмента чи середовища — `FAILED`. Ні те, ні інше не є verdict `BLOCKED` (`workflow-state-model.md` §3).

#### 7.5.1. CLARIFICATION — inline

1. Прочитай `.claude/skills/clarification/SKILL.md` повністю.
2. Виконай його **inline в основній сесії** з явними аргументами:
   `story=<ID> output=<output> version=<version> attempt=<attempt> mode=pilot`.
   `source` не передавай: Skill бере `story-source` з реєстру.
3. Очікування відповідей людини фіксуй за розділом 7.5.2.

#### 7.5.2. Очікування відповідей людини (CLARIFICATION)

Очікування відповідей під час CLARIFICATION — `AWAITING_HUMAN`, а не `BLOCKED` (`workflow-state-model.md` §3, §4). Для кожного раунду інтерв'ю:

**Перед тим як показати раунд** — один крок Orchestrator:

- `state.yaml`: `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`, `updated_at` і
  ```yaml
  pending_human_decision:
    kind: CLARIFICATION_ANSWERS
    stage: CLARIFICATION
    options: []           # відповіді вільні, варіантів рішення немає
    package: []           # зареєстрованих артефактів для перегляду ще немає
    requested_at: <date -u>
  ```
- подія `human_decision.requested`: `actor` — orchestrator, `details: { attempt, round, questions: [CQ-NN, …] }`.

Потім покажи раунд і чекай відповіді людини в цій самій сесії.

**Після відповіді людини** — один крок Orchestrator:

- `state.yaml`: `story_status: ACTIVE`, `execution_state: RUNNING`, `pending_human_decision: null`, `updated_at`;
- подія `human_decision.applied`: `actor: { kind: human, id: <decided_by> }`, `details: { attempt, round, answered: [CQ-NN, …], deferred: [CQ-NN, …] }`.

У History потрапляють лише ID питань. Зміст відповідей фіксує лише `clarification-record` (`stage-io-map.yaml` → `CLARIFICATION`), транскрипт чату не зберігається. `decided_by` Stage Skill запитує один раз. Доки його немає, подію `human_decision.applied` не записуй: спершу отримай ідентифікатор.

Якщо людина зупиняє інтерв'ю на очікуванні, спершу `pending_human_decision: null` і `story_status: ACTIVE`, далі — невдача `INTERRUPTED` за розділом 7.7.

#### 7.5.3. SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS і SOLUTION_DESIGN — subagent

1. Перед запуском запам'ятай `git status --porcelain --untracked-files=all` (знімок А) і `git hash-object` кожного зареєстрованого файлу артефакта чи запису пілоту (файли пілоту не відстежуються Git, тож їхні зміни `git status` не покаже).
2. Запусти **рівно один** subagent для поточної стадії інструментом Agent з `subagent_type: general-purpose`. Середовище може запускати subagent у фоні. Це не змінює порядку: виконання послідовне (кроки 3–5). Prompt — лише текст нижче для стадії, без контексту основної сесії (чат, інтерв'ю, міркування й самооцінка producer-стадії, думка Orchestrator про артефакти не передаються).

   SPECIFICATION:

   ```text
   Ти виконуєш стадію SPECIFICATION пілотної Story поза зареєстрованим Workflow.
   Прочитай .claude/skills/specification/SKILL.md повністю і виконай його точно.
   Аргументи: story=<ID> clarification=<clarification> output=<output> version=<version> attempt=<attempt> mode=pilot [previous=<previous> reason="<reason>"]
   Людини в цьому контексті немає: нічого не питай.
   [Якщо reason — від SPEC_REVIEW:] Звіт із reason прочитай сам. Findings — діагностика reviewer, а не джерело продуктових рішень: не вигадуй рішень, яких немає в clarification-record.
   [Якщо reason — від HUMAN_SPEC_APPROVAL:] reason — дослівна причина, з якою людина повернула специфікацію на gate. Вона вказує, що виправити, але не є джерелом продуктових рішень: не вигадуй рішень, яких немає в clarification-record.
   [Якщо є reason:] Створи нову версію в output; попередню версію не змінюй.
   Пиши лише файл output. Не змінюй state.yaml, history.jsonl та інші файли. Не запускай інші Skills чи subagents.
   Останнім у відповіді поверни result envelope — YAML-блок `result:` за розділом 6 Stage Skill.
   ```

   SPEC_REVIEW:

   ```text
   Ти виконуєш стадію SPEC_REVIEW пілотної Story поза зареєстрованим Workflow як незалежний reviewer.
   Прочитай .claude/skills/spec-review/SKILL.md повністю і виконай його точно, включно з обмеженнями з його front matter.
   Аргументи: story=<ID> spec=<spec> clarification=<clarification> output=<output> record_id=<record_id> attempt=<attempt> mode=pilot [previous_reports=<шляхи>]
   Входи читай самостійно з файлів. Інших відомостей про ці артефакти чи їхнього автора в тебе немає.
   Людини в цьому контексті немає: нічого не питай.
   Пиши лише файл output. Не змінюй перевірені артефакти, state.yaml, history.jsonl та інші файли. Bash — лише git log і date. Не запускай інші Skills чи subagents.
   Останнім у відповіді поверни result envelope — YAML-блок `result:` за розділом 6 Stage Skill.
   ```

   IMPACT_ANALYSIS:

   ```text
   Ти виконуєш стадію IMPACT_ANALYSIS пілотної Story поза зареєстрованим Workflow.
   Прочитай .claude/skills/impact-analysis/SKILL.md повністю і виконай його точно.
   Аргументи: story=<ID> spec=<spec> output=<output> version=<version> record_output=<record_output> record_id=<record_id> attempt=<attempt> mode=pilot [previous=<previous>] [reason="<reason>"]
   Людини в цьому контексті немає: нічого не питай.
   [Якщо є reason:] reason — причини цього входу: findings SOLUTION_DESIGN і/або дослівна причина людини, з якою вона відхилила запропонований Loopback (RESUME). Це не джерело вимог: не вигадуй вимог, яких немає в story-spec. Застосовність оціни заново.
   [Якщо є previous:] previous — попередня версія impact-analysis. Не змінюй її; для PASS створи нову версію в output.
   Пиши лише файл record_output і, лише для PASS, файл output. Не змінюй state.yaml, history.jsonl, story-spec, код та інші файли. Bash — лише git log, git status, git ls-files і date. Не запускай інші Skills чи subagents.
   Останнім у відповіді поверни result envelope — YAML-блок `result:` за розділом 6 Stage Skill.
   ```

   SOLUTION_DESIGN:

   ```text
   Ти виконуєш стадію SOLUTION_DESIGN пілотної Story поза зареєстрованим Workflow.
   Прочитай .claude/skills/solution-design/SKILL.md повністю і виконай його точно.
   Аргументи: story=<ID> spec=<spec> [impact=<impact>] output=<output> version=<version> attempt=<attempt> mode=pilot [reason="<reason>"]
   Людини в цьому контексті немає: нічого не питай.
   [Якщо impact не передано:] IMPACT_ANALYSIS для цієї Story незастосовна. Не шукай і не використовуй impact-analysis, навіть якщо такий файл існує.
   [Якщо є reason:] reason — дослівна причина, з якою людина відхилила запропонований Loopback (RESUME). Вона не є джерелом вимог: не вигадуй вимог, яких немає в story-spec.
   Пиши лише файл output і лише для PASS. Не змінюй state.yaml, history.jsonl, story-spec, impact-analysis, код та інші файли. Bash — лише git log, git status, git ls-files і date. Не запускай інші Skills чи subagents.
   Останнім у відповіді поверни result envelope — YAML-блок `result:` за розділом 6 Stage Skill.
   ```

3. Дочекайся **фактичного завершення** subagent і його фінальної відповіді (для фонового запуску — сповіщення про завершення). Запуск у фоні не означає, що subagent завершився. Поки він працює:
   - не обробляй його результат і не вгадуй його;
   - не змінюй `state.yaml`, `history.jsonl` і файли пілоту;
   - не запускай інших subagents і стадій.
4. Лише після завершення знову візьми `git status --porcelain --untracked-files=all` (знімок Б) і `git hash-object` тих самих файлів. Різниця А→Б має складатися рівно з нових файлів, що відповідають елементам `outputs` envelope; якщо `outputs` порожні — різниці немає. Для SPECIFICATION, SPEC_REVIEW і SOLUTION_DESIGN це файл `output`. Для IMPACT_ANALYSIS `{ record: <record_id> }` відповідає `record_output`, а `{ type: impact-analysis, version }` — `output`. Хеші зареєстрованих файлів не змінюються (`artifact-rules.md` §3, п. 2). Інша зміна — результат відхилено (`INVALID_RESULT`) з переліком файлів. Нічого не відкочуй: рішення ухвалює людина.
5. Envelope — останній блок `result:` у відповіді subagent. Збій інструмента Agent — `FAILED`. Відповідь без envelope — `INVALID_RESULT`.

### 7.6. Перевірка result envelope

Orchestrator приймає результат, лише коли виконано **все** (`workflow-state.md` §6):

- є блок `result` з полями `story_id`, `stage`, `attempt`, `verdict`, `outputs`, `loopback_target`, `blocker`, `summary`;
- `story_id`, `stage`, `attempt` збігаються з параметрами спроби;
- `verdict` дозволений для стадії за `stage-map.yaml`: `optional: false` → не `NOT_APPLICABLE`; `CHANGES_REQUIRED` — лише якщо `loopbacks` стадії не порожні. CLARIFICATION: `PASS | BLOCKED`. SPECIFICATION і SPEC_REVIEW: `PASS | CHANGES_REQUIRED | BLOCKED`. IMPACT_ANALYSIS (`optional: true`): `PASS | NOT_APPLICABLE | CHANGES_REQUIRED | BLOCKED`. SOLUTION_DESIGN (`optional: false`): `PASS | CHANGES_REQUIRED | BLOCKED`;
- `loopback_target`: для `CHANGES_REQUIRED` — ціль із `loopbacks` стадії (SPECIFICATION → `CLARIFICATION`; SPEC_REVIEW → `SPECIFICATION` або `CLARIFICATION`; IMPACT_ANALYSIS → `SPECIFICATION` або `CLARIFICATION`, обидві з `crosses_gates`; SOLUTION_DESIGN → `IMPACT_ANALYSIS` без `crosses_gates` або `SPECIFICATION` чи `CLARIFICATION` з `crosses_gates`); інакше `null`;
- для SPEC_REVIEW — розділ 7.6.1, для IMPACT_ANALYSIS — розділ 7.6.2 замість правил документа нижче;
- для `PASS`: `blocker: null`; `outputs` — рівно `[{ type: <вихідний тип>, version: <version> }]`; файл `output` існує; його front matter (`artifact-rules.md` §1):
  - `artifact_type`, `story_id`, `version` збігаються;
  - `produced_by: { stage: <current_stage>, attempt: <attempt> }`;
  - `created_at` — ISO-8601 UTC;
  - `inputs` не порожні, кожне джерело має `source`, `path`, `commit`;
  - для SPECIFICATION: `inputs` містить `{ type: clarification-record, version: <поточна версія> }`; `story_type` — одне зі `story_types`; `acceptance_criteria` — непорожній список, і кожен ID є в тілі;
  - для SOLUTION_DESIGN (`solution-design/SKILL.md` §5):
    - `inputs` містить `{ type: story-spec, version: <поточна версія> }`;
    - `{ type: impact-analysis, version: <версія з impact> }` є в `inputs` тоді й лише тоді, коли передано `impact` (розділ 7.2, крок 7). Посилання на будь-яку іншу версію `impact-analysis`, зокрема історичну після `NOT_APPLICABLE`, — порушення;
    - `source` кожного джерела — ключ із `sources` реєстру (`architecture-foundation`, `product-foundation`, `codebase`); `path` — не `.` (D-3);
    - кожен ID `AC-NN` з `acceptance_criteria` поточної `story-spec` є в тілі (трасування AC);
    - `version: 1`, тож `revision_reason` немає;
  - для SPECIFICATION, якщо `version > 1`: є `revision_reason`, а `inputs` містить `{ type: story-spec, version: <N-1> }` (`artifact-rules.md` §1, `specification/SKILL.md` §5). `revision_reason` посилається на кожен `BLOCKING` finding звіту-тригера (loopback від SPEC_REVIEW) або на рішення людини `HUMAN_SPEC_APPROVAL REQUEST_CHANGES` (тригер — рішення людини; посилань на findings не вимагається, розділ 7.3.1);
  - немає статусу, Currency чи approval;
  - перший рядок тіла — пілотний рядок Stage Skill;
- для `CHANGES_REQUIRED`: `outputs: []`, `blocker: null`;
- для `BLOCKED`: `outputs: []`, `blocker` має `reason`, `details`, `needed`.

Невиконана умова → результат відхилено: `INVALID_RESULT` з переліком порушень (розділ 7.7).

#### 7.6.1. Envelope і `review-report` SPEC_REVIEW

- `outputs`: для `PASS` і `CHANGES_REQUIRED` — рівно `[{ record: <record_id> }]`; для `BLOCKED` — `[{ record: <record_id> }]` (перевірку виконано) або `[]` (перевірки не було, `spec-review/SKILL.md` §6);
- `blocker`: `null` для `PASS` і `CHANGES_REQUIRED`; для `BLOCKED` — `reason`, `details`, `needed`;
- якщо `outputs` не порожні — файл `output` існує, і його front matter (`artifact-rules.md` §2):
  - `record_type: review-report`, `record_id` і `story_id` збігаються;
  - `produced_by: { stage: SPEC_REVIEW, attempt: <attempt> }`;
  - `created_at` — ISO-8601 UTC;
  - `subjects` — рівно поточні версії `{ type: story-spec, version }` і `{ type: clarification-record, version }`;
  - `verdict` збігається з envelope;
  - кожен finding має `id` `F-NNN`, `severity` ∈ `BLOCKING | NON_BLOCKING`, `summary`, `status` ∈ `OPEN | RESOLVED | DISPOSITIONED` (`artifact-rules.md` §6);
  - `PASS` — немає `OPEN` `BLOCKING`; `CHANGES_REQUIRED` — є хоча б один `OPEN` `BLOCKING`;
  - кожен `OPEN` finding з `previous_reports` присутній з новим станом;
  - перший рядок тіла — пілотний рядок Stage Skill.

#### 7.6.2. Envelope, `applicability-decision` і `impact-analysis` IMPACT_ANALYSIS

Правила — `impact-analysis/SKILL.md` §5–6, `artifact-rules.md` §1–2, `stage-map.yaml` → `APPLICABILITY_IMPACT_ANALYSIS` (AP-01, AP-03).

**`outputs` і `blocker`** (порядок елементів не важливий):

| Verdict | `outputs` | `blocker` |
|---|---|---|
| `PASS` | рівно `{ type: impact-analysis, version: <version> }` і `{ record: <record_id> }` | `null` |
| `NOT_APPLICABLE` | рівно `[{ record: <record_id> }]` | `null` |
| `CHANGES_REQUIRED` | рівно `[{ record: <record_id> }]` | `null` |
| `BLOCKED` | `[{ record: <record_id> }]` (оцінку завершено) або `[]` | `reason`, `details`, `needed` |

Файл `output` (`impact-analysis`) існує лише для `PASS`. Для інших verdicts його немає.

**`applicability-decision`** — якщо `outputs` містить запис, файл `record_output` існує, і його front matter:

- `record_type: applicability-decision`; `record_id` і `story_id` збігаються;
- `produced_by: { stage: IMPACT_ANALYSIS, attempt: <attempt> }`; `created_at` — ISO-8601 UTC;
- `subjects` — рівно `[{ type: story-spec, version: <поточна версія> }]`;
- `target_stage: IMPACT_ANALYSIS`, `kind: ENTRY`, `result` ∈ `APPLICABLE | NOT_APPLICABLE`;
- `triggers` — рівно ключі `IA-01`…`IA-05`, кожен зі `status` ∈ `met | not_met` і непорожнім `ref`;
- `result: NOT_APPLICABLE` ⇔ усі тригери `not_met`; `result: APPLICABLE` ⇔ хоча б один `met` (AP-01, AP-03);
- `basis` не порожній. Він містить `{ type: story-spec, version: <поточна версія> }` і факт `story_type`, який збігається зі `state.yaml` → `story_type`. Жоден елемент `basis` не посилається на корінь репозиторію (`path: .`);
- полів `verdict` і `findings` немає;
- узгодженість з verdict: `PASS` і `CHANGES_REQUIRED` — лише з `result: APPLICABLE`; `NOT_APPLICABLE` — лише з `result: NOT_APPLICABLE`; `BLOCKED` — будь-який `result`;
- перший рядок тіла — пілотний рядок Stage Skill.

**`impact-analysis`** — лише для `PASS`. Файл `output` існує, і його front matter (`artifact-rules.md` §1):

- `artifact_type: impact-analysis`, `story_id`, `version` збігаються; `produced_by: { stage: IMPACT_ANALYSIS, attempt: <attempt> }`; `created_at` — ISO-8601 UTC;
- `inputs` містить `{ type: story-spec, version: <поточна версія> }`. Кожне джерело має `source`, `path` і `commit`; `path` — не `.` (D-3);
- `version: 1` — `revision_reason` немає, і посилань на інші версії `impact-analysis` в `inputs` немає;
- `version > 1` (розділ 7.3.2) — є непорожній `revision_reason`, а `inputs` містить рівно одне посилання на тип `impact-analysis`: `{ type: impact-analysis, version: <N-1> }` (lineage, `artifact-rules.md` §1, §3);
- статусу, Currency чи approval немає;
- перший рядок тіла — пілотний рядок Stage Skill.

### 7.7. Невдача спроби

`INTERRUPTED`, `FAILED` або `INVALID_RESULT`, один крок:

- `execution_state` — відповідне значення, `retry.technical_attempts.<stage>` + 1, `updated_at`;
- подія `stage.failed` з `details: { attempt, execution_state, reason }`;
- файл спроби не реєструється: він лишається Working Draft;
- якщо лічильник досяг `stage-map.yaml` → `retry_policy.defaults.technical_attempts`: також `story_status: BLOCKED`, `blocker: { reason: "Retry Budget Exhausted", source: ORCHESTRATOR, details, needed: "рішення людини через Recovery" }` і подія `story.blocked`.

### 7.8. Застосування прийнятого результату

Маршрут — лише зі `stage-map.yaml` (`workflow-state.md` §7). Один крок: спершу `state.yaml` (розділ 7.10), потім події в `history.jsonl`. В усіх випадках `retry.technical_attempts.<stage>` видаляється (скидання після прийнятого результату, `workflow-state-model.md` §10) і оновлюється `updated_at`.

**Запис SPEC_REVIEW.** Якщо `outputs` містить запис, він реєструється за будь-якого verdict: `records.review-report.SPEC_REVIEW: <record_id>` (`workflow-state.md` §3). `refs` подій `stage.completed` містять `<record_id>`.

**Запис IMPACT_ANALYSIS.** Якщо `outputs` містить запис, він реєструється за будь-якого verdict: `records.applicability-decision.IMPACT_ANALYSIS: { ENTRY: <record_id> }` (`workflow-state.md` §3). Попереднє значення `ENTRY` замінюється, бо чинним стає останній запис за призначенням. Його файл лишається без змін. `refs` подій `stage.completed` містять `<record_id>`.

**`PASS`** SPEC_REVIEW → `HUMAN_SPEC_APPROVAL` (`next` стадії, тип `human_gate`). Gate не виконується; Orchestrator лише фіксує, що Story чекає на людину (`workflow-state-model.md` §3: для `human_gate` — `AWAITING_HUMAN`):

- `status: REVIEWED` для `story-spec` і `clarification-record` — лише тих версій, що є в `subjects` звіту (`workflow-state.md` §3, «Статуси артефактів»);
- `last_stage_verdict: PASS`, `current_stage: HUMAN_SPEC_APPROVAL`, `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`;
- `pending_human_decision` (`workflow-state.md` §3):
  ```yaml
  pending_human_decision:
    kind: GATE
    stage: HUMAN_SPEC_APPROVAL
    options: [APPROVE, REQUEST_CHANGES, CANCEL]   # decisions gate; RECONFIRM у MVP не пропонується (README.md)
    package:
      - { type: clarification-record, version: <N> }
      - { type: story-spec, version: <N> }
      - { record: <record_id> }
    requested_at: <date -u>
  ```
- `approvals` не змінюються;
- події `stage.completed` (`to_stage: HUMAN_SPEC_APPROVAL`) і `human_decision.requested` (`stage: HUMAN_SPEC_APPROVAL`, `refs` — елементи `package`).

Далі `next` зупиняється на кроці 7.2 (`story_status` не `ACTIVE`). Рішення gate `next` не приймає і не імітує: його ухвалює людина командою `decide` (розділ 8).

**`PASS`** CLARIFICATION і SPECIFICATION → `next` стадії (CLARIFICATION → `SPECIFICATION`, SPECIFICATION → `SPEC_REVIEW`):

- `artifacts.<вихідний тип>: { version: <version>, status: DRAFT, currency: CURRENT }` — Version Status нової версії до review (`workflow-state-model.md` §5). Попередня версія стає `SUPERSEDED` без окремого запису (`workflow-state.md` §3); її файл лишається без змін;
- для SPECIFICATION: `story_type` — з front matter `story-spec` (`workflow-state.md` §3);
- `last_stage_verdict: PASS`;
- `current_stage: <next>`, `execution_state: READY`, `story_status: ACTIVE`;
- подія:

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"orchestrator","id":"story-orchestrator"},"event":"stage.completed","stage":"SPECIFICATION","refs":["story-spec@1"],"details":{"attempt":1,"verdict":"PASS","outputs":["story-spec@1"],"to_stage":"SPEC_REVIEW","summary":"<summary з envelope>","mode":"pilot"}}
```

**`PASS`** і **`NOT_APPLICABLE`** IMPACT_ANALYSIS → `SOLUTION_DESIGN` (`next` стадії; для `NOT_APPLICABLE` — RR-02):

- для `PASS`: `artifacts.impact-analysis: { version: <version>, status: DRAFT, currency: CURRENT }`. На повторному вході попередня версія стає `SUPERSEDED` без окремого запису, а її файл лишається без змін;
- для `NOT_APPLICABLE` `artifacts` не змінюються. Якщо там є `impact-analysis` з попереднього застосовного оцінювання, він лишається як історична версія без зміни `status` і `currency`. Входом він більше не є, бо чинне рішення — `NOT_APPLICABLE` (розділ 7.1, «Умовний вхід»);
- запис — за правилом «Запис IMPACT_ANALYSIS» вище;
- `last_stage_verdict: <verdict>`;
- `current_stage: SOLUTION_DESIGN`, `execution_state: READY`, `story_status: ACTIVE`;
- `artifacts` інших типів, `approvals` і `retry.quality_loops` не змінюються;
- подія `stage.completed` з `refs` — виходи (`impact-analysis@<version>`, якщо є, і `<record_id>`), `details: { attempt, verdict, outputs, to_stage: SOLUTION_DESIGN, summary, mode }`.

Наступний `next` виконає SOLUTION_DESIGN.

**`PASS`** SOLUTION_DESIGN → `API_DESIGN` (`next` стадії):

- `artifacts.solution-design: { version: <version>, status: DRAFT, currency: CURRENT }`;
- `last_stage_verdict: PASS`;
- `current_stage: API_DESIGN`, `execution_state: READY`, `story_status: ACTIVE`;
- `artifacts` інших типів, `records`, `approvals` і `retry.quality_loops` не змінюються;
- подія `stage.completed` з `refs: [solution-design@<version>]`, `details: { attempt, verdict: PASS, outputs, to_stage: API_DESIGN, summary, mode }`.

Наступний `next` зупиниться на кроці 7.2, п. 3: API_DESIGN ще не підтримується.

**`CHANGES_REQUIRED`** IMPACT_ANALYSIS і **`CHANGES_REQUIRED`** SOLUTION_DESIGN з `loopback_target` `SPECIFICATION` чи `CLARIFICATION`. Ці цілі мають `crosses_gates: [HUMAN_SPEC_APPROVAL]`, тож Loopback виконується лише після рішення людини (RR-04, `workflow-state-model.md` §6). Orchestrator його не виконує, лише просить підтвердження (D-1). Нижче `<stage>` — `IMPACT_ANALYSIS` або `SOLUTION_DESIGN`:

- для IMPACT_ANALYSIS: запис — за правилом «Запис IMPACT_ANALYSIS» вище. SOLUTION_DESIGN записів не створює;
- `last_stage_verdict: CHANGES_REQUIRED`; `artifacts` і `approvals` не змінюються;
- `retry.quality_loops` і `retry.repeated_failure` не змінюються. Quality Loops рахують повернення, а повернення ще немає (`workflow-state-model.md` §10). У MVP облік виконає `ROUTE_BACK`, коли його буде реалізовано; `RESUME` поверненням не є (рішення D-1);
- `current_stage` не змінюється (`<stage>`); `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`;
- `pending_human_decision` (`workflow-state.md` §3):
  ```yaml
  pending_human_decision:
    kind: LOOPBACK_CONFIRMATION
    stage: <stage>
    options: [ROUTE_BACK, RESUME, CANCEL]          # loopback_confirmation_decisions; ROUTE_BACK у MVP відхиляється (розділ 8.7.2)
    package:
      - { type: story-spec, version: <N> }        # погоджений артефакт, який змінив би Loopback
      - { record: <record_id> }                   # лише IMPACT_ANALYSIS: applicability-decision цієї спроби
    requested_at: <date -u>
  ```
- події:
  - `stage.completed` з `details: { attempt, verdict: CHANGES_REQUIRED, outputs, loopback_target, crosses_gates: [HUMAN_SPEC_APPROVAL], summary, mode }`, без `to_stage`;
  - `human_decision.requested` (`stage: <stage>`, `refs` — елементи `package`, `details: { kind: LOOPBACK_CONFIRMATION, options, loopback_target, mode }`).

Далі `next` зупиняється на кроці 7.2 (`story_status` не `ACTIVE`). Рішення ухвалює людина командою `decide` (розділ 8.7).

**`CHANGES_REQUIRED`** SPECIFICATION, SPEC_REVIEW і SOLUTION_DESIGN з `loopback_target: IMPACT_ANALYSIS` → `loopback_target`. Ці loopbacks не мають `crosses_gates`, тож рішення людини не потрібне (`stage-map.yaml` RR-03, RR-04):

- `last_stage_verdict: CHANGES_REQUIRED`; Version Status артефактів не змінюється;
- `retry.quality_loops.target.<loopback_target>` + 1 і `retry.quality_loops.initiator.<stage>` + 1 (`workflow-state-model.md` §10);
- для SPEC_REVIEW: `retry.repeated_failure.<F-NNN>` + 1 для кожного `OPEN` `BLOCKING` finding звіту (`workflow-state.md` §3);
- якщо будь-який лічильник досяг відповідного значення `retry_policy.defaults` (`quality_loops`, `repeated_failure`) — `story_status: BLOCKED`, `blocker: { reason: "Retry Budget Exhausted", source: ORCHESTRATOR, details, needed }`, `execution_state: COMPLETED`, `current_stage` не змінюється, події `stage.completed` і `story.blocked`;
- інакше `current_stage: <loopback_target>`, `execution_state: READY`, `story_status: ACTIVE`, подія `stage.completed` з `details: { attempt, verdict, loopback_target, to_stage, summary, mode }`.

Стадію-ціль не запускай. Наступний `next` виконає її повторний вхід за розділом 7.3.1, якщо це SPECIFICATION, або за розділом 7.3.2, якщо це IMPACT_ANALYSIS. Для CLARIFICATION він зупиниться (обмеження MVP, 7.3.1).

Повторний вхід лічильники не змінює: Quality Loops уже враховано під час `CHANGES_REQUIRED`.

**`BLOCKED`** → маршруту немає (`stage-map.yaml` RR-05):

- для IMPACT_ANALYSIS: запис, якщо він є в `outputs`, — за правилом «Запис IMPACT_ANALYSIS» вище;
- `execution_state: COMPLETED`, `last_stage_verdict: BLOCKED`, `story_status: BLOCKED`;
- `blocker: { reason, source: STAGE, details, needed }` з envelope;
- `current_stage` не змінюється;
- події `stage.completed` (verdict `BLOCKED`, без `to_stage`) і `story.blocked`.

### 7.9. Завершення

Після кроку 7.7 або 7.8 зупинись (Step Mode). Покажи підсумок командою `status`. Наступну стадію не запускай.

### 7.10. Безпечне оновлення `state.yaml`

- Не використовуй `sed` (чи інші рядкові заміни в shell), щоб вставляти або переписувати багаторядкові YAML-mappings.
- Змінюй `state.yaml` інструментом Edit (точна заміна фрагмента) або перезапиши файл повністю інструментом Write. Зберігай наявні відступи, порядок ключів і коментарі.
- Після запису прочитай `state.yaml` знову і перевір структуру: усі ключі розділу 4.1 на місці, вкладені mappings мають правильні відступи, нові значення там, де мають бути.
- Лише після цього дописуй подію в `history.jsonl`. Якщо перевірка не пройшла — не пиши подію, покажи людині розбіжність і зупинись.

**Текст людини** (`reason`, `comment`, `by`) записується дослівно: після розекранування значення точно збігається з аргументом. Не скорочуй, не перекладай і не «виправляй» його.

- **YAML** (`state.yaml`, front matter записів): завжди рядок у подвійних лапках. Екрануй `\` → `\\`, `"` → `\"`, перенесення рядка → `\n`, табуляцію → `\t`. Відсутній необов'язковий аргумент — `null` без лапок, а не рядок `"null"`.
- **JSON** (`history.jsonl`): рядок JSON з тими самими екрануваннями, інші керівні символи — `\uXXXX`. Відсутній необов'язковий аргумент — `null` без лапок. Кирилиця й інші символи Unicode лишаються як є. Подія — рівно один рядок.
- **Prompt subagent** (`reason` у 7.5.3): у подвійних лапках з тими самими екрануваннями.
- Після запису прочитай записане значення і переконайся, що воно не розірвало структуру: YAML-mapping цілий, нова подія — один рядок JSON з усіма полями `workflow-state.md` §5.

## 8. Command `decide`

Застосовує рішення людини для поточного `pending_human_decision`. Працює в основній сесії, без subagent. Підтримуються два види очікування:

| `pending_human_decision.kind` | Стадія | Рішення в MVP | Розділи |
|---|---|---|---|
| `GATE` | HUMAN_SPEC_APPROVAL | `APPROVE`, `REQUEST_CHANGES`, `CANCEL` | 8.2–8.5 |
| `LOOPBACK_CONFIRMATION` | IMPACT_ANALYSIS, SOLUTION_DESIGN | `RESUME`, `CANCEL`; `ROUTE_BACK` відхиляється без змін | 8.2, 8.5, 8.7 |

Рішення ухвалює людина, а Orchestrator лише перевіряє і застосовує його (`workflow-state.md` §1).

Команда неінтерактивна: рішення, ідентифікатор людини, ціль, причина і коментар приходять лише з аргументів. Orchestrator не веде діалогу про зміст артефактів, нічого не уточнює і не пропонує рішень. Gate погоджує наявні версії, а не створює нових продуктових рішень (D-03).

Рішення в пілоті — симуляція Human Gate. Воно діє лише в межах пілоту і не є погодженням чи скасуванням Story в офіційному Workflow.

### 8.1. Аргументи

| Аргумент | Рішення | Правило |
|---|---|---|
| `decision` | усі | Для `GATE`: `APPROVE`, `REQUEST_CHANGES` або `CANCEL` (`gate_decisions` у `stage-map.yaml`); `RECONFIRM` у MVP не пропонується (`docs/contracts/README.md`, «Відкладено»). Для `LOOPBACK_CONFIRMATION`: `ROUTE_BACK`, `RESUME` або `CANCEL` (`loopback_confirmation_decisions`). Інше значення — помилка |
| `by` | усі, обов'язковий | Непорожній ідентифікатор людини, яка ухвалила рішення. У пілоті це заявлений ідентифікатор, а не перевірена автентифікація. Orchestrator його не вигадує, не підставляє замість людини і не бере ідентифікатор агента чи Skill |
| `target` | лише `REQUEST_CHANGES`, обов'язковий | Стадія-ціль повернення (розділ 8.4.1) |
| `reason` | `REQUEST_CHANGES`, `CANCEL` і `RESUME`, обов'язковий | Непорожня причина. Записується дослівно за правилами тексту людини в розділі 7.10 (`workflow-policies.md` §3, §17). Для `RESUME` її дослівно отримує наступна спроба стадії (розділ 7.3) |
| `comment` | лише `APPROVE`, необов'язковий | Записується дослівно за правилами розділу 7.10. Не є джерелом вимог (розділ 8.3.1, крок 3) |

Якщо обов'язкового аргументу бракує, є аргумент, недозволений для цього рішення, або `decision` недозволене — поясни, як повторити команду, і зупинись. `state.yaml` і `history.jsonl` не змінюються.

### 8.2. Спільна перевірка

Для кожного рішення. Перевіряй усе за порядком. Перше порушення — поясни людині, що саме не так і де (файл, поле), і зупинись. Стан, History і файли не змінюються.

1. **Узгодженість** State, History і файлів — розділ 6. Розбіжність — як у `status`.
2. **Вид очікування.** `story_status: WAITING_FOR_HUMAN`, `execution_state: AWAITING_HUMAN`, `pending_human_decision.stage` = `current_stage`. Далі — за `pending_human_decision.kind`:
   - `LOOPBACK_CONFIRMATION` — перевірка розділу 8.7.1 замість кроків 3–4 нижче;
   - `GATE` — кроки 3–4 нижче;
   - інший kind або `null` — рішення цим Skill не підтримується: поясни і зупинись.
3. **Стан gate** (лише `GATE`):
   - `current_stage: HUMAN_SPEC_APPROVAL`; у `stage-map.yaml` ця стадія має `type: human_gate`;
   - `pending_human_decision.stage: HUMAN_SPEC_APPROVAL`;
   - `decision` є в `pending_human_decision.options` і в `decisions` gate у `stage-map.yaml`;
   - **пакет відповідає поточним версіям.** `pending_human_decision.package` містить рівно:
     - `{ type, version }` для кожного типу з `approval_scope` gate (`clarification-record`, `story-spec`), де `version` — поточна версія цього типу в `artifacts`;
     - `{ record: <record_id> }`, де `<record_id>` — `records.review-report.SPEC_REVIEW`.

   `refs` останньої події `human_decision.requested` збігаються з елементами `package`.
4. **Попередній approval** (лише `GATE`). `approvals.HUMAN_SPEC_APPROVAL` відсутній або `null`. Якщо запис уже є — це проходження gate після APPROVE. **Обмеження MVP:** таке проходження (перенесення в `previous`, інвалідація approval за Rework Protocol) не реалізовано. Нічого не змінюй, поясни і зупинись.

Час рішення: `date -u +%Y-%m-%dT%H:%M:%SZ`, одне значення для всіх полів часу і `ts` подій цього рішення.

Застосування — один крок Orchestrator (`workflow-state.md` §7): спершу файл запису, якщо він є (лише CANCEL), потім `state.yaml` за розділом 7.10, потім події в `history.jsonl`. `reason`, `comment` і `by` в усіх файлах серіалізуються за правилами тексту людини (розділ 7.10). В усіх рішеннях `actor` події `human_decision.applied` — людина з `by` (`{kind: human, id: <by>}`), `stage` — `pending_human_decision.stage` (HUMAN_SPEC_APPROVAL або IMPACT_ANALYSIS), `refs` — елементи `package`. Файли артефактів і review-звітів не змінюються (`artifact-rules.md` §3, п. 2).

### 8.3. APPROVE

#### 8.3.1. Перевірка

1. **Артефакти Scope.** Для кожного типу з `approval_scope`:
   - `status: REVIEWED`, `currency: CURRENT`, файл поточної версії існує;
   - Currency підтверджено заново за правилами розділу 7.2, крок 5: upstream-артефакти з `inputs` — поточні версії, джерела не змінилися після записаного `commit`, посилання на попередню версію — лише lineage.
2. **Review-звіт** `records.review-report.SPEC_REVIEW` (`artifact-rules.md` §2, §5):
   - front matter: `verdict: PASS`, `subjects` — рівно поточні версії обох типів Scope;
   - немає finding з `severity: BLOCKING` і `status: OPEN` (`workflow-policies.md` §3, WF-V2-01);
   - останній `stage.completed` у History — стадії `SPEC_REVIEW` з `verdict: PASS`, `to_stage: HUMAN_SPEC_APPROVAL`, і його `refs` містять цей `<record_id>`.
3. **Коментар не змінює вимог** (D-03, `workflow-state-model.md` §5, Approved Content Identity). Approval стосується лише вмісту версій зі Scope. `comment` не тлумачиться і не переноситься в артефакти. Якщо `comment` явно формулює нову вимогу, змінює чинну вимогу чи Acceptance Criterion або обирає варіант, якого немає в `story-spec`, — не застосовуй рішення. Поясни, що зміну вмісту вносить лише нова версія через `REQUEST_CHANGES` (розділ 8.4), і зупинись.

#### 8.3.2. Застосування

Маршрут — лише зі `stage-map.yaml`: `decisions.APPROVE` gate → `IMPACT_ANALYSIS`.

**`state.yaml`:**

- `artifacts.clarification-record.status` і `artifacts.story-spec.status`: `APPROVED` — лише для версій зі Scope (`workflow-state.md` §3, «Статуси артефактів»). `version` і `currency` не змінюються;
- `approvals.HUMAN_SPEC_APPROVAL` (`workflow-state.md` §4):

  ```yaml
  approvals:
    HUMAN_SPEC_APPROVAL:
      decision: APPROVE
      decided_by: <by>
      decided_at: <date -u>
      scope:                                   # усі елементи approval_scope gate, точні версії
        - { type: clarification-record, version: <N> }
        - { type: story-spec, version: <N> }
      package: [<record_id>]                   # review-report SPEC_REVIEW: переглянуто, не входить до Scope
      comment: "<comment>"                     # або null без лапок, якщо comment не передано (розділ 7.10)
      approval_state: VALID
      previous: []
  ```

- `pending_human_decision: null`;
- **Quality Loops** (`workflow-state-model.md` §10, «Скидання»): повне APPROVE скидає лічильники producer-стадій, артефакти яких входять до Scope, і review-стадій перед цим gate. Лічильник стадії скидається в обох ролях, в яких вона може бути: ціль і ініціатор повернення:
  - стадії: producer-стадії `CLARIFICATION` і `SPECIFICATION` (`produced_by` типів Scope в `artifact-registry.yaml`) і review-стадія `SPEC_REVIEW` (`reviews` цих стадій, `next: HUMAN_SPEC_APPROVAL`);
  - `retry.quality_loops.target`: видалити ключі `CLARIFICATION` і `SPECIFICATION`, якщо вони є;
  - `retry.quality_loops.initiator`: видалити ключі `SPECIFICATION` (ініціатор loopback на CLARIFICATION) і `SPEC_REVIEW`, якщо вони є. `CLARIFICATION` ініціатором не буває (`loopbacks: []`);
  - **не змінюються:** `retry.repeated_failure` (не скидається в межах Story), `retry.technical_attempts`, `retry.extensions`;
- `current_stage: IMPACT_ANALYSIS`, `story_status: ACTIVE`, `execution_state: READY`, `updated_at`;
- `last_stage_verdict` не змінюється: Gate Decision не є Stage Verdict (`workflow-state-model.md` §4);
- `blocker`, `records`, `git` не змінюються.

**`history.jsonl`** — подія `human_decision.applied` (`workflow-state.md` §5). Окремої події для скидання лічильників немає, тому попередні значення фіксуються в `details.retry_reset`: історія лічильників не губиться (`workflow-state-model.md` §10).

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"human","id":"<by>"},"event":"human_decision.applied","stage":"HUMAN_SPEC_APPROVAL","refs":["clarification-record@1","story-spec@2","review-report/0002"],"details":{"decision":"APPROVE","to_stage":"IMPACT_ANALYSIS","scope":["clarification-record@1","story-spec@2"],"approval_state":"VALID","comment":"<comment або null>","retry_reset":{"quality_loops":{"target":{"SPECIFICATION":1},"initiator":{"SPEC_REVIEW":1}}},"mode":"pilot"}}
```

- `retry_reset` — значення ключів, які були до скидання; ключа, якого не було, немає і тут.

IMPACT_ANALYSIS не запускай. Її виконає наступний `next` (розділ 7).

### 8.4. REQUEST_CHANGES

Повернення через Rework Protocol (`workflow-state-model.md` §14, механізм «REQUEST_CHANGES на Human Gate»; `stage-map.yaml` RR-11).

#### 8.4.1. Перевірка

1. **Ціль.** `target` входить до `decisions.REQUEST_CHANGES` gate у `stage-map.yaml` (`SPECIFICATION` або `CLARIFICATION`). Інша стадія — маршрут не оголошено (RR-01): поясни і зупинись.
2. **Ціль підтримується в MVP.** Поточний Orchestrator виконує лише `target=SPECIFICATION`. Для `target=CLARIFICATION` відхили команду до будь-яких змін: `state.yaml`, `history.jsonl` і файли не змінюються. Поясни людині таке:
   - маршрут `REQUEST_CHANGES → CLARIFICATION` оголошено в авторитетному Workflow (`stage-map.yaml`), але цей Orchestrator MVP його не виконує, бо повторний вхід у CLARIFICATION не реалізовано (розділ 7.3.1);
   - зараз підтримується `target=SPECIFICATION`;
   - `target=CLARIFICATION` стане доступним, коли буде реалізовано повторний вхід у CLARIFICATION.

   Не перенаправляй таке рішення на SPECIFICATION. Змінити ціль може лише людина новою командою.
3. **Причина.** `reason` непорожня. Orchestrator її не тлумачить, не скорочує і не доповнює. Причина каже, що виправити, але не є джерелом продуктових рішень: їх і далі містить лише `clarification-record` (D-03). Якщо виправлення потребує нового продуктового рішення, його вносить лише нова версія `clarification-record`, а це в MVP не підтримується (крок 2). `target=SPECIFICATION` такого рішення не замінює. Людина може залишити Story на gate або обрати `CANCEL`.

#### 8.4.2. Застосування

Маршрут — `target` з `decisions.REQUEST_CHANGES`. У MVP це лише `SPECIFICATION` (розділ 8.4.1, крок 2). Перенаправлення на іншу стадію немає.

**`state.yaml`:**

- `current_stage: <target>`, `story_status: ACTIVE`, `execution_state: READY`, `pending_human_decision: null`, `updated_at`;
- `last_stage_verdict` не змінюється (лишається `PASS` SPEC_REVIEW): Gate Decision не є Stage Verdict (`workflow-state-model.md` §4);
- `retry` не змінюється: REQUEST_CHANGES не зараховується до агентних Quality Loops і не скидає лічильників (`workflow-state-model.md` §10);
- `artifacts` не змінюються: Version Status не повертається назад, нових версій поки немає, тож Currency теж не змінюється (Rework Protocol, кроки 4–5, діють, коли стадія-ціль створить нову версію: розділ 7.8, `PASS`);
- `approvals` не змінюються: попереднього approval немає (розділ 8.2, крок 4), тож інвалідувати нічого (Rework Protocol, крок 6);
- `records`, `blocker`, `git` не змінюються.

**`history.jsonl`** — подія `human_decision.applied`. Вона ж є подією входу в Rework Protocol (§14, крок 3): механізм — `decision`, джерело — `stage`, ціль — `to_stage`, причина — `reason`, хто — `actor`. Окремої події немає.

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"human","id":"<by>"},"event":"human_decision.applied","stage":"HUMAN_SPEC_APPROVAL","refs":["clarification-record@1","story-spec@2","review-report/0002"],"details":{"decision":"REQUEST_CHANGES","to_stage":"SPECIFICATION","reason":"<reason дослівно>","mode":"pilot"}}
```

Стадію-ціль не запускай. Наступний `next` виконає повторний вхід у SPECIFICATION за розділом 7.3.1 з тригером «рішення людини».

### 8.5. CANCEL

Cancellation Protocol (`workflow-policies.md` §17, `stage-map.yaml` RR-08). `CANCELLED` — термінальний стан: Story повторно не відкривається.

Процедура однакова для обох видів очікування: `GATE` на HUMAN_SPEC_APPROVAL і `LOOPBACK_CONFIRMATION` на IMPACT_ANALYSIS чи SOLUTION_DESIGN (`loopback_confirmation_decisions.CANCEL`). Приклади нижче наведено для gate. Для `LOOPBACK_CONFIRMATION`:
- місце стадії в записі й подіях займає `current_stage` (`IMPACT_ANALYSIS` або `SOLUTION_DESIGN`);
- `subjects`, `artifacts` і `refs` беруться з поточного стану і `package`, зокрема `applicability-decision` та `impact-analysis`, якщо вони є;
- маршрут — `loopback_confirmation_decisions.CANCEL` замість `decisions.CANCEL` gate.

#### 8.5.1. Перевірка

1. **Причина.** `reason` непорожня.
2. **Запис.** `record_id` — `cancellation-record/<NNNN>`, наступний номер типу в Story (`artifact-rules.md` §2): найбільший номер `cancellation-record/…` у `refs` History + 1, чотири цифри; якщо таких немає — `0001`. Шлях — `records/cancellation-record/<NNNN>.md` під коренем пілоту (розділ 3). Якщо файл уже існує — не перезаписуй, поясни і зупинись.

#### 8.5.2. `cancellation-record`

Пише Orchestrator за рішенням людини (`artifact-registry.yaml` → `produced_by: ORCHESTRATOR`, `artifact-rules.md` §7). Заголовок — спільний для записів (`artifact-rules.md` §2), поля — зі списку Cancellation Protocol (`workflow-policies.md` §17). Стан — до застосування рішення.

`produced_by` — значення власника з реєстру (`ORCHESTRATOR`), а не стадія. Приклад заголовка в `artifact-rules.md` §2 описує записи агентних стадій (`{ stage, attempt }`). Для запису Orchestrator формат не визначено, а gate не є власником і не має спроб. Тому стадію і спробу не вигадуємо: стадію, на якій ухвалено рішення, зафіксовано в `last_state.stage` і в History.

`last_state` — значення `current_stage`, `story_status` і `execution_state` зі `state.yaml` до застосування рішення. Тіло: «Story <ID> скасовано рішенням людини на <current_stage> (<pending_human_decision.kind>)».

```markdown
---
record_type: cancellation-record
record_id: cancellation-record/0001
story_id: US-001
produced_by: ORCHESTRATOR                  # власник типу з artifact-registry.yaml
created_at: <date -u>
subjects:                                  # поточні версії артефактів Story
  - { type: clarification-record, version: 1 }
  - { type: story-spec, version: 2 }
reason: "<reason>"                         # дослівно, YAML-екранування за розділом 7.10
last_state:                                # останній стан Story до скасування
  stage: HUMAN_SPEC_APPROVAL
  story_status: WAITING_FOR_HUMAN
  execution_state: AWAITING_HUMAN
artifacts:                                 # створені артефакти: поточні версії зі статусами і чинні записи
  - { type: clarification-record, version: 1, status: REVIEWED }
  - { type: story-spec, version: 2, status: REVIEWED }
  - { record: review-report/0002 }
feature_branch: { exists: false }          # пілот без feature branch (state.yaml → git)
open_pull_request: false
resources: []                              # робочих ресурсів (гілок, PR, незакомічених змін) немає; файли пілоту не видаляються
follow_up: []                              # погоджених FOLLOW_UP, що стають backlog-entry, немає
---

> PILOT — поза зареєстрованим Workflow. Скасування симульоване в пілоті, а не скасування Story в офіційному Workflow.

Story <ID> скасовано рішенням людини на HUMAN_SPEC_APPROVAL. Причина — у полі `reason`.
```

- `artifacts` — усі ключі `artifacts` зі `state.yaml` з їхніми `version` і `status`, а також кожен запис із `records`. `SUPERSEDED` версії не перелічуються (`workflow-state.md` §3);
- `feature_branch`, `open_pull_request` — з `state.yaml` → `git`. Якщо поля `git` не `null` — зупинись до запису файлу: скасування з Git-ресурсами в пілоті не підтримується;
- `resources`: агент нічого не видаляє, не закриває і не відкочує (`workflow-policies.md` §17, п. 3). Дозволів на такі дії не потрібно, бо ресурсів немає;
- `follow_up: []`: Disposition FOLLOW_UP на цих стадіях не ухвалюється, а канонічного Backlog ще немає (PRQ-01).

Після запису прочитай файл і перевір front matter. Якщо файл записати не вдалося — `state.yaml` і History не змінюй, поясни і зупинись.

#### 8.5.3. Застосування

Маршрут — `decisions.CANCEL` gate або `loopback_confirmation_decisions.CANCEL` → `CANCELLED` (`terminal_states`).

**`state.yaml`:**

- `story_status: CANCELLED`, `pending_human_decision: null`, `updated_at`;
- `execution_state: COMPLETED` — **рішення MVP, не правило контракту.** Контракти не визначають Execution State для `CANCELLED`, і жодне значення не підходить дослівно:
  - `AWAITING_HUMAN` означає очікування рішення лише «доки людина не ухвалить рішення» (`workflow-state-model.md` §3);
  - `READY` означає, що спробу можна почати;
  - `COMPLETED` у State Model описано як завершену спробу зі Stage Verdict, а gate спроб і verdict не має.

  `COMPLETED` обрано як найближче значення: роботу на поточній стадії завершено і нових спроб не буде. Уточнення — справа майбутньої зміни контракту;
- `current_stage` не змінюється (`HUMAN_SPEC_APPROVAL`, `IMPACT_ANALYSIS` або `SOLUTION_DESIGN`): `CANCELLED` — термінальний стан, а не Stage ID (`stage-map.yaml` → `terminal_states`). Наслідок рішення видно в `story_status` (`workflow-state.md` §4);
- `records.cancellation-record: <record_id>` — реєстрація нового запису (`workflow-state.md` §7, крок 3);
- `last_stage_verdict`, `artifacts`, `approvals`, `retry`, `blocker`, `git` не змінюються.

**`history.jsonl`** — дві події:

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"human","id":"<by>"},"event":"human_decision.applied","stage":"HUMAN_SPEC_APPROVAL","refs":["clarification-record@1","story-spec@2","review-report/0002"],"details":{"decision":"CANCEL","to_stage":"CANCELLED","reason":"<reason дослівно>","record":"cancellation-record/0001","mode":"pilot"}}
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"orchestrator","id":"story-orchestrator"},"event":"story.cancelled","stage":"HUMAN_SPEC_APPROVAL","refs":["cancellation-record/0001"],"details":{"story_status":"CANCELLED","execution_state":"COMPLETED","mode":"pilot"}}
```

Для `LOOPBACK_CONFIRMATION` в обох подіях `stage` — `current_stage`, `refs` першої — елементи `package`, а `details` першої містять також `kind: LOOPBACK_CONFIRMATION`.

Подальші стадії не запускаються. `next` і `decide` для `CANCELLED` зупиняються (розділ 7.2, крок 2; розділ 8.2, крок 2).

### 8.6. Завершення

Після 8.3.2, 8.4.2, 8.5.3 або 8.7.3 зупинись і покажи підсумок командою `status`. Наступну стадію не запускай.

### 8.7. LOOPBACK_CONFIRMATION (IMPACT_ANALYSIS, SOLUTION_DESIGN)

Підтвердження Loopback через gate (`stage-map.yaml` RR-04, `loopback_confirmation_decisions`; `workflow-state-model.md` §6). У MVP його створює `CHANGES_REQUIRED` IMPACT_ANALYSIS або SOLUTION_DESIGN з ціллю `SPECIFICATION` чи `CLARIFICATION` (розділ 7.8). Нижче `<stage>` — `current_stage`. Рішення ухвалює людина, Orchestrator лише перевіряє і застосовує його. Рішення в пілоті — симуляція, як і на gate.

#### 8.7.1. Перевірка

Замість кроків 3–4 розділу 8.2. Перше порушення — поясни і зупинись, нічого не змінюючи.

1. **Стан:**
   - `current_stage` — `IMPACT_ANALYSIS` або `SOLUTION_DESIGN`; `pending_human_decision.kind: LOOPBACK_CONFIRMATION`, `pending_human_decision.stage` = `current_stage`;
   - `last_stage_verdict: CHANGES_REQUIRED`;
   - `decision` є в `pending_human_decision.options` і в `loopback_confirmation_decisions` у `stage-map.yaml`.
2. **Пакет.** `pending_human_decision.package` містить рівно:
   - `{ type: story-spec, version: <поточна версія> }`;
   - лише для IMPACT_ANALYSIS — `{ record: <record_id> }`, де `<record_id>` — `records.applicability-decision.IMPACT_ANALYSIS.ENTRY`.

   `refs` останньої події `human_decision.requested` збігаються з `package`.
3. **Причина Loopback.** Остання подія `stage.completed` має `stage: <stage>`, `verdict: CHANGES_REQUIRED`, `loopback_target` з `loopbacks` стадії з `crosses_gates` і без `to_stage`. Для IMPACT_ANALYSIS її `refs` містять `<record_id>`.
4. **Approval не змінився** (лише для `RESUME`; `CANCEL` доступний з будь-якого нетермінального стану, RR-08). `approvals.HUMAN_SPEC_APPROVAL` має `approval_state: VALID`, а версія `story-spec` у його `scope` — поточна (розділ 7.2, крок 6).

Далі — за `decision`: `ROUTE_BACK` → 8.7.2, `RESUME` → 8.7.3, `CANCEL` → розділ 8.5.

#### 8.7.2. ROUTE_BACK — не підтримується

Відхили команду після перевірки 8.7.1, **до будь-яких змін**: `state.yaml`, `history.jsonl` і файли не змінюються. Story лишається в `WAITING_FOR_HUMAN` з тим самим `pending_human_decision`. Поясни людині таке:

- маршрут `ROUTE_BACK` оголошено в авторитетному Workflow (`loopback_confirmation_decisions`, `loopbacks` стадії `<stage>`), але цей Orchestrator MVP його не виконує;
- для нього потрібен повний Rework Protocol (`workflow-state-model.md` §14), зокрема:
  - повторний вхід у SPECIFICATION чи CLARIFICATION після Loopback через gate;
  - інвалідація `HUMAN_SPEC_APPROVAL` після нової версії `story-spec`;
  - повторне проходження gate після APPROVE (розділ 8.2, крок 4);
  - облік Quality Loops;
- зараз доступні `RESUME` (розділ 8.7.3) і `CANCEL` (розділ 8.5).

Не перенаправляй рішення на іншу стадію і не застосовуй замість нього інше рішення. Змінити рішення може лише людина новою командою.

#### 8.7.3. RESUME

Loopback відхилено: Story лишається на `<stage>` і починає нову спробу з урахуванням рішення людини (`loopback_confirmation_decisions.RESUME`).

**Перевірка:**

- `reason` непорожня, `target` і `comment` не передано (розділ 8.1);
- **причина не змінює вимог** (D-03, `workflow-policies.md` §2). `RESUME` не погоджує нових вимог і не скасовує verdict стадії: нова спроба знову оцінює Story за погодженим `story-spec`. Якщо `reason` явно формулює нову вимогу, змінює чинну вимогу чи Acceptance Criterion або обирає варіант, якого немає в `story-spec`, — не застосовуй рішення. Поясни, що зміна вимог можлива лише через нову версію `story-spec` (`ROUTE_BACK`, який у MVP не підтримується, розділ 8.7.2), і зупинись.

**`state.yaml`:**

- `story_status: ACTIVE`, `execution_state: READY`, `pending_human_decision: null`, `updated_at`;
- `current_stage` не змінюється (`<stage>`);
- `last_stage_verdict` не змінюється (`CHANGES_REQUIRED`): рішення людини не є Stage Verdict (`workflow-state-model.md` §4);
- `artifacts`, `records`, `approvals`, `retry`, `blocker`, `git` не змінюються. `story-spec` лишається погодженим. Лічильники `RESUME` не змінює і не скидає (`workflow-state-model.md` §10, рішення D-1).

**`history.jsonl`** — подія `human_decision.applied`:

```json
{"ts":"<date -u>","story_id":"US-003","actor":{"kind":"human","id":"<by>"},"event":"human_decision.applied","stage":"IMPACT_ANALYSIS","refs":["story-spec@2","applicability-decision/0001"],"details":{"decision":"RESUME","kind":"LOOPBACK_CONFIRMATION","to_stage":"IMPACT_ANALYSIS","rejected_loopback_target":"SPECIFICATION","reason":"<reason дослівно>","mode":"pilot"}}
```

- `stage` і `to_stage` — `<stage>`. Для SOLUTION_DESIGN `refs` — `package`, тобто лише `story-spec@<N>`;
- `rejected_loopback_target` — `loopback_target` події з кроку 3 розділу 8.7.1.

Стадію не запускай. Наступний `next` виконає нову спробу і передасть `reason`: для IMPACT_ANALYSIS — за розділом 7.3.2, для SOLUTION_DESIGN — дослівно за розділом 7.3. `version` не змінюється, бо артефакт цієї спроби не реєструвався. Для IMPACT_ANALYSIS `record_id` — наступний номер, а попередній `applicability-decision` замінить новий запис після прийнятого результату (розділ 7.8).

## 9. Constraints

- Orchestrator пише лише `state.yaml` і `history.jsonl` пілоту: в `init`, на кроках 7.2 (лише `RUNNING` → `INTERRUPTED`), 7.4, 7.5.2, 7.7, 7.8, 8.3.2, 8.4.2, 8.5.3 і 8.7.3. Єдиний запис, який він пише сам, — `cancellation-record` у 8.5.2. `status` нічого не пише.
- Файл артефакта пише лише Stage Skill під час 7.5. Orchestrator його не створює, не редагує і не видаляє.
- Нічого не пиши в `docs/stories/`, `docs/contracts/`, `docs/workflow/`, Foundation чи `.claude/skills/`.
- Не вигадуй статусів, подій, полів стану чи правил переходів. Чого бракує в контрактах — повідом людині.
- History лише доповнюється: рядки не змінюються і не видаляються.
- Не виправляй стан мовчки і не відновлюй його автоматично.
- Запускай лише стадії з таблиці 7.1: CLARIFICATION inline, SPECIFICATION, SPEC_REVIEW, IMPACT_ANALYSIS і SOLUTION_DESIGN — кожну в одному subagent `general-purpose`. Історичну версію `impact-analysis` після `NOT_APPLICABLE` не передавай як вхід (розділ 7.1). Не запускай інші стадії і додаткові subagents.
- Рішення людини застосовуй лише в `decide` (розділ 8), за рішенням людини з аргументів команди. Не імітуй і не вгадуй рішень людини, не підставляй `by`. Не застосовуй рішень інших gates, RECONFIRM і `ROUTE_BACK`. Loopback через gate не виконуй автоматично.
- Рішення в пілоті — симуляція Human Gate. Не подавай їх як погодження чи скасування Story в офіційному Workflow.
- Stage Skills не викликай інструментом Skill.
- `state.yaml` змінюй лише за розділом 7.10.
- Не роби `git add`, `git commit`, `git push` чи деструктивних операцій.
