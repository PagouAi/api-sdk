# Pagou.ai SDK

Cross-runtime SDK template for Node.js 18+, Bun, and Deno.

- Resource-first API (`client.transactions.*`)
- `fetch`-based (no axios)
- Typed errors
- Retries + timeout + idempotency
- Native `/v2` page/limit pagination

## Install

```bash
bun add @pagouai/api-sdk
# or npm i / pnpm add
```

## Quick Start

```ts
import { Client } from "@pagouai/api-sdk";

const client = new Client({
	apiKey: "YOUR_API_KEY",
	timeoutMs: 30_000,
	maxRetries: 2,
	telemetry: true,
});
```

## Configuration

```ts
new Client({
	apiKey: "YOUR_API_KEY",
	environment: "production", // default: "production"
	// environment: "sandbox",
	// baseUrl: "https://api.sandbox.pagou.ai", // explicit override if needed
	timeoutMs: 30_000,
	maxRetries: 2,
	telemetry: true,
	userAgent: "PagouTS-SDK/0.1.0 (Transactions; +https://pagou.ai)",
	fetch: customFetch,
	auth: { scheme: "bearer" }, // default
	// apiVersion: "2026-02-01",
});
```

Auth schemes:

- `bearer` (default) -> `Authorization: Bearer <apiKey>`
- `basic` -> `Authorization: Basic <base64(apiKey:x)>`
- `api_key_header` -> `apikey: <apiKey>` (or custom `headerName`)

Default base URLs:

- production: `https://api.pagou.ai`
- sandbox/test: `https://api.sandbox.pagou.ai`

## API Coverage (`/v2`)

- `POST /v2/transactions`
- `GET /v2/transactions`
- `GET /v2/transactions/{id}`
- `PUT /v2/transactions/{id}` (test/sandbox-only behavior)
- `PUT /v2/transactions/{id}/refund`

## Usage Examples

### 1) Create a transaction

For `method: "pix"`, `buyer.document` is required and `buyer.document.type` must be uppercase:
- `CPF`
- `CNPJ`

```ts
const created = await client.transactions.create({
	amount: 1500,
	method: "pix",
	currency: "BRL",
	buyer: {
		name: "Jane Doe",
		email: "jane@example.com",
		document: {
			type: "CPF",
			number: "12345678901",
		},
	},
	products: [{ name: "Pro Plan", price: 1500, quantity: 1 }],
});

console.log(created.data.id, created.meta.requestId);
```

### 2) Retrieve a transaction

```ts
const tx = await client.transactions.retrieve("tr_123");
console.log(tx.data.status);
```

### 3) List transactions with page/limit + filters

```ts
const page = await client.transactions.list({
	page: 1,
	limit: 20,
	status: ["pending", "paid"],
	paymentMethods: ["pix"],
});

console.log(page.data.metadata.total);
for (const item of page.data.data) {
	console.log(item.id, item.status);
}
```

### 4) Update a transaction (test/sandbox only)

```ts
const updated = await client.transactions.update(
	"tr_123",
	{ status: "paid" },
	{ idempotencyKey: "idem_update_tr_123_1" },
);

console.log(updated.data.status);
```

### 5) Auto-paging iterator

```ts
for await (const item of client.transactions.listAutoPagingIterator({ limit: 100 })) {
	console.log(item.id);
}
```

### 6) Refund a transaction

```ts
const refunded = await client.transactions.refund(
	"tr_123",
	{ amount: 500, reason: "requested_by_customer" },
	{ idempotencyKey: "idem_refund_tr_123_1" },
);

console.log(refunded.data);
```

## Request Options

All resource methods accept `opts?`:

```ts
{
	idempotencyKey?: string;
	requestId?: string;
	timeoutMs?: number;
	signal?: AbortSignal;
}
```

## Retries and Timeout

Retries happen for:

- Network errors
- `429`, `500`, `502`, `503`, `504`

Rules:

- Always retry for `GET`/`HEAD`
- Retry `POST`/`PUT` only when `idempotencyKey` is set
- Respect `Retry-After` header when available

Timeout uses `AbortController`.

## Error Handling

```ts
import {
	AuthenticationError,
	InvalidRequestError,
	NotFoundError,
	RateLimitError,
	ServerError,
	NetworkError,
} from "@pagouai/api-sdk";

try {
	await client.transactions.retrieve("tr_404");
} catch (error) {
	if (error instanceof NotFoundError) {
		console.error("Missing transaction", error.requestId);
	} else if (error instanceof RateLimitError) {
		console.error("Back off", error.status, error.code);
	} else if (error instanceof NetworkError) {
		console.error("Network issue", error.message);
	} else if (error instanceof InvalidRequestError || error instanceof AuthenticationError || error instanceof ServerError) {
		console.error(error.message);
	} else {
		throw error;
	}
}
```

## Response Shapes

Data endpoints:

```ts
{
	success: boolean;
	requestId: string;
	data: T;
}
```

List endpoints:

```ts
{
	success: boolean;
	requestId: string;
	metadata: { page: number; limit: number; total: number };
	data: T[];
}
```

Notes:
- This SDK uses `/v2` endpoint paths.
