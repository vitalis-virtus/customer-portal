---
name: spec-review
description: Review Skill стадії SPEC_REVIEW. Незалежно, в ізольованому контексті, перевіряє story-spec разом з clarification-record, записує review-report з findings і verdict та повертає result envelope. Інтерв'ю не веде і перевірені артефакти не змінює. Запускається лише вручну з готовими шляхами входів і виходу, record_id і номером спроби.
argument-hint: "story=<ID> spec=<шлях> clarification=<шлях> output=<шлях> record_id=review-report/<NNNN> attempt=<N> [source=<шлях>] [mode=pilot] [previous_reports=<шлях,...>] [focus=<гіпотези>]"
disable-model-invocation: true
user-invocable: true
context: fork
agent: general-purpose
background: false
allowed-tools: Read, Grep, Glob, Bash(git log:*), Bash(date:*)
disallowed-tools: Edit, NotebookEdit, AskUserQuestion, Agent, Skill
---

# SPEC_REVIEW — Review Skill

## 1. Purpose

Незалежно перевірити `clarification-record` і `story-spec` до HUMAN_SPEC_APPROVAL. Skill виконує стадію SPEC_REVIEW за `docs/workflow/stages.md` (розділ 7) і `docs/workflow/stage-map.yaml` (`reviews: [CLARIFICATION, SPECIFICATION]`). Ці файли й `docs/contracts/` — авторитетні; у разі розбіжності діють вони, а не цей Skill.

Skill виконується в окремому контексті (`context: fork`). Входи — лише файли за переданими шляхами. Робочий контекст, чат, міркування і самооцінка producer-стадії не є входом і не є доказом. `PASS` producer-стадії не означає, що review пройдено. Reviewer не виправляє артефакти: він фіксує findings і verdict (`workflow-policies.md` §4).

## 2. Inputs

Аргументи (`$ARGUMENTS`):

| Аргумент | Обов'язковий | Значення |
|---|---|---|
| `story` | так | ID Story, наприклад `US-001` |
| `spec` | так | Готовий шлях версії `story-spec`, яку перевіряють |
| `clarification` | так | Готовий шлях поточної версії `clarification-record` |
| `output` | так | Готовий шлях файлу `review-report`. Skill не обчислює шлях сам |
| `record_id` | так | `review-report/<NNNN>`. Номер не обчислюється, береться з аргументу |
| `attempt` | так | Номер спроби стадії |
| `source` | ні | Готовий шлях джерела Story. Без нього — `story-source` з реєстру |
| `mode=pilot` | ні | Пілот поза зареєстрованим Workflow |
| `previous_reports` | ні | Попередні `review-report` SPEC_REVIEW цієї Story |
| `focus` | ні | Гіпотези для перевірки. Це не готові findings |

Людини немає: Skill не питає. Якщо бракує обов'язкового аргументу або вхід не читається — `BLOCKED` (розділ 6).

Логічні входи — `docs/contracts/stage-io-map.yaml` → `SPEC_REVIEW`, шляхи — `docs/contracts/artifact-registry.yaml`:
- `story-spec` і `clarification-record` — обидва є предметом перевірки;
- `story-source` — опис Story;
- `product-foundation` — мінімум: Business Glossary, Business Rules і NFR, пов'язані зі Story; за потреби — інші документи Product Foundation.

`architecture-foundation` не є входом. Межу Technical Design оцінюй за змістом `story-spec` і `clarification-record`. Читай лише потрібні документи (Progressive Context Loading).

## 3. Procedure

1. **Перевір аргументи.** У `mode=pilot` `output` не може лежати в `docs/stories/`. Якщо файл `output` уже існує — `BLOCKED`, не перезаписуй.
2. **Прочитай артефакти напряму** з `spec` і `clarification`, потім Story і Foundation. Не покладайся на підсумки, Handoff чи таблиці трасування як на доказ: перевіряй їх за первинним текстом.
3. **Перевір попередні звіти**, якщо їх передано. Кожен `OPEN` finding отримує новий стан. Ту саму проблему позначай тим самим `F-NNN`. Нові ID продовжують найбільший наявний номер, без попередніх звітів — з `F-001`.
4. **Перевір `clarification-record`:**
   - рішення `CQ` має явну відповідь людини, `decided_by` і `decided_at`;
   - рекомендацію агента не подано як рішення;
   - відкладене питання має описаний вплив;
   - вирішене питання не значиться водночас відкритим;
   - Handoff не вирішує мовчки відкрите рішення;
   - Story Type запропоновано за `workflow-policies.md` §16.
