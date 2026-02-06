# error-async-try-catch: Error handling in async handlers

## Why it matters

Async route handlers can throw or reject. Uncaught errors leave the client hanging and make debugging harder. Consistent try/catch and logging keep responses predictable and errors traceable.

## Incorrect

```javascript
"GET /user/:id": {
	handler: async function (req, res) {
		const user = await Services.User.getById(req.params.id);
		res.json(user);
	},
},
```

If `getById` throws or rejects, the request never gets a response and the error may only appear in `unhandledRejection`. The client gets no status or body.

## Correct

```javascript
const { logger } = require("../utils");

"GET /user/:id": {
	handler: async function (req, res) {
		try {
			const user = await Services.User.getById(req.params.id);
			if (!user) {
				return res.status(404).json({ error: "User not found" });
			}
			res.json(user);
		} catch (e) {
			logger.error(e);
			res.status(500).json({ error: "Internal server error" });
		}
	},
},
```

- Errors are caught and logged with `logger.error`.
- Client always receives a response (404 for missing resource, 500 for server error).
- In production, avoid sending `e.message` or stack traces to the client; use a generic message or a safe error code.

## Notes

- Use the project’s `logger` from `utils.js` for errors so they appear in the same format as the rest of the app.
- For validation errors (e.g. invalid `id`), return 400 with a clear message instead of 500.
- Middleware that does async work should use try/catch and call `next(err)` so Express or a global error handler can respond.
