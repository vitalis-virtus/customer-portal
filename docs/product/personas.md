# Personas — Customer Portal

Initial proto-personas of the Customer Portal product. The canonical source of product context is the [Product Vision](product-vision.md).

**Important:**

- These are proto-personas. They were derived from the Product Vision, not from user research. No real user research has been conducted.
- Items marked **[Hypothesis]** are assumptions. They must be validated.
- A Persona describes a person and their goals. It is not the same as a technical role in the system. The related role is given for each persona for reference only.

---

## Persona 1: Portal User

### Persona

A person who registers in Customer Portal and uses it through their own account.

Related system role: `user`.

### Context

- Gains access to the platform through Customer Portal as the single entry point.
- Creates the account on their own and then manages it on their own.

### Goals

- Gain access to the portal.
- Keep their profile data up to date.
- Control access to their account.

### Needs

- Create an account on their own.
- Log in to the system securely.
- View and change their profile data.
- Change the password of their account.

### Frustrations

- **[Hypothesis]** Uncertainty that only the owner has access to the account.
- **[Hypothesis]** Dependence on other people if changing profile data or the password on their own is not possible.

### Relevant Product Capabilities

- User registration
- Authentication
- Own profile management
- Password change

---

## Persona 2: Platform Administrator

### Persona

A person responsible for managing the platform's users.

Related system role: `admin`.

### Context

- Controls users' access to the platform.
- Uses capabilities that are available only to administrators.

### Goals

- Control which users have access to the platform.

### Needs

- Block users.
- Unblock users.
- Be confident that administrative capabilities are available only to administrators.

### Frustrations

- **[Hypothesis]** Inability to quickly restrict a user's access when needed.
- **[Hypothesis]** Risk that administrative capabilities become available to people who are not administrators.

### Relevant Product Capabilities

- Authentication
- user and admin roles
- Administrative user management (blocking and unblocking)
