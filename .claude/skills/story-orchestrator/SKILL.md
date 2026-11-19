---
name: story-orchestrator
description: Story Orchestrator MVP (Phase 6, increments 6.4.1–6.4.3). Ініціалізує пілотну Story поза зареєстрованим Workflow, показує її статус з перевіркою State і History і виконує одну стадію за запуск (CLARIFICATION inline, SPECIFICATION і SPEC_REVIEW у subagent) у Step Mode, до межі HUMAN_SPEC_APPROVAL. Запускається лише вручну з mode=pilot.
argument-hint: "init|status|next story=<ID> mode=pilot"
disable-model-invocation: true
user-invocable: true
---

# Story Orchestrator — MVP (increments 6.4.1–6.4.3)

## 1. Purpose

Orchestrator веде Workflow State і History однієї Story: `state.yaml` і `history.jsonl` змінює **лише** він (`docs/contracts/workflow-state.md` §1). Маршрути, verdicts, loopbacks і gates визначає лише `docs/workflow/stage-map.yaml`; формат стану — `docs/contracts/workflow-state.md`; шляхи — `docs/contracts/artifact-registry.yaml`. Ці файли авторитетні; у разі розбіжності діють вони, а не цей Skill.

Skill працює в основній сесії.

**Межі.** Команди: `init`, `status`, `next`. `next` виконує одну стадію за запуск (Step Mode), лише CLARIFICATION, SPECIFICATION або SPEC_REVIEW. Після `PASS` SPEC_REVIEW Story зупиняється на межі HUMAN_SPEC_APPROVAL: gate не виконується. Повторний вхід після loopback підтримується лише для SPECIFICATION (розділ 7.3.1). Немає виконання інших стадій, повторного входу в CLARIFICATION, рішень Human Gates, Recovery і Auto Mode. Якщо людина просить щось інше — поясни, що цього ще немає, і зупинись.

## 2. Inputs

Аргументи (`$ARGUMENTS`):

| Аргумент | Обов'язковий | Значення |
|---|---|---|
| команда | так | `init`, `status` або `next` |
| `story` | так | ID Story, формат `US-NNN` |
| `mode=pilot` | так | Явна позначка пілоту. Без неї Skill не працює: реєстрація реальних Stories не підтримується |

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
   - Story ID і позначку «PILOT — не зареєстрована, не погоджена»;
   - `story_status`, `current_stage`, `execution_state`, `last_stage_verdict`, `execution_mode`, `story_type`, `updated_at`;
   - поточні версії артефактів зі `status` і `currency`, чинні записи з `records`;
   - `pending_human_decision`, `blocker`, ненульові `retry.technical_attempts` і `retry.quality_loops`;
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
- `story_status: BLOCKED` ⇔ `blocker` не `null`.

**`history.jsonl`** (`workflow-state.md` §5):

- кожен рядок — валідний JSON з полями `ts`, `story_id`, `actor`, `event`, `stage`, `refs`, `details`;
- `story_id` збігається зі станом;
- `event` — з таблиці подій §5;
- `actor.kind` ∈ `agent | human | orchestrator`;
- `ts` не спадає;
- перша подія — подія ініціалізації (розділ 4.2);
- `execution_state: RUNNING` ⇔ остання подія — `stage.started` або `human_decision.applied` для `current_stage`;
- `execution_state: AWAITING_HUMAN` ⇔ остання подія — `human_decision.requested` для `current_stage`.

**Файли** (`workflow-state-model.md` §12, крок 2):

- кожна версія з `artifacts` існує за шляхом пілоту, а її front matter має той самий `artifact_type`, `version` і `story_id`;
- кожен запис з `records` існує, а його front matter має той самий `record_id` і `story_id`.

**Git Binding** не перевіряється, поки всі поля `git` — `null` (пілот без feature branch). Скажи про це в статусі.

## 7. Command `next`

Виконує одну спробу поточної стадії і зупиняється.

### 7.1. Підтримувані стадії

