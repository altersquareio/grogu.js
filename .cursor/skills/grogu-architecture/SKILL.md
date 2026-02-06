---
name: grogu-architecture
description: Documents grogu.js controller, service, and middleware layout and dependency injection. Use when adding or changing controllers, services, middlewares, routes, API versioning, or when explaining grogu.js structure.
---

# Grogu Architecture

Project-specific layout and conventions for the grogu.js Express backend. Single source of truth for how to add or change endpoints, services, and middlewares.

## When to Apply

Reference these guidelines when:

- Adding a new controller, service, or middleware
- Defining or modifying routes
- Using `config` or `Services` in handlers or middlewares
- Configuring API versioning
- Explaining or onboarding to the grogu.js codebase

## Bootstrap Flow

From `app.js`, load order is:

1. **Config** — `config` = `{ CONSTANTS, ...conf, rootDir }` from `config/` (constants.js, conf.js). apiVersions loaded separately.
2. **Services** — All `.js` files in `services/` loaded via `utils.dirIterator`; each export is invoked with `{ config, Services }` and may return a Promise; result stored in `Services` by PascalCase filename (e.g. `myService.js` → `Services.MyService`).
3. **Middlewares** — All `.js` files in `middlewares/` loaded; stored in `Middlewares` by filename (e.g. `auth.js` → `Middlewares.auth`).
4. **HTTP middlewares** — If `config/http.js` exists, its exported array is applied with `app.use()` (e.g. body-parser, compression, cors).
5. **Controllers** — Each `.js` file in `controllers/` is loaded; `routes({ Services, config })` is called; returned route config is registered on an Express Router mounted at `/{baseRoute}`. Base route = PascalCase filename (e.g. `Public.js` → `/Public`).

## Controllers

- **Location:** `controllers/`
- **Filename = base route:** `Public.js` → base path `/Public`; `UserProfile.js` → `/UserProfile`.
- **Required export:** `routes({ Services, config })` — function that returns an object of route definitions.

### Route definition shape

Each key in the returned object is either:

- `"METHOD /path"` (e.g. `"GET /test"`, `"POST /login"`) — HTTP method is in the key, or
- `"/path"` with a `method` property on the value (e.g. `method: "GET"`).

Each value must have:

- **`handler`** — `(req, res)` or `(req, res, next)` function. Receives Express request/response; `Services` and `config` are not passed to the handler (use closure from `routes({ Services, config })`).

Optional:

- **`localMiddlewares`** — Array of middleware **names** (strings) from `middlewares/`. Applied in order before the handler. Example: `["auth", "rateLimit"]`.
- **`version`** — API version string (e.g. `"v2.0"`). Must be in `config/apiVersions.js` `allowedVersions`. If omitted, route uses `default` version.
- **`enabled`** — Set to `false` to disable the route (it will be skipped at startup).

### Route path and versioning

- Final path = `/{baseRoute}/{version}{path}`.
- Version comes from `config/apiVersions.js`: `default` and `allowedVersions`. Example: default `v1.0`, path `/test` → `/Public/v1.0/test`.

### Global middlewares (per controller)

- Optional export: `globalMiddlewares` — array of middleware names. These run for every route under this controller’s base path, before the router. Attached via `app.use("/" + baseRoute, middleware)`.

### Example controller

See `controllers/Public.js`:

```javascript
module.exports.routes = function ({ Services, config }) {
	return {
		"GET /test": {
			handler: async function (req, res) {
				try {
					res.json({ ok: true, message: "hello world" });
				} catch (e) {
					res.json({ ok: true, message: e.message });
					logger.error(e);
				}
			},
		},
	};
};
```

## Services

- **Location:** `services/`
- **One file per service.** Filename (PascalCase) = key in `Services` (e.g. `userService.js` → `Services.UserService`).
- **Export:** A single function that receives `{ config, Services }` and returns the service API (object or value). The function may be `async`; it is awaited at startup.
- **Use:** In controllers, use `Services.SomeService` from the `routes({ Services, config })` closure. In middlewares, `Services` and `config` are injected as the last argument.

## Middlewares

- **Location:** `middlewares/`
- **Signature:** `(req, res, next, deps)` where `deps` = `{ Services, config }`. The last argument is injected by `utils.injectDependencyArgument`; do not expect it from Express.
- **Referenced by name:** In controllers, use the **filename** (no `.js`), e.g. `localMiddlewares: ["auth"]` for `middlewares/auth.js`.
- **Errors:** Call `next(err)` to pass to Express error handling.

## Config

- **In handlers and middlewares:** `config` = `{ CONSTANTS, ...require("config/conf"), rootDir }`. `CONSTANTS` from `config/constants.js`.
- **API versions:** `config/apiVersions.js` exports `default` and `allowedVersions`. Do not put version config in `conf.js`; it is read separately in `app.js`.
- **Server-level middlewares:** `config/http.js` exports an array of Express middlewares (e.g. body-parser, compression, cors). Applied with `app.use()` before any controller routes.

## API versioning

- **File:** `config/apiVersions.js`
- **Fields:** `default` (string), `allowedVersions` (array of strings).
- **Behavior:** If a route does not set `version`, `default` is used. If a route sets `version`, it must be in `allowedVersions`; the path becomes `/{version}{path}` (e.g. `v2.0` → `/Public/v2.0/test`).

## Quick reference

| Task | Where | What to do |
|------|--------|------------|
| Add a route | Controller file in `controllers/` | Add an entry to the object returned by `routes({ Services, config })` with `handler`, optional `localMiddlewares`, `version`, `enabled`. |
| Add a controller | New file in `controllers/` | Create `SomeName.js`; export `routes` returning route definitions. Base path = `SomeName` → `/SomeName`. |
| Add a service | New file in `services/` | Export a function `({ config, Services }) => ...` (sync or async). Use `Services` and `config` inside. |
| Add a middleware | New file in `middlewares/` | Export function `(req, res, next, { Services, config }) => { ... }`. Reference by filename in `localMiddlewares` or `globalMiddlewares`. |
| Change default API version | `config/apiVersions.js` | Set `default` and ensure it is in `allowedVersions`. |
| Add server-wide middleware | `config/http.js` | Export array of middlewares; they are applied with `app.use()`. |

## Utils (project root)

- **`utils.dirIterator(dir, callback)`** — Iterates `.js` files in `dir`; callback `(filenameWithoutExt, filepath)`.
- **`utils.getValidHttpMethod(str)`** — Returns HTTP method if key starts with one (e.g. `"GET /x"` → `"get"`); otherwise `null`.
- **`utils.fatalError(msg)`** — Logs and `process.exit(1)`.
- **`utils.injectDependencyArgument(fn, deps)`** — Wraps `fn` so that when called as `(req, res, next)`, it is invoked with `(req, res, next, deps)`. Used for middlewares.
- **`utils.logger`** — `logger.info`, `logger.error`, `logger.warn`, `logger.debug`.

## Additional resources

- For request flow and diagram, see [ARCHITECTURE.md](ARCHITECTURE.md).
