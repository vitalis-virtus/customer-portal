---
record_type: review-report
record_id: review-report/0002
story_id: US-001
produced_by: { stage: SPEC_REVIEW, attempt: 2 }
created_at: 2026-11-19T08:49:15Z
subjects:
  - { type: story-spec, version: 2 }
  - { type: clarification-record, version: 4 }
verdict: PASS
findings:
  - { id: F-001, severity: BLOCKING, summary: "Обсяг trim email («пробіли на краях») не вирішено людиною; spec інтерпретує його як лише U+0020 і переносить підтвердження на gate (REQ-03, AC-18, O-01)", status: RESOLVED }
  - { id: F-002, severity: BLOCKING, summary: "Спільне визначення «Відмова валідації» не відрізняє відмову валідації від інших помилок; Verification Method AC-39 нерозв'язний, AC-32 і негативні AC валідації (AC-10–AC-18, AC-21–AC-23, AC-27–AC-30) не перевіряють причину відмови", status: RESOLVED }
  - { id: F-003, severity: NON_BLOCKING, summary: "Verification Method AC-33 «у жодному кодуванні» не має вимірюваної межі", status: RESOLVED }
  - { id: F-004, severity: NON_BLOCKING, summary: "Немає AC для запиту без поля email (AC-14 покриває лише порожнє значення й самі пробіли)", status: RESOLVED }
---

Пілотний артефакт поза зареєстрованим Workflow. Не зареєстрований, не погоджений, не є входом жодної стадії.

# Review Report — SPEC_REVIEW, US-001 Register Account (спроба 2)

Цей звіт не є погодженням. Verdict `PASS` лише відкриває шлях до HUMAN_SPEC_APPROVAL і не замінює рішення людини. Перевірені пілотні артефакти також не погоджені.

## 1. Предмет перевірки

| Артефакт | Шлях | Версія |
|---|---|---|
| `story-spec` | `docs/pilots/US-001/spec-v2.md` | 2 |
| `clarification-record` | `docs/pilots/US-001/clarification-v4.md` | 4 |

Попередній звіт: `docs/pilots/US-001/review-report-0001.md` (`review-report/0001`, щодо `story-spec` v1 і `clarification-record` v3).

Прочитані джерела:

- **Story source:** `docs/product/epic-map.md` (EPIC-001, US-001), commit `0f02d11`.
- **Product Foundation:** `business-rules.md` (`b873568`), `business-glossary.md` (`0f02d11`), `non-functional-requirements.md` (`0f02d11`). `personas.md` і `product-vision.md` перевірено лише за commit (`0f02d11`); їхній зміст для цієї спроби не потрібен.
- **Workflow Definition і контракти:** `docs/workflow/stages.md` §7, `docs/workflow/stage-map.yaml` (`stage_verdicts`, `story_types`), `docs/workflow/workflow-policies.md` (§2, §4, §7, §15, §16), `docs/contracts/artifact-rules.md`, `docs/contracts/stage-io-map.yaml` (`SPEC_REVIEW`), `docs/contracts/artifact-registry.yaml`, `docs/contracts/workflow-state.md` §6.
- **Не прочитано:** Architecture Foundation (`docs/architecture/`). Вона не є входом SPEC_REVIEW (`stage-io-map.yaml`). Технічні факти, на які спираються REQ-08, REQ-10, REQ-13, REQ-14, REQ-15, Argon2id в AC-33 і твердження, що категорії відповіді розрізняються за статусом і кодом у форматі API Conventions §8, звірено лише з CR §4 і §8, а не з первинним текстом Foundation.

## 2. Підсумок

- **Verdict:** `PASS`.
- **Findings:** нових немає. Попередні F-001, F-002 (BLOCKING), F-003, F-004 (NON_BLOCKING) — `RESOLVED` (розділ 4).
- **Пропонована ціль loopback:** немає (verdict `PASS`).

## 3. Findings

Нових findings немає.

## 4. Попередні findings

Стан визначено за первинним текстом `spec-v2.md` і `clarification-v4.md`. `revision_reason` і таблиці трасування доказом не вважалися.

### F-001 — Обсяг trim email не вирішено людиною → `RESOLVED`

