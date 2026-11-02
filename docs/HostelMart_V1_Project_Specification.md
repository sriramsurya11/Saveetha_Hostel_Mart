# HostelMart — V1 Product & Engineering Specification

> **Purpose:** This document is the source of truth for building V1 of HostelMart with GitHub Copilot. Read it before implementing features. Do not generate the entire application in one pass.
>
> **Status:** V1 planning / implementation
> **Initial audience:** Saveetha Engineering College hostel students
> **Initial stores:** 3–4 student-run snack stores
> **Expected growth:** Start small; design for around 100 stores without premature distributed infrastructure.

---

## 1. Product overview

HostelMart is a mobile-first hostel snack marketplace. Students browse independent hostel snack stores, add items from one store to a cart, submit their name and delivery location, and place an order. The relevant seller receives the order in their dashboard and delivers it.

The product replaces repeatedly updated WhatsApp stock messages with live inventory and structured orders.

### V1 goals

- Students can browse stores and products on mobile.
- Each seller can manage only their own store, products, stock, and orders.
- Students can place orders with name and delivery details.
- Orders are saved reliably in MongoDB.
- Stock cannot go below zero or be oversold by concurrent requests.
- Sellers receive live order notifications while online.
- Sellers can retrieve pending orders after reconnecting or refreshing.
- Students see honest order statuses.
- UI works on small mobile screens and has no dead buttons.
- The first release can be operated by 3–4 sellers.

### Out of scope for V1

- Online payment gateway and payment processing.
- Chat/messaging between seller and customer.
- Redis, Kafka, microservices, Kubernetes, or load balancing.
- Coupons, loyalty programs, ratings, recommendations.
- Multi-hostel marketplace (unless needed for initial data modeling).
- Automatic delivery assignment or route optimization.
- Push notifications as the only notification mechanism.

**Payment recommendation:** V1 should support cash/UPI paid directly to the seller on delivery. A seller QR can be displayed later, but a QR screenshot is not proof of payment. Do not mark a payment as verified based only on a customer-uploaded screenshot.

---

## 2. User roles and permissions

### Student / Customer

- Browse active stores and available products.
- Search/filter stores and products.
- Add products from a single store to the cart.
- Edit cart quantities and remove items.
- Set and edit delivery details.
- Place an order.
- View their order confirmation and status.
- View their own order history if customer identity/session is implemented.

### Store Owner / Seller

- Log in using an approved account.
- Access only their assigned store.
- Create, edit, hide, and delete/archive their own products.
- Set price and stock.
- Toggle store open/closed.
- Toggle room delivery availability.
- View their store’s orders.
- Accept, reject, prepare, dispatch, and complete orders according to allowed transitions.
- See customer name, hostel, block, room number, AC status, order items, quantities, and order notes.
- Cannot view or modify another seller’s private orders or inventory.

### Super Admin

- Create/approve seller accounts and stores.
- Assign a seller to a store.
- Suspend/reactivate a store or seller.
- View stores and platform health.
- Help resolve issues.
- Avoid viewing customer data unless necessary for operations.

### Authorization principle

Every seller operation must be authorized on the backend. Never trust a `storeId`, `ownerId`, price, stock, role, or payment status supplied by the browser.

---

## 3. User journeys

### First-time customer

1. Open HostelMart.
2. Browse stores without being forced to register immediately.
3. Add products from one store.
4. At checkout, enter or confirm:
   - Full name
   - Hostel
   - Block (e.g. NBH)
   - Room number (e.g. 216)
   - AC room: yes/no
   - Optional delivery note
5. Review order and total.
6. Place order.
7. See confirmation only after the backend has persisted the order.
8. Track order status.

### Returning customer

- Reuse delivery details from their profile/session.
- Show the saved delivery details at checkout.
- Let them edit them.
- Copy a **snapshot** of the details into each order so later profile changes do not alter old orders.

### Seller

