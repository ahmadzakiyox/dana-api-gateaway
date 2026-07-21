# CLEANCODE — Mandatory Code Quality Rules

> This document contains **mandatory rules** that MUST be followed when writing, modifying, or reviewing code in ANY project.
> These rules are universal — they apply to all programming languages, frameworks, and architectures.

---

## RULE 1: MEANINGFUL NAMING

Every variable, function, class, and constant MUST have a self-descriptive name.
Anyone reading the code MUST understand its purpose without additional explanation.

### 1.1 Variables — Use Nouns

Variables store data. Name them with nouns that describe WHAT they hold.

DO NOT:
```
d = new Date()
x = getUser()
temp = data.filter(i => i.s === 1)
flag = true
lst = []
a = 25
```

MUST:
```
currentDate = new Date()
loggedInUser = getUser()
activeItems = data.filter(item => item.status === 1)
isVerified = true
pendingOrders = []
userAge = 25
```

### 1.2 Functions — Use Verbs

Functions perform actions. Name them with verbs that describe WHAT they do.

DO NOT:
```
function data() { }
function user() { }
function validation(email) { }
function proc(a, b) { }
function hndl(r) { }
```

MUST:
```
function fetchData() { }
function getActiveUser() { }
function validateEmail(email) { }
function calculateDiscountedPrice(originalPrice, discountPercent) { }
function handleLoginRequest(request) { }
```

### 1.3 Booleans — Use Prefix is / has / can / should

Booleans hold true/false conditions. MUST be named like a yes/no question.

DO NOT:
```
login = true
admin = false
permission = true
```

MUST:
```
isLoggedIn = true
isAdmin = false
hasPermission = true
canEdit = true
shouldRedirect = false
```

### 1.4 Constants — Use UPPER_SNAKE_CASE

Fixed values that never change MUST be visually distinct from regular variables.

```
MAX_RETRY_COUNT = 100
API_BASE_URL = "https://api.example.com"
DEFAULT_PAGE_SIZE = 20
TAX_RATE = 0.11
```

### 1.5 Arrays / Lists — Use Plural Nouns

DO NOT:
```
user = [{ name: "Andi" }, { name: "Budi" }]
```

MUST:
```
users = [{ name: "Andi" }, { name: "Budi" }]
pendingOrders = []
errorMessages = []
```

### 1.6 Classes / Types — Use PascalCase Nouns

```
class UserManager { }
class PaymentProcessor { }
class OrderValidator { }
```

### 1.7 Abbreviations

DO NOT use non-universal abbreviations.

DO NOT: `usrMgr`, `btnClkHndlr`, `prodQty`, `crtDt`

MUST: `userManager`, `buttonClickHandler`, `productQuantity`, `createdDate`

EXCEPTION — these universal abbreviations are allowed:
`id`, `url`, `api`, `html`, `css`, `db`, `ip`, `http`, `ws`, `io`.

### 1.8 Language-Specific Conventions

Follow the standard convention of the language being used:

| Language | Variables | Functions | Classes | Constants |
|---|---|---|---|---|
| JavaScript / TypeScript | camelCase | camelCase | PascalCase | UPPER_SNAKE |
| Python | snake_case | snake_case | PascalCase | UPPER_SNAKE |
| Java / Kotlin | camelCase | camelCase | PascalCase | UPPER_SNAKE |
| Go | camelCase | camelCase / PascalCase | PascalCase | PascalCase |
| PHP | $camelCase | camelCase | PascalCase | UPPER_SNAKE |

NEVER mix conventions within the same project. If the project already uses camelCase, all new code MUST use camelCase.

---

## RULE 2: ONE FUNCTION, ONE RESPONSIBILITY

Every function MUST do exactly ONE thing.
DO NOT stack multiple responsibilities in a single function.
Use a coordinator function or class to combine multiple tasks.

### Why?

When an error occurs, you MUST immediately know WHICH function is broken.
If one function does 5 things, you have to read the entire body to find the bug.

### DO NOT — Function with Multiple Responsibilities