| Стадія | Stage Skill | Де виконується | Вихід | Входи-артефакти |
|---|---|---|---|---|
| `CLARIFICATION` | `.claude/skills/clarification/SKILL.md` | inline, основна сесія (інтерв'ю з людиною) | `clarification-record` | — |
| `SPECIFICATION` | `.claude/skills/specification/SKILL.md` | окремий subagent `general-purpose` | `story-spec` | `clarification-record` |
| `SPEC_REVIEW` | `.claude/skills/spec-review/SKILL.md` | окремий subagent `general-purpose` | запис `review-report` | `story-spec`, `clarification-record` |

Вихід і власник — з `artifact-registry.yaml` (`produced_by`), входи — з `stage-io-map.yaml`. SPEC_REVIEW перевіряє обидва входи (`stage-map.yaml` → `reviews: [CLARIFICATION, SPECIFICATION]`). Дозволені verdicts і маршрути — лише зі `stage-map.yaml` (розділ 7.6).

### 7.2. Перевірка перед запуском

1. **Перевір аргументи** і узгодженість (розділ 6). Розбіжність — зупинись, як у `status`.
2. **`story_status` має бути `ACTIVE`.** Інакше — покажи стан і зупинись: рішення людини, gates і Recovery ще не реалізовано.
   Окремий випадок — `WAITING_FOR_HUMAN` з `pending_human_decision.kind: CLARIFICATION_ANSWERS`: новий запуск `next` означає, що інтерв'ю цієї спроби перервалося разом із сесією. **Обмеження MVP:** продовження інтерв'ю в новій сесії не підтримується, а контракт не визначає переходу з `AWAITING_HUMAN` у `INTERRUPTED` під час відновлення сесії (`workflow-state-model.md` §12 описує лише `RUNNING`). Нічого не змінюй: поясни це людині і зупинись.
3. **`current_stage` має бути в таблиці 7.1.** Інша стадія — «ще не підтримується», зупинись.
4. **`execution_state`:**
   - `READY`, `INTERRUPTED`, `FAILED`, `INVALID_RESULT` — можна починати нову спробу;
   - `RUNNING` — попередню спробу перервано без результату (`workflow-state-model.md` §12, крок 3). Запиши невдачу `INTERRUPTED` (розділ 7.7) і зупинись. Нову спробу людина запускає наступним `next`;
   - інше — зупинись і поясни.
5. **Входи-артефакти** (`stage-io-map.yaml`, правило R1). Для кожного входу з таблиці 7.1:
   - тип зареєстровано в `state.yaml` → `artifacts`, `currency: CURRENT`, файл поточної версії існує;
   - кожен upstream-артефакт з його `inputs` (`type`, `version`, тип відмінний від типу самого входу) — поточна версія цього типу в `artifacts` (`artifact-rules.md` §4);
   - посилання на попередню версію того самого типу — це lineage, а не upstream-залежність (`artifact-rules.md` §3): дозволене лише для `version > 1`, лише на `version - 1`, і файл цієї версії існує за шляхом пілоту. Бути поточною вона не мусить: вона `SUPERSEDED`;
   - кожне джерело з його `inputs` (`source`, `path`, `commit`) досі має той самий `git log -1 --format=%h -- <path>` (`artifact-rules.md` §4).

   Невиконано — нічого не змінюй, поясни людині і зупинись. **Обмеження MVP:** подію `staleness.detected` і перехід у `BLOCKED` (Upstream Changed) поточна версія Skill не записує.

### 7.3. Параметри спроби

Обчислює Orchestrator, а не Stage Skill:

| Параметр | Як визначається |
|---|---|
| `attempt` | Кількість подій `stage.started` для `current_stage` у History + 1 |
| `version` | Для документа: поточна версія вихідного типу в `artifacts` + 1; якщо його немає — `1` |
| `record_id` | Для запису (SPEC_REVIEW): `review-report/<NNNN>`, де `NNNN` — наступний номер типу в Story (`artifact-rules.md` §2): найбільший номер `review-report/…` у `refs` подій `stage.completed` History + 1, чотири цифри; якщо таких немає — `0001` |
| `output` | Шлях вихідного типу за реєстром із префіксом пілоту (розділ 3); для запису — `records/review-report/<NNNN>.md` |
| `clarification`, `spec` | Шляхи поточних версій `clarification-record` і `story-spec` (для SPECIFICATION — лише `clarification`) |
| `previous_reports` | Для SPEC_REVIEW: шляхи попередніх `review-report` цієї стадії з History (`stage.completed` стадії SPEC_REVIEW); немає — не передавай |
| `previous`, `reason` | Лише для `version > 1` — за розділом 7.3.1 |

`focus` для SPEC_REVIEW не передається: Orchestrator не підказує гіпотез чи findings.

#### 7.3.1. Повторний вхід після loopback (`version > 1`)

Підтримується лише SPECIFICATION після прийнятого `CHANGES_REQUIRED` від SPEC_REVIEW. Умови:

- `last_stage_verdict: CHANGES_REQUIRED`;
- остання подія `stage.completed` у History — стадії `SPEC_REVIEW` з `verdict: CHANGES_REQUIRED` і `to_stage: SPECIFICATION`; її `refs` містять запис-тригер `review-report/<NNNN>`, і він зареєстрований у `records.review-report.SPEC_REVIEW`.

Не виконано, або `current_stage: CLARIFICATION` з `version > 1` — нічого не змінюй, поясни і зупинись. **Обмеження MVP:** повторний вхід у CLARIFICATION (нова версія через інтерв'ю, Staleness `story-spec` і повторне проходження SPECIFICATION за Rework Protocol, `workflow-state-model.md` §14) не реалізовано. Loopback не перенаправляється на іншу стадію.

Параметри (`specification/SKILL.md` §2):

| Параметр | Значення |
|---|---|
| `previous` | Шлях поточної зареєстрованої версії `story-spec` (`v<N-1>`). Вона не змінюється |
| `reason` | `"SPEC_REVIEW CHANGES_REQUIRED: <record_id> (<шлях звіту>); BLOCKING: <F-NNN, …>; NON_BLOCKING: <F-NNN, …>"` — ID `OPEN` findings з front matter звіту-тригера, без переказу їхнього змісту |
| `version`, `output` | За загальним правилом: `v<N>` — новий файл |
| `clarification` | Поточна версія `clarification-record` |

Findings — діагностика reviewer, а не джерело продуктових рішень. Продуктові рішення й далі беруться лише з `clarification-record`.

**Незареєстрований Working Draft.** Перед стартом будь-якої спроби перевір, чи існує файл `output`, якого немає в `state.yaml` → `artifacts` чи `records`. Якщо існує — це Working Draft попередньої спроби (`artifact-rules.md` §3, п. 1). Не перезаписуй, не видаляй і не реєструй його. Нічого не змінюй у стані: покажи шлях, поясни, що знайдено незареєстрований Working Draft, і зупинись. Рішення ухвалює людина поза Skill.

> **Тимчасове обмеження MVP.** Контракт дозволяє повторній спробі перезаписати Working Draft. Цей Skill свідомо цього не робить, доки не реалізовано Recovery. Контракт Working Draft не змінюється.

### 7.4. Старт спроби

Один крок Orchestrator:

- `state.yaml`: `execution_state: RUNNING`, `updated_at`;
- `history.jsonl`: подія `stage.started`.

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"orchestrator","id":"story-orchestrator"},"event":"stage.started","stage":"SPECIFICATION","refs":["clarification-record@1"],"details":{"attempt":1,"version":1,"output":"docs/pilots/orchestrator/US-001/artifacts/story-spec/v1.md","mode":"pilot"}}
```

`refs` — входи-артефакти спроби (`<type>@<version>`); для CLARIFICATION — `[]`. Для SPEC_REVIEW у `details` замість `version` — `record_id`. Для повторного входу (7.3.1) `refs` додатково містять попередню версію і запис-тригер (`story-spec@1`, `review-report/0001`), а `details` — `previous` і `reason`.

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

#### 7.5.3. SPECIFICATION і SPEC_REVIEW — subagent

1. Перед запуском запам'ятай `git status --porcelain --untracked-files=all` (знімок А) і `git hash-object` кожного зареєстрованого файлу артефакта чи запису пілоту (файли пілоту не відстежуються Git, тож їхні зміни `git status` не покаже).
2. Запусти **один** subagent інструментом Agent з `subagent_type: general-purpose`, у foreground. Prompt — лише текст нижче для стадії, без контексту основної сесії (чат, інтерв'ю, міркування й самооцінка producer-стадії, думка Orchestrator про артефакти не передаються).

   SPECIFICATION:

   ```text
   Ти виконуєш стадію SPECIFICATION пілотної Story поза зареєстрованим Workflow.
   Прочитай .claude/skills/specification/SKILL.md повністю і виконай його точно.
   Аргументи: story=<ID> clarification=<clarification> output=<output> version=<version> attempt=<attempt> mode=pilot [previous=<previous> reason="<reason>"]
   Людини в цьому контексті немає: нічого не питай.
   [Якщо є reason:] Звіт із reason прочитай сам. Findings — діагностика reviewer, а не джерело продуктових рішень: не вигадуй рішень, яких немає в clarification-record. Створи нову версію в output; попередню версію не змінюй.
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