1. Log in.
2. Dashboard loads their store and pending orders from the API.
3. New orders appear via Socket.IO when connected.
4. Seller accepts/rejects and updates order status.
5. Seller can refresh/reconnect and still see every pending order.
6. Seller changes stock or delivery availability.
7. Customers see updated availability after fetching/realtime update.

---

## 4. Important business rules

1. A cart/order in V1 belongs to **one store only**. If the customer adds an item from another store, ask them to clear/checkout the current cart or switch stores. Do not silently mix sellers.
2. Room number is **not unique**. Many students may share room 216.
3. Room identity is a combination of hostel + block + room number + AC status, but it is still not a unique student identity.
4. Store delivery availability is separate from product availability.
5. A seller may have stock while delivery is paused. Show whether pickup is available only if the seller explicitly enables it.
6. Prices and totals must be calculated on the backend using current database prices. Never trust frontend totals.
7. Product stock must be checked and reserved atomically during order creation.
8. Stock cannot be negative.
9. Order creation must be idempotent: duplicate submissions/retries with the same idempotency key must not create duplicate orders.
10. An order must store snapshots of product name, unit price, quantity, store name, and delivery address at purchase time.
11. A product name is not globally unique; use product IDs and store IDs.
12. Do not tell the customer “out for delivery” until the seller actually marks it so.
13. Do not delete historical order data when a product/store is archived.
14. Order cancellation and stock restoration must be controlled, transactional, and idempotent.
15. Do not accept an order if the store is closed, suspended, or unable to fulfill the selected delivery method.

---

## 5. Recommended technology stack

### Frontend

- React + Vite
- React Router
- CSS Modules or organized plain CSS
- React Hook Form (optional) + Zod validation
- Fetch or Axios (choose one and use consistently)
- Socket.IO client
- Vitest + React Testing Library
- Playwright for end-to-end tests

### Backend

- Node.js + Express
- MongoDB Atlas + Mongoose
- Zod for request validation
- Socket.IO
- bcrypt or Argon2id for password hashing
- JWT or secure session-based authentication (choose and document one)
- Helmet
- CORS with explicit allowed origins
- Rate limiting
- Pino/Winston or structured logging
- Vitest/Jest + Supertest

### Infrastructure

- GitHub repository and GitHub Actions
- Frontend hosting: any suitable static host
- Backend hosting: Node-compatible host
- MongoDB Atlas Free tier for initial MVP, subject to current limits
- Product images: initially curated image URLs or a storage service; do not store image binary data in MongoDB documents.

### Architecture

Use a **modular monolith**: one Express application, one database, separated modules. Do not split into microservices for V1.

Suggested backend layers:

- Routes: URL and middleware wiring
- Controllers: HTTP request/response
- Services: business logic and transaction orchestration
- Models: Mongoose schema/database interaction
- Validators: Zod schemas
- Middleware: auth, role checks, error handling
- Socket layer: authenticated event connections and store rooms

Do not put complex order logic directly in route handlers.

---

## 6. Suggested repository layout

```text
Saveetha_Hostel_Mart/
├── client/
│   ├── src/
│   │   ├── app/
│   │   ├── components/
│   │   ├── features/
│   │   │   ├── auth/
│   │   │   ├── stores/
│   │   │   ├── products/
│   │   │   ├── cart/
│   │   │   ├── checkout/
│   │   │   ├── orders/
│   │   │   └── seller/
│   │   ├── pages/
│   │   ├── services/
│   │   ├── hooks/
│   │   ├── styles/
│   │   └── main.jsx
│   └── package.json
├── server/
│   ├── src/
│   │   ├── config/
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   ├── users/
│   │   │   ├── stores/
│   │   │   ├── products/
│   │   │   └── orders/
│   │   ├── middleware/
│   │   ├── sockets/
│   │   ├── utils/
│   │   ├── app.js
│   │   └── server.js
│   ├── tests/
│   └── package.json
├── docs/
│   ├── requirements.md
│   ├── architecture.md
│   ├── database-schema.md
│   ├── api-contract.md
│   ├── security.md
│   └── testing-plan.md
├── .github/
│   ├── copilot-instructions.md
│   └── workflows/
├── .env.example
├── .gitignore
└── README.md
```