```javascript
async function processOrder(orderData) {
  // Task 1: validation
  if (!orderData.productId) throw new Error("Product ID required");
  if (!orderData.quantity || orderData.quantity < 1) throw new Error("Invalid quantity");
  const user = await db.findUser(orderData.userId);
  if (!user) throw new Error("User not found");

  // Task 2: price calculation
  const product = await db.findProduct(orderData.productId);
  let totalPrice = product.price * orderData.quantity;
  if (orderData.couponCode) {
    const coupon = await db.findCoupon(orderData.couponCode);
    if (coupon && coupon.isValid) {
      totalPrice = totalPrice - (totalPrice * coupon.discount / 100);
    }
  }
  totalPrice = totalPrice + (totalPrice * 0.11);

  // Task 3: database write
  const order = await db.createOrder({ userId: orderData.userId, productId: orderData.productId, quantity: orderData.quantity, totalPrice, status: "pending" });

  // Task 4: notification
  await emailService.send(user.email, "Order Created", `Order #${order.id} created.`);
  await pushNotification.send(user.deviceToken, "New order!", `Total: $${totalPrice}`);

  return order;
}
```

### MUST — Split Into Separate Functions

```javascript
function validateOrderInput(orderData) {
  if (!orderData.productId) throw new Error("Product ID required");
  if (!orderData.quantity || orderData.quantity < 1) throw new Error("Invalid quantity");
  if (!orderData.userId) throw new Error("User ID required");
}

async function calculateTotalPrice(productId, quantity, couponCode) {
  const product = await db.findProduct(productId);
  let totalPrice = product.price * quantity;
  if (couponCode) {
    const coupon = await db.findCoupon(couponCode);
    if (coupon && coupon.isValid) {
      totalPrice = totalPrice - (totalPrice * coupon.discount / 100);
    }
  }
  const TAX_RATE = 0.11;
  return totalPrice + (totalPrice * TAX_RATE);
}

async function saveOrder(userId, productId, quantity, totalPrice) {
  return await db.createOrder({ userId, productId, quantity, totalPrice, status: "pending" });
}

async function notifyUser(user, order, totalPrice) {
  await emailService.send(user.email, "Order Created", `Order #${order.id} created.`);
  await pushNotification.send(user.deviceToken, "New order!", `Total: $${totalPrice}`);
}
```

Then create a coordinator:

```javascript
async function processOrder(orderData) {
  validateOrderInput(orderData);
  const totalPrice = await calculateTotalPrice(orderData.productId, orderData.quantity, orderData.couponCode);
  const user = await validateUserBalance(orderData.userId, totalPrice);
  const order = await saveOrder(orderData.userId, orderData.productId, orderData.quantity, totalPrice);
  await notifyUser(user, order, totalPrice);
  return order;
}
```

### When to Use Classes

Use a class when a group of functions share state and logically belong together:

```javascript
class OrderProcessor {
  constructor(db, emailService) {
    this.db = db;
    this.emailService = emailService;
  }

  validateInput(orderData) { /* one task */ }
  calculatePrice(productId, quantity, couponCode) { /* one task */ }
  saveOrder(userId, productId, quantity, totalPrice) { /* one task */ }
  notifyUser(user, order) { /* one task */ }

  async process(orderData) {
    // coordinator that calls the methods above in sequence
  }
}
```

### Function Size Limits

| Condition | Action |
|---|---|
| Function exceeds 20 lines | Consider splitting |
| Function exceeds 40 lines | MUST split |
| More than 3 levels of nesting | Extract inner logic into separate functions |
| More than 4 parameters | Use an object/struct as parameter |
| Must scroll to read one function | Function is too long, MUST split |

---

## RULE 3: MINIMAL AND PRECISE COMMENTS

Well-named code does NOT need comments.
Comments MUST only explain WHY, NEVER WHAT.
If you feel the need to comment what a variable or function does, that means the name is not clear enough — FIX THE NAME, do not add a comment.

### Comments That MUST NOT Exist

All of these are unnecessary because the code already explains itself:

```javascript
// Get user data              ← DO NOT
const userData = getUserData();

// Check if user is logged in  ← DO NOT
if (isLoggedIn) { }

// Calculate total price       ← DO NOT
const totalPrice = calculateTotalPrice(items);

// Increment counter           ← DO NOT
counter++;

// Loop through all products   ← DO NOT
for (const product of products) { }

