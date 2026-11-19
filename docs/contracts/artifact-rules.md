# Artifact Rules

**Статус:** `APPROVED (MVP)`, Simplified Artifact Contracts MVP. Погоджено 2026-11-17 як дизайн MVP-контрактів; межі погодження — [README.md](README.md).

Документ визначає спільні метадані артефактів і правила роботи з версіями. Шляхи й власників визначає [artifact-registry.yaml](artifact-registry.yaml), а збереження статусів — [workflow-state.md](workflow-state.md).

## 1. Контрольовані документи

Кожна версія — окремий Markdown-файл `artifacts/<artifact_type>/v<N>.md` з YAML front matter.

```yaml
---
artifact_type: story-spec          # ключ з artifact-registry.yaml
story_id: US-001
version: 2                         # ціле число від 1
produced_by:
  stage: SPECIFICATION             # стадія з produced_by у реєстрі
  attempt: 1                       # номер спроби стадії
created_at: 2026-11-18T10:15:00Z   # фактичний час, а не приклад
inputs:                            # усе, на чому ґрунтується ця версія
  - { type: clarification-record, version: 1 }
  - { source: product-foundation, path: docs/product/business-rules.md, commit: c1565b1 }
revision_reason: "F-002: уточнено AC-03"   # лише для version > 1
---
```

| Поле | Правило |
|---|---|
| `artifact_type` | Ключ із `artifacts` реєстру |
| `story_id` | Збігається з `story_id` у `state.yaml` |
| `version` | `1` для першої версії, далі +1. Збігається з `v<N>` у назві файлу |
| `produced_by` | Стадія, указана власником типу в реєстрі |
| `created_at` | ISO-8601 UTC, час створення версії |
| `inputs` | Пряма залежність від **конкретної** версії артефакта (`type` + `version`), запису (`record: review-report/0002`) або джерела (`source` + `path` + `commit`) |
| `revision_reason` | Обов'язкове для `version > 1`: що змінилося і чому. Посилання на findings, якщо є |

Типоспецифічні поля додаються лише там, де від них залежить Workflow:

- `story-spec`: `story_type` (одне з `story_types` у `stage-map.yaml`) і `acceptance_criteria: [AC-01, AC-02]`;
- `implementation-changeset`: `feature_branch`, `base_commit`, `changeset_commit` і `files` (перелік змінених шляхів).

У front matter **немає** статусу, Currency, approval і висновків review. Усе це належить до `state.yaml`.

## 2. Supporting-записи

Кожен запис — окремий Markdown-файл `records/<record_type>/<NNNN>.md` зі спільним заголовком.

```yaml
---
record_type: review-report
record_id: review-report/0002      # <record_type>/<NNNN>, номер — наступний для цього типу в Story
story_id: US-001
produced_by: { stage: SPEC_REVIEW, attempt: 1 }
created_at: 2026-11-18T11:00:00Z
subjects:                          # що саме перевіряє, описує чи підтверджує запис
  - { type: story-spec, version: 2 }
  - { type: clarification-record, version: 1 }
verdict: CHANGES_REQUIRED          # для записів, що мають verdict
findings:
  - { id: F-002, severity: BLOCKING, summary: "AC-03 не має Verification Method", status: OPEN }
---
```

| Поле | Правило |
|---|---|
| `record_id` | Стабільне посилання на запис з інших файлів і `state.yaml` |
| `subjects` | Точні версії, до яких належить запис. Запис щодо однієї версії не зараховується для іншої |
| `verdict` | Для записів review і перевірок: одне зі `stage_verdicts` у `stage-map.yaml` |
| `findings` | Лише у звітах review і verification (див. розділ 6) |

Додаткові поля за типом:

| Тип | Поля |
|---|---|
| `revalidation-record` | `subjects`: версія, що лишається актуальною. `upstream`: нові версії upstream. `rationale` |
| `applicability-decision` | `target_stage`, `kind: PLANNED \| ENTRY`, `result: APPLICABLE \| NOT_APPLICABLE`, `triggers` (ID → `met`/`not_met` з посиланням на джерело), `basis` (версії й факти, на яких ґрунтується рішення) |
| `verification-report` | `verified_changeset: { version, changeset_commit }`, `verified_target_base` (commit), `dod` (стан умов DoD), `follow_up` |
| `pull-request` | `repository`, `number`, `source_branch`, `target_branch`, `changeset: { version, changeset_commit }` |
| `integration-record` | `merge_commit`, `target_before_merge`, `checks` і результати. Запис фіксує факти, але сам не доводить інтеграції (README.md, «Full Git Integration Verification») |
| `cancellation-record` | Поля зі списку Cancellation Protocol (`workflow-policies.md`, розділ 17) |
| `risk-acceptance-record`, `foundation-exception-record` | `decided_by`, `decided_at`, `finding` чи правило, межі прийняття |

