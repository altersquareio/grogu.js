# validation-input: Validate and sanitize user input

## Why it matters

User input (query, body, params) can be missing, malformed, or malicious. Validating and sanitizing it prevents bad data from reaching services, avoids security issues (e.g. injection), and lets you return clear 400 responses instead of 500s or undefined behavior.

## Incorrect

```javascript
"POST /order": {
	handler: async function (req, res) {
		const { productId, quantity } = req.body;
		const order = await Services.Order.create({ productId, quantity });
		res.status(201).json(order);
	},
},
```

- No check that `productId` or `quantity` exist or are valid types.
- Invalid or missing values can cause service errors or bad data in the database.
- Client gets no clear feedback about what was wrong.

## Correct

```javascript
"POST /order": {
	handler: async function (req, res) {
		try {
			const { productId, quantity } = req.body || {};
			if (productId == null || typeof productId !== "string" || !productId.trim()) {
				return res.status(400).json({ error: "productId is required and must be a non-empty string" });
			}
			const q = Number(quantity);
			if (Number.isNaN(q) || q < 1 || !Number.isInteger(q)) {
				return res.status(400).json({ error: "quantity must be a positive integer" });
			}
			const order = await Services.Order.create({ productId: productId.trim(), quantity: q });
			res.status(201).json(order);
		} catch (e) {
			logger.error(e);
			res.status(500).json({ error: "Internal server error" });
		}
	},
},
```

- Validates presence and type; sanitizes (e.g. trim, parse to number).
- Returns 400 with a clear message when input is invalid.
- Uses validated/sanitized values in the service call.

## Notes

- Prefer early returns with 400 for invalid input so the handler body only deals with valid data.
- For complex schemas, consider a validation library (e.g. Joi, Zod) and keep validation in one place (e.g. a middleware or a small helper).
- Use 404 when the resource is missing (e.g. valid id but no record); use 400 when the request shape or values are wrong.
- Do not trust `req.params` or `req.query` without validation; validate and coerce types (ids, numbers, booleans) before use.
