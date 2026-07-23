# Pharmacy Management System API

## A Complete Concept & Requirements Study Guide (Node.js + Express.js)

> **Purpose of this document:** Before writing a single line of code, this guide walks through _every concept_ I need to understand in order to build this project correctly. No implementation - just deep, beginner friendly understanding of the "why" and "how" behind each concept.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Software Requirements Analysis](#2-software-requirements-analysis)
3. [Understanding Node.js](#3-understanding-nodejs)
4. [Understanding Express.js](#4-understanding-expressjs)
5. [REST API Fundamentals](#5-rest-api-fundamentals)
6. [Authentication vs Authorization](#6-authentication-vs-authorization)
7. [JWT (JSON Web Tokens) Deep Dive](#7-jwt-json-web-tokens-deep-dive)
8. [Role-Based Access Control (RBAC)](#8-role-based-access-control-rbac)
9. [Password Security](#9-password-security)
10. [System Modules — Conceptual Breakdown](#10-system-modules--conceptual-breakdown)
11. [Databases & Relational Modeling](#11-databases--relational-modeling)
12. [Inventory Management Concepts](#12-inventory-management-concepts)
13. [Sales & Billing Concepts](#13-sales--billing-concepts)
14. [Pagination, Filtering & Sorting](#14-pagination-filtering--sorting)
15. [Validation & DTOs](#15-validation--dtos)
16. [Error Handling](#16-error-handling)
17. [Application Architecture (Layered Design)](#17-application-architecture-layered-design)
18. [Security Best Practices Summary](#18-security-best-practices-summary)
19. [Glossary of Key Terms](#19-glossary-of-key-terms)
20. [Summary & Study Checklist](#20-summary--study-checklist)

---

## 1. Project Overview

We are building the **backend API** for a Pharmacy Management System. Think of it as the "engine room" that powers a pharmacy's digital operations — no visual interface, just a set of well-organized endpoints (URLs) that a frontend app (web or mobile) would call to get work done.

The system needs to let a pharmacy:

- Manage **who can log in and what they're allowed to do** (Admin, Pharmacist, Staff)
- Keep track of **medicines** in stock, including expiry dates
- Record **prescriptions** written for customers
- Process **sales** and reduce stock automatically
- Manage **suppliers** who provide the medicines
- Let API consumers **search, filter, sort, and page through** large lists of data
- **Validate** every piece of incoming data and respond with clear, meaningful errors

This is a real-world backend project that makes me practice almost every core backend skill at once: auth, relational data design, business rules, and clean API design.

---

## 2. Software Requirements Analysis

Before writing code, professional developers do **Requirements Analysis** — figuring out precisely _what_ the system must do and _how well_ it must do it. This step prevents wasted work and confused design later.

### 2.1 Functional vs Non-Functional Requirements

**Functional Requirements** describe _what the system does_ — the actual features.
Examples from our project:

- A pharmacist can add a new medicine to inventory
- A staff member can process a sale
- The system must prevent selling an expired medicine
- An admin can create new user accounts

**Non-Functional Requirements** describe _how well_ the system performs — quality attributes, not features.
Examples:

- The API must respond within an acceptable time even with thousands of records (**performance**)
- Passwords must never be stored in plain text (**security**)
- The list endpoints must handle large datasets without crashing (**scalability**)
- Error messages must be clear and consistent (**usability/maintainability**)

Understanding this split matters because it changes _how_ you build things. Functional requirements shape your routes and business logic. Non-functional requirements shape your architecture decisions (e.g., why we use pagination, why we hash passwords, why we centralize error handling).

### 2.2 Identifying Actors (Who Uses the System)

An **actor** is any person or system that interacts with your API. For this project:

| Actor          | Description                                                                                                          |
| -------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Admin**      | Full control — manages users, staff, and everything else                                                             |
| **Pharmacist** | Manages medicines, prescriptions, and sales                                                                          |
| **Staff**      | Views data, processes sales assigned to them                                                                         |
| **Customer**   | Not a logged-in user of the API directly, but a data entity — someone whose prescriptions and purchases are recorded |
| **Supplier**   | Also a data entity, not a logged-in actor — provides medicines to the pharmacy                                       |

Notice the distinction: some "actors" in the real world (Customer, Supplier) are just **data** in our system, not people who log in. Only Admin, Pharmacist, and Staff authenticate into the API. This distinction is critical when designing your database — customers and suppliers don't need passwords or login logic, but they do need to be linked (related) to other data.

### 2.3 Use Cases

A **use case** is a specific scenario describing an actor achieving a goal. Writing these out in plain English _before_ coding helps you discover the exact API endpoints you'll need. A few examples:

- _"As a Pharmacist, I want to add a new medicine with its supplier, stock quantity, and expiry date, so that inventory stays accurate."_
- _"As a Staff member, I want to view a paginated list of medicines filtered by availability, so I can quickly find what's in stock."_
- _"As an Admin, I want to create a new staff account with a specific role, so they can log in with proper permissions."_
- _"As a Pharmacist, I want the system to reject a sale if the medicine is expired or out of stock, so we never sell unsafe or unavailable products."_

Every one of these later becomes a specific **API endpoint** with specific **business rules** — that's the bridge between requirements analysis and actual system design.

### 2.4 Entities Identified From Requirements

Reading through the requirements, we can already identify the core "things" (entities) our database will need to represent:

- **User** (login credentials + role)
- **Staff Profile** (extra details tied to a user who is Pharmacist/Staff)
- **Customer**
- **Supplier**
- **Medicine**
- **Prescription**
- **Sale**

We'll explore exactly how these relate to each other in Section 11.

---

## 3. Understanding Node.js

### 3.1 What Node.js Actually Is

JavaScript was originally built to run _only_ inside web browsers. **Node.js** is a runtime environment that lets JavaScript run outside the browser — directly on a server/computer. It's built on Google Chrome's **V8 engine** (the same engine that runs JS in Chrome), but it adds capabilities browsers don't have, like reading files, handling network requests, and talking to databases.

In short: **Node.js turns JavaScript into a language that can build backend servers.**

### 3.2 Single-Threaded but Non-Blocking

This is the single most important concept to understand about Node.js, and it's often confusing for beginners.

- Node.js runs your JavaScript code on **a single thread** (one "worker" doing the work, one line at a time).
- BUT — Node.js is **non-blocking** for I/O operations (I/O = input/output, like reading a file, querying a database, or waiting on a network request).

Here's the key mental model: imagine a single waiter (the thread) in a restaurant. If the waiter had to stand at your table and wait while the kitchen cooks your food (a "blocking" model), they could only serve one table at a time — terribly inefficient. Instead, the waiter takes your order, hands it to the kitchen, and **immediately moves to the next table** while the kitchen works in the background. When your food is ready, the kitchen notifies the waiter, who then serves it to you.

That's exactly how Node.js handles things like database queries or file reads: it delegates the "waiting" part to the underlying system (via something called **libuv**, Node's C++ library that manages background work and the OS), and your single JS thread is freed up to keep handling other requests. When the background work finishes, a **callback** (or a resolved Promise, if using `async/await`) is queued up to run.

This is why Node.js is a great fit for a pharmacy API that will get many concurrent requests (multiple pharmacists/staff hitting the API at once) — it can juggle many in-progress operations without needing a separate thread per request (unlike some older server technologies).

### 3.3 The Event Loop

The **event loop** is the mechanism that makes non-blocking behavior possible. It's essentially an infinite loop that:

1. Checks if there's any code ready to run (like a completed database query's callback)
2. Runs it
3. Goes back and checks again

You don't have to manage the event loop yourself — it runs automatically for the entire life of your Node.js application. Understanding that it _exists_ helps you understand why asynchronous code (`async/await`, Promises, callbacks) is the natural way to write Node.js applications — you're always cooperating with this loop rather than blocking it.

**Why this matters for our project:** every database call (fetching medicines, checking prescriptions, saving a sale) is an I/O operation. We'll write these using `async/await` syntax, which is just a cleaner way to write asynchronous, non-blocking code without deeply nesting callbacks.

### 3.3.1 Concurrent vs Parallel — Node.js Is Concurrent, Not Parallel

**The core difference**

- **Parallel** = two things *actually* happening at the exact same instant. This requires two separate workers (two CPU cores, two threads) each doing their own task simultaneously.
- **Concurrent** = two things *appear* to be happening at the same time, but really it's one worker rapidly switching between tasks — doing a little bit of A, then a little bit of B, then back to A — so fast it looks simultaneous, even though at any single moment only one thing is actually being worked on.

**The single-cashier analogy**

Imagine one cashier at a counter (this is your one JS thread).

**Parallel** would be: hire a second cashier. Now two customers really are being served at the exact same second, by two different people.

**Concurrent** (what Node.js does) is: there's still only **one** cashier. But instead of serving Customer A from start to finish before even looking at Customer B, the cashier does this:

1. Takes Customer A's order, sends it to the kitchen, and — instead of standing there waiting — immediately turns to Customer B and takes their order too.
2. Sends B's order to the kitchen as well.
3. Whichever order the kitchen finishes first, the cashier goes back, hands it over, then returns to whatever's next.

From the customers' point of view, it feels like they're both being served "at once" — but if you freeze time at any single moment, the cashier is only ever doing one specific action. Never two at literally the same instant.

### 3.4 npm and Modules

**npm** (Node Package Manager) is the tool that comes bundled with Node.js. It lets you:

- Install reusable packages (code other people wrote) like Express, JWT libraries, password hashing libraries, etc.
- Manage your project's dependencies via a file called `package.json`, which lists exactly which packages (and versions) your project needs.

Node.js code is organized into **modules** — each file is its own self-contained unit that can export functionality (functions, classes, objects) for other files to import. This is what allows us to split a large project into small, organized files (routes, controllers, services, models) instead of one giant file.

### 3.5 Callbacks, Promises, Async/Await, explaining each in the aspect of Event Loop

**The purpose of each, in one line:**
- **Callback** — "call this function when you're done." The original way JS handled async work.
- **Promise** — an object representing a value that *will exist eventually* (pending → fulfilled/rejected). Solves "callback hell" (deeply nested callbacks).
- **async/await** — syntax sugar *on top of* Promises. Lets async code *read* like normal, top-to-bottom synchronous code, while still being non-blocking underneath.

**What actually happens at the event-loop level:**

1. Calling an `async function` runs it **synchronously** — line by line — right up until it hits an `await`.
2. At `await somePromise`, the function's execution is **paused right there**, and — critically — **control is immediately handed back to whatever called it**. The async function itself has already returned a (still-pending) Promise to its caller by this point.
3. The call stack is now free. The event loop goes on running other code — handling other incoming requests, timers, other pending operations — completely unaffected.
4. When the awaited operation (e.g. a database query) actually finishes, the *rest* of that paused function is queued up (as a "microtask") to run next, and picks up exactly where it left off.

**So — is `await` blocking?** No. It only pauses *that specific function's own continuation*. It does not freeze Node, the thread, or any other request being handled. Think back to the single-waiter restaurant analogy (Section 3.2): `await` is the waiter saying "I'll come back to this table once the kitchen's done" — not the waiter standing frozen at the table.

**Is it "parallel"?** No — it's **concurrent**, not parallel. There is still only one thread executing your JS, one instruction at a time. What async/await achieves is *overlapping the waiting time* of multiple operations, not literally running multiple pieces of code at the same instant. True parallel execution (multiple cores actually computing simultaneously) requires separate threads or processes — outside what async/await does.

**Why async/await over raw callbacks/Promises today?** Mainly readability. The following are conceptually equivalent, but async/await avoids nested indentation and lets you use normal `try/catch` for error handling:

```js
// Callback style
getMedicine(id, (err, medicine) => {
  if (err) return handleError(err);
  console.log(medicine);
});

// Promise style
getMedicine(id)
  .then(medicine => console.log(medicine))
  .catch(handleError);

// async/await style
try {
  const medicine = await getMedicine(id);
  console.log(medicine);
} catch (err) {
  handleError(err);
}
```

### 3.5.1 How Promises Solve "Callback Hell"

When multiple async steps depend on each other using callbacks, you end up nesting one inside another — deeper and deeper with every step. This is called **"callback hell"** (or the "pyramid of doom"):

```js
// Callback hell — nested, hard to read, hard to error-handle
getCustomer(customerId, (err, customer) => {
  if (err) return handleError(err);
  getPrescription(customer.id, (err, prescription) => {
    if (err) return handleError(err);
    getMedicines(prescription.id, (err, medicines) => {
      if (err) return handleError(err);
      checkStock(medicines, (err, stockOk) => {
        if (err) return handleError(err);
        console.log('Ready to sell:', stockOk);
      });
    });
  });
});
```

Every new step indents further right, error handling is repeated at every level, and reading the *actual order of operations* means following an ever-deepening staircase.

**Promises flatten this** by letting each async step return an object you can `.then()` off of, chaining steps at the **same indentation level** instead of nesting inside each other:

```js
// Same logic, with Promises — flat, chained, one error handler for everything
getCustomer(customerId)
  .then(customer => getPrescription(customer.id))
  .then(prescription => getMedicines(prescription.id))
  .then(medicines => checkStock(medicines))
  .then(stockOk => console.log('Ready to sell:', stockOk))
  .catch(handleError); // one catch handles an error from ANY step above
```

No pyramid, no repeated `if (err)` checks at every level — just one `.catch()` at the end that catches a failure from *any* step in the chain.

---

### 3.5.2 Promise vs Async/Await in a Real Express Route

Async/await is just cleaner syntax over the same Promise chain above — it reads top-to-bottom like normal synchronous code:

```js
// Promise-chain style controller
function getMedicineById(req, res) {
  medicineService.findById(req.params.id)
    .then(medicine => {
      if (!medicine) {
        return res.status(404).json({ success: false, message: 'Medicine not found' });
      }
      res.status(200).json({ success: true, data: medicine });
    })
    .catch(err => {
      res.status(500).json({ success: false, message: 'Something went wrong' });
    });
}

// async/await style controller — same behavior, reads like sync code
async function getMedicineById(req, res) {
  try {
    const medicine = await medicineService.findById(req.params.id);
    if (!medicine) {
      return res.status(404).json({ success: false, message: 'Medicine not found' });
    }
    res.status(200).json({ success: true, data: medicine });
  } catch (err) {
    res.status(500).json({ success: false, message: 'Something went wrong' });
  }
}
```

Both versions do exactly the same thing under the hood — `findById` still returns a Promise either way. `async/await` just lets you write `try/catch` instead of `.then()/.catch()`, which reads far more naturally once you have several dependent steps (like the customer → prescription → medicines → stock-check chain above).

---

### 3.6 npm vs npx - What They're For and When to Use Which

- **npm (Node Package Manager)** — installs and manages packages. Running `npm install pg` downloads the PostgreSQL driver into your project's `node_modules` folder and adds it to `package.json` as a **permanent dependency** your app will need every time it runs.
- **npx (Node Package eXecute)** — *runs* a package's command-line tool. If that tool is already installed (locally in `node_modules/.bin`, or globally), `npx` just runs it. If it **isn't installed at all**, `npx` will temporarily download it, run it once, and not leave it installed in your project — useful for one-off commands you don't want cluttering your dependencies.

**Project examples:**
- `npm install express pg bcrypt jsonwebtoken` → these are libraries your API actually imports and runs on every request — permanent dependencies, so `npm install`.
- `npx prisma init` → a one-time scaffolding command to set up Prisma's folder structure — you're not "running your app" with this, just generating files once, so `npx` (no need to ever install a global `prisma` CLI).
- `npx nodemon index.js` → if you haven't installed `nodemon` as a project dependency yet, this runs it anyway without a permanent install. (Though in practice, since you'd use it constantly during development, most projects instead run `npm install --save-dev nodemon` once, then use `npm run dev` — a `package.json` script — to launch it.)

**Rule of thumb:** if your **code will `require()`/`import` it** → `npm install` it as a real dependency. If you just need to **run a CLI tool**, especially a one-off or something you don't want permanently installed → `npx`.

---

## 4. Understanding Express.js

### 4.1 What Express.js Is

Node.js by itself gives you very low-level tools to build a server (the built-in `http` module), but writing a full API with plain Node.js quickly becomes repetitive and messy. **Express.js** is a lightweight **framework** built on top of Node.js that gives you a clean, organized way to:

- Define routes (URLs your API responds to)
- Handle incoming requests and send responses
- Plug in reusable logic via **middleware** (explained below)

Think of Node.js as the raw engine, and Express.js as the steering wheel, pedals, and dashboard that make driving that engine actually practical.

### 4.2 The Request-Response Cycle

Every interaction with your API follows this basic cycle:

1. A **client** (a frontend app, mobile app, or tool like Postman) sends an **HTTP request** to a specific URL (e.g., `GET /medicines`)
2. Express receives the request and matches it to a route you've defined
3. The request may pass through several **middleware functions** first (checking auth, validating data, logging, etc.)
4. Your **route handler** (controller logic) runs, often talking to the database
5. Express sends back an **HTTP response** — typically JSON data plus a status code (like `200 OK` or `404 Not Found`)

### 4.3 Middleware — The Core Idea of Express

**Middleware** is simply a function that sits _between_ the incoming request and the final response. It has access to:

- `req` (the request object — data sent by the client)
- `res` (the response object — what you'll send back)
- `next` (a function you call to pass control to the _next_ middleware or route handler)

You can chain as many middleware functions as you like before the final handler runs. This is an incredibly powerful pattern because it lets you write reusable, composable pieces of logic. For our project, middleware will be used for:

- **Authentication middleware** — checks if a valid JWT was provided; if not, stops the request immediately with a 401 error
- **Authorization/role-guard middleware** — checks if the logged-in user's role is allowed to access this specific route
- **Validation middleware** — checks that the incoming request body matches the expected shape/rules before it reaches business logic
- **Error-handling middleware** — a special kind of middleware (placed at the very end) that catches any errors thrown anywhere in the chain and formats a consistent error response

The beauty of middleware is that each piece of logic is written **once** and reused across many routes, instead of being copy-pasted into every route handler.

### 4.4 Routing

Express lets you define routes tied to HTTP methods and URL patterns, e.g.:

- `GET /medicines` → list all medicines
- `POST /medicines` → create a new medicine
- `GET /medicines/:id` → get one specific medicine by ID
- `PATCH /medicines/:id` → update a medicine
- `DELETE /medicines/:id` (or a "deactivate" pattern, discussed later) → remove/deactivate a medicine

Express also supports **routers** — a way to group related routes into their own file/module (e.g., all medicine-related routes in `medicine.routes.js`), which keeps large projects organized. This ties directly into the layered architecture we'll cover in Section 17.

---

## 5. REST API Fundamentals

**REST** (Representational State Transfer) is a set of conventions/principles for designing APIs in a predictable, standardized way. Almost all modern backend APIs (including ours) follow REST principles.

### 5.1 Core Principles

- **Resources**: Everything is modeled as a "resource" — a noun, not a verb. In our system: `medicines`, `prescriptions`, `sales`, `suppliers`, `users`. You don't design an endpoint called `/getAllMedicines`; instead you use the resource name `/medicines` combined with an HTTP method.
- **HTTP Methods map to actions**:
  - `GET` → read/retrieve data (safe, doesn't change anything)
  - `POST` → create new data
  - `PUT`/`PATCH` → update existing data (`PUT` typically replaces the whole resource, `PATCH` updates part of it)
  - `DELETE` → remove data
- **Statelessness**: Each request must contain everything the server needs to understand it (like the auth token). The server does NOT remember anything about you between requests — there's no "session memory" sitting on the server. This is exactly why JWT-based authentication (Section 7) fits REST so well: the token itself carries the proof of who you are, every single request.
- **Uniform interface**: Predictable URL structure and consistent response formats make the API easy for any frontend to consume.

### 5.2 HTTP Status Codes

A well-designed API communicates meaning through status codes, not just messages. Some you'll use constantly:

| Code                      | Meaning                        | Example use in our project              |
| ------------------------- | ------------------------------ | --------------------------------------- |
| 200 OK                    | Request succeeded              | Successfully fetched medicine list      |
| 201 Created               | New resource created           | New medicine added                      |
| 400 Bad Request           | Invalid input                  | Missing required field in request body  |
| 401 Unauthorized          | Not authenticated              | No/invalid JWT provided                 |
| 403 Forbidden             | Authenticated but not allowed  | Staff trying to delete a medicine       |
| 404 Not Found             | Resource doesn't exist         | Medicine ID doesn't exist               |
| 409 Conflict              | Conflicting state              | Trying to sell an out-of-stock medicine |
| 500 Internal Server Error | Unexpected server-side failure | Unhandled bug/database failure          |

Understanding this table deeply matters because a huge part of "Error Handling" (Section 16) is simply: _picking the right status code and message for every possible failure scenario._

---

## 6. Authentication vs Authorization

These two words sound similar but mean very different things — and mixing them up is one of the most common beginner mistakes.

- **Authentication** = "Who are you?" — verifying identity. This is the login process: proving you are a real, registered user (checking email/password, then issuing a token).
- **Authorization** = "What are you allowed to do?" — checking permissions _after_ identity is already confirmed. This is where roles (Admin/Pharmacist/Staff) come in.

Think of it like entering an office building: **authentication** is showing your ID badge at the front door to prove you're an employee. **Authorization** is whether your badge lets you into the server room, or just the break room.

In our project:

- Authentication happens once at login — the server verifies credentials and issues a **JWT**.
- Authorization happens on _every subsequent request_ — middleware checks the role encoded in that JWT against what the specific route requires.

---

## 7. JWT (JSON Web Tokens) Deep Dive

### 7.1 Why JWT Instead of Sessions?

Traditionally, many web apps used **sessions**: the server creates a session record (stored in server memory or a database) when you log in, and gives your browser a session ID in a cookie. Every request, the server looks up that session ID to know who you are. This requires the server to **remember state** about every logged-in user.

REST APIs prefer to be **stateless** (Section 5.1) — the server shouldn't have to store anything about who's logged in. **JWT solves this**: instead of the server remembering you, it gives you a signed token that contains your identity information directly inside it. Every request, you send that token, and the server can verify it's authentic _without_ looking anything up in a database. This makes JWT auth easy to scale across multiple servers too, since no shared "session store" is needed.

### 7.2 The Structure of a JWT

A JWT is a string made of **three parts**, separated by dots: `header.payload.signature`

1. **Header** — metadata about the token, like which signing algorithm is used (commonly `HS256`).
2. **Payload** — the actual data ("claims") encoded into the token. This might include the user's ID, role, and an expiry timestamp. **Important beginner insight:** the payload is only _encoded_ (Base64), not encrypted — anyone can decode and read it. Never put sensitive secrets (like a password) in the payload. The security of a JWT comes from the signature, not secrecy of the payload content.
3. **Signature** — created by taking the header + payload and signing them with a **secret key** that only the server knows, using the chosen algorithm. This signature is what proves the token hasn't been tampered with.

### 7.3 How JWT Verification Works

When a request comes in with a JWT (typically sent in the `Authorization: Bearer <token>` header):

1. The server splits the token and re-computes the signature using the header + payload + its own secret key.
2. If the computed signature matches the one in the token, the token is valid and untampered.
3. If someone tried to edit the payload (e.g., change their role from "staff" to "admin"), the signature would no longer match, and verification would fail.
4. The server also checks the token's expiry claim — expired tokens are rejected even if the signature is valid.

This is the core trick that makes JWTs trustworthy without a database lookup: **cryptographic signing**, not secrecy of content.

### 7.4 Access Tokens vs Refresh Tokens (Concept)

Many production systems use two kinds of tokens:

- **Access Token**: short-lived (e.g., 15 minutes), sent with every API request, used for authorization.
- **Refresh Token**: longer-lived, stored more securely, used _only_ to request a new access token once the old one expires — without forcing the user to log in again.

This limits the damage if an access token is somehow stolen (it expires quickly), while still keeping the user experience smooth. This is a design decision worth understanding conceptually, even at a basic-project level.

### 7.5 Route Protection with Guards (Middleware)

A "**guard**" (mentioned in the requirements) is really just the term for **authentication/authorization middleware** that "guards" a route — it runs before the actual route logic and blocks the request if the token is missing, invalid, or the role doesn't have permission. Conceptually:

- **Auth Guard**: "Is there a valid JWT? If not, reject with 401."
- **Role Guard**: "Does this user's role match what's required for this route? If not, reject with 403."

These guards are typically written once as reusable middleware and then attached to whichever routes need protection — some routes (like login) need no guard at all, some need just the auth guard, and some need both auth + a specific role check.

---

## 8. Role-Based Access Control (RBAC)

### 8.1 The Core Idea

**RBAC** means permissions aren't assigned to individual users one-by-one — instead, users are assigned a **role**, and permissions are attached to that role. This project defines three roles:

| Role           | Permissions (conceptually)                                                           |
| -------------- | ------------------------------------------------------------------------------------ |
| **Admin**      | Full access to everything — user/staff management, all modules, all data             |
| **Pharmacist** | Manage medicines (add/update/deactivate), manage prescriptions, process/manage sales |
| **Staff**      | Read-only access to most data, but can process sales _assigned to them_              |

### 8.2 Why RBAC Instead of Per-User Permissions?

Imagine a pharmacy with 50 staff members. Assigning individual permissions to each person would be a maintenance nightmare. With RBAC, you define the rules **once per role**, and every user just gets tagged with a role. Adding a new staff member is instant — just assign them the "Staff" role and they immediately inherit the correct permission set.

### 8.3 Where RBAC Lives in the Code (Conceptually)

RBAC in an Express app is enforced through the **role-guard middleware** described in Section 7.5. Conceptually, each protected route declares which role(s) are allowed, e.g.: _"this route requires the user's role to be Admin or Pharmacist."_ The middleware reads the role from the decoded JWT payload and compares it against that list before allowing the request to continue.

### 8.4 RBAC and the Database Design

Since role determines behavior, the **User** entity in our database needs a `role` field (e.g., a string or enum: `admin`, `pharmacist`, `staff`). This is a small but critical detail in the data model — we'll revisit it in Section 11.

---

## 9. Password Security

### 9.1 Never Store Plain-Text Passwords

Storing a user's password as plain, readable text in the database is a severe security flaw — if the database is ever leaked or accessed by an attacker, every user's real password is instantly exposed (and since people often reuse passwords, this compromises their accounts elsewhere too).

### 9.2 Hashing (Not Encryption)

Instead, passwords are **hashed**. A hash function takes an input (the password) and produces a fixed-length, scrambled output that is:

- **One-way**: you cannot reverse a hash back into the original password.
- **Deterministic**: the same input always produces the same hash — which is what lets you verify a login later (hash the entered password, compare it to the stored hash).

This is different from **encryption**, which is reversible (meant to be decrypted later with a key). Passwords should never be _encrypted_ — they should be _hashed_, because we never need to recover the original password, only verify a match.

### 9.3 Salting and bcrypt

A plain hash alone is still somewhat vulnerable to precomputed "rainbow table" attacks (huge lookup tables of common passwords and their hashes). To defend against this, we add a **salt** — random data mixed into the password before hashing, unique per user. This ensures that even if two users have the identical password, their stored hashes look completely different.

**bcrypt** is the industry-standard hashing library for this exact purpose. It automatically handles salting for you and includes a "cost factor" (also called salt rounds) that controls how computationally expensive the hashing is — deliberately slow, to make brute-force attacks impractical. Conceptually, when a user registers:

1. Their plain password + a generated salt are run through bcrypt's hashing algorithm.
2. The resulting hash (which already includes the salt info) is what gets stored in the database — never the plain password.

When they log in:

1. The plain password they typed is hashed using the same algorithm and the stored salt.
2. The new hash is compared to the stored hash. If they match, the password was correct.

---

## 10. System Modules — Conceptual Breakdown

Let's walk through each module from the requirements and understand its _purpose_ and _responsibilities_ at a conceptual level.

### 10.1 User & Staff Management

This module handles **who can log into the system**. Two closely related but distinct ideas:

- **User**: the login identity — email/username, hashed password, and role.
- **Staff Profile**: additional information about a Pharmacist or Staff member that isn't related to login at all (e.g., contact info, employment details, department).

Why split these into two separate concepts instead of one giant "User" table with everything crammed in? Because they represent different concerns: authentication data vs. HR/profile data. This separation is also _exactly_ why the requirements mention a **One-to-One relationship** between User and Staff Profile — each user has exactly one staff profile (if they need one), and each staff profile belongs to exactly one user. We explore this in Section 11.

### 10.2 Medicine Inventory Management

This module is the heart of the pharmacy's stock-keeping. Each medicine record conceptually needs to track:

- Basic identity info (name, description, category)
- **Stock quantity** — how many units are currently available
- **Expiry date** — critical for a pharmacy; selling expired medicine is a safety and legal issue
- **Status** — active vs. deactivated (pharmacies don't usually _delete_ medicine records outright, since historical sales/prescriptions still reference them — instead they're "deactivated" so they stop appearing as sellable, while preserving historical data integrity)
- **Link to Supplier** — which supplier provides this medicine (a One-to-Many relationship, Section 11)

### 10.3 Sales & Billing

This module represents the actual transactions: a customer buying medicine, processed by a staff member. Conceptually, a **Sale** needs to:

- Know **which staff member processed it** and **which customer** it was for
- Know **which medicine(s)** were sold, and in what quantities (this is why sales-to-medicines is Many-to-Many — a single sale can include multiple different medicines, and a medicine can appear across many different sales)
- Track a **status** (e.g., `completed`, `cancelled`)
- Trigger a **side effect**: reducing the relevant medicine's stock quantity automatically when the sale completes

### 10.4 Supplier Management

Suppliers are the businesses that provide medicines to the pharmacy. This module is comparatively simple: supplier records (name, contact info, etc.) that are linked to the medicines they supply. This module matters because tracking supplier relationships supports real business needs — knowing where to reorder stock from, and being able to filter/report on medicines by supplier.

### 10.5 How the Modules Connect

None of these modules exist in isolation — the real complexity (and the real learning value) of this project is in how they **relate to each other** through the database, which brings us to the next major section.

---

## 11. Databases & Relational Modeling

### 11.1 What "Relational" Means

A **relational database** organizes data into **tables** (like spreadsheets), where each row is a record and each column is an attribute. The real power comes from **relationships** — tables can reference each other, so you don't duplicate data everywhere. Instead of writing a supplier's full contact info into every single medicine record, you store it once in a `Suppliers` table and just _reference_ it from `Medicines`.

### 11.2 Primary Keys and Foreign Keys

- A **Primary Key (PK)** is a unique identifier for each row in a table (commonly an auto-incrementing ID or a UUID). No two rows share the same primary key.
- A **Foreign Key (FK)** is a column in one table that stores the primary key of a row in _another_ table, creating a link between them.

Example: a `Medicine` row might have a `supplierId` column — that's a foreign key pointing to a specific row in the `Supplier` table.

### 11.3 One-to-One Relationship: User → Staff Profile

A **One-to-One** relationship means one row in Table A relates to exactly one row in Table B, and vice-versa.

**In our project**: one `User` has exactly one `StaffProfile`, and each `StaffProfile` belongs to exactly one `User`.

**How it's implemented**: the `StaffProfile` table has a foreign key column (e.g., `userId`) pointing back to the `User` table, and that foreign key is marked as **unique** — this uniqueness constraint is what actually enforces "one-to-one" instead of "one-to-many" at the database level. Without the uniqueness constraint, nothing would stop multiple staff profiles from pointing to the same user, which would technically make it one-to-many instead.

**Why separate them at all instead of one table?** As discussed in Section 10.1 — separation of concerns (auth data vs. profile data), and it also means not every `User` _needs_ a staff profile (e.g., if customers were ever given logins, they wouldn't need one).

### 11.4 One-to-Many Relationships

A **One-to-Many** relationship means one row in Table A can relate to _many_ rows in Table B, but each row in Table B relates back to only _one_ row in Table A.

**Example 1 — Supplier → Medicines**: One supplier can supply many different medicines, but each medicine record comes from exactly one supplier (in this simplified model). Implementation: the `Medicine` table has a `supplierId` foreign key.

**Example 2 — Customer → Prescriptions**: One customer can have many prescriptions over time (different doctor visits, different needs), but each individual prescription belongs to exactly one customer. Implementation: the `Prescription` table has a `customerId` foreign key.

The general pattern for one-to-many: **the foreign key always lives on the "many" side.** The "one" side doesn't need any extra column — its relationship to many children is discovered by _querying_ for all rows where the foreign key matches its ID.

### 11.5 Many-to-Many Relationships

A **Many-to-Many** relationship means rows in Table A can relate to _multiple_ rows in Table B, and rows in Table B can relate to _multiple_ rows in Table A.

**Example 1 — Prescriptions ↔ Medicines**: A single prescription often lists multiple medicines, and any given medicine can appear across many different prescriptions.

**Example 2 — Sales ↔ Medicines**: A single sale/transaction can include multiple different medicines being purchased at once, and a specific medicine will appear in many different sales over time.

**How it's implemented — the Junction Table**: You _cannot_ represent many-to-many with a simple foreign key column on either side (that only works for one-to-many). Instead, you introduce a **third table** — often called a **junction table** (or "join table" / "pivot table") — that sits between the two, with a foreign key pointing to each side.

For example, a junction table `PrescriptionMedicine` would have:

- `prescriptionId` (FK → Prescription)
- `medicineId` (FK → Medicine)
- Possibly extra columns specific to _that pairing_, like `dosageInstructions` or `quantityPrescribed` — data that doesn't belong to the medicine itself or the prescription itself, but to the _relationship between them_.

Similarly, a `SaleMedicine` junction table would connect `Sale` and `Medicine`, and is actually the perfect place to store `quantitySold` and `pricePerUnitAtTimeOfSale` for that specific line item — since price and quantity are specific to _that sale_, not to the medicine's general record.

**This is one of the most important concepts in relational design**, and pharmacies are a great real-world example of why junction tables exist — because "a sale contains multiple items, each with their own quantity" is such a common real-world pattern (think of any shopping receipt).

### 11.6 Data Integrity & Constraints

**Data integrity** means the data in your database stays accurate, consistent, and trustworthy over time. Relational databases give you tools to _enforce_ this automatically, rather than relying on your application code to always get it right:

- **NOT NULL constraint**: a column must always have a value (e.g., a medicine must always have a name).
- **UNIQUE constraint**: no two rows can share the same value in that column (e.g., two users can't share the same email; used for enforcing one-to-one relationships too, as in Section 11.3).
- **Foreign Key constraint**: the database physically refuses to let you insert a foreign key value that doesn't correspond to a real row in the referenced table (e.g., you cannot create a medicine pointing to a `supplierId` that doesn't exist).
- **CASCADE / RESTRICT rules**: define what happens when a referenced row is deleted. For example, should deleting a supplier also delete all their medicines (`CASCADE`), or should the database _block_ that deletion while medicines still reference it (`RESTRICT`)? For a pharmacy system, you'd typically **restrict or soft-delete** rather than cascade-delete, since medicine history needs to be preserved for audit/legal reasons — this connects back to the "deactivate, don't delete" idea from Section 10.2.

### 11.7 What Is an ORM?

Writing raw SQL queries by hand for every operation works, but becomes repetitive and error-prone at scale. An **ORM (Object-Relational Mapper)** — like Prisma, Sequelize, or TypeORM in the Node.js world — lets you define your database tables as JavaScript/TypeScript objects ("models") and interact with your database using regular JS code instead of raw SQL strings. The ORM translates your code into SQL behind the scenes.

Benefits relevant to our project:

- Relationships (one-to-one, one-to-many, many-to-many) can be **declared directly in your model definitions**, and the ORM handles generating the correct foreign keys, junction tables, and constraints for you.
- Querying related data (e.g., "get a sale along with all its medicines") becomes much simpler than writing manual JOIN queries.
- Many ORMs offer built-in **migration** systems — version-controlled, incremental changes to your database schema over time, so your team can evolve the database structure safely.

This is a conceptual note for when you move to implementation — you are not required to write raw SQL for this project, but understanding what the ORM does _for_ you (rather than treating it as magic) will make you a much stronger backend developer.

### 11.8 PostgreSQL Specifics (Coming From SQL Server / General SQL)

This project uses **PostgreSQL** as the database and **Express.js** as the framework (correcting the earlier note about the stack). The good news: everything in Sections 11.1–11.6 above (primary keys, foreign keys, one-to-one/one-to-many/many-to-many, constraints) is standard relational database theory — it applies identically in PostgreSQL. What changes is the _dialect_ and a handful of Postgres-specific tools and features. Since you already know SQL Server, here's what's genuinely different or new.

**What PostgreSQL is.** It's a free, open-source, object-relational database system. "Object-relational" means it's a relational database (tables, rows, SQL, joins) but with extra features borrowed from object-oriented concepts, like custom types and rich data types. It's one of the most popular databases for Node.js backends, largely because of how well it's supported by every major ORM and its excellent handling of complex data and constraints.

**Auto-incrementing IDs.** In SQL Server you're used to `IDENTITY(1,1)` columns. PostgreSQL's traditional equivalent is the `SERIAL` (or `BIGSERIAL` for larger ranges) pseudo-type, which automatically creates an internal **sequence** (a separate database object that generates the next number) and wires it to your column. Modern PostgreSQL (10+) also supports the SQL-standard `GENERATED ALWAYS AS IDENTITY` syntax, which behaves very similarly to what you already know from SQL Server. Many Node.js projects on Postgres instead use **UUIDs** (universally unique IDs, long random strings) as primary keys rather than incrementing integers — useful when you don't want IDs to reveal how many records exist, or when merging data from multiple sources.

**Case sensitivity of identifiers.** This trips up almost everyone coming from SQL Server. In PostgreSQL, unquoted table/column names are automatically folded to lowercase. If you write `SELECT * FROM Medicines`, Postgres actually looks for a table called `medicines` (lowercase) — and if you ever _quote_ an identifier with mixed case (`"Medicines"`), it becomes case-sensitive and must match exactly, forever, every time you reference it. The safe convention (and what you'll see in nearly all Node.js + Postgres projects) is: always use `snake_case`, all-lowercase names for tables and columns (e.g., `stock_quantity`, not `StockQuantity`), and never rely on quoted mixed-case identifiers.

**Data types worth knowing.** A few Postgres types you'll run into that don't exist quite the same way in SQL Server:

- `TEXT` — a string type with no length limit, commonly used instead of always specifying `VARCHAR(n)`.
- `BOOLEAN` — a real native true/false type (SQL Server famously has no boolean type and uses `BIT` instead).
- `NUMERIC` / `DECIMAL` — exact-precision numbers, important for money values like medicine prices (never use floating-point types for currency).
- `TIMESTAMP WITH TIME ZONE` (often called `timestamptz`) — the generally recommended way to store date/time values so they're unambiguous across time zones; relevant for expiry dates and sale timestamps.
- `ENUM` types — Postgres lets you define a custom type that only accepts a fixed list of values (e.g., a `sale_status` enum restricted to `completed` / `cancelled`), enforced by the database itself rather than just application code. This is a very natural fit for our `role` field (admin/pharmacist/staff) and sale status field.
- `JSONB` — a binary, indexable JSON column type. Not essential for this project's core structured data (which fits neatly into normal relational tables), but good to know it exists for cases where a field's shape is genuinely flexible/unstructured.

**Constraints and syntax differences.** The _concepts_ (NOT NULL, UNIQUE, FOREIGN KEY, CHECK) are identical to what you already know from Section 11.6 and from SQL Server — only small syntax differences exist. One handy Postgres-specific feature: the `RETURNING` clause, which lets an `INSERT`, `UPDATE`, or `DELETE` statement return the affected row(s) directly, without a separate follow-up `SELECT` (SQL Server's closest equivalent is the `OUTPUT` clause, so this will feel familiar).

**Transactions.** The ACID transaction concepts from Section 13.2 work exactly the same way conceptually — `BEGIN`, then your statements, then `COMMIT` (or `ROLLBACK` on failure). This is what you'll rely on for the "process a sale" flow.

**Connecting from Node.js/Express.** There are two common approaches, and it's worth understanding both exist:

1. **Raw driver** — the `pg` package (often called **node-postgres**) is the standard low-level library for connecting Node.js directly to PostgreSQL and running SQL queries. It manages a **connection pool** (a reusable set of open database connections, since opening a fresh connection per request would be slow) via `pg.Pool`.
2. **ORM on top of Postgres** — tools like **Prisma**, **Sequelize**, or **TypeORM** (introduced conceptually in Section 11.7) sit on top of a driver like `pg` and let you avoid writing raw SQL for most operations, while still using PostgreSQL as the underlying engine. Prisma in particular has become extremely popular specifically for Node.js + PostgreSQL projects, with strong support for exactly the kinds of relationships (one-to-one, one-to-many, many-to-many) this project needs, plus a built-in migration system.

**Migrations.** Whichever route you take, you'll want a way to track database schema changes over time as version-controlled files (e.g., "add `expiry_date` column to `medicines`"), so the schema can be recreated or updated consistently across environments (your machine, a teammate's machine, production). This is usually referred to as **migrations**, and it's a standard companion concept to any ORM you pick.

**GUI/CLI tools.** Where SQL Server has SSMS (SQL Server Management Studio), PostgreSQL's rough equivalents are **pgAdmin** (a graphical tool) and **psql** (a command-line client) — useful for inspecting your tables and running ad-hoc queries while developing, separate from your actual application code.

**Environment configuration.** Your Node.js app connects to Postgres using a **connection string** (also called a connection URI), typically something like `postgresql://user:password@host:port/database_name`. Just like the JWT secret (Section 18), this belongs in an environment variable (e.g., `DATABASE_URL` in a `.env` file) — never hard-coded into your source files.

---

## 12. Inventory Management Concepts

### 12.1 Stock Tracking as a Business Rule Problem

At its core, inventory management is about keeping a **quantity** field accurate as it changes over time (medicines being added, sold, adjusted). The trickiest part isn't the field itself — it's making sure **every possible action that changes stock does so correctly and consistently**, especially when a sale happens (see Section 13).

### 12.2 Expiry Awareness

Because this is a pharmacy, expiry dates aren't just informational — they're a **business rule that blocks certain actions**. Conceptually, "expiry awareness" means:

- The system should be able to identify medicines that are expired, or expiring soon (useful for filtering/reporting, e.g., "show me all medicines expiring in the next 30 days").
- Before allowing a sale to proceed, the system must **check the expiry date** and reject the sale if the medicine has already expired — this is a business-logic validation, separate from basic input validation (Section 15), because it depends on _stored data_ (today's date vs. the medicine's expiry date), not just the shape of the incoming request.

### 12.3 Preventing Invalid Sales (Business Logic Guards)

The requirements explicitly call out: _"Prevent sales of out-of-stock or expired medicines."_ This is a great example of a **domain rule** — a rule that comes from the real-world business (pharmacy operations), not from generic software patterns. Implementing this conceptually requires, at the moment a sale is attempted:

1. Look up the current stock quantity for the requested medicine.
2. Check the requested sale quantity does not exceed available stock.
3. Check the medicine's expiry date is in the future.
4. Only if both checks pass, allow the sale to proceed (and then reduce stock — Section 13).

This kind of rule lives in your **service layer** (Section 17) — not in the database, and not in simple request validation — because it requires business reasoning combined with a database lookup.

### 12.4 Deactivation vs Deletion

As mentioned earlier, "deactivating" a medicine (flipping an `isActive` flag to false) instead of deleting its row is the standard real-world approach, because:

- Historical sales and prescriptions still need to reference that medicine's data — deleting it would break that history or force awkward orphaned references.
- Deactivated medicines can be excluded from "available medicines" listings without losing any data.

---

## 13. Sales & Billing Concepts

### 13.1 A Sale as a Multi-Step Business Transaction

Processing a sale isn't just "insert one row into a Sales table." It typically involves several coordinated steps:

1. Validate that the requested medicines exist, are active, in stock, and not expired (Section 12.3).
2. Create the `Sale` record itself (with references to the customer and the staff member processing it).
3. Create the related `SaleMedicine` junction rows (one per medicine included in the sale, storing quantity and price at time of sale).
4. **Reduce the stock quantity** on each medicine by the amount sold.
5. Set the sale's status (e.g., `completed`).

### 13.2 Why This Needs to Be Atomic (Database Transactions)

Here's a subtle but crucial concept: what happens if step 4 (reducing stock) fails halfway through — say, after updating one medicine's stock but before updating a second one in the same sale? You'd end up with **inconsistent data**: a sale record exists, but stock wasn't properly reduced for all items.

This is exactly the problem that **database transactions** solve. A transaction lets you group multiple database operations into a single all-or-nothing unit: either _everything_ succeeds and is saved, or if _anything_ fails partway through, _everything_ is rolled back as if nothing happened. This property is part of what's called **ACID** compliance in relational databases:

- **Atomicity** — all operations in the transaction succeed, or none do.
- **Consistency** — the database moves from one valid state to another valid state.
- **Isolation** — concurrent transactions don't interfere with each other's intermediate states.
- **Durability** — once committed, the changes are permanently saved even if the system crashes right after.

For our project, wrapping the entire "process a sale" sequence in a transaction is a core design decision — it directly protects the inventory accuracy the requirements ask for.

### 13.3 Sale Status Tracking

The requirements mention supporting statuses like `completed` and `cancelled`. Conceptually, a **status field** models the _lifecycle_ of a sale over time. A few things worth thinking through at the requirements level (even without implementing yet):

- What happens to stock if a completed sale is later **cancelled**? (Conceptually: stock should likely be restored — another good example of business logic that needs careful thought, not just a status flip.)
- Should certain roles be restricted from cancelling sales (an RBAC + business-logic overlap)?

Thinking through these "what if" scenarios during requirements analysis (Section 2) — _before_ coding — is exactly what separates a robust system from a fragile one.

### 13.4 Associating Sales with Staff and Customers

Every sale needs to answer "who processed this?" and "who was it for?" — these are simple **one-to-many** relationships (Section 11.4): one staff member can process many sales, one customer can be part of many sales. Recording this is essential for accountability, reporting, and the filtering requirements discussed next.

---

## 14. Pagination, Filtering & Sorting

### 14.1 Why Pagination Exists

Imagine the `GET /medicines` endpoint with 10,000 medicines in the database. Returning _all_ of them in a single response would be slow, waste bandwidth, and overwhelm whatever frontend is displaying the data. **Pagination** solves this by returning data in smaller "pages" or "chunks."

### 14.2 Offset-Based Pagination (the common approach for this kind of project)

The client sends query parameters like:

```
GET /medicines?page=2&limit=20
```

- `limit` — how many records per page
- `page` — which page number to return

On the backend, this conceptually translates into "skip the first (page-1)×limit records, then take the next `limit` records" — most databases/ORMs support this directly (often called `offset` and `take`/`limit` under the hood).

A well-designed paginated response also typically includes **metadata**, such as:

```
{
  "data": [ ...medicines... ],
  "meta": {
    "totalRecords": 245,
    "currentPage": 2,
    "totalPages": 13,
    "limit": 20
  }
}
```

This metadata lets the frontend build page-navigation UI without needing a separate request just to count total records.

### 14.3 Filtering

**Filtering** lets the client narrow down results based on specific criteria, expressed as query parameters. For our project, the requirements call out filters like:

- **Availability** (e.g., `?inStock=true` — only show medicines currently in stock)
- **Expiry range** (e.g., `?expiryFrom=2026-01-01&expiryTo=2026-06-30`)
- **Date range** (for sales/prescriptions — e.g., `?dateFrom=...&dateTo=...`)
- **Customer** (e.g., `?customerId=...` — only show prescriptions/sales for a specific customer)
- **Staff** (e.g., `?staffId=...` — only show sales processed by a specific staff member)

Conceptually, each filter translates into an additional condition added to the underlying database query (a `WHERE` clause, in SQL terms) — and multiple filters can typically be combined together (AND logic) in a single request.

### 14.4 Sorting

**Sorting** lets the client control the _order_ of returned results, usually via a query parameter like:

```
GET /medicines?sortBy=expiryDate&order=asc
```

This tells the backend which column to sort by (`sortBy`) and in which direction (`order` — ascending or descending). A well-designed API often restricts `sortBy` to a specific allow-list of safe, indexed fields, rather than letting the client sort by _any_ arbitrary field — both for performance and security reasons (never directly interpolate user input into a raw query without validation).

### 14.5 Combining All Three

In a real request, pagination, filtering, and sorting typically all work together, e.g.:

```
GET /medicines?page=1&limit=10&inStock=true&sortBy=expiryDate&order=asc
```

This reads naturally as: _"Give me page 1, 10 results per page, only in-stock medicines, sorted by soonest-to-expire first."_ Designing your query-building logic to cleanly combine all three (rather than treating them as separate, conflicting features) is an important backend design skill.

---

## 15. Validation & DTOs

### 15.1 Why Validation Matters

Never trust data coming from a client — whether it's a malicious user or just an honest mistake (a typo, a missing field, wrong data type), incoming request data must always be checked before your business logic or database ever touches it. Skipping this leads to crashes, corrupted data, or security vulnerabilities.

### 15.2 What Is a DTO?

**DTO** stands for **Data Transfer Object**. It's a concept (not a Node.js-specific feature) describing a clearly defined "shape" that data must match when moving between layers of your system — most commonly, the shape of the data a client is expected to send in a request body.

For example, a "Create Medicine" DTO conceptually defines:

- `name`: required, string
- `stockQuantity`: required, number, must be zero or greater
- `expiryDate`: required, valid date, must be in the future
- `supplierId`: required, must reference an existing supplier

Defining this shape explicitly (rather than just hoping the client sends the right thing) gives you a single, clear contract to validate against — and it also serves as living documentation of exactly what each endpoint expects.

### 15.3 How Validation Fits Into Express (Conceptually)

In an Express app, validation is typically implemented as **middleware** (Section 4.3) that runs _before_ the route's main controller logic:

1. The request body arrives.
2. Validation middleware checks it against the DTO's rules.
3. If validation fails, the middleware immediately responds with a `400 Bad Request` and a clear message describing exactly what was wrong — the request never reaches your business logic.
4. If validation passes, `next()` is called, and the request proceeds normally.

In the Node.js ecosystem, libraries like `class-validator` (often paired with `class-transformer`), `zod`, or `joi` are commonly used to define these validation rules declaratively, rather than writing manual `if` checks for every field by hand.

### 15.4 Validation vs Business Logic — An Important Distinction

It's worth clearly separating two different kinds of "checking":

- **Validation** = checking the _shape and format_ of input data (Is `stockQuantity` actually a number? Is `expiryDate` a valid date?) — this doesn't require looking anything up in the database.
- **Business logic checks** = checking _rules that depend on stored data or domain knowledge_ (Is this specific medicine currently out of stock? Has this specific supplier already been deactivated?) — this requires querying the database and understanding pharmacy-specific rules.

Beginners often blur these together, but keeping them conceptually (and often structurally) separate leads to much cleaner code — validation belongs in middleware near the "edge" of the request, while business rules belong deeper in your service layer (Section 17).

---

## 16. Error Handling

### 16.1 Why Centralized Error Handling?

Without a centralized approach, every single route handler would need its own `try/catch` block with manually duplicated logic for formatting error responses — messy and inconsistent. Express supports a special kind of middleware specifically for this: an **error-handling middleware**, recognizable because it takes **four** parameters instead of the usual three: `(err, req, res, next)`.

This special middleware is defined _once_, placed at the very end of your middleware chain, and Express automatically routes any error there — whether it came from a thrown exception, a rejected Promise (when using `async/await` with proper error-forwarding), or an explicit `next(err)` call anywhere in your app.

### 16.2 Custom Error Classes

A very common, clean pattern is to define your own custom error classes (extending JavaScript's built-in `Error`) that carry extra information like an HTTP status code, e.g., a conceptual `NotFoundError`, `ValidationError`, `UnauthorizedError`. Throwing these specific, meaningful errors throughout your business logic (e.g., `throw new NotFoundError('Medicine not found')`) lets your centralized error handler inspect the error's type/status and format an appropriate, consistent response automatically — instead of scattering `res.status(...).json(...)` calls everywhere.

### 16.3 Consistent Error Response Format

A well-designed API returns errors in a predictable, consistent JSON shape every single time, e.g.:

```
{
  "success": false,
  "statusCode": 404,
  "message": "Medicine not found",
  "errors": []
}
```

Consistency here matters enormously for whoever consumes your API (a frontend developer, or you demoing this project) — they can write generic error-handling logic once, trusting that every error from your API "looks the same," rather than needing special-case handling per endpoint.

### 16.4 Meaningful Messages, Not Leaky Details

Error messages should be genuinely helpful to the API consumer ("Stock quantity cannot be negative") without leaking internal implementation details that could help an attacker (e.g., raw database error messages, stack traces, or SQL syntax) — those belong only in server-side logs, never in the client-facing response, especially in production.

---

## 17. Application Architecture (Layered Design)

### 17.1 Why Organize Code Into Layers?

As a project grows, cramming database queries, business rules, and request/response handling all into one giant route handler function becomes unmanageable and hard to test. A **layered architecture** splits responsibilities into distinct layers, each with a single clear job. A very common pattern (sometimes loosely called MVC-inspired, though APIs don't literally have a "View") is:

| Layer                         | Responsibility                                                                                                                           |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **Routes**                    | Define URL patterns and HTTP methods; wire up middleware and controllers                                                                 |
| **Middleware**                | Cross-cutting concerns: auth, validation, error handling (Sections 4.3, 15, 16)                                                          |
| **Controllers**               | Receive the request, call the right service function(s), and shape the HTTP response — thin, no business logic                           |
| **Services**                  | The actual business logic (e.g., "process a sale," "check if a medicine can be sold") — this is where rules from Sections 12 and 13 live |
| **Models / Repository layer** | Talk directly to the database (often handled by the ORM) — fetching, saving, and updating records                                        |

### 17.2 Why This Separation Matters

- **Testability**: business logic in the service layer can be tested independently of HTTP concerns.
- **Reusability**: the same service function could be reused by multiple controllers, or even a background job, without duplicating logic.
- **Clarity**: when something breaks, you know exactly _which layer_ to look in — a wrong error message points you to controllers, a wrong business rule points you to services, a wrong query points you to the model/repository layer.
- **Team collaboration**: different people can work on different layers with a clear, agreed-upon contract between them.

### 17.3 Applying This to Our Project

For example, the "process a sale" flow (Section 13.1) would be organized like this conceptually:

- **Route**: `POST /sales` → points to `salesController.createSale`
- **Middleware chain**: auth guard → role guard (Pharmacist/Staff) → validation middleware (DTO for sale creation)
- **Controller**: extracts validated data from `req.body` and the authenticated user from `req.user`, calls `salesService.processSale(...)`, and returns the formatted response
- **Service**: contains all the real logic — checking stock/expiry, wrapping the operation in a database transaction, reducing stock, creating records
- **Model/Repository**: the actual database read/write calls that the service relies on

---

## 18. Security Best Practices Summary

A quick consolidated reference of security concepts relevant to this project (most already discussed in depth above, gathered here for easy review):

- **Hash passwords** with bcrypt — never store or log plain-text passwords (Section 9)
- **Sign and verify JWTs** with a strong secret key, and set sensible expiry times (Section 7)
- **Never trust client input** — always validate (Section 15)
- **Enforce RBAC** on every protected route, not just at the UI level — the API itself must never assume the frontend already blocked unauthorized actions (Section 8)
- **Use environment variables** (e.g., a `.env` file, never committed to version control) for secrets like the JWT signing key and database credentials — never hard-code secrets into source files
- **Return generic error messages** in production for unexpected server errors — don't leak stack traces or database details to clients (Section 16.4)
- **Rate limiting** (conceptually) — restricting how many requests a client can make in a given time window, to protect against brute-force login attempts or abuse
- **CORS (Cross-Origin Resource Sharing)** — a browser security mechanism; your Express app needs to explicitly allow which frontend origins are permitted to call your API

---

## 19. Glossary of Key Terms

| Term                          | Plain-English Meaning                                                                                                         |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **API**                       | A set of URLs/endpoints that let programs talk to each other                                                                  |
| **Node.js**                   | A runtime that lets JavaScript run outside the browser, e.g. on a server                                                      |
| **Express.js**                | A framework built on Node.js for building web servers/APIs easily                                                             |
| **Middleware**                | A function that runs between a request coming in and the final response                                                       |
| **Event Loop**                | The mechanism that lets Node.js handle many operations without blocking                                                       |
| **REST**                      | A standard style of designing APIs around resources and HTTP methods                                                          |
| **JWT**                       | A signed token that proves who a user is, without the server storing sessions                                                 |
| **Authentication**            | Verifying _who_ someone is                                                                                                    |
| **Authorization**             | Verifying _what_ someone is allowed to do                                                                                     |
| **RBAC**                      | Assigning permissions to roles, then assigning roles to users                                                                 |
| **Hashing**                   | A one-way scrambling of data (used for passwords), can't be reversed                                                          |
| **Salting**                   | Adding random data to a password before hashing, to make attacks harder                                                       |
| **Primary Key**               | The unique identifier for a row in a database table                                                                           |
| **Foreign Key**               | A column that links to a primary key in another table                                                                         |
| **One-to-One**                | One record relates to exactly one other record                                                                                |
| **One-to-Many**               | One record relates to many records in another table                                                                           |
| **Many-to-Many**              | Records on both sides can relate to multiple records on the other side                                                        |
| **Junction Table**            | A helper table used to implement many-to-many relationships                                                                   |
| **ORM**                       | A tool that lets you interact with your database using code objects instead of raw SQL                                        |
| **DTO**                       | A defined "shape" that incoming data must match                                                                               |
| **Pagination**                | Splitting large result sets into smaller pages                                                                                |
| **ACID Transaction**          | A group of database operations that all succeed together, or all fail together                                                |
| **Idempotent** _(bonus term)_ | An operation that produces the same result no matter how many times it's repeated (e.g., `GET` requests should be idempotent) |

---

## 20. Summary & Study Checklist

Use this checklist to self-test your understanding before moving to implementation. You should be able to explain each item **in your own words, out loud**, without looking back at the document:

- [ ] Difference between functional and non-functional requirements
- [ ] How Node.js handles many requests on a single thread (event loop, non-blocking I/O)
- [ ] What middleware is and why Express is built around it
- [ ] REST conventions: resources, HTTP methods, status codes, statelessness
- [ ] Difference between authentication and authorization
- [ ] The three parts of a JWT and what each does
- [ ] Why JWTs don't need a database lookup to be verified
- [ ] How RBAC middleware ("guards") protect routes
- [ ] Why passwords are hashed (with bcrypt/salting), not encrypted
- [ ] The difference between One-to-One, One-to-Many, and Many-to-Many relationships, with this project's real examples for each
- [ ] Why junction tables are needed for many-to-many relationships, and what extra data they can hold
- [ ] Why medicines get "deactivated," not deleted
- [ ] Why processing a sale needs a database transaction (ACID/atomicity)
- [ ] How pagination, filtering, and sorting work together via query parameters
- [ ] The difference between validation (DTOs) and business logic checks
- [ ] Why centralized error-handling middleware is better than scattered try/catch blocks
- [ ] The purpose of each layer in a layered architecture (routes → middleware → controllers → services → models)

Once every box here feels natural to explain, you have a genuinely solid conceptual foundation — implementation will simply be translating this understanding into code, one small piece at a time.
