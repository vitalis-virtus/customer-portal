# Epic Map — Customer Portal

## Purpose

The Epic Map divides the approved MVP Scope from the [Product Vision](product-vision.md#mvp-scope) into functional areas (Epics) and defines Candidate User Stories for each of them.

Terms are used in the meanings from the [Business Glossary](business-glossary.md). Constraints are set by the [Business Rules](business-rules.md) and [Non-Functional Requirements](non-functional-requirements.md).

**Status of Candidate Stories:**

- Candidate Stories have not yet gone through Clarification and are not approved specifications.
- Their identifiers are provisionally reserved for the future Backlog.
- Acceptance Criteria have not yet been defined for them.

Epic and Story identifiers are stable. An identifier does not change and is not reused, even if the Epic or Story is changed or removed.

## Overview

| Epic | Name | Candidate Stories |
|---|---|---|
| EPIC-001 | Account Access | US-001, US-002 |
| EPIC-002 | Self-Service Account Management | US-003, US-004, US-005 |
| EPIC-003 | User Access Administration | US-006, US-007 |

## EPIC-001 — Account Access

**Goal:** Enable a person to create an account on their own and log in to the system.

| ID | Candidate Story | Purpose |
|---|---|---|
| US-001 | Register Account | A person creates a new account in Customer Portal |
| US-002 | Log In | A user logs in to the system with their account |

**Related rules and requirements:** BR-003, BR-004, BR-005, BR-006, BR-007, NFR-001, NFR-002.

## EPIC-002 — Self-Service Account Management

**Goal:** Enable a user to keep their profile and password up to date on their own.

| ID | Candidate Story | Purpose |
|---|---|---|
| US-003 | View Own Profile | A user views their profile data |
| US-004 | Update Own Profile | A user changes their profile data |
| US-005 | Change Password | A user changes the password of their account |

**Related rules and requirements:** BR-002, BR-003, NFR-002.

## EPIC-003 — User Access Administration

**Goal:** Enable an administrator to control users' access to the platform.

| ID | Candidate Story | Purpose |
|---|---|---|
| US-006 | Block User | An administrator blocks a user |
| US-007 | Unblock User | An administrator cancels a user's blocking |

**Related rules and requirements:** BR-001, BR-006, BR-007, BR-008.

## Cross-Cutting Capabilities

The MVP capability **"user and admin roles"** is not allocated to a separate Epic. It is not a standalone user action; it is implemented through rules that apply in other Epics:

- BR-005: the `user` role is assigned during registration (US-001);
- BR-001: administrative capabilities are available only to the `admin` role (US-006, US-007);
- BR-006: a user has no more than one role. It applies during registration (US-001) and wherever the user's role matters (US-006, US-007);
- BR-008: only users with the `user` role can be blocked (US-006).

## Open Product Dependencies

Unresolved product dependencies of EPIC-003. They must be resolved before Clarification of the related Stories. No decisions have been approved yet.

| Dependency | Question | Related Candidate Stories |
|---|---|---|
| **Administrator Provisioning** | How does a user with the `admin` role appear in the system if self-registration assigns only the `user` role (BR-005)? | US-006, US-007 |
| **Target User Identification** | How does an administrator identify the user for whom blocking or unblocking is performed? | US-006, US-007 |
