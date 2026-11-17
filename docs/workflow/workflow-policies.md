# Workflow Policies — Workflow Definition

**Статус документа:** `APPROVED` (revision 3). Погоджено людиною 2026-11-16 після фінального Review.

## 1. Призначення

Документ визначає нормативні політики Story Workflow: межі автономії агента, Human Gates, Reviews, Optional Stages, зміни Foundation, Findings, Definition of Done, Git-інтеграцію, режими виконання та ізоляцію Stories.

Маршрути визначає [stage-map.yaml](stage-map.yaml). Стадії описано в [stages.md](stages.md). Модель стану — у [workflow-state-model.md](workflow-state-model.md).

Політики не залежать від технологій проєкту. Технологічні правила (мова, фреймворки, сховища, конкретні Security Rules) належать Project Foundation і застосовуються стадіями через посилання на нього. Ці політики описують, що Foundation має бути дотримано, але не дублюють його правил.

Нормативні правила документа ще не мають програмної перевірки. Їх забезпечуватимуть Orchestrator, Stage Skills і Harness Validation на наступних етапах.

## 2. Controlled Autonomy

Агент ухвалює технічні рішення в межах погоджених документів (Q7): Foundation, погодженої специфікації, погодженого дизайну та Implementation Plan.

**Агент вирішує сам:**

- деталі реалізації, які не суперечать погодженому дизайну й Foundation;
- внутрішню структуру коду в межах правил Foundation;
- склад і форму тестів у межах тестового підходу плану;
- формулювання артефактів, які створює його стадія.

**Потрібна ескалація до людини:**

| Ситуація | Як обробляється |
|---|---|
| Зміна Product Foundation | Foundation Change Request (розділ 6) |
| Зміна Architecture Foundation | Foundation Change Request (розділ 6) |
| Суттєва нова залежність (критерії нижче) | `BLOCKED` (Escalation Required). Якщо Foundation вимагає оновлення документа для нової технології, — Foundation Change Request |
| Послаблення чи порушення обов'язкового правила Foundation, зокрема Security Rules | `BLOCKED` (Foundation Rule Violation). Лише Foundation Change Request або дозволений виняток (розділ 7). Ескалація і Risk Acceptance правила не послаблюють |
| Зміна погоджених вимог | Loopback через gate з рішенням людини |
| Прийняття значного ризику, що не порушує обов'язкових правил Foundation | `BLOCKED` (Escalation Required) → Risk Acceptance людиною (розділ 7) |

Додаткові правила:

- Якщо вимога або важливе архітектурне рішення не визначені й відповіді немає в документації, агент пояснює невизначеність і просить рішення людини. Агент не вигадує вимог.
- Рішення людини за ескалацією записується в канонічний артефакт, якого воно стосується. Запису лише в History недостатньо.
- Агент не пропонує Risk Acceptance для порушення обов'язкового правила Foundation (`stage-map.yaml` RR-14).
### Risk-Based Dependency Approval (OD-09)

Нова залежність або зміна наявної потребує погодження людини, якщо вона суттєво впливає хоча б на одне:

- **архітектуру** — нова технологія в ролі компонента системи, заміна наявного компонента, новий runtime-сервіс;
- **безпеку** — криптографія, автентифікація, розбір недовірених даних, мережева взаємодія, доступ до секретів;
- **операційні вимоги** — новий сервіс чи інфраструктура для запуску, суттєві вимоги до ресурсів чи середовища;
- **production credentials** — потрібні нові облікові дані, ключі чи секрети для production;
- **ліцензійні зобов'язання** — ліцензія з обов'язками для проєкту або нестандартна чи комерційна ліцензія;
- **сумісність** — зміна версії мови чи runtime, major-оновлення наявної залежності, breaking changes.

Правила:

- Прості допоміжні залежності, які не відповідають жодному критерію, агент може запропонувати в `solution-design` або `implementation-plan` у межах Controlled Autonomy. Людина бачить їх на HUMAN_TECHNICAL_APPROVAL.
- Незапланована залежність, яка з'явилася під час IMPLEMENTATION, фіксується в changeset. CODE_REVIEW перевіряє її класифікацію. Суттєва незапланована залежність → `BLOCKED` (Escalation Required).
- Якщо класифікація неоднозначна, залежність вважається суттєвою.
- Суворіші правила Project Foundation мають пріоритет. Для Customer Portal діє правило Architecture Foundation щодо нових технологій.
- Формальної оцінки ризику залежностей (scoring) Workflow не передбачає.

## 3. Human Gates

Три Human Gates погоджують артефакти (Q5):

| Gate | Що погоджує |
|---|---|
| HUMAN_SPEC_APPROVAL | Вимоги Story: `story-spec`, `clarification-record` |
| HUMAN_TECHNICAL_APPROVAL | Технічний дизайн і Implementation Plan разом |
| HUMAN_PR_APPROVAL | Конкретну версію змін Story у PR |

HUMAN_MERGE — окрема дія доставки, яку підтверджує людина. Вона не є погодженням артефактів.

Правила:

- **Gate не можна обійти автоматично.** Жоден режим виконання, verdict чи конфігурація не замінює рішення людини.
- **Reviewer PASS не є Human Approval.** Gate відкривається лише після `PASS` відповідних review-стадій, але й тоді потребує рішення людини.
- **Approval прив'язаний до Approved Content Identity** кожного артефакта Approval Scope (`approval_scope` у `stage-map.yaml`). Правила валідності — [workflow-state-model.md](workflow-state-model.md#7-approval-validity).
- **Пакет рішення.** Людина отримує артефакти Scope, звіти review, `applicability-decision`, Findings Disposition, ризики та ескалації. Під час повторного проходження — зміни відносно погодженої identity, результати Change Impact Assessment і `revalidation-record`.
- **Рішення.** Людина обирає одне з рішень gate у `stage-map.yaml`. REQUEST_CHANGES має містити ціль із дозволених для цього gate і причину. Далі діє Rework Protocol.
- **Повторне проходження (OD-06).** Gate — обов'язкова зупинка під час кожного проходження. Якщо погоджений вміст не змінився, approval має стан `VALID` або `SUSPENDED`, а кожну upstream-зміну після останнього рішення людини покрито перевіреним `revalidation-record`, людина може виконати скорочене RECONFIRM. `SUSPENDED` approval не стає `VALID` без рішення людини. Якщо погоджена identity змінилася або approval має стан `INVALIDATED`, потрібне повне APPROVE. RECONFIRM не є погодженням зміненого вмісту і фіксується окремою подією з посиланням на початковий approval.
- **Auto Mode** не виконує APPROVE чи RECONFIRM і не обирає рішення замість людини.
- **Межа повноважень gate (WF-V2-01).** Рішення людини на gate не може прийняти порушення обов'язкового правила Foundation, порушення Acceptance Criterion чи невиконану умову DoD. APPROVE за наявності такого BLOCKING finding неможливий. Можливі рішення — REQUEST_CHANGES, CANCEL або окремий Foundation Change Request.
- Рішення людини фіксується в Workflow State і History разом з Approved Content Identity, до якої воно належить.