Keep the actual folder layout consistent with the existing repo. Do not rename or restructure the whole repository without asking.

---

## 7. Data model proposal

Use MongoDB ObjectIds. Validate all IDs and inputs.

### User

- `_id`
- `name`
- `email` or `username` (seller/admin login; do not require customer email unless needed)
- `passwordHash`
- `role`: `CUSTOMER`, `SELLER`, `SUPER_ADMIN`
- `storeId` for seller account (nullable for non-sellers)
- `isActive`
- timestamps

Do not allow public self-registration as `SELLER` or `SUPER_ADMIN`. Seller accounts should be created/approved by the super admin.

### Store

- `_id`
- `name`
- `slug`
- `description`
- `hostel`
- `ownerId`
- `imageUrl`
- `isOpen`
- `isActive`
- `deliveryAvailable`
- `pickupAvailable`
- `deliveryNote`
- timestamps

### Product

- `_id`
- `storeId`
- `name`
- `description`
- `category` (e.g. BISCUITS, NAMKEENS, DRINKS, OTHER)
- `imageUrl`
- `price` (integer paise preferred, e.g. ₹25 = 2500)
- `stock` (nonnegative integer)
- `isAvailable`
- timestamps

Use integer paise for money to avoid floating-point arithmetic errors. Format into rupees in the UI.

### Customer profile (optional for guest V1)

- `_id` or linked `userId`
- `name`
- `hostel`
- `block`
- `roomNumber`
- `isAC`
- `deliveryNote`
- timestamps

If guest checkout is used, store the customer details safely in the order. Avoid relying solely on localStorage for order ownership or identity.

### Order

- `_id`
- `orderNumber` (human-readable, non-secret)
- `storeId`
- `storeNameSnapshot`
- `customerId` (nullable for guest)
- `customerName`
- `deliveryAddressSnapshot`:
  - hostel
  - block
  - roomNumber
  - isAC
  - deliveryNote
- `items[]`:
  - productId
  - productNameSnapshot
  - unitPricePaise
  - quantity
  - lineTotalPaise
- `subtotalPaise`
- `totalPaise`
- `paymentMethod`: `CASH`, `UPI_ON_DELIVERY`
- `paymentStatus`: `UNPAID`, `PAID`, `REFUNDED` (V1 normally starts UNPAID)
- `status`: `PLACED`, `ACCEPTED`, `PREPARING`, `OUT_FOR_DELIVERY`, `DELIVERED`, `REJECTED`, `CANCELLED`
- `idempotencyKey`
- timestamps
- cancellation/rejection reason where applicable

Use an order-item snapshot so old orders remain correct if the product is renamed or repriced.

### Indexes to consider

Create and verify indexes based on actual query patterns:

- Product: `{ storeId: 1, isAvailable: 1, category: 1 }`
- Order: `{ storeId: 1, status: 1, createdAt: -1 }`
- Order: `{ customerId: 1, createdAt: -1 }` if customer history exists
- Store: `{ isActive: 1, isOpen: 1 }`
- Unique idempotency constraint scoped appropriately (e.g. customer/session + key)

Avoid creating indexes blindly; explain each index and test the relevant query.

---

## 8. API contract (initial proposal)

Use `/api/v1`. Return consistent JSON and meaningful HTTP status codes.

### Public

- `GET /api/v1/stores` — list active stores
- `GET /api/v1/stores/:storeId` — store details
- `GET /api/v1/stores/:storeId/products` — available products
- `GET /api/v1/products/:productId` — product details

### Authentication

