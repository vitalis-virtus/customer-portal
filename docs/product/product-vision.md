# Product Vision — Customer Portal

## Product Overview

Customer Portal is a backend platform through which users gain access to the portal and manage their account, and administrators manage users.

Product development starts with user management. This part will become the foundation to which other domains can later be added.

Customer Portal is an educational project.

## Product Vision

Customer Portal is the single entry point for the platform's users. Here a user can:

- create an account;
- log in to the system securely;
- manage their profile on their own.

In the portal, an administrator controls users' access to the platform.

Over time, the portal may expand with new domains, including Product Catalog, Orders, Support and others. All of them will rely on the shared user management foundation.

## Target Users

| User | Description | What they get from the product |
|---|---|---|
| **User** | A person who registers and uses the portal | Their own account, logging in to the system, profile and password management |
| **Admin** | A person responsible for managing the platform's users | The ability to block and unblock users |

## Product Goals / Success Criteria

### Product Goals

1. Enable users to create an account on their own and gain access to the portal.
2. Ensure secure access to accounts in accordance with users' roles and permissions.
3. Enable users to keep their profile and password up to date on their own.
4. Separate the capabilities of regular users and administrators.
5. Enable administrators to control users' access to the platform.

### MVP Success Criteria

The MVP is considered successful when all capabilities from the [MVP Scope](#mvp-scope) section are available and work in accordance with their roles:

- a user can register, log in to the system, view and change their profile, and change their password;
- administrator capabilities are available only to users with the admin role;
- an administrator can block and unblock a user.

No quantitative success metrics have been set at this stage.

## MVP Scope

The initial MVP includes:

- **User registration** — creation of a new account.
- **Authentication** — a user logging in to the system.
- **Own profile management** — viewing and changing one's own profile data.
- **Password change** — changing the password of one's own account.
- **user and admin roles** — separation of the capabilities of regular users and administrators.
- **Administrative user management** — blocking and unblocking users.

## Out of Scope

The initial MVP does not include:

- **Email verification** — confirmation of the email address.
- **Future domains** — Product Catalog, Orders, Support and others.
- **Any capabilities not listed in the [MVP Scope](#mvp-scope) section.** They require separate approval.

---

> **Note on the educational goal.** Separately from the product goals, the project has an engineering educational goal: to learn and apply Artifact-Driven Agentic SDLC in practice. This goal is not a business goal of the product and does not affect its scope.