Записи незмінні. Щоб виправити запис, створюють новий з `corrects: <record_id>`.

## 3. Версії і незмінність

1. **Чернетка до реєстрації.** Стадія записує файл версії, але канонічним він стає лише тоді, коли Orchestrator зареєструє його в `state.yaml`. Незареєстрований файл — Working Draft: він не є входом іншої стадії, і повторна спроба тієї самої стадії може його перезаписати.
2. **Опублікована версія незмінна.** Після реєстрації файл не редагують, зокрема й тоді, коли змінюється його статус.
3. **Виправлення — нова версія.** Будь-яка зміна вмісту дає `v<N+1>` з `revision_reason`. Попередня версія стає `SUPERSEDED`.
4. **Версія джерела** — git commit останньої зміни файлу на момент створення версії (`git log -1 --format=%h -- <path>`). Посилання на каталог (`docs/architecture/`) дозволене, але тоді будь-яка зміна в каталозі робить залежну версію `STALE`.
5. **Ідентичність погодженого вмісту** (Approved Content Identity) у MVP:
   - для контрольованого документа — його `version`;
   - для запису — `record_id`;
   - для `implementation-changeset` — `version` разом з `changeset_commit`;
   - для `pull-request` — `record_id`, тобто репозиторій, номер, гілки й версія changeset (метадані PR не входять).

## 4. Staleness і повторна перевірка

Version Status і Currency визначено у `workflow-state-model.md`, розділ 5. Їхні значення зберігаються лише в `state.yaml`.

**Правило Staleness.** Поточна версія X має Currency `STALE`, якщо хоча б для одного з її `inputs` виконується одне з двох:

- поточна версія upstream-артефакта більша за записану, а `revalidation-record`, який підтверджує X щодо цієї нової версії, немає;
- для джерела після записаного `commit` є нові коміти, що змінили `path`.

Інакше Currency — `CURRENT`.

Правила:

- `STALE` і `SUPERSEDED` версії не є чинними входами стадії. Читати їх можна лише для Revalidation.
- Коли upstream змінився, стадія-власник залежного артефакта на повторному вході робить одне з двох: створює `revalidation-record` (зміна не впливає) або нову версію (зміна впливає).
- Непрямий вплив обробляється ланцюгом: нова версія B робить C `STALE`, навіть якщо B залежала від A.
- Review-звіт, створений щодо інших версій, ніж поточні, не зараховується як `PASS`.

## 5. Review і Human Approval

- **Review PASS не є Human Approval.** Review-звіт відкриває шлях до Human Gate, але gate проходить лише рішення людини, записане в `state.yaml` → `approvals` ([workflow-state.md](workflow-state.md), розділ 4).
- Producer не перевіряє себе остаточно. Його артефакт перевіряє стадія з `reviewed_by` у `stage-map.yaml`.
- Human Approval прив'язаний до **всіх** відповідних елементів Approval Scope (`approval_scope` gate у `stage-map.yaml`): точних версій документів і `record_id` supporting-записів. Нова версія документа або заміна погодженого запису новим записом того самого призначення робить approval `INVALIDATED` ([workflow-state.md](workflow-state.md), розділ 4).
- Approval не стає чинним для зміненого вмісту без нового рішення людини.

## 6. Findings

- Finding має ID `F-NNN`, стабільний у межах Story. Ту саму проблему, знайдену повторно (зокрема іншою стадією), позначають тим самим ID. Збіг визначає reviewer.
- `severity`: `BLOCKING` або `NON_BLOCKING`.
- `status`: `OPEN`, `RESOLVED` (підтвердив незалежний review чи verification) або `DISPOSITIONED` (рішення людини чи погоджений Disposition, зокрема FOLLOW_UP).
- Наступний звіт тієї самої review-стадії щодо того самого артефакта перелічує всі попередні `OPEN` findings і їхній новий стан. Відкритий finding не зникає мовчки.
- Порушення Acceptance Criterion, обов'язкового правила Foundation чи умови DoD лишається `BLOCKING` (RR-14). Рішення на gate його не приймає.

## 7. Хто що пише

| Що | Хто пише | Чого не можна |
|---|---|---|
| Файли версій і записів свого типу | Стадія-власник із реєстру | Писати файли чужого типу, змінювати зареєстровані файли |
| `state.yaml`, `history.jsonl` | Лише Orchestrator | Stage Skill не змінює стан напряму |
| `risk-acceptance-record`, `foundation-exception-record`, `cancellation-record` | Orchestrator, лише за рішенням людини | Агент не ухвалює ці рішення сам |
| `docs/contracts/` | Людина | Агент не змінює й не послаблює контракти, щоб пройти стадію; така перешкода дає `BLOCKED` |
