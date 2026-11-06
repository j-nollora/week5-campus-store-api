# **WEEK 5: Campus Store API**
- **Name:** Janelyn G. Nollora
- **Section:** ACT2
- **Activity Title:** Design a Campus Store API

> **Scenario:** A campus store has thousands of products. Students may retry order requests after connection failures, and two students may attempt to purchase the last item.

---
## Endpoints & Pagination

### Retrieving one product
``` https
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
    "product_id": 67,
    "quantity": 1
}
```

### Pagination choice: page-based (offset)
I chose **page-based pagination**. Using page numbers make it easy for students to navigate back and forth through catalog lists and it naturally map to traditional frontend pagination UI elements like the pagination bar.

---