- `POST /api/v1/auth/login`
- `POST /api/v1/auth/logout` (if session/cookie design supports it)
- `GET /api/v1/auth/me`

### Customer orders

- `POST /api/v1/orders` — create order; requires validation and idempotency
- `GET /api/v1/orders/:orderId` — only the owner/customer or authorized seller/admin can view
- `GET /api/v1/me/orders` — customer history if authenticated

### Seller

- `GET /api/v1/seller/store`
- `PATCH /api/v1/seller/store`
- `GET /api/v1/seller/products`
- `POST /api/v1/seller/products`
- `PATCH /api/v1/seller/products/:productId`
- `DELETE /api/v1/seller/products/:productId` — preferably archive/hide rather than hard delete
- `GET /api/v1/seller/orders`
- `PATCH /api/v1/seller/orders/:orderId/status`
- `PATCH /api/v1/seller/products/:productId/stock`

### Super Admin

- `POST /api/v1/admin/stores`
- `PATCH /api/v1/admin/stores/:storeId`
- `POST /api/v1/admin/sellers`
- `PATCH /api/v1/admin/sellers/:sellerId/status`
- `GET /api/v1/admin/stores`

The backend must enforce roles and store ownership for every protected route. Do not rely on hiding buttons in React.

---

## 9. Inventory and concurrency requirements (critical)

### Single product

Use an atomic conditional update that checks `stock >= requestedQuantity` and decrements stock in the same MongoDB operation. Do not do “read stock, then later write stock” as separate unprotected operations.

Conceptual example:

```js
const updatedProduct = await Product.findOneAndUpdate(
  {
    _id: productId,
    storeId,
    isAvailable: true,
    stock: { $gte: quantity }
  },
  { $inc: { stock: -quantity } },
  { new: true }
);

if (!updatedProduct) {
  // Product missing, unavailable, or insufficient stock.
  // Return a clear conflict/unavailable response.
}
```

This example is **not a complete order implementation**. Order creation, idempotency, error recovery, and cancellation still need to be implemented.

### Multi-product order

- Use a MongoDB transaction.
- Validate all products and current prices inside the transaction.
- Reserve stock for each item.
- Create the order in the same transaction.
- If any product is unavailable, abort the entire transaction.
- Do not partially create an order unless the product explicitly defines partial fulfillment.
- Ensure the MongoDB deployment supports transactions (Atlas replica set supports them; confirm deployment configuration).

### Cancellation

- Define who can cancel and at which statuses.
- Restore inventory only once.
- Prevent repeated cancellation requests from incrementing stock multiple times.
- Use transaction/state transition safeguards.

### Idempotency

- Customer checkout creates an idempotency key.
- Persist the key with the order and enforce uniqueness within the relevant customer/session scope.
- If a request is retried with the same key, return the existing order result.
- Do not generate a new key for every retry of the same checkout attempt.

### Concurrency acceptance test

Seed a product with stock = 1. Run many simultaneous order attempts for quantity 1.

Expected:
- Exactly one order succeeds.
- All other requests receive a clear unavailable/insufficient-stock response.
- Stock is exactly 0, never negative.
- No duplicate order is created for retries.
- No successful order exists without its corresponding stock reservation.

---

## 10. Order statuses and transitions

Recommended transitions:

- `PLACED` → `ACCEPTED` or `REJECTED`
- `ACCEPTED` → `PREPARING` or `CANCELLED` (according to policy)
- `PREPARING` → `OUT_FOR_DELIVERY` or `CANCELLED` (according to policy)
- `OUT_FOR_DELIVERY` → `DELIVERED`
- `REJECTED`, `CANCELLED`, `DELIVERED` are terminal for V1

The backend must validate every transition. A seller cannot jump directly from `PLACED` to `DELIVERED`.

Show the customer:
- Order placed
- Seller accepted
- Preparing
- Out for delivery
- Delivered

