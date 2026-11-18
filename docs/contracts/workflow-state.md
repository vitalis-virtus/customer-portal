# Workflow State

**Статус:** `APPROVED (MVP)`, Simplified Artifact Contracts MVP. Погоджено 2026-11-17 як дизайн MVP-контрактів; межі погодження — [README.md](README.md).

Документ визначає формат стану Story (`state.yaml`), журнал подій (`history.jsonl`) і result envelope стадії. Значення статусів беруться з погодженої Workflow Definition: `stage-map.yaml` і `workflow-state-model.md`. Нових статусів документ не вводить.

## 1. Хто пише

- `state.yaml` і `history.jsonl` змінює **лише Orchestrator**.
- Stage Skill записує файли своїх артефактів (у статусі чернетки, [artifact-rules.md](artifact-rules.md), розділ 3) і повертає result envelope (розділ 6). Стан Skill не змінює.
- Рішення людини записує Orchestrator: в `approvals` або `pending_human_decision` → History.
- Історія чату не є джерелом стану.

## 2. Файли

```
docs/stories/<story_id>/
├── state.yaml        # поточний стан, змінний
├── history.jsonl     # журнал подій, лише додавання
├── artifacts/…       # версії документів
└── records/…         # supporting-записи
```

## 3. `state.yaml`

Приклад: Story чекає на HUMAN_SPEC_APPROVAL.

```yaml
story_id: US-001
story_type: FEATURE                 # з story-spec після його реєстрації; до того null
story_status: WAITING_FOR_HUMAN     # ACTIVE | WAITING_FOR_HUMAN | BLOCKED | DONE | CANCELLED
current_stage: HUMAN_SPEC_APPROVAL  # Stage ID зі stage-map.yaml
execution_state: AWAITING_HUMAN     # READY | RUNNING | AWAITING_HUMAN | COMPLETED | INTERRUPTED | FAILED | INVALID_RESULT
last_stage_verdict: PASS            # verdict останньої завершеної спроби поточної/попередньої стадії
execution_mode: STEP                # STEP | AUTO
updated_at: 2026-11-18T11:20:00Z

artifacts:                          # лише поточна версія кожного типу
  clarification-record: { version: 1, status: REVIEWED, currency: CURRENT }
  story-spec:           { version: 2, status: REVIEWED, currency: CURRENT }

records:                            # останній чинний запис за призначенням
  review-report:
    SPEC_REVIEW: review-report/0002
  applicability-decision: {}        # <STAGE>: { PLANNED: <record_id>, ENTRY: <record_id> }
  revalidation-record: []           # усі чинні revalidation-записи

approvals:
  HUMAN_SPEC_APPROVAL: null         # див. розділ 4

pending_human_decision:
  kind: GATE                        # GATE | HUMAN_ACTION | LOOPBACK_CONFIRMATION | RECOVERY | CLARIFICATION_ANSWERS
  stage: HUMAN_SPEC_APPROVAL
  options: [APPROVE, REQUEST_CHANGES, CANCEL]
  package:                          # що людина має переглянути: Approval Scope і звіти
    - { type: clarification-record, version: 1 }
    - { type: story-spec, version: 2 }
    - { record: review-report/0002 }
  requested_at: 2026-11-18T11:20:00Z

blocker: null                       # { reason, source: STAGE | ORCHESTRATOR, details, needed }

retry:
  technical_attempts: {}            # <STAGE>: n — скидається після прийнятого результату
  quality_loops:                    # окремо для цілі й ініціатора; скидаються лише повним APPROVE відповідного gate
    target: { SPECIFICATION: 1 }
    initiator: { SPEC_REVIEW: 1 }
  repeated_failure: { F-002: 1 }    # появи Finding ID в межах Story; не скидається
  extensions: []                    # { counter, key, extra, decided_by, decided_at }

git:
  feature_branch: story/US-001
  target_branch: main
  changeset_commit: null            # з поточної версії implementation-changeset
  verified_target_base: null        # з поточного verification-report
  merge_protection_status: null     # ACTIVE | NOT_SUPPORTED | UNAVAILABLE | DISABLED
```

### Статуси артефактів

- `status` — Version Status поточної версії: `DRAFT`, `REVIEWED` або `APPROVED`.
- Кожна версія з номером, меншим за поточний, — `SUPERSEDED`. Окремо такі версії не перелічуються.
- `currency` — `CURRENT` або `STALE`, обчислюється за правилом [artifact-rules.md](artifact-rules.md), розділ 4.
- Статус і Currency зберігаються **лише тут**. У файлах артефактів їх немає, тож розійтися вони не можуть.
- Version Status не повертається назад. `REVIEWED` встановлюється, коли review-стадія повернула `PASS` для цієї версії. `APPROVED` — коли людина ухвалила APPROVE на gate, у Scope якого входить версія.