// Return the result           ← DO NOT
return result;
```

### Comments That MUST Exist

Comments are REQUIRED in these situations:

**1. Explaining WHY, not WHAT:**
```javascript
// 30s timeout because third-party API is often slow during peak hours
const API_TIMEOUT = 30000;
```

**2. Workarounds or temporary hacks:**
```javascript
// Workaround: Instagram API returns URLs that expire after 24 hours,
// so we proxy images through our own server
const proxiedUrl = proxyImage(originalUrl);
```

**3. Critical execution order dependencies:**
```javascript
// IMPORTANT: payment MUST be recorded before notification is sent.
// If notification fails, payment data is still preserved.
await recordPayment(paymentData);
await sendNotification(userId, paymentData);
```

**4. Non-obvious technical decisions:**
```javascript
// Using regex because string.split() fails for multi-delimiter formats
const parts = input.match(/[^,;|]+/g);
```

**5. Constraints or assumptions other developers must know:**
```javascript
// This function MUST be called after initDatabase() completes
// Max 50 entries due to memory constraints on free-tier hosting
```

### TODO / FIXME Format

Temporary notes in code MUST include a target date:

```
// TODO(2026-08-01): Replace with API v2 after migration
// FIXME(2026-07-20): Race condition when 2 requests write simultaneously
// HACK: Temporary solution until library is updated to v4.x
```

### Comment Policy Summary

| Situation | Comment Required? |
|---|---|
| Variable/function name is already clear | NO |
| Complex business logic | YES |
| Workaround or temporary hack | YES |
| Reason for choosing a specific approach | YES |
| Critical execution order dependency | YES |
| Standard operations (loop, return, assign) | NO |

---

## RULE 4: MODULAR ARCHITECTURE

Code MUST be organized into independent, self-contained modules.
Each module MUST have a single clear purpose and well-defined boundaries.
Modules MUST communicate through explicit interfaces, NOT by reaching into each other's internals.

This is the foundation for scalability — a system that cannot be broken into independent pieces cannot grow.

### 4.1 What is a Module?

A module is any self-contained unit of code: a file, a folder, a class, a package, or a service.
The key property is **independence** — a module can be understood, tested, and modified without needing to understand the entire system.

### 4.2 Module Design Rules

**Rule: High Cohesion**
Everything inside a module MUST be closely related. If a function doesn't belong to the module's core purpose, move it out.

DO NOT:
```
lib/
└── utils.js  ← 2000 lines containing date formatting, API calls,
                 string manipulation, database queries, image processing,
                 email sending, and logging all in one file
```

MUST:
```
lib/
├── dateFormatter.js      ← only date formatting
├── apiClient.js          ← only HTTP request logic
├── stringUtils.js        ← only string manipulation
├── imageProcessor.js     ← only image processing
├── emailSender.js        ← only email sending
└── logger.js             ← only logging
```

**Rule: Loose Coupling**
Modules MUST NOT depend on each other's internal implementation. They MUST communicate through explicit exports and imports.

DO NOT — tight coupling (module A reaches into module B's internals):
```javascript
// orderService.js — directly manipulating another module's internal data
import { _internalCache } from "./userService.js";

function getOrderUser(orderId) {
  // BAD: accessing private cache of another module
  return _internalCache.users[orderId];
}
```

MUST — loose coupling (module A uses module B's public interface):
```javascript
// orderService.js — using the public API of another module
import { findUserById } from "./userService.js";

function getOrderUser(orderId) {
  return findUserById(orderId);
}
```

**Rule: Explicit Dependencies**
Every module MUST declare its dependencies at the top of the file through imports.
NEVER use global variables to share data between modules.

DO NOT:
```javascript
// BAD: relying on a global variable set somewhere else
function getApiKey() {
  return global.API_KEY; // where is this set? who controls it?
}
```

MUST:
```javascript
// GOOD: dependency is explicit and traceable
import { API_KEY } from "./config.js";

function getApiKey() {
  return API_KEY;
}
```

### 4.3 File and Folder Organization — Group by Feature

Organize files by FEATURE, NOT by file type.

DO NOT — group by type (hard to navigate at scale):
```
src/
├── controllers/
│   ├── userController.js
│   ├── orderController.js
│   └── productController.js
├── models/
│   ├── userModel.js
│   ├── orderModel.js
│   └── productModel.js
└── services/
    ├── userService.js
    ├── orderService.js
    └── productService.js
```

Problem: to work on "Order", you must open 3 different folders.

MUST — group by feature (scales naturally):
```
src/
├── user/
│   ├── userController.js
│   ├── userModel.js
│   └── userService.js
├── order/
│   ├── orderController.js
│   ├── orderModel.js
│   └── orderService.js
└── product/
    ├── productController.js
    ├── productModel.js
    └── productService.js