Only display “Out for delivery” after seller action.

---

## 11. Real-time notifications

Use Socket.IO for live seller notifications, but treat MongoDB as the source of truth.

### Event

- `order:new` — notify the seller's authenticated store room
- `order:status_updated` — notify the customer/session if supported
- `product:stock_updated` — optional V1 event

### Requirements

- Authenticate Socket.IO connections.
- Authorize store room membership on the server.
- Do not allow clients to join arbitrary seller rooms.
- Emit to the correct store only.
- Do not include secrets or unnecessary customer data in events.
- Seller dashboard must fetch pending orders from the API on initial load.
- On reconnect, fetch orders again.
- Socket event is a notification to refresh/update state, not the only copy of the order.
- If seller is offline, the order remains saved and appears when they return.

### Reliability consideration

A database write followed by a socket emit can fail between those operations. V1 should at minimum persist first and make seller dashboard fetch persisted orders. For stronger delivery guarantees, add a transactional outbox in a later iteration.

---

## 12. Customer delivery details and privacy

Checkout details:

- Name
- Hostel
- Block
- Room number
- AC/non-AC
- Optional delivery note

Rules:

- Room number is not unique.
- Never use room number as a primary key or account identity.
- Validate lengths and allowed characters.
- Do not expose delivery details to other customers.
- Only the relevant store owner and authorized admin should see delivery details needed to fulfill the order.
- Avoid logging full customer room details in application logs.
- Use HTTPS in production.
- Do not store sensitive delivery information in analytics.
- If the user changes profile details, old order snapshots remain unchanged.

Customer setup should not block browsing. Collect/confirm delivery details at checkout, and optionally save them for next time.

---

## 13. Security checklist

- Hash passwords using bcrypt or Argon2id.
- Use HTTPS in production.
- Use secure, HttpOnly, SameSite cookies if using cookie-based auth; protect against CSRF where needed.
- If using JWT, define expiry and refresh/revocation strategy; do not put secrets or personal data in the token.
- Never store secrets in Git.
- Keep `.env` ignored.
- Add `.env.example` with placeholder values only.
- Validate every request body, query, and route parameter.
- Add Helmet.
- Configure CORS with exact frontend origins, not wildcard with credentials.
- Rate-limit login and order creation.
- Enforce authorization in backend middleware/services.
- Never trust price, stock, role, store owner, or payment status from frontend.
- Avoid mass assignment: whitelist fields allowed in each update.
- Return generic login failure messages.
- Do not leak stack traces or database details in production.
- Use a dedicated database user with least privilege.
- Do not expose MongoDB to all IPs unless unavoidable; use provider-recommended network controls.
- Do not commit `.env`, credentials, tokens, or real student data.
- Seed only fake data in development.

---

## 14. Mobile-first UI and accessibility

The application is primarily used on phones.

### Customer pages

- Home/store listing
- Store details/product catalog
- Cart
- Checkout
- Order confirmation/tracking
- Profile/order history (if included)

### Seller pages

- Login
- Dashboard/pending orders
- Product management
- Stock management
- Store settings (open/closed, delivery status)
- Order details/status updates

### Super Admin pages

- Login
- Stores/sellers management
- Store approval/suspension

### UI requirements

- Responsive from 320px width.
- No horizontal overflow.
- Product cards use responsive CSS Grid.
- Buttons and controls are touch-friendly (target roughly 44px minimum).
- Show loading state during network requests.
- Show empty states for no stores/products/orders.
- Show clear error messages and retry actions.
- Disable submit while a request is in progress, but still rely on backend idempotency.
- Prevent duplicate clicks.
- Show success only after server response.
- Keep cart/store identity clear.
- Use semantic buttons and labels.
- Keyboard navigation and visible focus.
- Maintain reasonable color contrast.
- Don't use fake static data in production views.
- Every visible button must perform a real action or be clearly disabled/not shown.

### Test screen widths

