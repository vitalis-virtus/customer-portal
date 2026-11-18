---
name: specification
description: Stage Skill стадії SPECIFICATION. Перетворює clarification-record на перевірювану story-spec (Story Type, вимоги, Acceptance Criteria з Verification Method, scope) і повертає result envelope. Інтерв'ю не веде. Запускається лише вручну з готовими шляхами входу й виходу, версією і номером спроби.
argument-hint: "story=<ID> clarification=<шлях> output=<шлях до v<N>.md> version=<N> attempt=<N> [source=<шлях>] [mode=pilot] [previous=<шлях>] [reason=<причина повернення>]"
disable-model-invocation: true
user-invocable: true
---

# SPECIFICATION — Stage Skill

## 1. Purpose

Сформувати перевірювану специфікацію Story з уточнених вимог. Skill виконує стадію SPECIFICATION за `docs/workflow/stages.md` (розділ 7) і `docs/workflow/stage-map.yaml`. Ці файли й `docs/contracts/` — авторитетні; у разі розбіжності діють вони, а не цей Skill.

Інтерв'ю немає. Продуктові рішення живуть лише в `clarification-record`. Якщо рішення бракує — це повернення на CLARIFICATION (розділ 6), а не питання до людини.

## 2. Inputs

Аргументи (`$ARGUMENTS`):

| Аргумент | Обов'язковий | Значення |
|---|---|---|
| `story` | так | ID Story, наприклад `US-001` |
| `clarification` | так | Готовий шлях поточної версії `clarification-record`. Її номер версії бери з front matter цього файлу |
| `output` | так | Готовий шлях файлу `story-spec`. Skill не обчислює шлях і версію сам |
| `version`, `attempt` | так | Версія `story-spec` (збігається з `v<N>` в `output`, не залежить від версії `clarification-record`) і номер спроби стадії |
| `source` | ні | Готовий шлях джерела Story. Без нього — `story-source` з реєстру |
| `mode=pilot` | ні | Пілот поза зареєстрованим Workflow |
| `previous`, `reason` | для `version > 1` | Попередня версія `story-spec` і причина повернення |

Людину питай лише про відсутній обов'язковий аргумент. Не вгадуй.

Логічні входи — `docs/contracts/stage-io-map.yaml` → `SPECIFICATION`, шляхи — `docs/contracts/artifact-registry.yaml`:
- `clarification-record` — рішення людини (`CQ`), встановлені факти, ризики, Handoff;
- `story-source` — опис Story;
- `product-foundation` — мінімум: Business Glossary, Business Rules і NFR, пов'язані зі Story; за потреби — інші документи Product Foundation.

`architecture-foundation` не є входом цієї стадії. Технічні факти Foundation потрапляють у специфікацію лише через `clarification-record`, з посиланням на нього. Історія чату не є входом. Читай лише потрібні документи (Progressive Context Loading).

## 3. Procedure

1. **Перевір аргументи.** У `mode=pilot` `output` не може лежати в `docs/stories/`. Якщо файл `output` уже існує — не перезаписуй без дозволу людини.
2. **Прочитай `clarification-record`, Story і Foundation.** Виділи: рішення `CQ` з `decided_by`; відкладені питання з їхнім впливом; встановлені факти з ID (BR-, NFR-); ризики; Handoff. Рекомендація агента без рішення людини — не вимога.
3. **Зафіксуй Story Type** зі `story_types` у `stage-map.yaml` за таблицею `docs/workflow/workflow-policies.md` §16. Стартова точка — пропозиція CLARIFICATION. Розбіжність із нею обґрунтуй. Для технічних типів не вигадуй User Persona.
4. **Сформулюй вимоги** `REQ-NN`: спостережувана поведінка або технічний результат, кожна з джерелом (`CQ-NN` або ID правила Foundation). Посилайся на правила Foundation, а не переписуй їх. Покрий обов'язковий зміст типу (§16).
5. **Визнач scope і поза scope.** Відкладені `CQ` і явно виключене в `clarification-record` — поза scope, з успадкованим впливом.
6. **Сформулюй Acceptance Criteria** `AC-NN` (розділ 3.1).
7. **Перевір межу Clarification.** Якщо вимогу неможливо сформулювати без нового продуктового рішення, або рішення суперечать одне одному чи Foundation — зупинись і поверни verdict за розділом 6. Не обирай варіант за людину і не записуй «припущень».
8. **Перевір межу Technical Design.** Технічне, що вже встановлене Foundation і записане в `clarification-record`, — лише посилання. Нового (маршрути, коди помилок поза Foundation, схеми, алгоритми, структура коду, бібліотеки) не вирішуй — винеси в розділ «Відкрито для Technical Design».
9. **Перевір якість** (розділ 4). Невідповідність виправ; якщо вона походить з `clarification-record` — крок 7.
10. **Запиши `story-spec`** в `output` (розділ 5) і поверни envelope.

### 3.1. Acceptance Criteria

