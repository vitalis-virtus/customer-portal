# Business Rules — Customer Portal

## Purpose

This document contains the global business rules of Customer Portal. These are product rules and invariants that apply regardless of any specific User Story.

Only rules that have one of two justifications are included here:

- they follow directly from approved sources: [Product Vision](product-vision.md), [Personas](personas.md), [Business Glossary](business-glossary.md);
- they were explicitly approved by the product owner. The "Sources" field of such a rule states "Product owner decision" with a date.

Terms in the rules are used in the meanings from the [Business Glossary](business-glossary.md).

Rule identifiers are stable. An identifier does not change and is not reused, even if the rule is changed or retired.

## Rules

### BR-001 — Admin-Only Capabilities

Administrative capabilities are available only to users with the `admin` role. In the MVP, these are blocking and unblocking users.

**Sources:**

- [Product Vision — MVP Success Criteria](product-vision.md#mvp-success-criteria)
- [Product Vision — Target Users](product-vision.md#target-users)
- [Personas — Platform Administrator](personas.md#persona-2-platform-administrator)

### BR-002 — Own Account Self-Service

Profile management and password change available to a user apply only to that user's own Account.

**Sources:**

- [Product Vision — MVP Scope](product-vision.md#mvp-scope): "Own profile management", "Changing the password of one's own account"
- [Personas — Portal User](personas.md#persona-1-portal-user)

### BR-003 — Unique Email

An email uniquely corresponds to one Customer Portal user. Two users cannot have the same email.

**Sources:**

- Product owner decision (2026-11-10)

### BR-004 — Open Registration

Anyone can register in Customer Portal. Registration must comply with the other business rules in this document.

**Sources:**

- Product owner decision (2026-11-10)
- [Product Vision — Product Goals](product-vision.md#product-goals), goal 1

### BR-005 — Default Role on Registration

A user who has registered receives the `user` role.

**Sources:**

- Product owner decision (2026-11-10)

### BR-006 — Single Role

A user cannot have more than one role at the same time.

**Sources:**

- Product owner decision (2026-11-10)

### BR-007 — Blocked User Access

A blocked user immediately loses access to their Account: none of their requests sent after the blocking is completed is executed. After unblocking, access is restored.

**Sources:**

- Product owner decision (2026-11-10)
- Product owner decision (2026-11-13): clarified that the loss of access is immediate
- Product owner decision (2026-11-13): clarified that the guarantee applies to requests sent after the blocking is completed
- [Business Glossary — User Blocking, User Unblocking](business-glossary.md#administration)

### BR-008 — Administrators Cannot Be Blocked

An administrator cannot block a user with the `admin` role. This also applies to the administrator themselves. Only users with the `user` role can be blocked.

**Sources:**

- Product owner decision (2026-11-10)