5. **Перевір `story-spec`** за критеріями розділу 4.
6. **Перевір гіпотези `focus`.** Підтверди або відхили кожну за документами й поясни висновок у звіті. Гіпотеза без підтвердження не стає finding.
7. **Класифікуй findings** (розділ 3.1), визнач verdict і пропоновану ціль loopback (розділ 6).
8. **Запиши `review-report`** в `output` (розділ 5) і поверни envelope.

### 3.1. Findings

- Finding — лише матеріальна проблема: вона впливає на коректність, повноту, перевірюваність або трасування вимог. Стилістичних findings немає.
- `severity` — лише `BLOCKING` або `NON_BLOCKING` (`artifact-rules.md` §6).
  - `BLOCKING`: дефект вимоги, що не дозволяє погодити специфікацію. Сюди належать вигадане продуктове рішення, відкрите продуктове питання, передане в Technical Design, неперевірюваний AC або AC без Verification Method, суперечність, відсутнє обов'язкове покриття, порушення правила Foundation чи метаданих контракту.
  - `NON_BLOCKING`: необов'язкове покращення, що не порушує вимог і правил.
  - Порушення AC чи обов'язкового правила Foundation не знижуй. У сумнівному випадку, чи порушено правило Foundation, — `BLOCKING` (`workflow-policies.md` §7).
- Кожен finding містить:
  - `F-NNN`;
  - severity;
  - місце: файл, розділ, ID (`REQ`/`AC`/`CQ`);
  - пояснення;
  - доказ: цитату чи посилання на джерело;
  - конкретну рекомендацію, що змінити. Reviewer рекомендує, але сам не виправляє.
- Якщо finding потребує нового або іншого рішення людини, позначай це в рекомендації (ціль — CLARIFICATION). Варіант за людину не обирай.

## 4. Quality Criteria

Перевіряй:

1. **Повнота.** Кожне рішення `CQ` відображене в `REQ`/`AC` або позначене як таке, що не впливає, з причиною. Нічого з Handoff не загублено.
2. **Узгодженість з Foundation.** Специфікація не суперечить правилам BR-, NFR- і Glossary. Вона посилається на правила, а не переписує їх зі зміною змісту.
3. **Трасування.** Кожна `REQ` і кожен `AC` мають джерело: `CQ` з рішенням людини або ID правила Foundation. Посилання існують і означають те, що стверджується.
4. **Перевірюваність AC.** Кожен AC має умову, дію, спостережуваний результат і Verification Method. У Verification Method вказано, через що спостерігати і що перевіряти, і так перевірку справді можна виконати. Розмитих слів без вимірюваного значення немає.
5. **Негативні й граничні сценарії.** Невалідний ввід, конфлікт, заборонена дія, значення на межі та за нею — там, де їх вимагають рішення чи правила.
6. **Внутрішня узгодженість.** `REQ` і `AC` не суперечать одна одній. Немає матеріального дублювання, яке дає різні очікування для однієї поведінки.
7. **Жодних вигаданих рішень.** Немає вимог без джерела. Відкрите питання не інтерпретовано мовчки. Відкладене не потрапило в scope. Позначене «відкритим» справді не вирішене у `clarification-record` чи Foundation.
8. **Межа Technical Design.** Немає нових технічних рішень (маршрути, коди помилок поза Foundation, схеми, алгоритми, бібліотеки). Розділ «Відкрито для Technical Design» не містить продуктових рішень, від яких залежить спостережувана поведінка чи AC.
9. **Story Type.** Тип зі `story_types`, обґрунтований за `workflow-policies.md` §16. Обов'язковий зміст типу покрито (для FEATURE — роль і потреба; для технічних типів — Technical Goal, Scope, Constraints, без вигаданої Persona).
10. **Метадані й посилання** (`artifact-rules.md` §1):
    - поля front matter відповідають контракту;
    - версія `clarification-record` в `inputs` збігається з переданою;
    - `commit` кожного джерела збігається з `git log -1 --format=%h -- <path>`; інакше вхід `STALE`, і це `BLOCKING`;
    - `acceptance_criteria` збігається з ID AC у тілі;
    - `story_type` збігається з тілом;
    - `revision_reason` є для `version > 1`;
    - у `mode=pilot` є пілотний рядок.

## 5. Output

Один файл `review-report` за шляхом `output`, мова — українська.

