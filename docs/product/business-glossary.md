# Business Glossary — Customer Portal

## Purpose

The Business Glossary defines a single business terminology for Customer Portal. When agents and people describe the product, they must use the terms exactly in the meanings given here.

Canonical sources of product context:

- [Product Vision](product-vision.md) defines the product concept and scope.
- [Personas](personas.md) describes user needs.
- The Business Glossary is the canonical source of the meanings of product terms.

If a contradiction arises between these documents, the agent must report it. The agent must not automatically prefer one of the documents.

The glossary defines only the meanings of terms. It does not establish business rules, technical models, or relationships between concepts beyond those already present in the sources.

## Participants

### Customer Portal

The product: a platform through which users gain access to the portal and manage their account, and administrators manage users.

### User

A person who uses Customer Portal or is going through the account creation process in order to use it.

> An administrator also belongs to the general concept of User. To distinguish the roles, people say "a user with the `user` role".

### Administrator

A user responsible for managing the platform's users. Abbreviation — **Admin**.

> Administrator is a person. `admin` is a role (Role) that grants that person administrative capabilities.

## Accounts

### Account

A user's account. Through it, the user gains access to the portal.

### Email

An email address that uniquely corresponds to one Customer Portal user.

> Email uniqueness is established by rule [BR-003](business-rules.md#br-003--unique-email).

### Profile

User data that the user can view and change on their own.

> The composition of profile data is not defined by the product documents.

### Password

Secret credentials associated with an Account that the user can change.

## Access

### Registration

Creation of a new account by a user.

### Authentication

A user logging in to the system.

### Role

A user attribute that determines which portal capabilities are available to the user. The MVP has two roles: `user` and `admin`.

> Role differs from Persona. A Persona describes a person, their goals and needs (see [Personas](personas.md)). A Role describes capabilities in the product.

### Authorization

Separation of portal capabilities according to the user's role. In particular, administrative capabilities are available only to users with the `admin` role.

> Authentication answers the question "who has logged in". Authorization answers the question "what is available to them".

## Administration

### User Blocking

An administrative action that restricts a user's access to their account.

> Blocking behavior is defined by rule [BR-007](business-rules.md#br-007--blocked-user-access).

### User Unblocking

An administrator action by which the administrator cancels a user's blocking.
