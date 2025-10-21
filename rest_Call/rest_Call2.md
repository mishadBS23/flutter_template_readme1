# REST API Basics - Complete Guide

## What is REST API?

**REST** (Representational State Transfer) is an architectural style for designing networked applications. It uses HTTP requests to access and manipulate data.

---

## HTTP Methods (CRUD Operations)

### 1. GET - Read/Retrieve Data
- **Purpose**: Fetch data from server
- **Has Body**: ❌ No
- **Idempotent**: ✅ Yes (same request = same result)
- **Safe**: ✅ Yes (doesn't change server state)

**Example**:
```http
GET /api/users
GET /api/users/123
GET /api/products?category=electronics&page=1
```

**When to Use**:
- Fetch a list of items
- Get details of a single item
- Search/filter data

---

### 2. POST - Create New Data
- **Purpose**: Create new resources
- **Has Body**: ✅ Yes
- **Idempotent**: ❌ No (multiple requests = multiple resources)
- **Safe**: ❌ No (changes server state)

**Example**:
```http
POST /api/users
Body: {
  "name": "John Doe",
  "email": "john@example.com"
}
```

**When to Use**:
- Create new user
- Submit a form
- Upload data
- Login/Authentication

---

### 3. PUT - Update/Replace Entire Resource
- **Purpose**: Replace entire resource
- **Has Body**: ✅ Yes
- **Idempotent**: ✅ Yes
- **Safe**: ❌ No

**Example**:
```http
PUT /api/users/123
Body: {
  "id": 123,
  "name": "John Updated",
  "email": "john.new@example.com",
  "age": 30,
  "address": "123 Street"
}
```

**When to Use**:
- Replace all fields of a resource
- Update complete profile

---

### 4. PATCH - Partial Update
- **Purpose**: Update specific fields
- **Has Body**: ✅ Yes
- **Idempotent**: ✅ Yes
- **Safe**: ❌ No

**Example**:
```http
PATCH /api/users/123
Body: {
  "email": "newemail@example.com"
}
```

**When to Use**:
- Update only name
- Change password
- Update specific fields

---

### 5. DELETE - Remove Resource
- **Purpose**: Delete resources
- **Has Body**: ❌ Usually No
- **Idempotent**: ✅ Yes
- **Safe**: ❌ No

**Example**:
```http
DELETE /api/users/123
DELETE /api/posts/456
```

**When to Use**:
- Remove user account
- Delete a post
- Remove item from cart

---

### 6. HEAD - Get Headers Only
- **Purpose**: Get metadata without body
- **Has Body**: ❌ No
- **Idempotent**: ✅ Yes
- **Safe**: ✅ Yes

**Example**:
```http
HEAD /api/users/123
```

**When to Use**:
- Check if resource exists
- Get file size before download

---

### 7. OPTIONS - Get Allowed Methods
- **Purpose**: Get available HTTP methods
- **Has Body**: ❌ No
- **Idempotent**: ✅ Yes
- **Safe**: ✅ Yes

**Example**:
```http
OPTIONS /api/users
Response: GET, POST, PUT, DELETE
```

**When to Use**:
- CORS preflight requests
- Check what operations are allowed

---

## Parameter Types

### 1. Path Parameters
Embedded in the URL path to identify specific resources.

**Syntax**: `/resource/{id}/subresource/{subId}`

**Examples**:
```http
GET /api/users/123
GET /api/users/123/posts/456
DELETE /api/products/789
```

**When to Use**:
- Identify specific resource (user ID, product ID)
- Required parameters
- Hierarchical relationships

---

### 2. Query Parameters
Appended to URL after `?` for filtering/options.

**Syntax**: `/resource?param1=value1&param2=value2`

**Examples**:
```http
GET /api/users?role=admin&status=active
GET /api/products?category=electronics&price_min=100&price_max=500
GET /api/posts?page=2&limit=10&sort=date
```

**Common Query Parameters**:
- **Pagination**: `page`, `limit`, `offset`
- **Filtering**: `status`, `category`, `type`
- **Sorting**: `sort`, `order`
- **Search**: `search`, `q`, `query`
- **Date Range**: `start_date`, `end_date`

**When to Use**:
- Optional parameters
- Filtering lists
- Pagination
- Searching

---

### 3. Body Parameters
Data sent in request body (JSON, XML, Form Data).

**JSON Example**:
```http
POST /api/users
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30,
  "address": {
    "street": "123 Main St",
    "city": "New York"
  }
}
```

**When to Use**:
- POST/PUT/PATCH requests
- Complex data structures
- Nested objects
- Arrays

---

### 4. Header Parameters
Metadata sent in HTTP headers.

**Examples**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
Accept: application/json
X-API-Key: abc123xyz
```

**Common Headers**:
- **Authorization**: Authentication tokens
- **Content-Type**: Data format (JSON, XML)
- **Accept**: Expected response format
- **User-Agent**: Client information
- **Custom Headers**: `X-Custom-Header`

**When to Use**:
- Authentication
- API keys
- Content negotiation
- Custom metadata

---

### 5. Form Data
Data sent as form fields (multipart/form-data).

**Example**:
```http
POST /api/upload
Content-Type: multipart/form-data

name=John Doe
email=john@example.com
file=<binary data>
```

**When to Use**:
- File uploads
- Traditional form submissions

---

## Request/Response Structure

### Request Structure
```http
METHOD /path/to/resource?query=value HTTP/1.1
Host: api.example.com
Authorization: Bearer token123
Content-Type: application/json

{
  "key": "value"
}
```

### Response Structure
```http
HTTP/1.1 200 OK
Content-Type: application/json
X-RateLimit-Remaining: 99

{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com"
}
```

---

## HTTP Status Codes

### 2xx Success
- **200 OK**: Request succeeded
- **201 Created**: Resource created successfully
- **204 No Content**: Success but no response body

### 3xx Redirection
- **301 Moved Permanently**: Resource moved
- **304 Not Modified**: Cached version is valid

### 4xx Client Errors
- **400 Bad Request**: Invalid request data
- **401 Unauthorized**: Authentication required
- **403 Forbidden**: No permission
- **404 Not Found**: Resource doesn't exist
- **422 Unprocessable Entity**: Validation failed
- **429 Too Many Requests**: Rate limit exceeded

### 5xx Server Errors
- **500 Internal Server Error**: Server crashed
- **502 Bad Gateway**: Gateway/proxy error
- **503 Service Unavailable**: Server overloaded

---

## Real-World Examples

### E-Commerce API

#### Get Products
```http
GET /api/products?category=electronics&price_max=1000&page=1&limit=20
```

#### Create Order
```http
POST /api/orders
Body: {
  "customer_id": 456,
  "items": [
    { "product_id": 123, "quantity": 2 },
    { "product_id": 789, "quantity": 1 }
  ],
  "payment_method": "credit_card"
}
```

#### Update Order Status
```http
PATCH /api/orders/789
Body: {
  "status": "shipped",
  "tracking_number": "ABC123XYZ"
}
```

#### Delete Product
```http
DELETE /api/products/123
```

---

### Social Media API

#### Get Posts
```http
GET /api/users/123/posts?page=1&limit=10&sort=date
```

#### Create Post
```http
POST /api/posts
Body: {
  "title": "My New Post",
  "content": "This is awesome!",
  "tags": ["tech", "coding"]
}
```

#### Like Post
```http
POST /api/posts/456/like
```

#### Delete Comment
```http
DELETE /api/posts/456/comments/789
```

---

### File Upload API

#### Upload Single File
```http
POST /api/upload
Content-Type: multipart/form-data

file=<binary>
description=Profile picture
```

#### Upload Multiple Files
```http
POST /api/upload/multiple
Content-Type: multipart/form-data

files[]=<binary1>
files[]=<binary2>
folder=documents
```

---

## Best Practices

### 1. URL Structure
```
✅ Good:
/api/users
/api/users/123
/api/users/123/posts
/api/products?category=electronics

❌ Bad:
/api/getUsers
/api/user_list
/api/deleteUser?id=123  (use DELETE method instead)
```

### 2. Use Proper HTTP Methods
```
✅ GET /api/users/123        (Fetch user)
✅ POST /api/users           (Create user)
✅ PATCH /api/users/123      (Update user)
✅ DELETE /api/users/123     (Delete user)

❌ POST /api/getUser         (Wrong method)
❌ GET /api/deleteUser/123   (Should use DELETE)
```

### 3. Use Query Parameters for Filtering
```
✅ GET /api/products?category=electronics&price_min=100
❌ GET /api/products/electronics/price/100
```

### 4. Return Appropriate Status Codes
```
✅ 200 for successful GET
✅ 201 for successful POST (created)
✅ 204 for successful DELETE
✅ 400 for validation errors
✅ 404 for not found

❌ Always returning 200
```

### 5. Versioning
```
✅ /api/v1/users
✅ /api/v2/users

❌ /api/users (no version)
```

---

## Common Patterns

### Pagination
```http
GET /api/posts?page=2&limit=20
GET /api/posts?offset=40&limit=20
```

**Response**:
```json
{
  "data": [...],
  "pagination": {
    "current_page": 2,
    "total_pages": 10,
    "total_items": 200,
    "per_page": 20
  }
}
```

### Filtering
```http
GET /api/users?role=admin&status=active&created_after=2024-01-01
```

### Sorting
```http
GET /api/products?sort=price&order=asc
GET /api/products?sort=-price  (descending)
```

### Searching
```http
GET /api/users?search=john
GET /api/products?q=laptop&category=electronics
```

### Nested Resources
```http
GET /api/users/123/posts
GET /api/users/123/posts/456/comments
POST /api/users/123/posts
```

---

## Security Best Practices

### 1. Authentication
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
X-API-Key: your-api-key-here
```

### 2. HTTPS Only
```
✅ https://api.example.com
❌ http://api.example.com
```

### 3. Rate Limiting
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1635724800
```

### 4. Input Validation
Always validate:
- Email formats
- Required fields
- Data types
- String lengths
- Allowed values

---

## Quick Reference Table

| Method | Purpose | Body | Idempotent | Safe |
|--------|---------|------|------------|------|
| GET | Read | ❌ | ✅ | ✅ |
| POST | Create | ✅ | ❌ | ❌ |
| PUT | Replace | ✅ | ✅ | ❌ |
| PATCH | Update | ✅ | ✅ | ❌ |
| DELETE | Remove | ❌ | ✅ | ❌ |
| HEAD | Headers | ❌ | ✅ | ✅ |
| OPTIONS | Methods | ❌ | ✅ | ✅ |

---

## Learning Resources

1. **Practice APIs**:
   - JSONPlaceholder: https://jsonplaceholder.typicode.com
   - ReqRes: https://reqres.in
   - The Cat API: https://thecatapi.com

2. **Tools**:
   - Postman (API testing)
   - cURL (command line)
   - Swagger/OpenAPI (documentation)

3. **Test Example**:
```bash
# GET request
curl https://jsonplaceholder.typicode.com/users

# POST request
curl -X POST https://jsonplaceholder.typicode.com/posts \
  -H "Content-Type: application/json" \
  -d '{"title":"Test","body":"Content","userId":1}'
```

---

This guide covers 90% of REST API concepts you'll encounter in real projects!