3. Дочекайся завершення subagent. Не продовжуй і не вгадуй результат.
4. Знову візьми `git status --porcelain --untracked-files=all` (знімок Б) і `git hash-object` тих самих файлів. Різниця А→Б має бути лише файлом `output`, якщо envelope має його в `outputs`; інакше — порожньою. Хеші зареєстрованих файлів не змінюються (`artifact-rules.md` §3, п. 2). Інша зміна — результат відхилено (`INVALID_RESULT`) з переліком файлів. Нічого не відкочуй: рішення ухвалює людина.
5. Envelope — останній блок `result:` у відповіді subagent. Збій інструмента Agent — `FAILED`. Відповідь без envelope — `INVALID_RESULT`.

### 7.6. Перевірка result envelope

Orchestrator приймає результат, лише коли виконано **все** (`workflow-state.md` §6):

- є блок `result` з полями `story_id`, `stage`, `attempt`, `verdict`, `outputs`, `loopback_target`, `blocker`, `summary`;
- `story_id`, `stage`, `attempt` збігаються з параметрами спроби;
- `verdict` дозволений для стадії за `stage-map.yaml`: `optional: false` → не `NOT_APPLICABLE`; `CHANGES_REQUIRED` — лише якщо `loopbacks` стадії не порожні. CLARIFICATION: `PASS | BLOCKED`. SPECIFICATION і SPEC_REVIEW: `PASS | CHANGES_REQUIRED | BLOCKED`;
- `loopback_target`: для `CHANGES_REQUIRED` — ціль із `loopbacks` стадії (SPECIFICATION → `CLARIFICATION`; SPEC_REVIEW → `SPECIFICATION` або `CLARIFICATION`); інакше `null`;
- для SPEC_REVIEW — розділ 7.6.1 замість правил документа нижче;
- для `PASS`: `blocker: null`; `outputs` — рівно `[{ type: <вихідний тип>, version: <version> }]`; файл `output` існує; його front matter (`artifact-rules.md` §1):
  - `artifact_type`, `story_id`, `version` збігаються;
  - `produced_by: { stage: <current_stage>, attempt: <attempt> }`;
  - `created_at` — ISO-8601 UTC;
  - `inputs` не порожні, кожне джерело має `source`, `path`, `commit`;
  - для SPECIFICATION: `inputs` містить `{ type: clarification-record, version: <поточна версія> }`; `story_type` — одне зі `story_types`; `acceptance_criteria` — непорожній список, і кожен ID є в тілі;
  - якщо `version > 1`: є `revision_reason`, він посилається на кожен `BLOCKING` finding звіту-тригера, а `inputs` містить `{ type: story-spec, version: <N-1> }` (`artifact-rules.md` §1, `specification/SKILL.md` §5);
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