## 4. Approvals

Запис створюється або замінюється, коли людина ухвалює рішення на gate.

**Approval Scope.** `scope` містить **кожен** відповідний елемент `approval_scope` цього gate зі `stage-map.yaml`, розгорнутий до точної ідентичності:

- контрольований документ — `{ type, version }`;
- supporting-запис — `{ record: <record_id> }`;
- `implementation-changeset` — `{ type, version, changeset_commit }`.

Елемент з умовою «якщо стадія застосовна» для незастосовної стадії замінює запис `{ record: <applicability-decision> }` з `NOT_APPLICABLE`. Пропускати елемент Scope мовчки не можна. Звіти, які людина переглянула, але які не входять до Scope (наприклад, `review-report` SPEC_REVIEW для HUMAN_SPEC_APPROVAL), зберігаються окремо в `package`.

```yaml
approvals:
  HUMAN_SPEC_APPROVAL:
    decision: APPROVE
    decided_by: human-architect            # хто: ідентифікатор людини
    decided_at: 2026-11-18T12:05:00Z      # коли
    scope:                                # що саме погоджено: усі елементи approval_scope gate
      - { type: clarification-record, version: 1 }
      - { type: story-spec, version: 2 }
    package: [review-report/0002]         # переглянуто, але не входить до Scope
    comment: "OD-01: варіант A"
    approval_state: VALID                 # VALID | SUSPENDED | INVALIDATED
    previous: []                          # попередні approvals цього gate, що стали INVALIDATED

  HUMAN_PR_APPROVAL:                      # приклад Scope із записами
    scope:
      - { type: implementation-changeset, version: 3, changeset_commit: 4f2a9c1 }
      - { record: review-report/0007 }            # CODE_REVIEW
      - { record: applicability-decision/0009 }   # SECURITY_REVIEW ENTRY: NOT_APPLICABLE
      - { record: applicability-decision/0010 }   # RECONCILIATION ENTRY: NOT_APPLICABLE
      - { record: verification-report/0002 }
      - { record: pull-request/0001 }
```

**Заміна запису.** Запис у Scope вважається заміненим, коли для того самого призначення стає чинним новий запис. Наприклад, новий `verification-report` після повторної VERIFICATION або новий `review-report` CODE_REVIEW для нової версії changeset (у `state.yaml` → `records` змінюється посилання). `revalidation-record` запис не замінює: він лише підтверджує, що висновок досі чинний.

Обчислення `approval_state` (`workflow-state-model.md`, розділ 7) — лише в один бік:

| Перехід | Хто і коли |
|---|---|
| → `VALID` | **Лише** рішення людини APPROVE. Воно створює новий запис approval, а попередній переходить у `previous` |
| `VALID` → `SUSPENDED` | Orchestrator: ідентичності Scope не змінилися, але після `decided_at` змінився upstream хоча б одного елемента. Документ став `STALE`; для запису — версія з його `subjects` більше не поточна, а нового запису ще немає |
| `VALID` / `SUSPENDED` → `INVALIDATED` | Orchestrator: хоча б один елемент Scope отримав новішу версію, новий `changeset_commit` або його замінив новий запис |

Виняток з Workflow: `post-merge-validation-record` не входить до жодного Scope і не змінює HUMAN_PR_APPROVAL (`stage-map.yaml`, `approval_effect`).

Orchestrator ніколи не повертає approval у `VALID`. Зокрема, `revalidation-record` повертає артефакту Currency `CURRENT`, але approval лишається `SUSPENDED` (`workflow-state-model.md`, розділ 7).

**Обмеження MVP.** RECONFIRM не підтримується. Approval у стані `SUSPENDED` чи `INVALIDATED` знову стає чинним лише через нове повне APPROVE. Це суворіше за Workflow Definition (див. README.md).

Рішення людини, які не є approval (REQUEST_CHANGES, CANCEL, CONFIRM_MERGED, рішення Recovery і підтвердження Loopback), записуються лише в History (розділ 5). Наслідок кожного з них видно в стані: `current_stage`, `story_status`, `blocker`.

## 5. `history.jsonl`

Один рядок — одна подія JSON. Файл лише доповнюється: рядки не змінюють і не видаляють.

```json
{"ts":"2026-11-18T12:05:00Z","story_id":"US-001","actor":{"kind":"human","id":"human-architect"},"event":"human_decision.applied","stage":"HUMAN_SPEC_APPROVAL","refs":["clarification-record@1","story-spec@2","review-report/0002"],"details":{"decision":"APPROVE","to_stage":"IMPACT_ANALYSIS","comment":"OD-01: варіант A"}}
```