```

Benefit: everything related to "Order" is in one folder. Want to delete a feature? Delete one folder.

### 4.4 Module Interface Design

Every module MUST have a clear public interface. Only export what other modules need.

DO NOT — exporting everything:
```javascript
// userService.js
export function findUserById(id) { }
export function _hashPassword(pw) { }     // internal helper, should NOT be exported
export function _formatUserObject(raw) { } // internal helper, should NOT be exported
export const _cache = {};                  // internal state, should NOT be exported
```

MUST — export only the public API:
```javascript
// userService.js

// --- INTERNAL (not exported) ---
function hashPassword(pw) { }
function formatUserObject(raw) { }
const cache = {};

// --- PUBLIC API (exported) ---
export function findUserById(id) { }
export function createUser(data) { }
export function deleteUser(id) { }
```

### 4.5 Shared Utilities — The lib/ or shared/ Pattern

Code that is genuinely used by 3 or more modules MUST be placed in a shared location (`lib/`, `shared/`, or `utils/`).

```
src/
├── lib/                    ← shared utilities
│   ├── logger.js           ← used by many modules
│   ├── httpClient.js       ← used by many modules
│   └── validators.js       ← used by many modules
├── user/                   ← feature module
│   └── ...
├── order/                  ← feature module
│   └── ...
```

If a utility is only used by ONE module, keep it inside that module's folder. DO NOT put it in `lib/`.

### 4.6 Dependency Direction — Always Top-Down

Dependencies MUST flow in one direction: from higher-level modules down to lower-level modules.
Lower-level modules MUST NEVER import from higher-level modules.

```
ALLOWED dependency direction:

  Controller (high level)
      ↓
  Service (mid level)
      ↓
  Repository / Model (low level)
      ↓
  Shared Utilities (lowest level)

NEVER:
  Utility ← imports from → Controller  ← FORBIDDEN
  Model   ← imports from → Service     ← FORBIDDEN
```

### 4.7 Scaling Checklist

Before adding new features or modules, verify:

1. Can this module be understood WITHOUT reading other modules?
2. Can this module be tested in isolation?
3. If this module is deleted, does only the import statement in other modules break (not their logic)?
4. Does this module have a SINGLE clear purpose described in 1 sentence?
5. Are all dependencies explicitly imported at the top?

If any answer is NO, refactor before proceeding.

---

## RULE 5: CONSISTENT CODE STYLE

Pick one code style and use it consistently across the ENTIRE project.
Consistent style is MORE IMPORTANT than "perfect" style.

### DO NOT Mix Styles

```javascript
// DO NOT — mixed conventions
const user_name = "Andi";        // snake_case
const userAge = 25;              // camelCase
const UserEmail = "a@b.com";     // PascalCase

function GetUser() { }           // PascalCase
function fetch_orders() { }      // snake_case
function validateEmail() { }     // camelCase
```

### MUST Be Consistent

```javascript
// MUST — consistent camelCase for JavaScript
const userName = "Andi";
const userAge = 25;
const userEmail = "a@b.com";

function getUser() { }
function fetchOrders() { }
function validateEmail() { }
```

### Use Formatters and Linters

MUST use automated formatters to enforce consistency without manual effort:

| Language | Formatter | Linter |
|---|---|---|
| JavaScript | Prettier | ESLint |
| Python | Black / Ruff | Ruff / Pylint |
| Java | google-java-format | Checkstyle |
| Go | gofmt (built-in) | golint |
| PHP | PHP-CS-Fixer | PHPStan |

### File Naming Rules

- One file MUST contain one primary module or class.
- File name MUST match the primary export name.
- Pick ONE file naming convention (camelCase OR kebab-case), NEVER mix.

---

## RULE 6: CLEAR ERROR HANDLING

NEVER swallow errors silently.
NEVER catch errors without a specific and informative message.
Every error MUST provide enough information to find the root cause.

### DO NOT — Swallowing Errors

```javascript
try {
  await processPayment(orderData);
} catch (e) {
  console.log("error");  // DO NOT — no useful information
}