Front matter — поля `docs/contracts/artifact-rules.md` §2 і лише вони:
- `record_type: review-report`;
- `record_id` з аргументу;
- `story_id`;
- `produced_by: { stage: SPEC_REVIEW, attempt }`;
- `created_at` — фактичний UTC з `date -u +%Y-%m-%dT%H:%M:%SZ`;
- `subjects` — `{ type: story-spec, version }` і `{ type: clarification-record, version }` з front matter перевірених файлів;
- `verdict`;
- `findings` — `{ id, severity, summary, status }` для кожного нового finding і кожного попереднього `OPEN`.

Розділи тіла:

1. **Предмет перевірки** — шляхи й версії, прочитані джерела Foundation.
2. **Підсумок** — verdict, кількість findings за severity, пропонована ціль loopback.
3. **Findings** — підрозділ на кожен `F-NNN` за розділом 3.1.
4. **Попередні findings** — новий стан кожного `OPEN` finding з попередніх звітів, якщо вони є.
5. **Гіпотези `focus`** — підтверджено чи відхилено, з посиланням на `F-NNN` або на доказ.
6. **Перевірені критерії** — коротко за пунктами 1–10 розділу 4: виконано або `F-NNN`.

У `mode=pilot` перший рядок тіла: «Пілотний артефакт поза зареєстрованим Workflow. Не зареєстрований, не погоджений, не є входом жодної стадії». Перевірені пілотні артефакти теж не погоджені; не подавай їх і review як погодження.

## 6. Verdict and Return

SPEC_REVIEW — обов'язкова review-стадія з loopbacks `SPECIFICATION` і `CLARIFICATION`, без `crosses_gates`. Тому допустимі **`PASS`**, **`CHANGES_REQUIRED`** і **`BLOCKED`**. `NOT_APPLICABLE` недопустимий (`workflow-policies.md` §4).

| Ситуація | Verdict |
|---|---|
| Немає BLOCKING findings | `PASS` (NON_BLOCKING findings допустимі) |
| Є BLOCKING finding, що потребує нового чи іншого рішення людини | `CHANGES_REQUIRED`, `loopback_target: CLARIFICATION` |
| Є BLOCKING findings, які усуваються лише в `story-spec` | `CHANGES_REQUIRED`, `loopback_target: SPECIFICATION` |
| Суперечність із Foundation, яку не усунути зміною артефактів Story | `BLOCKED` (Foundation Change Required) |
| Потрібне рішення з переліку ескалацій (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) |
| Бракує аргументу, вхід не читається або `output` уже існує | `BLOCKED` до перевірки, файл не записується |

`loopback_target` — лише пропозиція. Маршрут визначає Orchestrator зі `stage-map.yaml`. Для `PASS`, `CHANGES_REQUIRED` і `BLOCKED`, знайденого під час перевірки, звіт записується. `outputs: []` — лише коли перевірки не було.

Envelope — формат `docs/contracts/workflow-state.md` §6, без змін і без додаткових полів:

```yaml
result:
  story_id: US-001
  stage: SPEC_REVIEW
  attempt: 1
  verdict: CHANGES_REQUIRED          # PASS | CHANGES_REQUIRED | BLOCKED
  outputs:
    - { record: review-report/0001 }
  loopback_target: SPECIFICATION     # лише для CHANGES_REQUIRED: SPECIFICATION | CLARIFICATION
  blocker: null                      # для BLOCKED: { reason, details, needed }
  summary: "2 BLOCKING (F-001, F-002), 1 NON_BLOCKING"
```

Для `BLOCKED` опиши потрібне в `blocker.needed`; `foundation-change-request` Skill не створює. У `mode=pilot` додай у `summary` позначку «pilot».

## 7. Constraints

- Пиши лише файл `output`. Не змінюй `story-spec`, `clarification-record`, Foundation, Workflow Definition, `docs/contracts/` та інші артефакти.
- Не змінюй `state.yaml`, `history.jsonl`, статуси, Currency чи approvals.
- Не обчислюй шлях чи номер запису — бери їх з аргументів.
- Не веди інтерв'ю, не питай людину і не ухвалюй продуктових рішень.
- Не спирайся на робочий контекст producer-стадії чи її самооцінку.
- Не проходь Human Gate, не імітуй Orchestrator чи рішень людини; не запускай інші стадії та Skills.
- Не роби `git commit`, `git push` чи деструктивних операцій.
