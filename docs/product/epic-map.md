# Epic Map — Customer Portal

## Призначення

Epic Map ділить погоджений MVP Scope з [Product Vision](product-vision.md#mvp-scope) на функціональні області (Epics) і визначає для кожної з них Candidate User Stories.

Терміни вживаються у значеннях із [Business Glossary](business-glossary.md). Обмеження задають [Business Rules](business-rules.md) та [Non-Functional Requirements](non-functional-requirements.md).

**Статус Candidate Stories:**

- Candidate Stories ще не проходили Clarification і не є погодженими специфікаціями.
- Їхні ідентифікатори попередньо зарезервовано для майбутнього Backlog.
- Acceptance Criteria для них ще не визначено.

Ідентифікатори Epics і Stories стабільні. Ідентифікатор не змінюється і не використовується повторно, навіть якщо Epic чи Story змінено або вилучено.

## Огляд

| Epic | Назва | Candidate Stories |
|---|---|---|
| EPIC-001 | Account Access | US-001, US-002 |
| EPIC-002 | Self-Service Account Management | US-003, US-004, US-005 |
| EPIC-003 | User Access Administration | US-006, US-007 |

## EPIC-001 — Account Access

**Goal:** Дати людині змогу самостійно створити обліковий запис і входити в систему.

| ID | Candidate Story | Призначення |
|---|---|---|
| US-001 | Register Account | Людина створює новий обліковий запис у Customer Portal |
| US-002 | Log In | Користувач входить у систему через свій обліковий запис |

**Пов'язані правила та вимоги:** BR-003, BR-004, BR-005, BR-006, BR-007, NFR-001, NFR-002.

## EPIC-002 — Self-Service Account Management

**Goal:** Дати користувачу змогу самостійно підтримувати актуальність свого профілю та пароля.

| ID | Candidate Story | Призначення |
|---|---|---|
| US-003 | View Own Profile | Користувач переглядає дані свого профілю |
| US-004 | Update Own Profile | Користувач змінює дані свого профілю |
| US-005 | Change Password | Користувач змінює пароль свого облікового запису |

**Пов'язані правила та вимоги:** BR-002, BR-003, NFR-002.

## EPIC-003 — User Access Administration

**Goal:** Дати адміністратору змогу контролювати доступ користувачів до платформи.

| ID | Candidate Story | Призначення |
|---|---|---|
| US-006 | Block User | Адміністратор блокує користувача |
| US-007 | Unblock User | Адміністратор скасовує блокування користувача |

**Пов'язані правила та вимоги:** BR-001, BR-006, BR-007, BR-008.

## Наскрізні можливості

Можливість MVP **«Ролі user та admin»** не виділено в окремий Epic. Вона не є самостійною дією користувача, а реалізується через правила, які діють в інших Epics:

- BR-005: роль `user` призначається під час реєстрації (US-001);
- BR-001: адміністративні можливості доступні лише ролі `admin` (US-006, US-007);
- BR-006: користувач має не більше однієї ролі. Діє під час реєстрації (US-001) і там, де важлива роль користувача (US-006, US-007);
- BR-008: блокувати можна лише користувачів із роллю `user` (US-006).

## Open Product Dependencies

Невирішені продуктові залежності EPIC-003. Їх треба вирішити до Clarification пов'язаних Stories. Рішень поки не погоджено.

| Залежність | Питання | Пов'язані Candidate Stories |
|---|---|---|
| **Administrator Provisioning** | Як у системі з'являється користувач із роллю `admin`, якщо самостійна реєстрація призначає лише роль `user` (BR-005)? | US-006, US-007 |
| **Target User Identification** | Як адміністратор визначає користувача, для якого виконується блокування або розблокування? | US-006, US-007 |
