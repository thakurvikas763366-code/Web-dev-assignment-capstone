# E-Commerce and Inventory Management System (Backend)

Capstone Project - Node.js, Express.js, MongoDB, Mongoose, JWT, bcrypt, EventEmitter, and Socket.IO.

This REST API enables **customers** to explore products, maintain a shopping cart, and submit orders. **Sellers** can handle their own product listings, inventory, and customer orders, while **administrators** oversee users, categories, all orders, and platform reports.

The payment process is **simulated**. No actual payment gateway or card information is involved, and every order includes `payment.simulated = true`.

## Features
- Authentication through JWT **access and refresh tokens**, including refresh-token rotation, database-side token hashing, reuse detection, and logout
- Password protection with **bcrypt** and route restrictions using **RBAC** for `customer`, `seller`, and `admin` roles
- Product and category management, text search, filtering, sorting, pagination, and soft deletion
- Shopping cart calculations performed and validated by the server
- Checkout splitting a cart into **separate orders for each seller** while preserving item prices
- **Atomic stock reservations** that prevent overselling and negative inventory, with automatic rollback and stock restoration after cancellation
- An order lifecycle state machine with permissions for different roles
- **Three MongoDB aggregation pipelines** for sales, best-selling products, and inventory summaries
- An **EventEmitter** event bus and **Socket.IO** integration for real-time notifications
- Joi request validation, centralized error handling, HTTP request logging, Helmet, CORS, and authentication rate limiting

## Prerequisites
Node.js 18 or newer and MongoDB 6 or newer, running locally or through a MongoDB Atlas connection string.

## Setup
```bash
npm install
cp .env.example .env        # update the secrets and MONGO_URI
npm run seed                # create sample users, categories, and products
npm run dev                 # alternative: npm start -> http://localhost:5000
```

The `.env` file is excluded by Git and must not be committed.

### Environment Variables
| Variable | Description | Default |
|---|---|---|
| PORT | Server port | 5000 |
| MONGO_URI | MongoDB connection string | mongodb://127.0.0.1:27017/ecommerce_inventory |
| JWT_ACCESS_SECRET | Secret used to sign access tokens (required) | - |
| JWT_REFRESH_SECRET | Secret used to sign refresh tokens (required) | - |
| JWT_ACCESS_EXPIRES | Access token lifetime | 15m |
| JWT_REFRESH_EXPIRES | Refresh token lifetime | 7d |
| BCRYPT_SALT_ROUNDS | bcrypt work factor | 10 |
| CLIENT_ORIGIN | CORS and Socket.IO allowed origin | * |

### Seed Accounts
Every seed account uses the password: `Password123`

| Role | Email |
|---|---|
| admin | admin@shop.com |
| seller | seller1@shop.com (electronics, home), seller2@shop.com (fashion, books) |
| customer | customer1@shop.com, customer2@shop.com |

Use `npm run seed:destroy` to erase all seeded and application data.

## Project Structure (MVC)
```
src/
  config/        env.js, db.js
  models/        User, Category, Product, Cart, Order        (Mongoose schemas and indexes)
  controllers/   auth, user, category, product, cart, order, report   (application logic)
  routes/        resource-specific routers                   (URL -> middleware -> controller)
  middleware/    auth (protect/restrictTo), validate, errorHandler, logger
  validators/    schemas.js                                   (Joi schemas)
  listeners/     notificationListeners.js                     (EventEmitter -> Socket.IO)
  utils/         ApiError, asyncHandler, response, pagination, tokens, events, socket
  app.js         Express application
  server.js      HTTP and Socket.IO bootstrap
scripts/         seed.js, generatePostman.js
postman/         Ecommerce_API.postman_collection.json
```

Typical request path: **Route -> Middleware (authentication, RBAC, validation) -> Controller -> Model -> MongoDB -> JSON response**.

## Response Format
```json
{ "success": true, "message": "optional", "data": {}, "meta": { "page": 1, "limit": 10, "total": 42, "totalPages": 5 } }
{ "success": false, "message": "Validation failed", "errors": [ { "field": "price", "message": "price must be a positive number" } ] }
```

## API Documentation
Authentication values in the tables are represented as follows: `-` means public, `Bearer` means any signed-in user, and named roles indicate the roles that are permitted.