## 4. Review Policies

**Обов'язковість (Q28).** Review обов'язковий, якщо застосовна producer-стадія, яку він перевіряє. Відповідність producer → review задає поле `reviewed_by` у `stage-map.yaml`. Обійти Review через `NOT_APPLICABLE` не можна:

- обов'язкові review-стадії (SPEC_REVIEW, DESIGN_REVIEW, PLAN_REVIEW, CODE_REVIEW) не мають `optional: true` і не можуть повернути `NOT_APPLICABLE`;
- кожна обов'язкова producer-стадія перевіряється обов'язковою review-стадією або Human Gate;
- SECURITY_REVIEW — додаткова спеціалізована перевірка. Її `NOT_APPLICABLE` не звільняє зміни від DESIGN_REVIEW, CODE_REVIEW і базових security-перевірок у VERIFICATION.
- Зміна Changeset Content Identity після CODE_REVIEW потребує нового CODE_REVIEW, а якщо SECURITY_REVIEW застосовна, — і нового SECURITY_REVIEW. Цього вимагає Review Binding Guard (`stage-map.yaml` RR-10). Зміна лише метаданих PR нового review не потребує (RR-12).

**Дворівнева security-перевірка (OD-05):**

1. DESIGN_REVIEW перевіряє security-рішення на рівні архітектури для кожної Story.
2. SECURITY_REVIEW, якщо застосовна, виконує поглиблену перевірку реалізації після CODE_REVIEW.

Окремої Security Design Review Stage немає.

**Незалежність (Q6) — гібридна модель:**

- Producer і Reviewer мають різні логічні ролі й відповідальності.
- Reviewer не виправляє артефакт, який перевіряє. Він фіксує findings і verdict. Виправлення робить producer-стадія після повернення.
- Reviewer отримує канонічні артефакти, а не робочий контекст producer-стадії.

**Ізоляція контексту (OD-10).** Ізольований контекст означає, що reviewer виконується окремо від producer-стадії і отримує лише канонічні артефакти, `revalidation-record` та джерела Foundation. Робочого контексту, міркувань і чернеток producer-стадії він не бачить. Механізм реалізації (наприклад, Subagents) визначає наступний етап.

- Ізоляція обов'язкова завжди для DESIGN_REVIEW, CODE_REVIEW і SECURITY_REVIEW.
- Для SPEC_REVIEW і PLAN_REVIEW ізоляція обов'язкова, якщо спрацьовує хоча б один тригер підвищеного ризику з `review_context_isolation.risk_triggers` у `stage-map.yaml`: Story Type SECURITY чи INFRASTRUCTURE; зміни автентифікації, авторизації, сесій, облікових даних чи чутливих даних; заплановано SECURITY_REVIEW (для PLAN_REVIEW); артефакт створено після повернення через Human Gate; спрацював Repeated Failure або розширено Retry Budget; ізоляції вимагає людина.
- Якщо застосовність тригера неоднозначна, ізоляція потрібна.

**Прив'язка до версій.** Review-звіт дійсний лише для тих версій артефактів і upstream-версій, щодо яких виконано перевірку. Після нової версії потрібен новий review. Для незмінної версії з новим `revalidation-record` review перевіряє цей запис.

## 5. Optional Stage Policies

Workflow один для всіх Stories (Q21). Складність Story впливає лише на застосовність Optional Stages: IMPACT_ANALYSIS, API_DESIGN, DB_DESIGN, SECURITY_REVIEW, RECONCILIATION.

Правила:

Умови застосовності кожної Optional Stage визначено в `applicability_conditions` у `stage-map.yaml` як перелік тригерів (OD-07). Там само визначено спільні правила AP-01…AP-07. Коротко:

- Стадія застосовна, якщо виконується хоча б один тригер. Якщо застосовність неоднозначна, стадія вважається застосовною.
- `NOT_APPLICABLE` потребує `applicability-decision`: для кожного тригера — чому він не виконується, з посиланням на джерело.
- `applicability-decision` прив'язаний до Applicability Basis — усіх релевантних фактів рішення: версій артефактів, Changeset Content Identity, фактів Workflow State, релевантних подій History, Retry Counters, рішень людини, Story Risk Assessment (WF-V2-02, AP-04). Релевантні категорії для кожної стадії задає `basis_facts`.
- Якщо релевантний факт змінився, рішення переоцінюється перед використанням (AP-08). Зокрема, повернення через Human Gate, Repeated Failure, розширення Retry Budget, нові security-findings, рішення людини застосувати Review, Recovery після BLOCKED чи зміна Story Risk Assessment не дозволяють повторно використати попередній `NOT_APPLICABLE` для SECURITY_REVIEW чи RECONCILIATION. Перевіряє стадія на вході, стадія з `na_checked_by` і Orchestrator у Review Binding Guard.
- Події поза `basis_facts` стадії її рішення не інвалідують (AP-09).
- Заплановане рішення в Implementation Plan — прогноз, що ґрунтується лише на артефактах плану. Після входу в стадію чинне рішення на вході (AP-10).
- Обґрунтування `NOT_APPLICABLE` перевіряє стадія з поля `na_checked_by` і бачить людина на найближчому Human Gate.
- Implementation Plan декларує очікувану застосовність SECURITY_REVIEW і RECONCILIATION. Людина погоджує її на HUMAN_TECHNICAL_APPROVAL і може вимагати застосування стадії.
- Стадію, яку погоджено як застосовну, не можна пізніше оголосити `NOT_APPLICABLE` без рішення людини. Незастосовну стадію агент може оголосити застосовною без додаткового погодження.
- Автоматичного Risk Scoring Engine немає: тригери оцінює стадія, а перевіряє reviewer і людина.