### 7.7. Невдача спроби

`INTERRUPTED`, `FAILED` або `INVALID_RESULT`, один крок:

- `execution_state` — відповідне значення, `retry.technical_attempts.<stage>` + 1, `updated_at`;
- подія `stage.failed` з `details: { attempt, execution_state, reason }`;
- файл спроби не реєструється: він лишається Working Draft;
- якщо лічильник досяг `stage-map.yaml` → `retry_policy.defaults.technical_attempts`: також `story_status: BLOCKED`, `blocker: { reason: "Retry Budget Exhausted", source: ORCHESTRATOR, details, needed: "рішення людини через Recovery" }` і подія `story.blocked`.

### 7.8. Застосування прийнятого результату

Маршрут — лише зі `stage-map.yaml` (`workflow-state.md` §7). Один крок: спершу `state.yaml` (розділ 7.10), потім події в `history.jsonl`. В усіх випадках `retry.technical_attempts.<stage>` видаляється (скидання після прийнятого результату, `workflow-state-model.md` §10) і оновлюється `updated_at`.

**Запис SPEC_REVIEW.** Якщо `outputs` містить запис, він реєструється за будь-якого verdict: `records.review-report.SPEC_REVIEW: <record_id>` (`workflow-state.md` §3). `refs` подій `stage.completed` містять `<record_id>`.

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

