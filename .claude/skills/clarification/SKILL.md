---
name: clarification
description: Stage Skill стадії CLARIFICATION. Інтерактивно уточнює Candidate Story з людиною, фіксує її рішення в clarification-record і повертає result envelope. Запускається лише вручну з готовим шляхом виходу, версією і номером спроби.
argument-hint: "story=<ID> output=<шлях до v<N>.md> version=<N> attempt=<N> [source=<шлях>] [mode=pilot] [previous=<шлях>] [reason=<причина повернення>]"
disable-model-invocation: true
user-invocable: true
---

# CLARIFICATION — Stage Skill

## 1. Purpose

Усунути неоднозначності Story до SPECIFICATION: намір, межі, правила й обмеження. Skill виконує стадію CLARIFICATION за `docs/workflow/stages.md` (розділ 6) і `docs/workflow/stage-map.yaml`. Ці файли й `docs/contracts/` — авторитетні; у разі розбіжності діють вони, а не цей Skill.

Skill працює в основній сесії, бо веде інтерв'ю з людиною.

## 2. Inputs

Аргументи (`$ARGUMENTS`):

| Аргумент | Обов'язковий | Значення |
|---|---|---|
| `story` | так | ID Story, наприклад `US-001` |
| `output` | так | Готовий шлях файлу `clarification-record`. Skill не обчислює шлях і версію сам |
| `version`, `attempt` | так | Номер версії (збігається з `v<N>` у `output`) і номер спроби стадії |
| `source` | ні | Готовий шлях джерела Story. Без нього — `story-source` з реєстру |
| `mode=pilot` | ні | Пілот поза зареєстрованим Workflow |
| `previous`, `reason` | для `version > 1` | Попередня версія `clarification-record` і причина повернення |

Якщо обов'язкового аргументу немає — запитай людину. Не вгадуй.

Логічні входи — `docs/contracts/stage-io-map.yaml` → `CLARIFICATION`, шляхи — `docs/contracts/artifact-registry.yaml`:
- `story-source` — опис Story з її джерела (для FEATURE — `docs/product/epic-map.md`);
- `product-foundation` — мінімум: Business Glossary, Business Rules і NFR, пов'язані зі Story; за потреби — Product Vision, Personas;
- `architecture-foundation` — обов'язково для TECHNICAL та INFRASTRUCTURE; для інших типів — лише щоб виявити технічні обмеження, які впливають на вимоги.

Історія чату не є входом. Читай лише потрібні документи (Progressive Context Loading).

## 3. Procedure

1. **Перевір аргументи.** У `mode=pilot` `output` не може лежати в `docs/stories/`. Якщо файл `output` уже існує — не перезаписуй без дозволу людини.
2. **Прочитай Story** з джерела. Перевір передумови `stages.md` §6: якщо Product Foundation позначає залежність, яку треба вирішити до Clarification цієї Story (наприклад, Open Product Dependencies у Epic Map), і вона не вирішена — `BLOCKED` без інтерв'ю.
3. **Запропонуй Story Type** зі `story_types` у `stage-map.yaml` за таблицею `docs/workflow/workflow-policies.md` §16 (джерело Story і зміст). Тип лише пропонується: фіксує його SPECIFICATION, погоджує людина на HUMAN_SPEC_APPROVAL. Для технічних типів не вигадуй User Persona.
4. **Прочитай Foundation** і розклади знайдене на:
   - **встановлені факти** — погоджені правила й рішення з ID (BR-, NFR-, тощо);
   - **нерозв'язані питання** — відкриті залежності, «рекомендації», невизначені пункти. Це не факти.
5. **Знайди прогалини**: неоднозначності, суперечності, відсутні вимоги, потрібні для обов'язкового змісту специфікації цього типу (`workflow-policies.md` §16).
6. **Відсортуй питання.** До інтерв'ю — лише ті, без яких неможлива специфікація. Некритичне — у розділ ризиків або handoff. Не питай про те, що вже вирішено у Foundation.
   **Межа Clarification:** питання, що впливає на спостережувану поведінку продукту чи Acceptance Criteria, вирішується з людиною до `PASS`. Далі передаються лише суто технічні деталі реалізації.
7. **Проведи інтерв'ю** (розділ 3.1). Повторюй раунди, доки кожне критичне питання не матиме відповіді або явного рішення людини відкласти його.
8. **Суперечність із Foundation** → `BLOCKED` (розділ 6). Не виправляй Foundation і не обходь його.
9. **Фінальна перевірка узгодженості** перед `PASS`:
   - рішення людини й похідні вимоги не суперечать одне одному;
   - питання, позначене вирішеним, не значиться водночас відкритим чи прогалиною;
   - Handoff не вирішує мовчки відкрите рішення людини;
   - рекомендації не подано як рішення людини.
   Знайдена невідповідність → повернись до інтерв'ю.
10. **Запиши `clarification-record`** у `output` (розділ 5) і поверни envelope.

### 3.1. Інтерв'ю