**SECURITY_REVIEW (Q22).** Для змін, що відповідають умові застосовності, стадія обов'язкова. Базові security-перевірки залишаються частиною VERIFICATION для всіх Stories.

**RECONCILIATION (Q27).** Повний Reconciliation виконується для Stories, які відповідають умові застосовності. Базова перевірка відповідності вимогам у VERIFICATION обов'язкова для всіх Stories.

## 6. Foundation Change Management

Stage Skill не змінює погоджену Product чи Architecture Foundation (Q19).

Якщо Story потребує зміни Foundation:

1. **Story зупиняється.** Стадія повертає `BLOCKED` з причиною Foundation Change Required.
2. **Формується Change Request** (`foundation-change-request`): що змінити, чому, які погоджені рішення це зачіпає, альтернативи в межах чинної Foundation.
3. **Людина розглядає зміну** і погоджує або відхиляє її.
4. **Foundation оновлюється** після погодження. Оновлення — окрема робота поза Story Workflow, яку погоджує людина.
5. **Виконується Change Impact Assessment** для артефактів цієї Story. Інші нетермінальні Stories перевіряють Staleness під час свого наступного запуску (розділ 10, [Session Recovery](workflow-state-model.md#12-session-recovery)), бо один запуск Orchestrator працює лише з однією Story.
6. **Залежні результати проходять Revalidation** ([workflow-state-model.md](workflow-state-model.md#8-staleness-і-revalidation)). Для незмінних версій створюються `revalidation-record`, для зачеплених — нові версії.
7. **Story відновлюється** через Recovery Protocol: `RESUME` або `ROUTE_BACK` за правилами Rework Protocol ([workflow-state-model.md](workflow-state-model.md#14-rework-protocol)). Для `SUSPENDED` approvals попередніх gates виконується Out-of-Stage Revalidation ([workflow-state-model.md](workflow-state-model.md#8-staleness-і-revalidation)), після чого людина підтверджує їх через RECONFIRM у межах цього рішення. Якщо approval має стан `INVALIDATED`, потрібен `ROUTE_BACK` на producer-стадію відповідного артефакта.

Якщо зміну відхилено, Story або адаптується в межах чинної Foundation (`ROUTE_BACK`), або скасовується (`CANCEL`).

Суперечність між документами Foundation агент повідомляє і не вирішує сам на користь одного з документів.

## 7. Findings Disposition

Кожен finding review- чи verification-стадії має клас (Q23):

| Клас | Значення | Наслідок |
|---|---|---|
| BLOCKING | Порушення, яке не дозволяє рухатися далі | Verdict `CHANGES_REQUIRED` або `BLOCKED`. Має бути усунений до `PASS` |
| NON_BLOCKING | Недолік, який не порушує вимог і правил | Виправляється або приймається з обґрунтуванням у Findings Disposition |
| FOLLOW_UP | Робота поза scope Story | Потребує погодження людини на HUMAN_PR_APPROVAL. Погоджений FOLLOW_UP стає `backlog-entry` у канонічному Backlog |

Правила:

- Порушення обов'язкового Acceptance Criterion завжди BLOCKING. Його не можна перенести у FOLLOW_UP.
- Порушення обов'язкового правила Product, Architecture чи Security Foundation завжди BLOCKING, доки його не вирішено належно (див. нижче, `stage-map.yaml` RR-14).
- Клас пропонує reviewer. Агент не знижує клас BLOCKING. Людина може знизити клас лише для finding, який не є порушенням Acceptance Criterion, обов'язкового правила Foundation чи умови DoD. Рішення фіксується.
- BLOCKING не переводиться у FOLLOW_UP, якщо це дозволило б обійти обов'язкову вимогу. Автоматичного зниження класу немає.
- Findings Disposition (як вирішено кожен finding) входить до `verification-report` і пакета HUMAN_PR_APPROVAL.

### Risk Acceptance і Foundation Rules (WF-V2-01)

| Випадок | Що можливо | Запис |
|---|---|---|
| **A. Risk Acceptance без порушення Foundation** | Людина явно приймає обґрунтований ризик, який не суперечить обов'язковим правилам Product, Architecture чи Security Foundation. Finding може отримати Disposition «ризик прийнято» | `risk-acceptance-record` (Finding Identity, ризик, обґрунтування, хто ухвалив, межі) і Findings Disposition. Подія в History |
| **B. Порушення обов'язкового правила Foundation** | Risk Acceptance і рішення на gate порушення не усувають. Варіанти: виправити реалізацію; погоджений Foundation Change Request з оновленням Foundation (розділ 6); контрольований виняток, лише якщо саме це правило Foundation прямо допускає виняток, — за процедурою, визначеною в Foundation | `foundation-change-request` або `foundation-exception-record` з посиланням на положення Foundation, яке допускає виняток. Подія в History |

Правила:

- Якщо правило Foundation прямо не допускає винятку, Human Approval не може його створити. Залишаються лише виправлення або Foundation Change Request.
- Finding випадку B лишається BLOCKING, доки немає виправлення, оновленої Foundation або чинного `foundation-exception-record`.
- Для Customer Portal Foundation не містить загальної процедури винятків. Окремі правила самі визначають умови свого винятку (наприклад, «лише окремим погодженням»). Workflow не розширює цих умов.
- Якщо неочевидно, чи порушує ризик обов'язкове правило Foundation, застосовується випадок B.
- Кожен finding має Finding Identity ([workflow-state-model.md](workflow-state-model.md#10-retry-policy)). Повторна поява тієї самої проблеми — той самий finding, навіть якщо його виявила інша стадія.

### FOLLOW_UP Backlog (OD-11)

- Кожен погоджений людиною FOLLOW_UP створює відстежуваний запис `backlog-entry` у канонічному Backlog проєкту.
- `backlog-entry` посилається на початкову Story і на Finding Identity.
- Записи створює INTEGRATION_CONFIRMATION до переходу Story в DONE. Story не стає DONE, поки для кожного погодженого FOLLOW_UP немає `backlog-entry`.
- `backlog-entry` живе незалежно від Story: перехід Story в DONE чи CANCELLED його не закриває і не видаляє.
- Backlog — спільний артефакт проєкту, а не артефакт Story. Створення `backlog-entry` — дозволений виняток з ізоляції Stories (розділ 10).
- Канонічного Backlog ще немає. Його наявність — передумова (розділ 15). Структуру запису визначає етап Artifact Contracts.

## 8. Definition of Done

Story отримує `DONE` лише тоді, коли виконано всі умови (Q4, Q23).

**Перевіряє VERIFICATION:**

1. Усі обов'язкові стадії мають `PASS`, а Optional Stages — `PASS` або `NOT_APPLICABLE` з перевіреним `applicability-decision`.
2. Обов'язкові review виконано для поточної Changeset Content Identity і поточних версій артефактів (Review Binding Guard).
3. Усі контрольовані артефакти мають Currency `CURRENT`, а review-звіти прив'язані до поточних версій.
4. Кожен Acceptance Criterion має фактичний доказ виконання.
5. Перевірки, які вимагають Testing Strategy проєкту, Story Risk Assessment і Testing Approach плану, виконано й пройдено (розділ 18).
6. Нерозв'язаних BLOCKING findings немає. Порушення обов'язкових правил Foundation вирішено виправленням, Foundation Change Request або чинним `foundation-exception-record`, а не Risk Acceptance. NON_BLOCKING і FOLLOW_UP мають Disposition.
7. Рішення людини, зафіксовані в History, відображені в канонічних артефактах. Це перевіряє лише VERIFICATION. RECONCILIATION цього не дублює.

**Підтверджує людина:**

8. HUMAN_PR_APPROVAL має стан `VALID` для поточної Changeset Content Identity.
9. HUMAN_MERGE виконано.

**Підтверджують MERGE_READINESS_CHECK і INTEGRATION_CONFIRMATION:**

10. Перед merge перевірено актуальність feature branch і target branch (розділ 9).
11. Merge вніс у target branch саме погоджені зміни, без непогодженого вмісту.
12. Якщо Target Base Identity на момент merge відрізнялася від перевіреної, Post-Merge Validation пройдена.
13. Для кожного погодженого FOLLOW_UP створено `backlog-entry` (розділ 7).

Готовність до PR чи відкритий PR не означають DONE.

## 9. Git Integration

Правила (Q8):

- Кожна Story працює в окремій feature branch.
- Агент може створювати commits у feature branch і готувати PR до основної гілки (target branch).
- Merge в target branch виконує лише людина.
- DONE можливий лише після підтвердженої інтеграції погоджених змін.

### Ідентичність змін

Перевірки й approvals прив'язуються не лише до Git SHA, а до двох ідентичностей ([workflow-state-model.md](workflow-state-model.md#5-artifact-status)):

- **Changeset Content Identity** — зміст змін Story. Перебудова commits (squash, rebase без конфліктів, перейменування commits) її не змінює, якщо зміст змін той самий.
- **Target Base Identity** — версія target branch, щодо якої перевірено інтеграційний результат.

Нову Changeset Content Identity створює лише IMPLEMENTATION. Зокрема, розв'язання конфлікту з target branch — це зміна вмісту, яка виконується в IMPLEMENTATION і проходить CODE_REVIEW, SECURITY_REVIEW (якщо застосовна) і VERIFICATION (`stage-map.yaml` RR-10).

### Merge Validation Protocol (OD-12)

1. **Прив'язка перевірки.** VERIFICATION прив'язує `verification-report` до Changeset Content Identity і Target Base Identity.
2. **Перевірка перед merge.** MERGE_READINESS_CHECK після HUMAN_PR_APPROVAL перевіряє, що feature branch містить погоджену Changeset Content Identity, PR відповідає погодженій identity, а target branch не змінилася відносно перевіреної Target Base Identity.
3. **Target branch змінилася.** Якщо вміст змін Story той самий, потрібна перевірка інтеграційного результату: повернення на VERIFICATION з condition `CHANGESET_CONTENT_UNCHANGED`. Новий `verification-report` змінює Approval Scope, тому HUMAN_PR_APPROVAL стає `INVALIDATED`.
4. **Потрібна зміна коду для інтеграції.** Повернення на IMPLEMENTATION. Далі CODE_REVIEW, SECURITY_REVIEW (якщо застосовна), RECONCILIATION (якщо застосовна) і VERIFICATION.
5. **Застарілий approval.** `INVALIDATED` HUMAN_PR_APPROVAL потребує повного APPROVE. RECONFIRM можливий лише тоді, коли жоден артефакт Scope не змінився.
6. **Merge виконує лише людина** (HUMAN_MERGE). Агент не виконує merge, revert чи push в target branch.
7. **Підтвердження після merge.** INTEGRATION_CONFIRMATION перевіряє фактичний результат merge: зміни Story в target branch відповідають погодженій Changeset Content Identity, непогодженого вмісту немає. Потім порівнює Target Base Identity на момент merge з перевіреною.
8. **DONE** — лише після `PASS` INTEGRATION_CONFIRMATION.

Перевірка перед merge (крок 2) і підтвердження після merge (крок 7) — різні стадії. Між ними target branch може змінитися. Цей проміжок закривається двома рівнями (OD-16): Merge Protection запобігає неперевіреній інтеграції, а Post-Merge Validation контрольовано її перевіряє, якщо вона все-таки відбулася.

### Merge Protection — Prevention (OD-16)

- Якщо Git-платформа підтримує механізми захисту target branch (Branch Protection, Required Checks, заборона merge із застарілими перевірками або еквівалент), вони використовуються відповідно до **Merge Protection Policy** проєкту (`stage-map.yaml` RR-13). Merge Protection Policy визначає, які механізми обов'язкові. Це правило проєкту, тож на нього поширюється WF-V2-01. Workflow Core не залежить від конкретної платформи чи стратегії merge.
- MERGE_READINESS_CHECK фіксує **Merge Protection Status** і діє за ним (`merge_protection_statuses`, FR-02):

| Status | Випадок | Дія |
|---|---|---|
| `ACTIVE` | Механізм увімкнено | Звичайна перевірка перед merge |
| `NOT_SUPPORTED` | Платформа не підтримує механізм | Дозволена контрольована ручна процедура (нижче). Обмеження фіксується. Якщо Merge Protection Policy вимагає саме цього механізму, Story обробляється як `DISABLED` |
| `UNAVAILABLE` | Механізм підтримується, але об'єктивно недоступний | Обмеження фіксується. `BLOCKED` (Merge Protection Unavailable). Людина в Recovery погоджує допустимий порядок інтеграції. Рішення фіксується у Workflow State і History |
| `DISABLED` | Механізм доступний і обов'язковий за Merge Protection Policy, але вимкнений | `BLOCKED` (Merge Protection Disabled). Merge неможливий, доки людина не ввімкне захист або доки немає чинного `foundation-exception-record`, якщо Merge Protection Policy прямо допускає виняток. Ручна процедура цей випадок не замінює |

Механізм, доступний, але не обов'язковий за політикою і вимкнений, обробляється як `UNAVAILABLE`.

- **Контрольована ручна процедура** (`NOT_SUPPORTED` або погоджений порядок для `UNAVAILABLE`): людина виконує merge одразу після MERGE_READINESS_CHECK і разом із CONFIRM_MERGED підтверджує, що target branch не змінилася після перевірки. Якщо змінилася, людина не виконує merge, а обирає REQUEST_CHANGES → MERGE_READINESS_CHECK. Ручна процедура не обходить обов'язкових правил проєкту.
- Агент не змінює налаштування захисту репозиторію без дозволу людини.
- Merge Protection знижує ризик, але не замінює INTEGRATION_CONFIRMATION.

### Post-Merge Validation — Recovery (OD-16)

Post-Merge Validation — частина INTEGRATION_CONFIRMATION, а не окрема стадія. Вона використовує ті самі результати стадії (`PASS` → DONE, `BLOCKED`, технічний збій → `FAILED`), тож нових маршрутів, gates чи Loopbacks не потребує. Порядок перевірок у стадії обов'язковий (`post_merge_validation.check_order` у `stage-map.yaml`):

1. **Integration Match.** Оцінюється результат merge Story, а не пізніший стан target branch. Merge мав внести саме погоджену Changeset Content Identity без непогодженого вмісту. Інакше — `BLOCKED` (Integration Mismatch), а Post-Merge Validation не виконується. Так Story не може стати DONE з невідповідним changeset.
2. **Base Comparison.** Якщо Target Base Identity на момент merge дорівнює перевіреній, Post-Merge Validation не потрібна.
3. **Post-Merge Validation** — лише якщо база змінилася:
   - перевіряється фактичний інтегрований результат на поточному стані target branch;
   - виконуються релевантні інтеграційні перевірки й тести за Testing Strategy, Story Risk Assessment і Testing Approach;
   - результат і перевірений стан target branch фіксуються в незмінному `post-merge-validation-record`;
   - `PASS` дозволяє продовжити, `FAIL` дає `BLOCKED` (Post-Merge Validation Failed) з діагностикою в `post-merge-validation-record`;
   - технічний збій запуску перевірок — це `FAILED` у межах Technical Attempts, а не verdict.
4. **FOLLOW_UP Backlog.** Створюються `backlog-entry`.

Правила:

- Агент не змінює target branch: не виконує merge, revert чи push і не виправляє автоматично вже інтегрований код.
- Post-Merge Validation не інвалідує HUMAN_PR_APPROVAL, якщо Changeset Content Identity не змінилася. `post-merge-validation-record` не входить до Approval Scope ([workflow-state-model.md](workflow-state-model.md#7-approval-validity)).
- Невідповідність інтегрованого результату або зміна погодженого коду обробляються чинними правилами Approval Validity, Recovery Protocol і Rework Protocol.

**Незалежність від стратегії merge.** Протокол не залежить від того, чи використовується merge commit, squash merge чи rebase:

| Стратегія | Що перевіряє INTEGRATION_CONFIRMATION |
|---|---|
| Merge commit | Інтегрована гілка має погоджену Changeset Content Identity, а merge не додав іншого вмісту |
| Squash merge | Зміни squash-commit відносно Target Base Identity відповідають погодженій Changeset Content Identity |
| Rebase | Сукупні зміни перенесених commits відповідають погодженій Changeset Content Identity |

Конкретний спосіб порівняння визначають Stage Skills та Artifact Contracts.

**Integration Mismatch і Post-Merge Validation Failed.** В обох випадках INTEGRATION_CONFIRMATION повертає `BLOCKED`, і Story не стає DONE. Агент не виправляє вже виконаний merge автоматично. Рішення ухвалює людина через Recovery Protocol ([workflow-state-model.md](workflow-state-model.md#9-blocked-і-recovery-protocol)).

**Завершення після зовнішнього виправлення (OD-17).** Зовнішнє виправлення саме по собі Story не завершує.

- **Post-Merge Validation Failed.** DONE можливий, якщо первісний Integration Match = `PASS`, причину виправила інша Story або людина, людина дозволила `RESUME`, повторна Post-Merge Validation = `PASS` і виконано всі інші умови DoD.
- **Integration Mismatch.** Успішна інша Story не виправдовує первісної інтеграції. DONE можливий лише за доказу, що погоджена Changeset Content Identity цієї Story інтегрована належно, і за виконання всіх умов DoD. Інакше Story лишається `BLOCKED` або завершується `CANCELLED` за рішенням людини. Доказ — новий merge погоджених змін після корекції: `ROUTE_BACK` → HUMAN_MERGE → новий CONFIRM_MERGED. Integration Match завжди оцінює merge, зафіксований останнім CONFIRM_MERGED.

Іменування гілок і формат commits та PR визначають Stage Skills і Artifact Contracts.

## 10. Multiple Story Isolation

Правила (Q12):

- Кожна Story має незалежний Workflow State, History, артефакти та feature branch.
- Один запуск Orchestrator працює лише з однією Story.
- Перемикання між Stories дозволене. Стан Story, з якої перемкнулися, зберігається, і її можна відновити пізніше ([workflow-state-model.md](workflow-state-model.md#12-session-recovery)).
- Стадія однієї Story не змінює стан, артефакти чи гілку іншої Story. Читати інші Stories можна (наприклад, для IMPACT_ANALYSIS).
- Єдиний виняток — створення `backlog-entry` у спільному Backlog проєкту для погодженого FOLLOW_UP (розділ 7). Наявні записи інших Stories при цьому не змінюються.
- Спільні для всіх Stories джерела — Foundation і Workflow Definition. Їхня зміна запускає перевірку Staleness в інших нетермінальних Stories під час їхнього наступного запуску.
- Автоматична паралельна оркестрація кількох Stories не входить до поточної версії.

## 11. Step Mode і Auto Mode

| Режим | Поведінка |
|---|---|
| Step Mode | Виконується одна стадія. Після переходу запуск зупиняється |
| Auto Mode | Стадії виконуються послідовно до обов'язкової зупинки |

**Обов'язкові зупинки Auto Mode** (Q25):

- Human Gate, зокрема повторне проходження, на якому можливий RECONFIRM;
- human action;
- Loopback, що перетинає Human Gate;
- очікування відповідей людини;
- `BLOCKED`, зокрема вичерпання Retry Budget, ескалація, Foundation Change Request, Foundation Rule Violation, порушення цілісності, Review Binding Guard, Merge Protection Unavailable чи Disabled, Integration Mismatch, Post-Merge Validation Failed;
- термінальний стан.

Auto Mode не обходить Human Gates і не ухвалює рішень людини: ні APPROVE, ні RECONFIRM, ні рішень Recovery. Loopback без перетину gate в межах Retry Budget Auto Mode виконує без зупинки.

## 12. Context Management

- **Progressive Context Loading (Q17).** Кожна стадія має мінімальний набір джерел ([stages.md](stages.md#2-джерела-контексту)). Додаткові документи завантажуються лише за потреби.
- **Persistent Artifacts (Q18).** Наступні стадії отримують контекст із канонічних артефактів попередніх стадій. Історія чату не є джерелом стану, вимог чи погоджених рішень.

## 13. Відповідність Workflow Invariants

| # | Інваріант | Де забезпечено |
|---|---|---|
| 1 | Немає переходу без дозволеного маршруту | `stage-map.yaml` RR-01; state model §6 |
| 2 | Human Gate не можна обійти автоматично | §3, §11; `stage-map.yaml` RR-07 (зокрема RECONFIRM) |
| 3 | Reviewer PASS ≠ Human Approval | §3; state model §4 |
| 4 | Approval прив'язаний до актуальних артефактів | §3; state model §5 (Approved Content Identity), §7 |
| 5 | Рішення людини — у канонічному джерелі | §2; state model §2 |
| 6 | Stale artifacts не використовуються без Revalidation | state model §5, §8, §14 |
| 7 | Недозволений Loopback не виконується | `stage-map.yaml` RR-03; state model §6 |
| 8 | BLOCKED має Recovery Protocol | state model §9 |
| 9 | Retry Budget скінченний | state model §10 |
| 10 | NOT_APPLICABLE потребує обґрунтування | §5; `stage-map.yaml` RR-02 |
| 11 | Stage Skills не змінюють Workflow State | state model §2 |
| 12 | DONE лише після Git Integration | §8, §9; `stage-map.yaml` RR-09, RR-10, RR-13, `post_merge_validation` |
| 13 | Chat History не є джерелом істини | §12; state model §2 |
| 14 | Workflow Core не залежить від Go, Gin, PostgreSQL | §1; [stages.md](stages.md#2-джерела-контексту) (прив'язка до проєкту винесена окремо) |
| 15 | Технологічні правила — у Foundation і Stage Skills | §1 |

Нормативне визначення інваріантів не є їхньою програмною перевіркою. Перевірку забезпечать Orchestrator і Harness Validation.

## 14. Рішення OD

### Погоджені рішення OD-01…OD-17

Усі сімнадцять рішень погоджено людиною: OD-01…OD-15 — після Review Draft v1, OD-16 — після Review Draft v2, OD-17 — після фінального Review Draft v3. Статус кожного — `APPROVED_DECISION`. Ідентифікатори OD-* лишаються стабільними і використовуються в усіх документах Workflow Definition для traceability.

| ID | Статус | Рішення | Нормативні правила | Де застосовано |
|---|---|---|---|---|
| OD-01 | `APPROVED_DECISION` | Обов'язкова універсальна стадія SOLUTION_DESIGN: загальна технічна структура рішення — Domain, Application, Integration та Infrastructure Design, якщо це потрібно Story | Стадія не повертає `NOT_APPLICABLE`. DESIGN_REVIEW завжди застосовна | `stage-map.yaml` (SOLUTION_DESIGN, DESIGN_REVIEW); stages.md §8 |
| OD-02 | `APPROVED_DECISION` | Окрема фаза DELIVERY відокремлює технічну готовність Story від інтеграції змін | Фаза 7 у `phases` | `stage-map.yaml` (`phases`); stages.md §4, §12 |
| OD-03 | `APPROVED_DECISION` | Окремої Clarification Review Stage немає. SPEC_REVIEW перевіряє `clarification-record` і `story-spec` | `reviewed_by: SPEC_REVIEW` для CLARIFICATION | `stage-map.yaml`; stages.md §6, §7 |
| OD-04 | `APPROVED_DECISION` | Окрема стадія PLAN_REVIEW для незалежної перевірки Implementation Plan | PLAN_REVIEW обов'язкова | `stage-map.yaml`; stages.md §9 |
| OD-05 | `APPROVED_DECISION` | Дворівнева security-перевірка: DESIGN_REVIEW — архітектура; SECURITY_REVIEW — поглиблена перевірка реалізації після CODE_REVIEW, якщо застосовна | Окремої Security Design Review немає | §4; `stage-map.yaml` (DESIGN_REVIEW, SECURITY_REVIEW); stages.md §8, §11 |
| OD-06 | `APPROVED_DECISION` | Gates — обов'язкові зупинки. Незмінний вміст після Revalidation → скорочене RECONFIRM. Змінений вміст або `INVALIDATED` → повне APPROVE. Auto Mode не виконує Reconfirmation | `gate_decisions.RECONFIRM`, RR-07 | §3, §11; state model §7; `stage-map.yaml` |
| OD-07 | `APPROVED_DECISION` | Явні нормативні умови застосовності Optional Stages. Неоднозначність → застосовна. `NOT_APPLICABLE` з обґрунтуванням і перевіркою. Без Risk Scoring Engine | `applicability_conditions`, AP-01…AP-07 | §5; `stage-map.yaml` |
| OD-08 | `APPROVED_DECISION` | Конфігурована Retry Policy з погодженими початковими значеннями 3 / 3 / 3 (Technical Attempts, Quality Loops, Repeated Failure). Правила обліку й скидання. Розширення — лише людиною через Recovery | `retry_policy` | state model §10; `stage-map.yaml` |
| OD-09 | `APPROVED_DECISION` | Risk-Based Dependency Approval. Прості допоміжні залежності — у межах Controlled Autonomy. Суворіші правила Foundation мають пріоритет | Критерії суттєвості | §2 |
| OD-10 | `APPROVED_DECISION` | Ізоляція контексту обов'язкова для DESIGN_REVIEW, CODE_REVIEW, SECURITY_REVIEW. Для SPEC_REVIEW і PLAN_REVIEW — за risk triggers | `review_context_isolation`, ISO-01…ISO-06 | §4; `stage-map.yaml` |
| OD-11 | `APPROVED_DECISION` | Погоджені FOLLOW_UP створюють `backlog-entry` у канонічному Backlog з посиланням на Story і Finding | Умова DoD 12 | §7, §8, §10, §15; stages.md §12 |
| OD-12 | `APPROVED_DECISION` | Merge Validation Protocol: прив'язка до feature і target branch, перевірка перед merge, перевірка інтеграційного результату, повторні review за зміни коду, повне погодження застарілого approval, merge лише людиною, підтвердження після merge | RR-10, RR-12, `route_conditions`, MERGE_READINESS_CHECK | §9; state model §5, §9; `stage-map.yaml`; stages.md §12 |
| OD-13 | `APPROVED_DECISION` | Cancellation Protocol. CANCELLED — термінальний стан, Story не відкривається повторно. Без явного дозволу нічого не видаляється | RR-08 | §17; `stage-map.yaml` |
| OD-14 | `APPROVED_DECISION` | Один Workflow для типів FEATURE, BUG, TECHNICAL, INFRASTRUCTURE, SECURITY. Clarification і Specification адаптуються до типу. Без окремих Workflow Profiles | `story_types` | §16; stages.md §6, §7; `stage-map.yaml` |
| OD-15 | `APPROVED_DECISION` | Перевірки визначають Testing Strategy проєкту, Story Risk Assessment і Testing Approach плану. Технологічні правила тестування не входять у Workflow Core. Відсутність Testing Strategy — Foundation Gap | FG-01 | §8, §15, §18; stages.md §9, §11 |

| OD-16 | `APPROVED_DECISION` | Option C: Merge Protection (Prevention) і Post-Merge Validation (Recovery). Post-Merge Validation — частина INTEGRATION_CONFIRMATION, без нової стадії. Агент не змінює target branch і налаштування захисту | RR-13, `post_merge_validation`, DoD 11–12 | §9; state model §7, §9; `stage-map.yaml` (INTEGRATION_CONFIRMATION); stages.md §12 |
| OD-17 | `APPROVED_DECISION` | Завершення після зовнішнього виправлення: Post-Merge Validation Failed → DONE лише за первісного Integration Match PASS, RESUME людини, повторної Post-Merge Validation PASS і виконання DoD. Integration Mismatch → DONE лише з доказом належної інтеграції погодженої Changeset Content Identity, інакше BLOCKED або CANCELLED | `post_merge_validation.resume_rules` | §9; state model §9; stages.md §12 |

### Погоджені Review Findings

| ID | Статус | Суть | Де застосовано |
|---|---|---|---|
| WF-01 | `RESOLVED` | Незмінність версій, Currency, `revalidation-record` | state model §5, §7, §8 |
| WF-02 | `RESOLVED` | Review Binding Guard | `stage-map.yaml` RR-10, RR-12 |
| WF-03 | `RESOLVED` | Approved Content Identity для Approval Scope | state model §5; `stage-map.yaml` |
| WF-04 | `RESOLVED` | Єдиний Rework Protocol | state model §14; RR-11 |
| WF-05 | `RESOLVED` | Розподіл RECONCILIATION і VERIFICATION | stages.md §11 |
| WF-V2-01 | `APPROVED_DECISION` | Risk Acceptance не усуває порушень обов'язкових правил Foundation | §2, §3, §7, §8; RR-14; stages.md §8, §11 |
| WF-V2-02 | `APPROVED_DECISION` | Applicability Decision прив'язана до всіх релевантних фактів (Applicability Basis) | §5; AP-04, AP-08…AP-10, `basis_facts`, RR-10; state model §6, §14 |

### Рішення фінального Review

| ID | Статус | Рішення | Де застосовано |
|---|---|---|---|
| FR-01 | `APPROVED_DECISION` | Погоджено колишні `DERIVED` елементи: CODE_REVIEW, PR_PREPARATION, MERGE_READINESS_CHECK і конкретні Applicability Triggers | `stage-map.yaml` (`definition_status`, `applicability_conditions`); stages.md §4 |
| FR-02 | `APPROVED_DECISION` | Merge Protection розрізняє `NOT_SUPPORTED` (ручна процедура), `UNAVAILABLE` (людина погоджує порядок інтеграції) і `DISABLED` за обов'язкової політики (merge блокується). Ручна процедура не обходить обов'язкових правил | RR-13, `merge_protection_statuses`; §9; state model §9; stages.md §12 |
| FR-03 | `APPROVED_DECISION` | SEC-08 стосується лише змін, що безпосередньо впливають на security controls або поведінку, регульовану спеціалізованими Security Rules. Базові security-перевірки лишаються в DESIGN_REVIEW і VERIFICATION | `stage-map.yaml` (SEC-08) |

### Відкриті питання

Відкритих і блокувальних питань немає. Нові питання реєструються тут з наступним ідентифікатором OD-18 і проходять окреме погодження.

## 15. Prerequisites і Foundation Gaps

Передумови, без яких Workflow не можна застосувати до реальної Story. Вони не змінюють погодженої Foundation і не створюються в межах Workflow Definition.

| ID | Передумова | Тип | Коли потрібна | Наслідок, якщо немає |
|---|---|---|---|---|
| FG-01 | Testing Strategy проєкту. Architecture Foundation Customer Portal посилається на неї, але документа ще немає | Foundation Gap | До першої реальної Story | Перша реальна Story не реєструється у Workflow, доки Testing Strategy не створено й не погоджено окремою роботою поза Story Workflow |
| PRQ-01 | Канонічний Backlog проєкту | Prerequisite | До першої Story, джерелом якої не є Product Foundation, і до першого погодженого FOLLOW_UP | Story такого типу не реєструється. INTEGRATION_CONFIRMATION не може створити `backlog-entry`, тому Story не стає DONE |
| PRQ-02 | Merge Protection Configuration: Merge Protection Policy проєкту (які механізми обов'язкові) і фактичний стан механізмів на платформі (OD-16, FR-02) | Prerequisite | До першої HUMAN_MERGE | Без визначеної політики жоден механізм не вважається обов'язковим, і діють випадки `NOT_SUPPORTED` / `UNAVAILABLE` (розділ 9). Обмеження фіксується в кожній Story. Налаштування змінює лише людина |

## 16. Story Types (OD-14)

Один канонічний Workflow підтримує всі типи зі `story_types` у `stage-map.yaml`. Окремих Workflow Profiles немає. Тип впливає на зміст CLARIFICATION і SPECIFICATION та на тригери Optional Stages і ізоляції review.

| Story Type | Джерело Story | Обов'язковий зміст специфікації |
|---|---|---|
| FEATURE | Product Foundation (для Customer Portal — Epic Map) або Backlog | Користувач чи роль, потреба, поведінка, scope |
| BUG | Backlog | Спостережувана і очікувана поведінка, умови відтворення, зачеплений scope, регресійна перевірка |
| TECHNICAL | Backlog, зокрема відкладені рішення Foundation | Technical Goal, Scope, Constraints |
| INFRASTRUCTURE | Backlog, зокрема відкладені рішення Foundation | Technical Goal, Scope, Constraints, середовища, вплив на запуск і відкат |
| SECURITY | Backlog або findings безпеки | Ризик чи загроза, security goal, Scope, Constraints |

Правила:

- Кожна Story, незалежно від типу, має перевірювані Acceptance Criteria і для кожного з них — Verification Method.
- Для TECHNICAL, INFRASTRUCTURE і SECURITY не потрібно штучно створювати User Persona.
- Тип пропонує CLARIFICATION. Тип фіксується в `story-spec`, а людина погоджує його на HUMAN_SPEC_APPROVAL.
- Зміна типу після погодження — це зміна `story-spec`, тобто повернення через Rework Protocol.

## 17. Cancellation Protocol (OD-13)

`CANCELLED` — термінальний стан. Скасована Story не відкривається повторно. Якщо роботу треба продовжити, створюється нова Story з посиланням на скасовану.

Процедура:

1. **Рішення людини.** CANCEL ухвалює лише людина: на gate, на human action, у Recovery чи в рішенні щодо Loopback через gate.
2. **Cancellation Record.** Orchestrator створює `cancellation-record`, який фіксує:
   - причину скасування;
   - останній стан Story: стадію, Story Status, Execution State;
   - створені артефакти, їхні версії і статуси;
   - стан feature branch: чи існує, чи є commits поза target branch, чи є незакомічені зміни;
   - наявність відкритого PR;
   - рішення людини щодо робочих ресурсів: що зберегти, що закрити чи видалити;
   - рішення щодо погоджених FOLLOW_UP: які стають `backlog-entry`.
3. **Робочі ресурси.** Агент не видаляє гілки, не закриває PR і не скасовує незакомічені зміни без явного дозволу людини на кожну дію. Дозвіл фіксується в `cancellation-record`.
4. **Перехід** у `CANCELLED` і подія в History.

Якщо зміни Story вже інтегровано в target branch (CANCEL після HUMAN_MERGE), `cancellation-record` явно це фіксує. Скасування не відкочує інтегрованих змін.

## 18. Testing Requirements (OD-15)

Обов'язкові перевірки Story визначають три джерела:

1. **Testing Strategy проєкту** — частина Project Foundation (FG-01).
2. **Story Risk Assessment** — розділ `implementation-plan`. Він визначає категорії ризику, яких торкається Story, на основі `solution-design`, `impact-analysis` і Story Type. Його перевіряє PLAN_REVIEW, а погоджує людина на HUMAN_TECHNICAL_APPROVAL.
3. **Testing Approach** — розділ `implementation-plan`, що визначає перевірки для кожного Acceptance Criterion і кожної категорії ризику.

Ризикові зміни отримують додаткові перевірки. Workflow Core визначає категорії ризику, а не технології тестування:

| Категорія ризику | Приклад додаткової перевірки |
|---|---|
| Authentication, authorization | Security tests |
| Persistence | Integration tests зі сховищем |
| Concurrency | Concurrent scenario / race tests |
| Infrastructure | Configuration / deployment validation |

Конкретні види, інструменти й обсяги тестів визначає Testing Strategy проєкту. VERIFICATION перевіряє, що всі перевірки з трьох джерел виконано і вони пройшли.