try {
  const result = await fetchData();
} catch (error) {
  return res.status(500).json({ message: "Something went wrong" });  // DO NOT — too generic
}
```

### MUST — Specific and Informative Errors

```javascript
try {
  const result = await fetchExternalApi(url);
  return result;
} catch (error) {
  if (error.code === "ECONNREFUSED") {
    throw new Error(`Failed to connect to external server: ${url}`);
  }
  if (error.response?.status === 429) {
    throw new Error(`Rate limit reached for: ${url}. Try again later.`);
  }
  throw new Error(`Failed to fetch data from ${url}: ${error.message}`);
}
```

### Use Custom Errors for Specific Cases

```javascript
class ValidationError extends Error {
  constructor(field, message) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
    this.statusCode = 400;
  }
}

class NotFoundError extends Error {
  constructor(resource, identifier) {
    super(`${resource} with ID "${identifier}" not found.`);
    this.name = "NotFoundError";
    this.statusCode = 404;
  }
}

// Usage
throw new ValidationError("email", "Invalid email format.");
throw new NotFoundError("User", "usr_12345");
```

---

## RULE 7: NO MAGIC NUMBERS OR MAGIC STRINGS

Numbers and strings that appear directly in code without context MUST be replaced with named constants.

### DO NOT — Magic Numbers

```javascript
if (user.role === 1) { }
if (retryCount > 3) { }
setTimeout(callback, 86400000);
const price = amount * 0.11;
if (status === "s") { }
```

### MUST — Named Constants

```javascript
const ROLE_ADMIN = 1;
const MAX_RETRY_ATTEMPTS = 3;
const ONE_DAY_IN_MS = 86400000;
const TAX_RATE = 0.11;
const STATUS_SUCCESS = "s";

if (user.role === ROLE_ADMIN) { }
if (retryCount > MAX_RETRY_ATTEMPTS) { }
setTimeout(callback, ONE_DAY_IN_MS);
const priceWithTax = amount * TAX_RATE;
if (status === STATUS_SUCCESS) { }
```

### Use Enums for Related Value Sets

```javascript
const OrderStatus = Object.freeze({
  PENDING:    "pending",
  PROCESSING: "processing",
  SHIPPED:    "shipped",
  DELIVERED:  "delivered",
  CANCELLED:  "cancelled",
});

const UserRole = Object.freeze({
  GUEST: 0,
  USER:  1,
  ADMIN: 2,
  SUPER: 3,
});

if (order.status === OrderStatus.SHIPPED) { }
if (user.role >= UserRole.ADMIN) { }
```

---

## RULE 8: DON'T REPEAT YOURSELF (DRY)

If the same code appears 3 or more times in different places, MUST extract it into a shared function.

### DO NOT — Copy-Pasting the Same Code

```javascript
// In orderController.js
const timestamp = new Date().toISOString().split("T")[0];
fs.appendFileSync("logs.txt", `[${timestamp}] Order created by ${userId}\n`);

// In paymentController.js — same code
const timestamp = new Date().toISOString().split("T")[0];
fs.appendFileSync("logs.txt", `[${timestamp}] Payment received from ${userId}\n`);

// In userController.js — same code again
const timestamp = new Date().toISOString().split("T")[0];
fs.appendFileSync("logs.txt", `[${timestamp}] User registered: ${userId}\n`);
```

### MUST — Extract to a Shared Function

```javascript
// lib/logger.js
function writeLog(action, userId) {
  const timestamp = new Date().toISOString().split("T")[0];
  fs.appendFileSync("logs.txt", `[${timestamp}] ${action} by ${userId}\n`);
}

// Usage anywhere
writeLog("Order created", userId);
writeLog("Payment received", userId);
writeLog("User registered", userId);
```

### EXCEPTION: Do Not Over-DRY

If two code blocks look similar but have FUNDAMENTALLY DIFFERENT logic, DO NOT force them into one function.
Duplication of only 2 occurrences with different contexts MAY be left as-is.

---

## PRE-COMMIT CHECKLIST

Before saving or pushing code, ALL of these MUST pass:

1. Can someone else read this code WITHOUT your explanation?
2. Does every function do exactly ONE thing?
3. Are there comments that are UNNECESSARY because the naming is already clear?
4. Are variable and function names MEANINGFUL enough?
5. Are there magic numbers or strings that NEED a named constant?
6. Is there code that was COPY-PASTED elsewhere?
7. Are errors handled with SPECIFIC messages?
8. Is the code style CONSISTENT throughout the file?
9. Is the code organized into INDEPENDENT modules with clear boundaries?
10. Do all dependencies flow TOP-DOWN?