Send an access token in this header:

`Authorization: Bearer <accessToken>`

### Auth
| Method | Endpoint | Auth | Body | Notes |
|---|---|---|---|---|
| POST | /api/auth/register | - | name, email, password (8+ characters, letter and number), role (`customer`\|`seller`), phone? | 201; 400 validation; 409 duplicate email |
| POST | /api/auth/login | - | email, password | 200 `{ user, accessToken, refreshToken }`; 401 invalid credentials |
| POST | /api/auth/refresh | - | refreshToken | Replaces the old token; reuse revokes every session |
| POST | /api/auth/logout | - | refreshToken | Invalidates the supplied refresh token |

### Users
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | /api/users/me | Bearer | Retrieve the signed-in user's profile |
| PATCH | /api/users/me | Bearer | Update name, phone, or password |
| GET | /api/users?role=&q=&page=&limit= | admin | Retrieve the user list |
| GET | /api/users/:id | admin | Retrieve one user |
| PATCH | /api/users/:id | admin | Change `role` or `isActive` |
| DELETE | /api/users/:id | admin | Deactivate one user |

### Categories
| Method | Endpoint | Auth |
|---|---|---|
| GET | /api/categories, /api/categories/:id | - |
| POST | /api/categories | admin |
| PATCH / DELETE | /api/categories/:id | admin (deletion is blocked while products remain) |

### Products
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | /api/products | - | Supports `q` text search, `category`, `seller`, `minPrice`, `maxPrice`, `inStock`, `sort` (`newest`,`price_asc`,`price_desc`,`name`,`relevance`), `page`, and `limit` |
| GET | /api/products/:id | - | Retrieve product details |
| GET | /api/products/mine | seller, admin | Retrieve seller inventory (all inventory for an admin) |
| GET | /api/products/low-stock | seller, admin | Retrieve products at or below `lowStockThreshold` |
| POST | /api/products | seller, admin | name, description, price, category, stock, lowStockThreshold?, images? |
| PATCH | /api/products/:id | owner seller, admin | Apply a partial update |
| PATCH | /api/products/:id/stock | owner seller, admin | `{ "delta": 10 }` for atomic restock or adjustment (stock cannot drop below zero) |
| DELETE | /api/products/:id | owner seller, admin | Soft-delete by setting `isActive=false` |

### Cart
Available to customers only.

| Method | Endpoint | Body |
|---|---|---|
| GET | /api/cart | - (server-calculated prices, totals, and availability `issues`) |
| POST | /api/cart/items | productId, quantity |
| PATCH | /api/cart/items/:productId | quantity |
| DELETE | /api/cart/items/:productId | - |
| DELETE | /api/cart | - |

### Orders
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | /api/orders/checkout | customer | Body: `{ shippingAddress: { fullName, phone, line1, line2?, city, state, postalCode, country } }`. Returns 201 with one order per seller; 400 for an empty cart; 409 for a stock conflict |
| GET | /api/orders?status=&from=&to=&page=&limit= | Bearer | Customer: own orders; seller: assigned orders; admin: every order |
| GET | /api/orders/:id | owner customer / assigned seller / admin | Returns 404 when a customer requests another customer's order |
| PATCH | /api/orders/:id/status | customer (cancel only), seller, admin | `{ "status": "confirmed" \| "shipped" \| "delivered" \| "cancelled" }` |

**Status rules:** `pending -> confirmed | cancelled`, `confirmed -> shipped | cancelled`, and `shipped -> delivered`.

The `delivered` and `cancelled` states are final. Any invalid transition returns 409. Cancellation restores stock and changes the simulated payment to `refunded`.

### Reports
Seller reports contain only the seller's data. Admin reports contain all data, with an optional `?seller=` filter.

| Endpoint | Query | Aggregation |
|---|---|---|
| GET /api/reports/sales | from, to, groupBy=`day`\|`month` | Uses `$match`, `$facet`, `$group`, and `$dateToString` to calculate revenue, order count, and average order value over time |
| GET /api/reports/best-sellers | from, to, limit | Uses `$unwind`, `$group`, `$sort`, and `$lookup` to identify the strongest-selling products |
| GET /api/reports/inventory | - | Uses `$group` and `$lookup` to summarize units, stock value, and low-stock totals by category |