- **Severity:** BLOCKING (без змін).
- **Доказ у CR v4:**
  - CQ-08 містить варіанти (A — лише U+0020; B — також TAB, CR, LF), явну відповідь людини («CQ-08 — Option A. Trim only ordinary ASCII space U+0020 from both ends of the email. Do not trim tabs, CR, LF or other whitespace characters…»), `decided_by: owner` і `decided_at`. Рекомендацію агента не подано як рішення («не надавалась»).
  - Похідне CQ-08 пояснює відхилення TAB, CR, LF (керівні символи за CQ-06) і U+00A0 (не-ASCII за CQ-06). Воно випливає з рішення людини («After trimming, the existing CQ-06 validation rules apply. Therefore, an email containing these characters must be rejected as invalid»).
  - §7 більше не стверджує хибно, що прогалин немає: «Прогалину F-001 закрито рішенням CQ-08».
  - Handoff п. 3 і теми AC відображають CQ-08.
- **Доказ у spec v2:**
  - REQ-03 посилається на CQ-08 і більше не називає поведінку «наслідком поєднання двох рішень».
  - AC-18 посилається на CQ-08, CQ-06 і покриває TAB, CR, LF і U+00A0 на краю.
  - O-01 знято (§6: «O-01 з v1 знято: обсяг trim email вирішила людина (CQ-08)»). Перенесення рішення на HUMAN_SPEC_APPROVAL у тексті немає.
  - AC-05, AC-08 і AC-20 (U+0020 на краях обрізається) узгоджені з CQ-08.
- **Примітка:** `decided_at` CQ-08 (`2026-11-19T08:31:09Z`) збігається з `created_at` CR v4. Сам артефакт прямо зазначає, що це час запису рішення, а раніший час не зафіксовано. Відповідь людини наведено дослівно і `decided_by` є. Тому це не робить рішення рекомендацією агента і на коректність вимог не впливає. Окремим finding не є.

### F-002 — «Відмова валідації» не відрізняється від інших відмов → `RESOLVED`

- **Severity:** BLOCKING (без змін).
- **Доказ у spec v2, §4:**
  - «Категорії відповіді»: успіх, відмова валідації (порушення REQ-05, REQ-07, REQ-08, зокрема відсутній чи порожній email і відсутній пароль), конфлікт (`409`), відмова CSRF (`403 csrf_rejected`), тимчасово недоступно. Категорії «взаємно розрізнювані за HTTP-статусом і кодом помилки». «Відповідь поза цими категоріями, зокрема внутрішня помилка, не задовольняє жодного AC».
  - «Правило одного порушення»: решта даних валідна, email не зареєстрований, `Origin` дозволений, ліміт не вичерпано. Тому очікувана категорія однозначно випливає з порушеного правила.
- **AC-39:** Verification Method вимагає категорію «тимчасово недоступно» — «не відмова валідації, не внутрішня помилка й не інша категорія». Нерозв'язність зникла, бо категорії взаємовиключні.
- **AC-32:** Verification Method — «категорія «відмова валідації»» замість «не успіх».
- **AC-10–AC-18, AC-21–AC-23, AC-27–AC-30:** Verification Method кожного — «категорія «відмова валідації»». `409`, `403` чи внутрішня помилка ці AC більше не задовольняють.
- **Межа Technical Design:** конкретні статус і код для відмови валідації та «тимчасово недоступно» залишено Technical Design (§4, §7) з обмеженням розрізнюваності. Нових статусів чи кодів специфікація не вигадує: `409` — з CQ-02, `403 csrf_rejected` — з Foundation через CR §4.

### F-003 — Verification Method AC-33 «у жодному кодуванні» не має межі → `RESOLVED`

- **Severity:** NON_BLOCKING (без змін).
- **Доказ:** AC-33, Verification Method: «не містять пароля `Abcdef12` у таких представленнях: байти UTF-8 як є, hex (у нижньому й верхньому регістрі), base64». Множину обмежено, перевірку можна виконати. Перевірку одностороннього перетворення збережено.

### F-004 — Немає AC для запиту без поля email → `RESOLVED`

- **Severity:** NON_BLOCKING (без змін).
- **Доказ:** AC-14: «з email, що містить лише пробіли (`␠␠␠`), з порожнім email або з валідним паролем без поля email → відмова валідації для кожного з варіантів». Спільне визначення категорії «відмова валідації» явно охоплює відсутній email.

## 5. Гіпотези `focus`

Немає: гіпотез `focus` не передано.

## 6. Перевірені критерії