- Кожен AC: ID `AC-NN`, посилання на `REQ` і джерело, умова → дія → очікуваний спостережуваний результат, вид (позитивний / негативний / граничний), Verification Method.
- Покривай позитивні сценарії, відмови (невалідний ввід, конфлікт, заборонена дія) і межі (значення на межі та за нею). Перелік тем для AC у Handoff `clarification-record` — мінімум, не стеля.
- Один AC перевіряє одну поведінку. Без слів «коректно», «швидко», «зручно» без вимірюваного значення.
- Verification Method описує, як спостерігати результат: через який інтерфейс чи дію і що саме перевіряється (відповідь, збережений стан, відсутність даних у відповідях і логах). Інструменти й рівні тестів не обирай — їх визначають Testing Strategy (FG-01) і Testing Approach `implementation-plan`.
- ID `AC-NN` і `REQ-NN` стабільні між версіями. Вилучений ID не перевикористовуй.

## 4. Quality Criteria

- Кожна вимога й AC має джерело: Foundation або `CQ` з рішенням людини. Нічого вигаданого.
- Кожне рішення `CQ` відображене в `REQ`/`AC` або явно позначене як таке, що не впливає на специфікацію, з причиною.
- Кожен AC конкретний, перевірюваний і має Verification Method; важливі негативні й граничні сценарії покрито.
- Вимоги не суперечать одна одній, Foundation і `clarification-record`.
- Межі scope однозначні; відкладене не потрапило в scope.
- Жодне відкрите продуктове питання не інтерпретовано мовчки; рекомендацію не подано як рішення.
- Немає технічних рішень, що належать Technical Design.

## 5. Output

Один файл `story-spec` за шляхом `output`, мова — українська.

Front matter — поля `docs/contracts/artifact-rules.md` §1 і лише вони: `artifact_type: story-spec`, `story_id`, `version`, `produced_by: { stage: SPECIFICATION, attempt }`, `created_at` (фактичний UTC з `date -u +%Y-%m-%dT%H:%M:%SZ`), `inputs`, `story_type`, `acceptance_criteria: [AC-01, …]`. В `inputs` — `{ type: clarification-record, version }` і кожне прочитане джерело як `{ source, path, commit }`, commit — `git log -1 --format=%h -- <path>`. Для `version > 1` — `revision_reason` і попередня версія `story-spec` в `inputs`. Статусів, approval і позначки пілоту у front matter немає.

Розділи тіла:

1. **Story** — ID, джерело, Story Type з обґрунтуванням; обов'язковий зміст типу (для FEATURE — роль і потреба).
2. **Scope** — у scope / поза scope, з джерелами.
3. **Вимоги** — `REQ-NN` з джерелами.
4. **Acceptance Criteria** — `AC-NN` за розділом 3.1.
5. **Трасування рішень** — таблиця `CQ` → `REQ`/`AC` або «не впливає: причина».
6. **Ризики й обмеження** — посилання на ризики `clarification-record` і прогалини Foundation, що діють на Story.
7. **Відкрито для Technical Design** — технічні питання, які специфікація свідомо не вирішує.

У `mode=pilot` перший рядок тіла: «Пілотний артефакт поза зареєстрованим Workflow. Не зареєстрований, не погоджений, не є входом жодної стадії». Пілотний `clarification-record` теж не зареєстрований; не подавай жоден із них як погоджений.

## 6. Verdict and Return

SPECIFICATION має `optional: false` і єдину ціль loopback — `CLARIFICATION` (без `crosses_gates`), тому допустимі **`PASS`**, **`CHANGES_REQUIRED`** і **`BLOCKED`**. `NOT_APPLICABLE` недопустимий.

| Ситуація | Verdict |
|---|---|
| Специфікацію записано, критерії розділу 4 виконано | `PASS`, `outputs: [{ type: story-spec, version }]` |
| Бракує продуктового рішення, вимоги неповні або суперечать одна одній | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION`, `outputs: []` |
| Суперечність із Foundation | `BLOCKED` (Foundation Change Required) |
| Потрібне рішення з переліку ескалацій (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) |

Envelope — формат `docs/contracts/workflow-state.md` §6, без змін і без додаткових полів:

```yaml
result:
  story_id: US-001
  stage: SPECIFICATION
  attempt: 1
  verdict: PASS                      # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { type: story-spec, version: 1 }
  loopback_target: null              # CLARIFICATION лише для CHANGES_REQUIRED
  blocker: null                      # для BLOCKED: { reason, details, needed }
  summary: "Type FEATURE, 9 REQ, 18 AC; усі CQ відображено"
```

Для `CHANGES_REQUIRED` файл не записуй. У `summary` коротко перелічи прогалини з посиланнями на `CQ` чи правила, деталі покажи людині в чаті. ID `F-NNN` не присвоюй — це робить review. Для `BLOCKED`: `outputs: []`; `foundation-change-request` Skill не створює — потрібну зміну опиши в `blocker.needed`. У `mode=pilot` додай у `summary` позначку «pilot». На повторному вході Skill завжди створює нову версію з `revision_reason`; `revalidation-record` цей Draft не створює.

## 7. Constraints

- Не змінюй `state.yaml`, `history.jsonl`, статуси, Currency чи approvals.
- Пиши лише файл `output`. Не змінюй `clarification-record`, Foundation, Workflow Definition, `docs/contracts/` та інші артефакти.
- Не обчислюй номер версії чи шлях — бери їх з аргументів.
- Не веди інтерв'ю і не ухвалюй продуктових рішень замість людини.
- Не проходь Human Gate, не імітуй SPEC_REVIEW, Orchestrator чи рішень людини; не запускай інші стадії та Skills.
- Не роби `git commit`, `git push` чи деструктивних операцій.