Далі `next` зупиняється на кроці 7.2 (`story_status` не `ACTIVE`). Рішення gate цей Skill не приймає і не імітує.

**`PASS`** CLARIFICATION і SPECIFICATION → `next` стадії (CLARIFICATION → `SPECIFICATION`, SPECIFICATION → `SPEC_REVIEW`):

- `artifacts.<вихідний тип>: { version: <version>, status: DRAFT, currency: CURRENT }` — Version Status нової версії до review (`workflow-state-model.md` §5). Попередня версія стає `SUPERSEDED` без окремого запису (`workflow-state.md` §3); її файл лишається без змін;
- для SPECIFICATION: `story_type` — з front matter `story-spec` (`workflow-state.md` §3);
- `last_stage_verdict: PASS`;
- `current_stage: <next>`, `execution_state: READY`, `story_status: ACTIVE`;
- подія:

```json
{"ts":"<date -u>","story_id":"US-001","actor":{"kind":"orchestrator","id":"story-orchestrator"},"event":"stage.completed","stage":"SPECIFICATION","refs":["story-spec@1"],"details":{"attempt":1,"verdict":"PASS","outputs":["story-spec@1"],"to_stage":"SPEC_REVIEW","summary":"<summary з envelope>","mode":"pilot"}}
```

**`CHANGES_REQUIRED`** (SPECIFICATION або SPEC_REVIEW) → `loopback_target`. Ці loopbacks не мають `crosses_gates`, тож рішення людини не потрібне (`stage-map.yaml` RR-03, RR-04):

- `last_stage_verdict: CHANGES_REQUIRED`; Version Status артефактів не змінюється;
- `retry.quality_loops.target.<loopback_target>` + 1 і `retry.quality_loops.initiator.<stage>` + 1 (`workflow-state-model.md` §10);
- для SPEC_REVIEW: `retry.repeated_failure.<F-NNN>` + 1 для кожного `OPEN` `BLOCKING` finding звіту (`workflow-state.md` §3);
- якщо будь-який лічильник досяг відповідного значення `retry_policy.defaults` (`quality_loops`, `repeated_failure`) — `story_status: BLOCKED`, `blocker: { reason: "Retry Budget Exhausted", source: ORCHESTRATOR, details, needed }`, `execution_state: COMPLETED`, `current_stage` не змінюється, події `stage.completed` і `story.blocked`;
- інакше `current_stage: <loopback_target>`, `execution_state: READY`, `story_status: ACTIVE`, подія `stage.completed` з `details: { attempt, verdict, loopback_target, to_stage, summary, mode }`.

Стадію-ціль не запускай. Наступний `next` виконає її повторний вхід за розділом 7.3.1, якщо це SPECIFICATION. Для CLARIFICATION він зупиниться (обмеження MVP, 7.3.1).

Повторний вхід лічильники не змінює: Quality Loops уже враховано під час `CHANGES_REQUIRED`.

**`BLOCKED`** → маршруту немає (`stage-map.yaml` RR-05):

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

## 8. Constraints

- Orchestrator пише лише `state.yaml` і `history.jsonl` пілоту: в `init`, на кроках 7.2 (лише `RUNNING` → `INTERRUPTED`), 7.4, 7.5.2, 7.7 і 7.8. `status` нічого не пише.
- Файл артефакта пише лише Stage Skill під час 7.5. Orchestrator його не створює, не редагує і не видаляє.
- Нічого не пиши в `docs/stories/`, `docs/contracts/`, `docs/workflow/`, Foundation чи `.claude/skills/`.
- Не вигадуй статусів, подій, полів стану чи правил переходів. Чого бракує в контрактах — повідом людині.
- History лише доповнюється: рядки не змінюються і не видаляються.
- Не виправляй стан мовчки і не відновлюй його автоматично.
- Запускай лише стадії з таблиці 7.1: CLARIFICATION inline, SPECIFICATION і SPEC_REVIEW — кожну в одному subagent `general-purpose`. Не запускай інші стадії, Human Gates і додаткові subagents; не створюй approvals і не імітуй рішень людини.
- Stage Skills не викликай інструментом Skill.
- `state.yaml` змінюй лише за розділом 7.10.
- Не роби `git add`, `git commit`, `git push` чи деструктивних операцій.