| Поле | Значення |
|---|---|
| `ts` | Час події, ISO-8601 UTC |
| `actor` | `{kind: agent \| human \| orchestrator, id}`. Для агента — назва стадії або Skill |
| `event` | Тип події з таблиці нижче |
| `stage` | Стадія, до якої належить подія |
| `refs` | Посилання: `<type>@<version>` для документів, `<record_id>` для записів |
| `details` | Короткі дані події: verdict, рішення, маршрут, причина |

| `event` | Коли |
|---|---|
| `story.registered` | Story зареєстровано, стан ініціалізовано з `initial_state` |
| `stage.started` | Почалася спроба стадії |
| `stage.completed` | Orchestrator прийняв result envelope: verdict, зареєстровані виходи, маршрут |
| `stage.failed` | Спроба `INTERRUPTED`, `FAILED` або `INVALID_RESULT`, з причиною |
| `human_decision.requested` | Story чекає на людину (`pending_human_decision`) |
| `human_decision.applied` | Рішення людини: хто, що, коли, щодо яких версій, маршрут |
| `staleness.detected` | Зміна upstream поза переходом стадії, зокрема Foundation |
| `story.blocked` | Story → `BLOCKED`: причина і джерело |
| `retry.extended` | Людина розширила Retry Budget |
| `story.done` / `story.cancelled` | Термінальний стан |

Окремої події для зміни режиму немає: `execution_mode` змінюється в `details` будь-якої події, яку ініціює людина.

## 6. Result envelope стадії

Stage Skill завершує роботу, повертаючи Orchestrator такий блок:

```yaml
result:
  story_id: US-001
  stage: SPEC_REVIEW
  attempt: 1
  verdict: CHANGES_REQUIRED          # PASS | CHANGES_REQUIRED | BLOCKED | NOT_APPLICABLE
  outputs:                           # файли, які стадія записала
    - { record: review-report/0003 }
  loopback_target: SPECIFICATION     # лише для CHANGES_REQUIRED; ціль із loopbacks стадії
  blocker: null                      # для BLOCKED: { reason, details, needed }
  summary: "1 BLOCKING finding (F-002)"
```

Orchestrator приймає результат лише тоді, коли виконано все з переліку:

- verdict дозволений для стадії (`NOT_APPLICABLE` — лише для `optional: true` разом з `applicability-decision`);
- усі обов'язкові виходи з [stage-io-map.yaml](stage-io-map.yaml) існують і мають коректні метадані;
- `loopback_target` входить до `loopbacks` стадії.

Інакше `execution_state` стає `INVALID_RESULT` і подається подія `stage.failed`.

## 7. Як Orchestrator записує перехід

1. Перевірити result envelope або рішення людини (розділ 6).
2. Визначити маршрут **лише** зі `stage-map.yaml`.
3. Зареєструвати нові версії й записи в `state.yaml`, оновити `status`, `currency` і `approval_state`.
4. Оновити поля стадії, `pending_human_decision`, `blocker`, `retry`.
5. Дописати одну або кілька подій у `history.jsonl`.

Рядок History і `state.yaml` змінюються в одному кроці Orchestrator. Протоколу довговічних транзакцій і replay немає. Якщо під час відновлення сесії `state.yaml`, History і файли суперечать одне одному, Story переходить у `BLOCKED` (Integrity Error), а розбіжність розв'язує людина. Автоматичного виправлення немає.

## 8. Перевірка зв'язку review зі змінами коду

Для стадій з order більшим за CODE_REVIEW (RR-10) Orchestrator перевіряє, що виконуються всі умови:

1. `review-report` CODE_REVIEW (а також SECURITY_REVIEW і `reconciliation-report`, якщо їх застосовано) має в `subjects` поточну версію `implementation-changeset`;
2. після `changeset_commit` у feature branch змінювались лише файли Story:

   ```
   git diff --quiet <changeset_commit> HEAD -- . ':(exclude)docs/stories/<story_id>'
   ```

3. поза `docs/stories/<story_id>/` немає незакомічених чи невідстежених змін (`git status --porcelain` порожній для цих шляхів).

Те саме правило застосовується до head PR у MERGE_READINESS_CHECK: між `changeset_commit` і head PR змінювались лише файли Story.

Якщо хоча б одна умова не виконується, Story переходить у `BLOCKED` (Integrity Error). Rebase чи squash змінює `changeset_commit`, тож потребує нової версії changeset через IMPLEMENTATION. Це суворіше за Workflow Definition (див. README.md).