### Common Status Codes
200 OK, 201 Created, 400 validation or invalid ID, 401 missing, invalid, or expired token, 403 role or ownership denied,
404 not found, 409 conflict (duplicate value, insufficient stock, or invalid transition), 429 rate limit, and 500 server error.

## Database Design
| Collection | Key fields | Relationships | Indexes |
|---|---|---|---|
| users | name, email, password (hash), role, isActive, refreshTokens[{tokenHash, expiresAt}] | - | unique `email`, `role` |
| categories | name, slug, description | - | unique `name`, unique `slug` |
| products | name, description, price, stock, lowStockThreshold, isActive | category -> categories, seller -> users | text(`name`,`description`), `{category, price}`, `{seller, isActive}`, `{stock}` |
| carts | items[{product, quantity}] | user -> users (1:1), product -> products | unique `user` |
| orders | orderNumber, checkoutId, items[{product, name, price, quantity}], totalAmount, shippingAddress, payment{method, status, simulated}, status, statusHistory[] | customer -> users, seller -> users, product -> products | `{customer, createdAt}`, `{seller, status, createdAt}`, `{status, createdAt}`, unique `orderNumber` |

**Design decisions:** Product information is referenced so price and stock remain consistent in one location. Order items store snapshots of the product name and price, keeping historical records accurate if a product later changes. A cart does not save prices.

## Preventing Overselling
Checkout reserves every item with one atomic database operation:

```js
Product.findOneAndUpdate({ _id, isActive: true, stock: { $gte: qty } }, { $inc: { stock: -qty } }, { new: true })
```

The availability check and stock decrement occur together inside MongoDB. Consequently, two customers racing for the final unit cannot both complete their purchases. If any reservation fails, earlier reservations are restored through compensating rollback and no order is created.

Cancellation first performs a conditional status update, preventing repeated cancellation requests from restoring stock more than once.

MongoDB multi-document transactions could provide a comparable result, but they require a replica set. This implementation is designed to run against a standalone MongoDB instance.

## EventEmitter and Socket.IO
**EventEmitter** (`src/utils/events.js`) separates core business operations from side effects. Controllers publish events, and listeners respond to them.

| Event | Emitted when | Listener action |
|---|---|---|
| `order.placed` | Checkout creates an order | Notify the seller, customer, and administrators |
| `order.statusChanged` | An order changes status | Notify the customer and seller |
| `stock.low` | Stock reaches or drops below the configured threshold during checkout, update, or adjustment | Notify the seller and administrators |

**Socket.IO** authenticates by using the access token:

```js
const socket = io('http://localhost:5000', { auth: { token: accessToken } });
socket.on('order:new', console.log);
```

Connections without a valid access token are refused. Each socket automatically joins `user:<id>` and `role:<role>` rooms.

| Server -> client event | Receiver | Payload |
|---|---|---|
| `connected` | the socket | `{ userId, role }` |
| `order:placed` | customer | `{ message, orderId, orderNumber, totalAmount, itemCount, status }` |
| `order:new` | seller and administrators | same as above |
| `order:status` | customer and seller | `{ orderId, orderNumber, previousStatus, status, message }` |
| `stock:low` | seller and administrators | `{ productId, name, stock, threshold, message }` |

## Testing with Postman
1. Run `npm run seed`, followed by `npm run dev`.
2. Import `postman/Ecommerce_API.postman_collection.json`, or regenerate it with `npm run postman`.
3. Open the Collection Runner and execute folders in the supplied order. Tokens and IDs are saved in collection variables.
4. After completing a run, execute `npm run seed` again to restore fresh data before rerunning the collection.

The tests cover successful requests, validation errors, malformed IDs, 404 responses, missing tokens (401), unauthorized roles (403), duplicate emails (409), ownership violations, insufficient stock, invalid status transitions, unavailable items, and report access restrictions.

To observe real-time events, connect a Socket.IO client as `seller1@shop.com`, then submit a checkout request as a customer.

## Security Notes
The project uses bcrypt password hashing, short-lived access tokens, SHA-256-only refresh-token storage, refresh-token rotation, generic login failures, backend role and ownership checks, Joi `stripUnknown` protection against mass assignment, Helmet, authentication route limits, and environment-only secrets.