- 320px
- 360px
- 375px
- 390px
- 430px
- Tablet and desktop widths

---

## 15. Testing strategy

### Unit tests

- Price/total calculations
- Order state transitions
- Stock validation
- Delivery validation
- Permission checks

### API/integration tests

- Login and authentication
- Seller can manage own store
- Seller cannot manage another store
- Customer order creation
- Store closed rejects orders
- Out-of-stock rejects orders
- Price tampering is ignored/rejected
- Multi-item transaction rollback
- Cancellation restores stock exactly once
- Idempotent checkout

### E2E tests (Playwright)

1. Student opens store listing.
2. Student selects a store.
3. Student adds products.
4. Student enters delivery details.
5. Student submits order.
6. Student sees confirmation.
7. Seller sees the order.
8. Seller accepts it.
9. Student sees updated status.
10. Seller marks delivered.
11. Student sees delivered status.

### Failure tests

- Server returns 500.
- Network timeout.
- Double click checkout.
- Refresh after checkout.
- Seller offline.
- Socket disconnect/reconnect.
- Product stock changes while customer has cart open.
- Two customers purchase the last item simultaneously.
- Invalid or expired login.
- Customer attempts to access another customer's order.
- Seller attempts to access another seller's order.

### CI

GitHub Actions should run:
- Lint (if configured)
- Unit tests
- Backend integration tests
- Frontend tests
- Build

Do not merge/deploy a change if required checks fail.

---

## 16. Free-tier hosting and performance

V1 can start with free-tier services, but free tiers have limits and may sleep, throttle, or change. Verify current provider limits before launch.

### Performance principles

- Keep the backend stateless so it can scale later.
- Use MongoDB indexes for common queries.
- Paginate seller order history.
- Return only necessary fields.
- Do not fetch every order/product for every request.
- Avoid N+1 database queries.
- Keep product images outside MongoDB.
- Compress HTTP responses if useful.
- Cache only data that is safe and useful to cache.
- Don't cache stock or order status in a way that causes stale checkout decisions.
- Use connection pooling through the MongoDB driver/Mongoose.
- Reuse one MongoDB connection per backend process; do not connect for every request.
- Add a lightweight health endpoint.
- Measure before optimizing.

### Important: “10–20 users” clarification

10–20 students using the website at the same time is not automatically a large workload. The important factor is how many requests they send concurrently and what those requests do.

A free database may be sufficient for an MVP with a small catalog and low order volume, but it is not a performance guarantee. Latency can be affected by:
- Free-tier resource limits
- Database region and network distance
- Backend cold starts/sleeping
- Slow queries or missing indexes
- Connection setup
- Hosting resource limits

Do not promise instant response. Measure actual response times and show a clear loading state. Keep order confirmation dependent on a successful database commit.

### Load testing

Before launch, use a tool such as k6 or Artillery against a staging environment with fake data.

Test:
- Store/product reads
- Checkout/order creation
- Seller order dashboard
- Concurrent attempts for the last stock item

Set reasonable goals based on actual measurements, not assumptions. Avoid running load tests against free-tier services beyond their limits or against production without permission.

---

## 17. Deployment and environment variables

Use separate environments:

- Local development
- Staging/test
- Production

Example `.env.example` (placeholders only):

```env
NODE_ENV=development
PORT=5000
MONGODB_URI=
CLIENT_URL=http://localhost:5173
JWT_SECRET=
JWT_EXPIRES_IN=
```

If using cookie sessions, replace/add the appropriate session configuration.

Never commit real `.env` files.

### Deployment checklist

- MongoDB Atlas database and least-privilege database user configured.
- Backend environment variables configured in hosting dashboard.
- Frontend API base URL configured.
- CORS matches frontend URL.
- HTTPS enabled.
- Production error handling enabled.
- Database indexes created.
- Seed/admin creation is secure (no public default admin password).
- Seller accounts approved and credentials delivered securely.
- Smoke test checkout after deployment.
- Test seller reconnect and pending order retrieval.
- Verify backups/export strategy and understand free-tier limits.
- Document how to rollback a deployment.

