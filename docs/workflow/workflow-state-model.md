# Workflow State Model — Workflow Definition

**Статус документа:** `APPROVED` (revision 3). Погоджено людиною 2026-11-16 після фінального Review.

## 1. Призначення

Документ визначає логічну модель стану Story Workflow: з чого складається стан, як він змінюється, як працюють Recovery, Retry, валідність approvals, Staleness, History і відновлення після перерваної сесії.

Маршрути переходів визначає [stage-map.yaml](stage-map.yaml). Опис стадій — [stages.md](stages.md). Політики — [workflow-policies.md](workflow-policies.md).

### Що не входить

- Технічні схеми файлу поточного стану (YAML) і файлу історії (JSONL): поля, типи, шляхи, формат подій. Їх визначає етап Artifact Contracts.
- Механізм обчислення версій артефактів і перевірки залежностей. Програмний Dependency Engine на цьому етапі не створюється.
- Реалізація Orchestrator.

Документ задає, **що** має зберігатися і **які** правила діють. Як саме це зберігається, визначають наступні етапи.

## 2. Складові Workflow State

Кожна Story має власний незалежний Workflow State (Q12).

| Складова | Значення |
|---|---|
| Story Status | Загальний стан Story (розділ 4) |
| Story Type | Тип Story зі `story_types` у `stage-map.yaml` (OD-14). Фіксується в `story-spec` |
| Current Stage | Стадія зі `stage-map.yaml`, на якій перебуває Story |
| Execution State | Стан поточної спроби виконання стадії (розділ 3) |
| Last Stage Verdict | Останній verdict поточної стадії, якщо спробу завершено |
| Pending Human Decision | Рішення, на яке чекає Story: gate, human action, Recovery, Loopback через gate, ескалація, відповіді на питання |
| Artifact Registry | Контрольовані артефакти Story, їхні версії, Version Status, Currency і зафіксовані залежності (розділ 5) |
| Evidence Records | Supporting-записи Story: `review-report`, `revalidation-record`, `applicability-decision` та інші (розділ 5) |
| Approvals | Рішення Human Gates, прив'язані до Approved Content Identity артефактів Scope, і їхній Approval State (розділ 7) |
| Optional Stage Applicability | Чинні `applicability-decision` для Optional Stages |
| Retry Counters | Лічильники Retry Policy (розділ 10) |
| Blocker | Причина `BLOCKED`, якщо Story заблокована (розділ 9) |
| Execution Mode | Step Mode або Auto Mode поточного запуску ([workflow-policies.md](workflow-policies.md#11-step-mode-і-auto-mode)) |
| Git Binding | Feature branch Story, Changeset Content Identity і Target Base Identity, до яких прив'язані перевірки та approval (OD-12) |

**Джерела істини (Q9, Q18):**

- Поточний стан зберігається у файлі стану Story у форматі YAML. Це канонічне джерело поточного стану.
- Історія подій і переходів зберігається append-only у форматі JSONL. Це канонічне джерело того, що відбулося.
- Погоджені рішення людини записуються в канонічні артефакти (наприклад, у `clarification-record`, approval у стані). Запис лише в History недостатній.
- Історія чату та пам'ять сесії Claude Code не є джерелом стану, рішень чи вимог.

**Хто змінює стан.** Workflow State і History змінює лише Orchestrator. Stage Skill повертає результат (verdict, виходи, пропозицію Loopback, обґрунтування), але не змінює стан напряму.

## 3. Execution State

Execution State описує технічний перебіг однієї спроби виконання стадії. Він не є оцінкою результату (Q10).

| Execution State | Значення |
|---|---|
| `READY` | Стадія поточна, спробу ще не розпочато |
| `RUNNING` | Спробу розпочато, результату ще немає |
| `AWAITING_HUMAN` | Стадія або Orchestrator чекає на рішення чи відповідь людини |
| `COMPLETED` | Спроба повернула прийнятий результат зі Stage Verdict |
| `INTERRUPTED` | Спроба не повернула результату: сесію перервано, процес зупинено |
| `FAILED` | Технічний збій інструмента або середовища під час спроби |
| `INVALID_RESULT` | Результат повернуто, але Orchestrator його відхилив: бракує обов'язкових виходів, verdict недозволений для стадії, ціль Loopback не дозволена |

Правила:

- `INTERRUPTED`, `FAILED` і `INVALID_RESULT` не є verdict `BLOCKED`. Вони обробляються як технічні спроби в межах Retry Policy (розділ 10).
- Для `human_gate` і `human_action` Execution State — `AWAITING_HUMAN`, доки людина не ухвалить рішення.
- Очікування відповідей людини під час CLARIFICATION — це `AWAITING_HUMAN`, а не `BLOCKED`.

## 4. Stage Verdict і Story Status

### Stage Verdict

Stage Verdict — оцінка результату стадії. Повертає агентна стадія після завершеної спроби (Q10).

| Verdict | Значення | Обов'язковий супровід |
|---|---|---|
| `PASS` | Стадія виконала відповідальність, обов'язкові виходи створено | Виходи стадії |
| `CHANGES_REQUIRED` | Потрібні зміни в артефакті цієї стадії або upstream-стадії | Findings і пропонована ціль Loopback |
| `BLOCKED` | Стадія не може продовжити без рішення чи дії людини | Причина блокування і що потрібно для відновлення |
| `NOT_APPLICABLE` | Optional Stage незастосовна для цієї Story | Обґрунтування за умовою застосовності |

Правила:

- `NOT_APPLICABLE` дозволений лише стадіям з `optional: true` у `stage-map.yaml`.
- Рішення людини на Human Gate — це **Gate Decision**, а не Stage Verdict. Reviewer `PASS` не дорівнює Human Approval.
- Producer-стадія повертає `CHANGES_REQUIRED`, коли виявила проблему в upstream-артефакті. Review-стадія — коли виявила BLOCKING findings у перевіреному артефакті чи вище.

### Story Status

| Story Status | Значення |
|---|---|
| `ACTIVE` | Story може виконуватися без рішення людини |
| `WAITING_FOR_HUMAN` | Story чекає на Gate Decision, human action, підтвердження Loopback через gate або відповіді людини |
| `BLOCKED` | Story заблокована. Вийти можна лише через Recovery Protocol (розділ 9) |
| `DONE` | Термінальний. Story завершена за Definition of Done |
| `CANCELLED` | Термінальний. Story скасована рішенням людини |

Story переходить у `BLOCKED` у таких випадках:

- стадія повернула verdict `BLOCKED`;
- вичерпано Retry Budget (розділ 10);
- Orchestrator виявив порушення цілісності стану (розділ 12) або спрацював Review Binding Guard (`stage-map.yaml` RR-10);
- потрібна ескалація або Foundation Change Request ([workflow-policies.md](workflow-policies.md));
- INTEGRATION_CONFIRMATION виявила Integration Mismatch або Post-Merge Validation не пройшла.

Для кожного блокування фіксується **джерело**: Stage Verdict чи Orchestrator. Так блокування стадії не плутається з технічною чи процесною зупинкою.

## 5. Artifact Status

### Класи артефактів (Q11)

- **Controlled artifacts** проходять погодження або можуть застарівати. Вони мають версії, Version Status, Currency і зафіксовані залежності від версій upstream-артефактів. Перелік — у [stages.md](stages.md#5-логічні-артефакти).
- **Supporting artifacts** (`review-report`, `revalidation-record`, `applicability-decision`, `reconciliation-report`, `verification-report`, `pull-request`, `integration-record`, `post-merge-validation-record`, `risk-acceptance-record`, `foundation-exception-record`, `cancellation-record`) не мають власного Lifecycle. Це **незмінні записи**: після створення запис не редагується, а виправлення створює новий запис. Кожен запис прив'язаний до версій артефактів, які він перевіряв чи описував, і до версій upstream-артефактів, щодо яких виконано перевірку. Запис щодо однієї версії не зараховується для іншої.

Product Foundation, Architecture Foundation і Workflow Definition — upstream-джерела для всіх Stories. Їхні версії враховуються в залежностях контрольованих артефактів, але змінюються вони лише через власні процеси погодження, а не через Story Workflow.

### Approved Content Identity (WF-03)

**Інваріант:** кожен артефакт, на який поширюється Human Approval, має стабільну ідентичність погодженого вмісту — **Approved Content Identity**. Вона однозначно визначає вміст, який бачила людина, і дає змогу пізніше перевірити, що вміст не змінився, зокрема під час Git Integration.

- Для контрольованого артефакта Approved Content Identity — це його версія.
- Для supporting-запису — ідентичність незмінного запису.
- Для `implementation-changeset` — **Changeset Content Identity**: ідентичність змісту змін Story, незалежна від Git SHA окремих commits. Окремо фіксується **Target Base Identity** — версія target branch, щодо якої виконано перевірку (OD-12).
- Для `pull-request` — репозиторій, ідентифікатор PR, source і target branch та Changeset Content Identity, яку PR містить. Метадані PR (заголовок, опис, мітки) до неї не входять (`stage-map.yaml` RR-12).

Механізм забезпечення (immutable snapshot, content hash чи інший) визначає етап Artifact Contracts. Workflow вимагає лише, щоб identity була стабільною і перевірюваною.

### Version Status

Version Status описує факти про конкретну незмінну версію контрольованого артефакта.

| Version Status | Значення |
|---|---|
| `DRAFT` | Версію створено, review для неї ще немає або review не пройдено |
| `REVIEWED` | Відповідна review-стадія повернула `PASS` для цієї версії |
| `APPROVED` | Людина погодила цю версію (APPROVE) на Human Gate |
| `SUPERSEDED` | Існує новіша версія цього артефакта |

### Currency

Currency — окремий вимір актуальності версії щодо upstream-артефактів. Він не змінює Version Status.

| Currency | Значення |
|---|---|
| `CURRENT` | Кожна upstream-залежність версії або збігається з поточною версією upstream, або покрита `revalidation-record` для поточної версії upstream |
| `STALE` | Хоча б одна upstream-залежність змінилася, а `revalidation-record` для нової версії upstream немає |

### Правила версіонування (WF-01)

- **Незмінність версії.** Вміст версії і зафіксовані під час її створення залежності не змінюються ніколи. Будь-яка зміна вмісту створює нову версію зі статусом `DRAFT`, а попередня отримує `SUPERSEDED`.
- **Revalidation не змінює версію.** Підтвердження актуальності незмінної версії щодо нових upstream-версій фіксується окремим `revalidation-record`. Історичний вміст і залежності погодженої версії при цьому не переписуються.
- **Version Status і Currency незалежні.** Версія може бути `APPROVED` і `STALE` водночас. Це означає: людина погодила саме цей вміст, але його актуальність щодо змінених upstream ще не підтверджено.
- **`SUPERSEDED` має пріоритет.** Для `SUPERSEDED` версії Currency не оцінюється. Її не можна використати як вхід чи підставу approval.
- **Version Status не повертається назад.** Немає переходів `APPROVED` → `REVIEWED` чи `REVIEWED` → `DRAFT`. Чинність approval визначає Approval State (розділ 7), а не Version Status.
- Артефакти без власного gate (`implementation-changeset` до HUMAN_PR_APPROVAL, `clarification-record` до HUMAN_SPEC_APPROVAL) отримують `APPROVED` через gate, у Scope якого вони входять.
- Зміна артефакта поза Workflow (ручне редагування, зміна в Git без участі Orchestrator) виявляється як нова версія. Далі діє Rework Protocol (розділ 14).

### Використання `STALE` артефактів

- Звичайне виконання стадії не використовує `STALE` чи `SUPERSEDED` версії як чинні входи.
- Процедура Revalidation (розділ 8) і Change Impact Assessment можуть читати `STALE` версію, її попередні upstream-версії та нові upstream-версії, щоб оцінити вплив. Результат такого читання — лише `revalidation-record` або нова версія, але не інші виходи стадії.

## 6. Transitions

Перехід виконує Orchestrator за такою процедурою:

1. **Отримати результат** стадії або рішення людини.
2. **Перевірити результат:** обов'язкові виходи наявні; verdict дозволений для типу й optionality стадії; для `NOT_APPLICABLE` є обґрунтування; для `CHANGES_REQUIRED` ціль Loopback входить до `loopbacks` поточної стадії.
3. **Визначити маршрут** лише зі `stage-map.yaml`. Stage Skill пропонує ціль Loopback, але не визначає маршрут (Q13).
4. **Перевірити передумови** цільової стадії: вхідні артефакти мають Currency `CURRENT` і не є `SUPERSEDED`; потрібні approvals мають стан `VALID`; для стадій після CODE_REVIEW виконується Review Binding Guard (`stage-map.yaml` RR-10); для маршрутів із `condition` умова підтверджена.
5. **Застосувати перехід:** оновити Workflow State і записати подію в History. Якщо перехід є поверненням на попередню стадію, застосовується Rework Protocol (розділ 14).

Правила:

- Недозволений перехід не виконується. Orchestrator записує подію відхилення, Execution State стає `INVALID_RESULT`.
- Невиконаний Review Binding Guard не є `INVALID_RESULT` стадії: Story переходить у `BLOCKED` (Integrity Error), бо вміст змін не відповідає пройденим review.
- Loopback з `crosses_gates` спершу переводить Story у `WAITING_FOR_HUMAN`. Перехід виконується лише після рішення людини, бо він змінює погоджений артефакт. Людина обирає одне з `loopback_confirmation_decisions` у `stage-map.yaml`: `ROUTE_BACK` (запропонована або інша дозволена ціль), `RESUME` (Loopback відхилено, нова спроба поточної стадії з урахуванням рішення) або `CANCEL`. Рішення фіксується в History.
- Нова Story починає з `initial_state` у `stage-map.yaml`.
- `applicability-decision` прив'язаний до **Applicability Basis** — усіх релевантних фактів, на яких ґрунтувалося рішення (WF-V2-02): версій артефактів, Changeset Content Identity, фактів Workflow State, релевантних подій History, Retry Counters, рішень людини, Story Risk Assessment. Які категорії фактів релевантні для кожної Optional Stage, визначає `basis_facts` у `stage-map.yaml`. Перед використанням рішення (на вході стадії, у перевірці `na_checked_by`, у Review Binding Guard, перед Human Gate) перевіряється, що жоден релевантний факт не змінився (AP-08). Якщо змінився, рішення переоцінюється, і попередній `NOT_APPLICABLE` автоматично не переноситься. Події поза `basis_facts` рішення не інвалідують (AP-09).
- Після повернення Story проходить усі наступні стадії за порядком. Кожна стадія на повторному вході виконує Revalidation (розділ 8): або створює `revalidation-record` для незмінної версії, або створює нову версію.
- Review-стадії на повторному вході перевіряють нові версії артефактів і `revalidation-record` для незмінних версій. Звіт щодо попередньої версії або щодо інших upstream-версій не зараховується.

## 7. Approval Validity

Human Approval прив'язаний до Approved Content Identity кожного артефакта **Approval Scope** відповідного gate (`approval_scope` у `stage-map.yaml`) (Q16, WF-03).

### Approval State

Approval State обчислюється відносно **Approval Basis** — версій upstream, чинних на момент останнього рішення людини щодо цього approval (APPROVE або RECONFIRM). Він не виводиться з Currency: `revalidation-record` робить артефакт `CURRENT`, але сам не відновлює approval.

| Approval State | Умова | Наслідок |
|---|---|---|
| `VALID` | Кожен артефакт Scope має погоджену Approved Content Identity, а upstream не змінився від Approval Basis | На approval можна спиратися для переходів |
| `SUSPENDED` | Identity не змінилася, але хоча б один upstream змінився від Approval Basis | На approval не можна спиратися, доки людина не виконає RECONFIRM |
| `INVALIDATED` | Хоча б один артефакт Scope отримав нову версію або новий запис замість погодженого, або Revalidation показала, що потрібна нова версія | Approval більше не чинний. Потрібне повне APPROVE нових версій |

Правила:

- Перейти з `SUSPENDED` у `VALID` можна лише через RECONFIRM людини. Після нього Approval Basis — поточні версії upstream, і це фіксує подія RECONFIRM.

- `INVALIDATED` не повертається у `VALID`. Нове рішення людини створює новий approval.
- Approval ніколи не стає чинним для зміненого вмісту без рішення людини APPROVE.
- Рішення людини та його прив'язка до identity фіксуються в Workflow State і History.

### Reconfirmation (OD-06)

Human Gate — обов'язкова точка зупинки під час кожного проходження. Під час повторного проходження gate людина обирає одне з двох рішень:

- **APPROVE** — повне погодження. Обов'язкове, якщо хоча б один артефакт Scope має нову identity або approval має стан `INVALIDATED`.
- **RECONFIRM** — скорочене повторне підтвердження. Доступне лише тоді, коли одночасно:
  - для gate є попередній approval у стані `VALID` або `SUSPENDED`;
  - кожен артефакт Scope має ту саму Approved Content Identity, що й під час попереднього APPROVE;
  - кожна upstream-зміна після Approval Basis покрита `revalidation-record` для кожного зачепленого артефакта Scope, і review-стадія перевірила ці записи.

RECONFIRM фіксується окремою подією. Вона посилається на початковий approval і на `revalidation-record`, які людина прийняла. Початковий approval не замінюється, і RECONFIRM не вважається погодженням нового вмісту. Пакет рішення явно показує, що вміст не змінився, і які upstream-зміни перевірено.

Auto Mode не виконує ні APPROVE, ні RECONFIRM.

**Approvals попередніх gates, які Story не проходитиме знову.** Якщо upstream змінився, коли Story вже пройшла gate (наприклад, Foundation Change Request під час IMPLEMENTATION), approval цього gate може стати `SUSPENDED`. Тоді:

- потрібні `revalidation-record` створює Out-of-Stage Revalidation (розділ 8), не повертаючи Story на попередні стадії;
- RECONFIRM для такого approval людина ухвалює в межах рішення Recovery чи Foundation Change Request, окремо для кожного approval, з тими самими умовами;
- `RESUME` неможливий, поки хоча б один approval попередніх gates має стан `SUSPENDED` без RECONFIRM або стан `INVALIDATED`. Для `INVALIDATED` потрібен `ROUTE_BACK` на producer-стадію відповідного артефакта.

**Target branch після merge (OD-16).** Після CONFIRM_MERGED зміни target branch, зокрема відмінність Target Base Identity на момент merge від перевіреної, обробляє лише Post-Merge Validation в INTEGRATION_CONFIRMATION. Вони не змінюють Currency `verification-report` і Approval State HUMAN_PR_APPROVAL. `post-merge-validation-record` не входить до жодного Approval Scope і не замінює `verification-report`. HUMAN_PR_APPROVAL лишається чинним, доки не змінилася Changeset Content Identity. До merge зміна target branch обробляється як завжди: MERGE_READINESS_CHECK → повторна VERIFICATION → новий `verification-report` → `INVALIDATED`.

## 8. Staleness і Revalidation

### Залежності

Кожна версія контрольованого артефакта фіксує версії upstream-артефактів, на яких вона ґрунтується (Q20). Наприклад, `solution-design` залежить від версії `story-spec`, а `implementation-plan` — від версій артефактів дизайну. Конкретний перелік залежностей кожного типу і спосіб їх запису визначає етап Artifact Contracts.

Залежності фіксуються під час створення версії і далі не змінюються (розділ 5). Supporting-записи також фіксують версії upstream, щодо яких виконано перевірку.

### Staleness

Коли upstream-артефакт отримує нову версію, кожна залежна версія, що посилається на попередню версію upstream і не має `revalidation-record` для нової, отримує Currency `STALE`. Version Status при цьому не змінюється. Approvals, у Scope яких входить `STALE` артефакт, переходять у `SUSPENDED`. Це позначка «потрібна перевірка», а не автоматичне скидання всіх approvals.

Review-звіт, виконаний щодо інших версій upstream, ніж поточні, не зараховується як `PASS` для подальших переходів.

### Change Impact Assessment і Revalidation

**Change Impact Assessment** — оцінка, як зміна upstream-артефакта впливає на залежні артефакти. Це процедура, а не стадія. Її не слід плутати зі стадією IMPACT_ANALYSIS, яка оцінює вплив Story на наявну систему.

- Assessment виконує стадія, яка змінила upstream-артефакт, або Orchestrator у межах Foundation Change Request.
- Результат Assessment входить до пакета найближчого рішення людини (gate, Recovery, Foundation Change Request). Тому людина бачить, які погоджені артефакти зачепила зміна.

**Revalidation** виконує стадія-власник залежного артефакта під час повторного входу. Вона може читати `STALE` версію (розділ 5).

| Результат Revalidation | Наслідок |
|---|---|
| Зміна не впливає на артефакт | Створюється `revalidation-record`: версія X лишається актуальною щодо upstream-версій Y′, з обґрунтуванням. Вміст, залежності й Version Status версії X не змінюються. Currency стає `CURRENT`. Approval State не змінюється: `SUSPENDED` approval стає `VALID` лише після RECONFIRM людини (розділ 7) |
| Зміна впливає на артефакт | Стадія створює нову версію (`DRAFT`). Попередня стає `SUPERSEDED`. Approvals, у Scope яких входить попередня версія, стають `INVALIDATED` |

`revalidation-record` — незмінний supporting-запис. Його перевіряє review-стадія артефакта на повторному проходженні. Остаточно його бачить людина під час APPROVE чи RECONFIRM (розділ 7).

**Supporting-записи в Approval Scope** (`review-report`, `applicability-decision`, `verification-report` та інші) підпорядковуються тому самому правилу. Якщо висновок незмінного запису лишається чинним щодо нових upstream-версій, автор запису (стадія, що його створила) фіксує це в `revalidation-record`, а не створює новий запис. Новий запис створюється лише тоді, коли висновок змінився, і тоді approval, у Scope якого був попередній запис, стає `INVALIDATED`.

### Out-of-Stage Revalidation

Коли upstream змінився, а Story перебуває на стадії після артефакта, який треба перевірити (наприклад, Foundation Change Request під час IMPLEMENTATION), Revalidation можна виконати без повернення Story:

1. Роль-власник артефакта виконує Revalidation і створює `revalidation-record` або повідомляє, що потрібна нова версія.
2. Review-роль цього артефакта перевіряє `revalidation-record`.
3. Current Stage не змінюється. Подія фіксується в History.

Якщо потрібна нова версія, Out-of-Stage Revalidation завершується, а людина обирає `ROUTE_BACK` на producer-стадію цього артефакта. Out-of-Stage Revalidation запускається лише в межах рішення людини щодо Recovery або Foundation Change Request.

## 9. BLOCKED і Recovery Protocol

Кожен `BLOCKED` має визначений шлях відновлення (Q14).

### Причини блокування

| Причина | Приклад |
|---|---|
| Missing Information | Потрібна інформація, яку може дати лише людина, а стадія не може продовжити |
| Escalation Required | Рішення з переліку ескалацій Controlled Autonomy ([workflow-policies.md](workflow-policies.md#2-controlled-autonomy)) |
| Foundation Change Required | Story потребує зміни Product або Architecture Foundation |
| External Dependency | Недоступна зовнішня система, доступ чи дозвіл |
| Retry Budget Exhausted | Вичерпано ліміт спроб або Loopbacks (розділ 10) |
| Upstream Changed | Під час Session Recovery або між запусками змінився upstream (Foundation, upstream-артефакт), і потрібна Revalidation, перш ніж Story продовжить |
| Integrity Error | Стан, History або артефакти суперечать одне одному (розділ 12), або не виконано Review Binding Guard (`stage-map.yaml` RR-10) |
| Integration Mismatch | Merge вніс у target branch не ті зміни, які погодила людина, або разом із ними непогоджений вміст |
| Post-Merge Validation Failed | Target Base Identity змінилася до merge, а перевірка фактичного інтегрованого результату не пройшла (OD-16). Діагностика — у `post-merge-validation-record` |
| Foundation Rule Violation | Порушено обов'язкове правило Foundation, яке не можна усунути через Risk Acceptance (WF-V2-01). Вирішення — виправлення, Foundation Change Request або дозволений виняток |
| Merge Protection Unavailable | Механізм захисту підтримується, але об'єктивно недоступний (`merge_protection_statuses.UNAVAILABLE`, FR-02). Людина в Recovery погоджує допустимий порядок інтеграції |
| Merge Protection Disabled | Механізм доступний і обов'язковий за Merge Protection Policy, але вимкнений (`merge_protection_statuses.DISABLED`, FR-02). `RESUME` лише після ввімкнення захисту або за чинним `foundation-exception-record` |

Перелік причин — робочий. Остаточні назви визначає етап Artifact Contracts.

### Дії Recovery

| Дія | Наслідок |
|---|---|
| `RESUME` | Story повертається в `ACTIVE` на тій самій стадії і починає нову спробу. Людина може водночас дозволити додаткові спроби понад Retry Budget і виконати RECONFIRM для `SUSPENDED` approvals (розділ 7) |
| `ROUTE_BACK` | Story переходить на ціль із `recovery_route_back_targets` у `stage-map.yaml`. Рішення людини водночас є підтвердженням повернення через gates. Далі — Rework Protocol |
| `CANCEL` | Story переходить у `CANCELLED` через Cancellation Protocol ([workflow-policies.md](workflow-policies.md#17-cancellation-protocol-od-13)) |
| `KEEP_BLOCKED` | Story лишається `BLOCKED`. Рішення і причина фіксуються |

**Integration Mismatch і Post-Merge Validation Failed.** Агент не виконує автоматичного виправлення вже виконаного merge: ні merge, ні revert, ні force-push, ні нових commits у target branch. Людина обирає:

| Дія | Коли |
|---|---|
| `RESUME` | Людина виправила target branch поза Story (сама або через іншу Story). INTEGRATION_CONFIRMATION повторює перевірки |
| `KEEP_BLOCKED` | Корекція ще триває, наприклад через нову пов'язану BUG-Story |
| `CANCEL` | `cancellation-record` фіксує, що зміни вже інтегровано і перевірку не пройдено |

`ROUTE_BACK` можливий лише на HUMAN_MERGE (`recovery_route_back_targets`).

Integration Match завжди оцінює результат merge, зафіксований останнім CONFIRM_MERGED, а не поточний стан target branch. Post-Merge Validation перевіряє поточний стан target branch. `RESUME` повторює перевірки для того самого merge. Повторний merge після корекції проходить через `ROUTE_BACK` → HUMAN_MERGE → новий CONFIRM_MERGED.

**Завершення Story після зовнішнього виправлення (OD-17).** Зовнішнє виправлення саме по собі Story не завершує.

- **Post-Merge Validation Failed.** Story може перейти в DONE лише тоді, коли одночасно:
  1. первісний Integration Match = `PASS`;
  2. причину невдачі виправила інша Story або людина;
  3. людина дозволила `RESUME` через Recovery Protocol;
  4. повторна Post-Merge Validation на поточному стані target branch = `PASS`;
  5. виконано всі інші умови Definition of Done.
- **Integration Mismatch.** Успішне виконання іншої Story не виправдовує первісної інтеграції. DONE можливий лише за доказу, що погоджена Changeset Content Identity цієї Story інтегрована належно, і за виконання всіх умов DoD. Таким доказом є новий merge саме погоджених змін після корекції людиною: `ROUTE_BACK` → HUMAN_MERGE → новий CONFIRM_MERGED → Integration Match `PASS`. `RESUME` для первісного невідповідного merge дає лише повторне `BLOCKED`. Якщо цього не можна підтвердити, Story лишається `BLOCKED` або завершується `CANCELLED` за рішенням людини.

Ці правила задає також `post_merge_validation.resume_rules` у `stage-map.yaml`.

### Процедура

1. **Зафіксувати блокування:** причина, джерело (Stage Verdict / Orchestrator), докази, що потрібно для відновлення.
2. **Отримати рішення людини** з переліку дій Recovery. Агент не ухвалює рішення Recovery сам.
3. **Перевірити передумови:** для `RESUME` — причину блокування усунено або рішення ескалації зафіксоване в канонічному джерелі, Review Binding Guard виконується для поточної стадії, а approvals, на які спирається поточна стадія, мають стан `VALID` (після RECONFIRM, якщо потрібно); для `ROUTE_BACK` — ціль дозволена `recovery_route_back_targets`.
4. **Змінити стан** згідно з дією.
5. **Записати подію** Recovery в History: рішення, хто його ухвалив, обґрунтування.

Якщо передумова не виконується, дія не застосовується, а Story лишається `BLOCKED`.

## 10. Retry Policy

Retry Budget не може бути нескінченним (Q15). Retry Policy конфігурована (OD-08). Погоджені початкові значення конфігурації задає `retry_policy` у `stage-map.yaml`: Technical Attempts — 3, Quality Loops — 3, Repeated Failure — 3. Це не незмінні глобальні константи: їх можна змінити лише зміною Workflow Definition, яку погоджує людина.

### Облік

| Лічильник | Що рахує | Ключ обліку |
|---|---|---|
| Technical Attempts | Послідовні спроби стадії, що завершилися `INTERRUPTED`, `FAILED` або `INVALID_RESULT` | Стадія |
| Quality Loops | Повернення через `CHANGES_REQUIRED` від агентних стадій | Окремо: producer-стадія-ціль і стадія-ініціатор повернення. Ліміт діє для кожного ключа |
| Repeated Failure | Кількість появ тієї самої **Finding Identity**. Ліміт 3 досягається на третій появі проблеми, тобто на другому повторенні після того, як її позначили виправленою | Finding Identity в межах усієї Story, незалежно від стадії, яка його виявила |

**Finding Identity** — логічна ідентичність проблеми: що порушено (Acceptance Criterion, правило, вимога дизайну) і де. Формат визначає етап Artifact Contracts. Та сама проблема, виявлена іншою стадією або після повернення на іншу стадію, має ту саму Finding Identity. Однакова технічна помилка (той самий тип збою тієї самої стадії) обліковується так само.

Облік за двома ключами Quality Loops не дає обійти ліміт: review-стадія, що повертає Story на різні producer-стадії, вичерпує ліміт стадії-ініціатора, а producer-стадія, на яку повертають різні review, — ліміт цілі.

### Скидання

- Technical Attempts скидається, коли стадія повертає прийнятий результат (`COMPLETED`).
- Quality Loops для producer-стадій, артефакти яких входять до Approval Scope gate, і для review-стадій перед цим gate скидаються лише повним рішенням людини APPROVE на цьому gate.
- Repeated Failure не скидається в межах Story.
- RECONFIRM, `RESUME`, `ROUTE_BACK`, REQUEST_CHANGES і нова сесія лічильників не скидають.

### Вичерпання і розширення

- Після вичерпання будь-якого ліміту Story переходить у `BLOCKED` з причиною Retry Budget Exhausted. Повторення Finding Identity до ліміту Repeated Failure веде до ранньої ескалації, не чекаючи вичерпання Quality Loops.
- Агент не збільшує бюджет. Розширення можливе лише рішенням людини через Recovery: `RESUME` з указаною кількістю додаткових спроб для конкретного лічильника або `ROUTE_BACK` на інший маршрут. Розширення фіксується в History.
- Очікування рішення людини не витрачає Retry Budget.
- Рішення людини REQUEST_CHANGES на gate не зараховується до агентних лічильників. Наступні повернення від агентних стадій зараховуються як звичайно.
- Зміна значень за замовчуванням — це зміна Workflow Definition, яку погоджує людина.
- Історія спроб, лічильників і розширень бюджету не видаляється. Скидання лічильника — це подія в History, а не видалення попередніх записів.

## 11. Event History

History — append-only журнал подій Story (Q9). Подію не змінюють і не видаляють.

Події фіксуються для:

- створення Story і зміни Execution Mode;
- початку, завершення, переривання, збою та відхилення результату спроби стадії;
- застосованих і відхилених переходів та Loopbacks;
- запитів і рішень Human Gates (APPROVE, RECONFIRM, REQUEST_CHANGES, CANCEL), human actions і підтверджень Loopback через gate;
- кожного входу в Rework Protocol: механізм, джерело, ціль, причина (розділ 14);
- створення версій контрольованих артефактів, зміни Currency, `revalidation-record`, змін Approval State;
- `applicability-decision` і їхньої повторної оцінки;
- спрацювання Review Binding Guard і змін метаданих PR;
- блокувань, рішень Recovery, вичерпання і розширення Retry Budget;
- Foundation Change Request і його вирішення;
- перевірки перед merge (зокрема Merge Protection Status), підтвердження merge людиною та результату INTEGRATION_CONFIRMATION, зокрема `post-merge-validation-record`;
- `risk-acceptance-record` і `foundation-exception-record`;
- створення `backlog-entry` для FOLLOW_UP;
- відновлення сесії та виявлених порушень цілісності;
- завершення (`DONE`) і скасування (`CANCELLED`) Story, зокрема `cancellation-record`.

Кожна подія Gate Decision і Recovery фіксує, хто ухвалив рішення і щодо яких Approved Content Identity. Склад полів подій визначає етап Artifact Contracts.

## 12. Session Recovery

Стан процесу не залежить від сесії Claude Code (Q9). Нова сесія відновлює роботу лише з Workflow State, History і канонічних артефактів.

Під час відновлення Orchestrator:

1. **Завантажує стан** обраної Story.
2. **Перевіряє цілісність:** стан узгоджений з останніми подіями History; артефакти, на які посилається стан, існують і мають записані версії; feature branch відповідає Git Binding. Розбіжність не виправляється автоматично: Story переходить у `BLOCKED` з причиною Integrity Error.
3. **Обробляє незавершену спробу:** якщо Execution State — `RUNNING`, але результату немає, спроба отримує `INTERRUPTED`. Це не verdict `BLOCKED`. Часткові результати перерваної спроби не вважаються завершеними виходами. Нова спроба може використати їх лише після перевірки.
4. **Перевіряє Staleness і зміни поза Workflow:** чи змінилися з останнього запуску Foundation, upstream-артефакти, вміст артефактів, Changeset Content Identity чи Target Base Identity. Виявлені зміни обробляє Rework Protocol (розділ 14).
5. **Продовжує роботу** згідно з Execution Mode: нова спроба стадії в межах Technical Attempts або очікування рішення людини.

## 13. Що визначають наступні етапи

| Тема | Етап |
|---|---|
| Схеми файлу стану і файлу History, шляхи, ідентифікатори подій | Artifact Contracts |
| Спосіб версіонування артефактів і запису залежностей | Artifact Contracts |
| Механізм Approved Content Identity, Changeset Content Identity, Target Base Identity | Artifact Contracts |
| Формат Finding Identity, Applicability Basis, `revalidation-record`, `applicability-decision`, `post-merge-validation-record`, `risk-acceptance-record`, `foundation-exception-record`, `cancellation-record` | Artifact Contracts |
| Виявлення змін Applicability Basis, Merge Protection Status | Story Orchestrator, Stage Skills |
| Перелік залежностей для кожного типу артефакта | Artifact Contracts |
| Порядок запису стану і History та виявлення розбіжностей | Story Orchestrator |
| Блокування одночасної роботи кількох сесій з однією Story | Story Orchestrator |
| Обчислення Currency, Approval State і Review Binding Guard | Story Orchestrator |
| Тести переходів, gates, Recovery, Retry, Staleness, Session Recovery, Review Binding Guard | Harness Validation |

## 14. Rework Protocol

Rework Protocol — єдина процедура для будь-якого повернення Story на попередню стадію або повторної перевірки після змін (WF-04, `stage-map.yaml` RR-11). Окремі механізми не мають власних правил інвалідації.

### Процедура

1. **Перевірити маршрут** для механізму: ціль дозволена `stage-map.yaml` (loopbacks, decisions gate, `recovery_route_back_targets`), а `condition` маршруту виконується.
2. **Отримати рішення людини**, якщо повернення перетинає Human Gate або механізм за визначенням є рішенням людини.
3. **Записати подію** входу в Rework Protocol: механізм, джерело, ціль, причина, хто ухвалив рішення.
4. **Перейти на ціль.** Стадія-ціль виконує роботу. Якщо вона створює нову версію артефакта, попередня стає `SUPERSEDED`.
5. **Оновити Currency** залежних версій: ті, що посилаються на попередню версію upstream і не мають `revalidation-record` для нової, стають `STALE` (розділ 8).
6. **Оновити Approval State:** approvals, у Scope яких входить `SUPERSEDED` версія або замінений supporting-запис, стають `INVALIDATED`; approvals, для яких upstream змінився від Approval Basis, стають `SUSPENDED` (розділ 7).
7. **Пройти наступні стадії** за порядком. Кожна стадія виконує Revalidation (розділ 8). Review-стадії перевіряють нові версії й нові `revalidation-record`. Optional Stages заново оцінюють застосовність, якщо змінився будь-який релевантний факт Applicability Basis (розділ 6), зокрема через саме це повернення, рішення людини чи зміну Retry Counters. Review Binding Guard діє для стадій після CODE_REVIEW.
8. **На кожному Human Gate** людина ухвалює APPROVE (якщо approval `INVALIDATED`) або може ухвалити RECONFIRM (якщо виконуються умови розділу 7). Auto Mode зупиняється.

### Застосування до механізмів

| Механізм | Кроки 1–2 (маршрут і рішення людини) | Кроки 3–8 |
|---|---|---|
| Stage Loopback (`CHANGES_REQUIRED`) | Ціль із `loopbacks`. Людина — якщо є `crosses_gates` | Однаково |
| REQUEST_CHANGES на Human Gate | Ціль із decisions gate. Рішення людини | Однаково |
| REQUEST_CHANGES на HUMAN_MERGE | Ціль із decisions HUMAN_MERGE. Рішення людини | Однаково |
| `loopback_confirmation` ROUTE_BACK | Ціль із `loopbacks`. Рішення людини | Однаково |
| Recovery ROUTE_BACK | Ціль із `recovery_route_back_targets`. Рішення людини | Однаково |
| Recovery RESUME | Ціль — поточна стадія. Рішення людини. Передумови розділу 9 | Кроки 5–6 для змін, що сталися під час блокування. Для approvals попередніх gates — Out-of-Stage Revalidation і RECONFIRM у межах рішення Recovery (розділи 7, 8) |
| Відновлення після Foundation Change Request | `RESUME` або `ROUTE_BACK` за результатом Change Impact Assessment. Рішення людини | Однаково. Foundation — upstream для всіх артефактів. Для `RESUME` — Out-of-Stage Revalidation і RECONFIRM |
| Staleness, виявлена під час Session Recovery | Orchestrator фіксує зміну. Story переходить у `BLOCKED` (Upstream Changed), далі — Recovery | Кроки 5–6 одразу, далі — однаково |
| Зміна артефакта поза Workflow | Виявляється як нова версія або нова Changeset Content Identity. Story переходить у `BLOCKED` (Integrity Error). Рішення людини через Recovery | Однаково. Нова Changeset Content Identity потребує ROUTE_BACK на IMPLEMENTATION або раніше (RR-10) |
