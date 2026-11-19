# Non-Functional Requirements — Customer Portal

## Purpose

This document contains the non-functional requirements (NFRs) for Customer Portal. These are requirements for the product's quality attributes, not for its capabilities.

Only requirements that have one of two justifications are included here:

- they reasonably follow from approved sources: [Product Vision](product-vision.md), [Personas](personas.md), [Business Glossary](business-glossary.md), [Business Rules](business-rules.md);
- they were explicitly approved by the product owner. The Source field of such a requirement states "Product owner decision" with a date.

NFRs do not repeat Business Rules and do not define technical decisions.

Identifiers are stable. An identifier does not change and is not reused, even if the requirement is changed or removed.

### Statuses

- `APPROVED` — the requirement is approved and has an objective verification criterion.
- `NEEDS_CLARIFICATION` — the requirement is supported by sources, but its verification criteria have not yet been defined. They must be defined before the corresponding functionality is implemented.

## Requirements

### NFR-001 — Secure Authentication

- **Status:** `NEEDS_CLARIFICATION`
- **Category:** Security
- **Requirement:** A user's login to the system must be secure.
- **Verification:** Not defined. What exactly counts as secure authentication must be defined before authentication is implemented.
- **Source:**
  - [Product Vision — Product Vision](product-vision.md#product-vision): "log in to the system securely"
  - [Product Vision — Product Goals](product-vision.md#product-goals), goal 2
  - [Personas — Portal User, Needs](personas.md#needs)

### NFR-002 — Password Confidentiality

- **Status:** `APPROVED`
- **Category:** Security
- **Requirement:** The system must not store passwords in plain text or in a recoverable form. Stored credentials must not allow the original password value to be obtained. Passwords must not be disclosed in system responses or logs.
- **Verification:**
  - Stored credentials do not contain the password in plain text, and the original password value cannot be obtained from them.
  - No system response contains a password.
  - No log contains a password.
- **Source:**
  - Product owner decision (2026-11-10)
  - [Business Glossary — Password](business-glossary.md#password): "secret credentials"

## Retired

Identifiers of removed requirements are reserved. They are not reused.

| ID | Name | Reason for removal |
|---|---|---|
| NFR-003 | Foundation for Future Domains | Duplicates the strategic intent from the [Product Vision](product-vision.md#product-vision) and has no verification criteria |