---

## 18. GitHub and Copilot workflow

### Git

- Use a private repository while developing with real customer data or credentials.
- Never commit student personal information or secrets.
- Commit small, working changes.
- Use branches for features when useful.
- Commit messages should describe the change.
- Review Copilot-generated code before accepting it.

Example branch names:
- `feature/store-listing`
- `feature/seller-auth`
- `feature/inventory`
- `feature/checkout`
- `feature/realtime-orders`
- `test/concurrent-inventory`

### Copilot development rules

1. Read this specification before coding.
2. Inspect existing code before editing.
3. Work on one feature at a time.
4. Explain planned file changes.
5. Do not rewrite unrelated code.
6. Do not invent API routes or schema fields; update docs when decisions change.
7. Do not hardcode production data.
8. Add tests with business logic.
9. Run tests and report actual results.
10. Never claim tests passed without executing them.
11. Ask before adding major dependencies or changing architecture.
12. Keep code readable and suitable for a student developer to understand.
13. Explain nontrivial concurrency/security decisions.
14. Do not mark a feature complete until acceptance criteria are met.

### Recommended Copilot task sequence

1. Inspect repo; create a plan; identify missing documentation.
2. Scaffold frontend and backend.
3. Add linting, environment handling, error handling, and health endpoint.
4. Implement User/Store models and seller authentication.
5. Implement seller/store authorization tests.
6. Implement product CRUD and inventory.
7. Implement customer store/product browsing.
8. Implement cart UI.
9. Implement transactional/idempotent order creation.
10. Implement seller order dashboard.
11. Implement Socket.IO notifications and reconnect recovery.
12. Implement order status transitions.
13. Add mobile responsive polish.
14. Add E2E tests.
15. Add CI and staging deployment.
16. Run concurrency/security checks.
17. Pilot with 2–3 sellers and a small group of students.
18. Fix feedback before wider rollout.

---

## 19. Definition of done for V1

V1 is ready for a small pilot only when:

- [ ] Customer can browse real stores/products from the backend.
- [ ] Seller login works.
- [ ] Super admin can create/approve sellers and stores.
- [ ] Sellers cannot access other stores' private data.
- [ ] Product prices and stock are controlled by backend.
- [ ] Student can place an order with delivery details.
- [ ] Order is persisted before confirmation.
- [ ] Stock never goes negative.
- [ ] Duplicate checkout requests do not create duplicate orders.
- [ ] Multi-product checkout is atomic.
- [ ] Seller receives live notification while online.
- [ ] Seller can retrieve pending orders after reconnect.
- [ ] Order status transitions are validated.
- [ ] Cancellation does not restore stock twice.
- [ ] UI works at small mobile widths.
- [ ] No visible button is inert.
- [ ] Loading, error, empty, and success states exist.
- [ ] Automated tests pass.
- [ ] Secrets are not in Git.
- [ ] Production smoke test passes.
- [ ] Sellers understand how to operate the dashboard.
- [ ] A small pilot has been completed and feedback reviewed.

---

## 20. Copilot's first task

Do not implement the entire project yet. Start by asking Copilot:

> Read `docs/requirements.md` (this specification) and inspect the repository. Do not write application code yet. Summarize the product, identify missing decisions, propose a V1 architecture, and create a phased implementation plan. Flag security, inventory concurrency, and order reliability risks. Ask me before making architectural changes.

After reviewing the plan, start with project scaffolding and authentication.

---

## Final engineering principle

HostelMart should be simple to operate, reliable under concurrent orders, secure between sellers, and easy to use on mobile.

Prioritize correctness and observability over adding infrastructure. A well-tested modular monolith is a better V1 than a distributed system that is difficult to debug.
