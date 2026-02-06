---
name: express-node-best-practices
description: Node.js and Express backend guidelines for routing, error handling, middleware, security, validation, and async code. Use when writing or reviewing Express routes, middleware, error handling, security, validation, or backend API code in this project.
---

# Express & Node Best Practices

Guidelines for writing and reviewing backend code in this Express/grogu.js project. Keeps handlers, middleware, and config consistent and secure.

## When to Apply

Reference these guidelines when:

- Adding or changing routes and handlers
- Writing or reviewing middleware
- Implementing or reviewing error handling
- Using env/config or security-related code (CORS, body size, etc.)
- Validating or sanitizing input
- Working with async/await or promises in handlers or services

## Rule Categories by Priority

| Priority | Category              | Impact     | Prefix        |
|----------|------------------------|------------|---------------|
| 1        | Routing and handlers   | CRITICAL   | `routing-`    |
| 2        | Error handling         | CRITICAL   | `error-`      |
| 3        | Middleware             | HIGH       | `middleware-` |
| 4        | Security and config    | HIGH       | `security-`   |
| 5        | Validation and input   | MEDIUM-HIGH| `validation-` |
| 6        | Async                  | MEDIUM     | `async-`      |

## Quick Reference

### 1. Routing and handlers (CRITICAL)

- **routing-thin-handlers** — Keep handlers thin; delegate business logic to services. Handlers should orchestrate (call service, send response), not contain complex logic.
- **routing-http-methods** — Use correct HTTP methods: GET for read, POST for create, PUT/PATCH for update, DELETE for remove. Avoid overloading GET with mutations.
- **routing-status-codes** — Use appropriate status codes: 200/201 for success, 400 for bad input, 401 for unauthorized, 403 for forbidden, 404 for not found, 500 for server errors.

### 2. Error handling (CRITICAL)

- **error-async-try-catch** — Wrap async handler logic in try/catch; catch errors and send a consistent error response (e.g. 500) and log with `logger.error`. See [rules/error-handling-async.md](rules/error-handling-async.md).
- **error-no-leak-stack** — Do not send stack traces or internal details in production responses. Log them server-side only.
- **error-centralize** — Prefer a shared error-response shape (e.g. `{ error: message }`) and, where useful, a small helper or middleware for formatting errors.

### 3. Middleware (HIGH)

- **middleware-order** — Order matters: body parsing and global security (e.g. CORS) first (already in `config/http.js`), then app-specific middleware. Do not block the event loop with sync heavy work.
- **middleware-next-err** — In middleware, call `next(err)` to pass errors to Express error handling; do not swallow errors.
- **middleware-deps** — In this project, middlewares receive `{ Services, config }` as the last argument (injected). Use them for cross-cutting concerns (auth, logging, etc.).

### 4. Security and config (HIGH)

- **security-env-only** — Use `process.env` and dotenv for secrets and environment-specific config (already used in this project). Never commit secrets or hardcode them.
- **security-cors-body** — CORS and body-parser limits are configured in `config/http.js`. Adjust whitelist and `limit` there; do not disable security for convenience.
- **security-whitelist** — When changing CORS, update the whitelist in `config/http.js` explicitly; avoid broad `origin: true` in production unless intended.

### 5. Validation and input (MEDIUM-HIGH)

- **validation-sanitize** — Validate and sanitize all user input (query, body, params). Reject invalid input with 400 and a clear message. See [rules/validation-input.md](rules/validation-input.md).
- **validation-status** — Use 400 for bad or invalid input, 404 for missing resource, 409 for conflict when applicable.

### 6. Async (MEDIUM)

- **async-prefer-await** — Prefer async/await in handlers and services. Handle promise rejections (try/catch or .catch); ensure unhandled rejections are logged (project already has `unhandledRejection` handler in app.js).
- **async-service-init** — Services in this project are async-initialized at startup; keep init logic in the service factory and avoid blocking there.

## How to Use

- Apply these rules when writing or reviewing backend code in `controllers/`, `services/`, `middlewares/`, and `config/`.
- For deeper guidance on specific topics, read the rule files under `rules/`:
  - [rules/error-handling-async.md](rules/error-handling-async.md)
  - [rules/validation-input.md](rules/validation-input.md)