1. **Повнота** — виконано.
   - CQ-01…CQ-08 відображено в REQ і AC (CQ-08 — REQ-03, AC-14, AC-18; AC-05, AC-08, AC-20 — обрізання U+0020).
   - BR-007 і NFR-001 явно визнано такими, що не впливають, з причиною.
   - Пункти Handoff 1–8 і теми AC з Handoff покрито, зокрема нові теми CQ-08 (TAB, CR, LF і U+00A0 на краю).
   - Clarification-record: кожне CQ має явну відповідь людини, `decided_by` і `decided_at`; відкладених питань немає (§6), тож опис впливу не потрібен; вирішене не значиться відкритим. Щодо `decided_at` CQ-08 — примітка в F-001.
2. **Узгодженість з Foundation** — виконано. Суперечностей з BR-003…BR-007, NFR-001, NFR-002 і Glossary не виявлено. Роль `user` і лише вона (BR-005, BR-006; AC-02, AC-40). NFR-002 покрито REQ-09 і AC-33–AC-35 за всіма трьома пунктами Verification NFR-002.
3. **Трасування** — виконано. REQ і AC мають джерела, посилання на CQ, BR і NFR існують і означають те, що стверджується. REQ-03 і AC-18 спираються на рішення людини CQ-08. Технічні джерела звірено лише з CR (розділ 1).
4. **Перевірюваність AC** — виконано. Кожен AC має умову, дію, спостережуваний результат і Verification Method. Категорії відповіді й правило одного порушення роблять негативні AC розв'язними (F-002). Граничні значення перераховано вручну: AC-20 (242 + 12 = 254; 256 до trim), AC-21 (243 + 12 = 255), AC-23 (7 байтів), AC-24 (8), AC-25 (`Abcdé12` — 7 символів, 8 байтів), AC-26 (8 байтів із пробілами), AC-30 (`Ábcdefg1` і `Пabcdef1` — по 9 байтів, без `A–Z`). Значення коректні.
5. **Негативні й граничні сценарії** — виконано. Покрито:
   - невалідний формат і набір символів email, зокрема керівні символи й пробіли на краю й усередині та відсутнє поле email;
   - межі 254/255 символів, 7/8 і 1024/1025 байтів;
   - дубль з іншим регістром і пробілами, збережений email у змішаному регістрі (AC-41), одночасні запити;
   - відсутній пароль, недозволений `Origin`, вичерпаний ліміт хешування.
6. **Внутрішня узгодженість** — виконано. Суперечність REQ-03 і O-01 усунуто (F-001). AC-39 узгоджений зі спільним визначенням категорій (F-002). Суперечностей між REQ і AC не виявлено.
7. **Жодних вигаданих рішень** — виконано. Відкладених питань у CR немає. Позначене як поза scope відповідає CR і Product Vision. Віднесення пароля понад 1024 байти (REQ-08) до категорії «відмова валідації» — уточнення «відхиляється» в межах категорій специфікації; конкретний статус і код лишаються за Technical Design.
8. **Межа Technical Design** — виконано. Маршрут, статуси й коди для відмови валідації і «тимчасово недоступно», пріоритет відмов, обробку невідомих полів, форму unique constraint і параметри Argon2id залишено Technical Design. Від пунктів розділу 7 AC не залежать: правило одного порушення виключає залежність від пріоритету відмов, а AC-37 перевіряє лише те, що дані Profile не збережено.
9. **Story Type** — виконано. FEATURE зі `story_types`, джерело — Epic Map (`workflow-policies.md` §16). Роль, потребу, поведінку і scope покрито. Тип збігається в CR §2, front matter і тілі spec.
10. **Метадані й посилання** — виконано.
    - Поля front matter обох артефактів відповідають `artifact-rules.md` §1. `produced_by` — SPECIFICATION і CLARIFICATION відповідно до реєстру.
    - `inputs` spec містить `clarification-record` v4, тобто передану версію, а також `story-spec` v1 і `review-report/0001`.
    - `commit` кожного джерела spec і CR збігається з `git log -1 --format=%h -- <path>`: `epic-map.md`, `business-glossary.md`, `non-functional-requirements.md`, `personas.md`, `product-vision.md` — `0f02d11`; `business-rules.md`, `security-conventions.md`, `api-conventions.md`, `persistence-conventions.md` — `b873568`. `STALE` немає.
    - `acceptance_criteria` (41 ID, AC-01…AC-41) збігається з ID AC у тілі; кожен ID трапляється в таблицях AC рівно один раз.
    - `story_type` збігається з тілом.
    - `revision_reason` є в spec v2 і CR v4.
    - Пілотний рядок є в обох артефактах.
