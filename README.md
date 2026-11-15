# **WEEK 5: Campus Store API**
- **Name:** Janelyn G. Nollora
- **Section:** ACT2
- **Activity Title:** Design a Campus Store API

> **Scenario:** A campus store has thousands of products. Students may retry order requests after connection failures, and two students may attempt to purchase the last item.

---
## Endpoints & Pagination

### Retrieving one product
```http
GET /api/v1/products/67 HTTP/1.1
Host: lvcc.campus.store
Accept: application/json
```
**Path Parameter:** 67 representing the product ID being fetched.

### List filtered and sorted products
```http
GET /api/v1/products?category=textbooks&course_code=ACT2&sort=name&order=asc&page=1&limit=10 HTTP/1.1
Host: lvcc.campus.store
Accept: application/json
```

- **Query parameters:**
  - `category`: filter by the textbooks category
  - `course_code`: filter by course
  - `sort`: field to sort by `name`
  - `order`: sort the textbooks alphabetically in `asc` order
  - `limit`: batch size limit set to 10
  - `page`: return the very first page of the result

### Next-batch request
```http
GET /api/v1/products?category=textbooks&course_code=ACT2&sort=name&order=asc&page=2&limit=10 HTTP/1.1
Host: lvcc.campus.store
Accept: application/json
```

The client sends back the same filters and sort query parameters, incrementing the page from 1 to 2.

### Create an order

```http
POST /api/v1/orders HTTP/1.1
Host: lvcc.campus.store
Content-Type: application/json
Accept: application/json

{
  "items"L [{
    "product_id": 67,
    "quantity": 1
  }]
}
```

### Pagination choice: page-based (offset)
I chose **page-based pagination**. Using page numbers makes it easy for students to navigate back and forth through catalog lists and it naturally maps to traditional frontend pagination UI elements like the pagination bar.

---

## Reliable Order Creation

### POST request with `Idempotency-Key`

```http
POST /api/v1/orders HTTP/1.1
Host: lvcc.campus.store
Content-Type: application/json
Authorization: Bearer <token>
Idempotency-Key: 3f1c9a52-7d0e-4b8a-9b1e-5c2f6a7d8e90

{
  "items": [
    { "product_id": 67, "quantity": 1 }
  ]
}
```

**Success response:**

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "order_id": 1001,
  "status": "confirmed",
  "total_centavos": 8900
}
```

The client generates a unique key (UUID v4) **once per order attempt** and reuses the same key on every retry of that attempt.

### How key reuse is handled

The server stores each key in an `idempotency_keys` table with a **unique constraint** on `(user_id, key)`. It saves a hash of the request body, the status, and the response.

| Situation | Server behavior |
|---|---|
| New key | Process the order, store the response with the key. |
| Same key, same body (retry after connection failure) | Return the **stored response** without creating another order. Adds header `Idempotent-Replay: true`. |
| Same key, **different** body | Reject with `422 Unprocessable Entity` because the key was misused. |
| Same key, first request still processing | Return `409 Conflict` so the client retries shortly. |

Keys expire after 24 hours and can be cleaned up by a scheduled job.

### Database transaction steps for stock reduction

```
BEGIN TRANSACTION
 1. INSERT idempotency key row (unique constraint rejects duplicates)
 2. SELECT id, stock, price_centavos FROM products
      WHERE id IN (...) ORDER BY id FOR UPDATE      -- lock rows
 3. Check stock >= requested quantity for every item
      - if not: ROLLBACK and return 409 "Insufficient stock"
 4. UPDATE products SET stock = stock - :qty WHERE id = :id
 5. INSERT INTO orders (...)
 6. INSERT INTO order_items (...)
 7. UPDATE idempotency key row with response status and body
COMMIT
```

If any step fails, everything is rolled back. Stock is never reduced without an order, and an order never exists without reduced stock.

### Concurrency control: pessimistic locking

I chose **pessimistic locking** with `SELECT ... FOR UPDATE` (in Laravel: `lockForUpdate()`).

- **Scenario:** Two students try to buy the last item. Student A locks the product row. Student B's transaction waits. A decrements stock from 1 to 0 and commits. B then reads stock = 0 and gets `409 Insufficient stock`. Overselling is impossible.
- **Why this over optimistic locking?** For a scarce, hot item like the last unit, many conflicts are expected, so optimistic retries would waste work. Locking is simpler to reason about.
- **Deadlock prevention:** Rows are always locked in ascending `id` order.
- Locks are held only for the short transaction, so throughput stays acceptable.

As a safety net, the database also has `CHECK (stock >= 0)`.

---

## Response DTO

### Internal fields vs. public fields

| Internal field | Public field | Exposed? |
|---|---|---|
| `id` | `id` | Yes |
| `product_name` | `name` | Yes (renamed) |
| `price_centavos` | `price_centavos` | Yes |
| `cost` | none | **No** |
| `notes` | none | **No** |

### Public product JSON schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "PublicProduct",
  "type": "object",
  "properties": {
    "id": { "type": "integer" },
    "name": { "type": "string", "description": "Mapped from internal product_name" },
    "price_centavos": { "type": "integer", "minimum": 0 }
  },
  "required": ["id", "name", "price_centavos"],
  "additionalProperties": false
}
```

Example output:

```json
{ "id": 42, "name": "School Lanyard", "price_centavos": 8900 }
```

### Why `cost` and `notes` are excluded

- **`cost` (supplier cost):** This is confidential business information. If customers see it, they can compute the store's margin, and suppliers' pricing could leak.
- **`notes` (internal admin notes):** These are meant for staff and may contain sensitive remarks, such as supplier issues or stock problems.
- **Security and stability:** A DTO acts as a whitelist. New database columns are not exposed automatically, and the API contract stays independent of the database schema, so internal renames like `product_name` don't break clients.

---

## Versioning

### Version 1 vs. Version 2 responses

**`GET /api/v1/products/42`** (uses `name`)

```json
{ "id": 42, "name": "School Lanyard", "price_centavos": 8900 }
```

**`GET /api/v2/products/42`** (uses `title`)

```json
{ "id": 42, "title": "School Lanyard", "price_centavos": 8900 }
```

### Chosen approach: URI-based versioning

I chose **URI-based versioning** (`/api/v1/...`, `/api/v2/...`).

- It is visible and easy to understand, test, and share (a browser or cURL can call it directly).
- It is easy to cache and route, and to document separately.
- Header-based versioning is cleaner in theory but harder for students and tools to debug.

**Standard requests:**

```http
GET /api/v1/products/42 HTTP/1.1
Host: lvcc.campus.store
Accept: application/json
```

```http
GET /api/v2/products/42 HTTP/1.1
Host: lvcc.campus.store
Accept: application/json
```

### How version 1 clients keep working

- Old clients keep calling `/api/v1/...`, and **v1's contract is frozen** (`name` stays `name`).
- Both versions read the same `product_name` column. Each version has its own DTO / resource class: `ProductV1Resource` maps `product_name` to `name`, and `ProductV2Resource` maps it to `title`.
- The `title` field is added only in v2, so no existing client breaks.
- v1 responses can include `Deprecation` and `Sunset` headers to announce a retirement date and give clients time to migrate.
- Only breaking changes require a new version. Non-breaking additions can be made within a version.