- Раунд — 1–4 пов'язані питання. Для кожного:
  - ID `CQ-NN`, стабільний між раундами й версіями;
  - контекст: що неоднозначно і звідки це видно (посилання на документ чи правило);
  - варіанти, якщо вони є;
  - рекомендація з обґрунтуванням, окремо позначена як рекомендація.
- У кожному раунді показуй два блоки: «Вирішено людиною» і «Відкрито».
- Рішення — лише явна відповідь людини щодо конкретного питання. Мовчання чи загальне «ок» не погоджує рекомендацію; уточни, до яких `CQ` воно стосується.
- Не обирай рекомендований варіант за людину.
- Людина може явно відкласти питання. Тоді запитай або запропонуй на підтвердження вплив: що випадає зі scope, що успадковує SPECIFICATION і подальші стадії.
- Ідентифікатор того, хто ухвалює рішення (`decided_by`), запитай один раз, якщо його не надано.
- Час рішення бери з `date -u +%Y-%m-%dT%H:%M:%SZ`, а не з прикладів.

## 4. Quality Criteria

- Кожне твердження про вимогу має джерело: документ Foundation або `CQ` з рішенням людини.
- Жодної вигаданої вимоги, бізнес-правила чи архітектурного рішення.
- Рекомендація агента ніде не подана як рішення людини.
- Кожне критичне питання має відповідь або рішення відкласти з описаним впливом.
- Обов'язковий зміст для запропонованого типу покрито або явно позначено прогалиною.

## 5. Output

Один файл `clarification-record` за шляхом `output`, мова — українська.

Front matter — **лише** поля `docs/contracts/artifact-rules.md` §1: `artifact_type: clarification-record`, `story_id`, `version`, `produced_by: { stage: CLARIFICATION, attempt }`, `created_at` (фактичний UTC), `inputs`. Для джерела — `{ source, path, commit }`, commit — `git log -1 --format=%h -- <path>`. Для `version > 1` — також `revision_reason` і попередня версія в `inputs`. Статусів, Story Type чи позначки пілоту у front matter немає.

Розділи тіла:

1. **Candidate Story** — ID, джерело, опис.
2. **Запропонований Story Type** — тип і обґрунтування.
3. **Розуміння scope** — у scope / поза scope.
4. **Встановлені факти** — з посиланнями на Foundation.
5. **Питання і рішення** — для кожного `CQ`: контекст, варіанти, рекомендація агента, рішення людини (близько до дослівного), `decided_by`, `decided_at`.
6. **Відкладені питання** — `CQ`, хто і коли відклав, вплив на scope і подальші стадії.
7. **Суперечності та ризики**.
8. **Handoff для SPECIFICATION** — вимоги й рішення з посиланнями на `CQ` чи Foundation.

У `mode=pilot` перший рядок тіла: «Пілотний артефакт поза зареєстрованим Workflow. Не зареєстрований, не погоджений, не є входом жодної стадії».

## 6. Verdict and Return

CLARIFICATION має `optional: false` і `loopbacks: []`, тому допустимі лише **`PASS`** і **`BLOCKED`**. `CHANGES_REQUIRED` і `NOT_APPLICABLE` для цієї стадії недопустимі.

| Ситуація | Дія |
|---|---|
| Чекаємо відповіді людини | Не verdict і не `BLOCKED`. Продовжуй інтерв'ю, envelope не повертай |
| Усі критичні питання вирішено або явно відкладено, файл записано | `PASS` |
| Невирішена передумова Product Foundation | `BLOCKED` |
| Суперечність із Foundation | `BLOCKED`: Foundation Change Required або уточнення Foundation людиною |
| Потрібне рішення з переліку ескалацій (`workflow-policies.md` §2) | `BLOCKED` (Escalation Required) |

Envelope — формат `docs/contracts/workflow-state.md` §6, без змін:

```yaml
result:
  story_id: US-001
  stage: CLARIFICATION
  attempt: 1
  verdict: PASS                      # PASS | BLOCKED
  outputs:
    - { type: clarification-record, version: 1 }
  loopback_target: null
  blocker: null                      # для BLOCKED: { reason, details, needed }
  summary: "4 питання вирішено, 1 відкладено (CQ-03)"
```

Для `BLOCKED`: `outputs: []`, а вже отримані рішення людини — у `blocker.details`. `foundation-change-request` Skill не створює: потрібну зміну Foundation опиши в `blocker.needed`. У `mode=pilot` додай у `summary` позначку «pilot».

## 7. Constraints

- Не змінюй `state.yaml`, `history.jsonl`, статуси, Currency чи approvals.
- Пиши лише файл `output`. Не змінюй Foundation, Workflow Definition, `docs/contracts/` та інші артефакти.
- Не обчислюй номер версії, номер запису чи шлях — бери їх з аргументів.
- Не проходь Human Gate і не імітуй рішень людини чи Orchestrator.
- Не запускай інші стадії та Skills.
- Не роби `git commit`, `git push` чи деструктивних операцій.
