# FastAPI — Complete In-Depth Notes

A structured, in-depth reference for learning FastAPI — core concepts, request/response handling, dependency injection, async, security, databases, testing, deployment, and a glossary of technical terms used throughout.

---

## 1. What is FastAPI & Why It Matters

- **FastAPI** is a modern, high-performance Python web framework for building APIs, built on top of **Starlette** (for the web/ASGI parts) and **Pydantic** (for data validation/serialization).
- Key selling points:
  - **Speed**: one of the fastest Python frameworks available, comparable to Node.js/Go frameworks, because it's built on **ASGI** (async-native) rather than the older synchronous WSGI model.
  - **Automatic interactive docs**: generates **OpenAPI** (Swagger) schema automatically from your code (type hints), giving you a browsable `/docs` (Swagger UI) and `/redoc` (ReDoc) interface for free, with zero extra configuration.
  - **Type-hint driven**: uses standard Python type hints to define request/response schemas, get automatic data validation, and get editor autocompletion — one source of truth (your function signature) drives validation, serialization, and docs simultaneously.
  - **Data validation built-in**: powered by **Pydantic** — invalid input is automatically rejected with a clear `422 Unprocessable Entity` error before your business logic even runs.
  - **Async-native**: first-class support for `async def` path operations, letting you handle high-concurrency I/O-bound workloads (calling external APIs, DB queries) efficiently.
- Compared to alternatives:
  - **Flask**: simpler, synchronous (WSGI) by default, more manual work for validation/docs (though extensions like Flask-RESTX exist) — FastAPI is generally preferred for new API-first projects needing async/high performance.
  - **Django REST Framework**: more batteries-included (ORM, admin panel, auth) but heavier and traditionally synchronous — good for full-featured monolithic web apps; FastAPI is leaner and more focused purely on building fast APIs.

---

## 2. Core Concepts & Technical Terms

### 2.1 WSGI vs ASGI
- **WSGI (Web Server Gateway Interface)**: the traditional synchronous standard interface between Python web servers and web applications (used by Flask, Django by default). Each request is handled by blocking a worker (thread/process) until it completes — to handle concurrency, you need many worker processes/threads, which is memory/resource-heavy.
- **ASGI (Asynchronous Server Gateway Interface)**: the modern successor standard, supporting both synchronous **and** asynchronous applications. Enables a single worker/event loop to handle many concurrent connections efficiently (especially I/O-bound work like waiting on a DB or external API call) without needing a thread per request — this is what allows FastAPI to be highly concurrent with fewer resources.
- FastAPI apps run via an ASGI server (see Uvicorn below), not directly like a plain script.

### 2.2 Starlette
- The lightweight ASGI framework/toolkit that FastAPI is built on top of — it provides the core web-framework machinery: routing, middleware support, WebSocket handling, background tasks, testing utilities, and the request/response objects.
- FastAPI adds on top of Starlette: the Pydantic-based data validation/serialization layer, automatic OpenAPI schema generation, and the dependency injection system.
- Practical implication: anything Starlette can do (e.g., raw `Request`/`Response` objects, WebSockets, low-level middleware), FastAPI can do too, since FastAPI *is* a Starlette application under the hood.

### 2.3 Pydantic
- A Python data validation and settings-management library using Python type hints. You define a `BaseModel` subclass with typed fields, and Pydantic automatically validates, parses/coerces, and serializes data against that schema.
- FastAPI uses Pydantic models to define request bodies, response bodies, and even environment/config settings.
- Pydantic v2 (the current major version) rewrote its validation core in Rust (via `pydantic-core`), giving major performance improvements over v1.
- Validation errors raised by Pydantic are automatically caught by FastAPI and turned into a structured `422` HTTP response listing exactly which fields failed and why — you don't have to write that error-handling logic yourself.

### 2.4 Uvicorn (and Gunicorn)
- **Uvicorn**: a lightning-fast ASGI server implementation (built on `uvloop` and `httptools`) — this is the actual process that runs your FastAPI app and speaks HTTP, translating incoming requests into ASGI calls into your app. You typically run `uvicorn main:app --reload` during development.
- **Gunicorn**: a mature, battle-tested process manager traditionally used for WSGI apps; in production, it's common to run **Gunicorn managing multiple Uvicorn worker processes** (`gunicorn -k uvicorn.workers.UvicornWorker`) to get Gunicorn's robust process management (auto-restarting crashed workers, graceful reloads) combined with Uvicorn's ASGI/async performance.
- **Hypercorn**: an alternative ASGI server supporting HTTP/2 and HTTP/3, sometimes used instead of Uvicorn when those protocol features are needed.

### 2.5 Type Hints as the Foundation
- FastAPI leans heavily on Python's type hint syntax (`def get_item(item_id: int)`), introduced in PEP 484 and expanded since — this isn't just documentation, FastAPI actually **reads these hints at runtime** to know how to validate, convert, and serialize data.
- Example: declaring `item_id: int` on a path parameter means FastAPI will automatically convert the incoming string from the URL into an `int`, and reject the request with a clear error if it isn't a valid integer — no manual `int(request.args['item_id'])` and manual error handling needed.

---

## 3. Path Operations & Routing

- A **path operation** is FastAPI's term for a function decorated to handle a specific HTTP method + path combination:
```python
from fastapi import FastAPI
app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}
```
- **Path parameters**: parts of the URL path itself, declared in `{curly_braces}` and matched to function parameters by name (e.g., `item_id` above). Type-annotate them for automatic validation/conversion.
- **Query parameters**: function parameters *not* declared in the path are automatically treated as query string parameters (e.g., `?skip=0&limit=10`), with optional defaults (`skip: int = 0`) making them optional in the request.
- **Path ordering matters**: more specific/fixed paths must be declared *before* more generic dynamic ones if they could otherwise match ambiguously (e.g., `/users/me` must be declared before `/users/{user_id}`, or `/users/me` would incorrectly be interpreted as a request for a user with ID `"me"`).
- **APIRouter**: lets you organize path operations into separate modules/files (e.g., a `users.py` router, an `items.py` router) and then include them into the main app (`app.include_router(users.router, prefix="/users", tags=["users"])`) — essential for structuring any non-trivial FastAPI project instead of putting everything in one file.
- **Path convertors**: special path parameter types like `{file_path:path}` that allow the parameter to contain slashes (matching an entire sub-path), useful for things like serving arbitrary file paths.
- **Tags**: group related endpoints together in the auto-generated docs UI (purely for documentation organization, no functional effect on routing).

---

## 4. Request Handling

### 4.1 Request Body & Pydantic Models
- To accept a JSON request body, declare a Pydantic model and use it as a parameter type:
```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    is_offer: bool | None = None

@app.post("/items/")
async def create_item(item: Item):
    return item
```
- FastAPI automatically: reads the request body as JSON, validates it against the `Item` schema, converts it into an actual `Item` Python object (with attribute access like `item.name`), and returns a `422` error automatically if validation fails.
- **Field validation**: Pydantic's `Field()` lets you add constraints (`price: float = Field(gt=0)`), and custom **validators** (`@field_validator` in Pydantic v2) let you write custom validation logic beyond simple types/constraints.
- **Nested models**: Pydantic models can contain other Pydantic models as fields, and lists of them, letting you validate deeply nested JSON structures declaratively.

### 4.2 Query, Path, Header, Cookie Parameters with Metadata
- FastAPI provides `Query()`, `Path()`, `Header()`, `Cookie()` helper functions to add validation/metadata to these parameter types (min/max length, regex patterns, descriptions for docs, deprecation flags), e.g.:
```python
from fastapi import Query
async def read_items(q: str | None = Query(default=None, max_length=50)):
    ...
```

### 4.3 Form Data & File Uploads
- **Form data** (`application/x-www-form-urlencoded` or `multipart/form-data`): use FastAPI's `Form()` to declare fields coming from an HTML form submission rather than JSON.
- **File uploads**: use `UploadFile` (preferred over the simpler `bytes` type for larger files, since `UploadFile` uses a spooled file under the hood and doesn't load the entire file into memory at once) — `async def create_file(file: UploadFile)`.

### 4.4 Request Validation Order & Automatic Errors
- FastAPI validates path parameters, then query parameters, then the request body, using each declared type's rules; the first validation failures encountered are collected and returned together in a single structured `422` response — the client gets one comprehensive error response rather than having to fix and resubmit one field error at a time.

---

## 5. Response Handling

### 5.1 Response Models
- Declaring `response_model=SomeModel` on a path operation decorator tells FastAPI to **filter/shape the output** to match that schema — even if your function returns extra internal fields (e.g., a hashed password field on a DB object), only the fields declared in the response model are actually serialized and sent to the client. This is a key security/data-hygiene mechanism, not just documentation.
- `response_model_exclude_unset=True` can be used to omit fields that weren't explicitly set (useful for PATCH-style partial updates where you don't want default values overwriting things unintentionally in the response representation).

### 5.2 Status Codes
- Set explicit HTTP status codes via the decorator (`@app.post("/items/", status_code=201)`) or by returning a `Response` object / raising `HTTPException` with a specific code — FastAPI includes the `starlette.status` module with named constants (`status.HTTP_404_NOT_FOUND`) to avoid hardcoding magic numbers.

### 5.3 Error Handling
- **`HTTPException`**: the standard way to return an error response from within a path operation — `raise HTTPException(status_code=404, detail="Item not found")` — FastAPI catches this and converts it into a proper JSON error response.
- **Custom exception handlers**: register a handler for a custom exception class (`@app.exception_handler(MyCustomError)`) to centralize how a specific error type is converted into an HTTP response across the whole app, instead of repeating `try/except` + manual response-building in every path operation.
- **Validation exception handling**: you can override FastAPI's default handler for `RequestValidationError` to customize the shape/format of validation error responses if the default format doesn't match your API's conventions.

---

## 6. Dependency Injection (FastAPI's `Depends`)

- FastAPI has a built-in **Dependency Injection (DI)** system — a way to declare reusable pieces of logic (a "dependency") that FastAPI automatically calls and injects the result of into your path operation function, via `Depends()`.
```python
async def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/items/")
async def read_items(db: Session = Depends(get_db)):
    ...
```
- **Why it matters**: promotes code reuse (shared logic like "get the current authenticated user", "get a DB session", "check pagination params" is written once and reused across many endpoints) and keeps path operation functions focused on their actual business logic (Single Responsibility Principle in practice) rather than repeating boilerplate.
- **Dependencies with `yield`**: a dependency can use `yield` instead of `return` to define setup/teardown logic (like the DB session example above) — code before `yield` runs before the path operation, code after `yield` (typically in a `finally` block) runs after the response has been sent, useful for cleanup (closing DB sessions, releasing resources) that should happen regardless of whether the request succeeded or raised an exception.
- **Sub-dependencies**: a dependency can itself depend on other dependencies (`Depends()` inside a dependency function) — FastAPI resolves the whole chain automatically.
- **Dependency caching**: by default, if the same dependency is required multiple times within handling a single request (e.g., used directly and also as a sub-dependency of another dependency), FastAPI calls it only **once** per request and reuses the cached result (can be disabled with `use_cache=False` if you genuinely need fresh calls).
- **Global/router-level dependencies**: dependencies can be applied to an entire `APIRouter` or the whole `FastAPI` app (e.g., `dependencies=[Depends(verify_api_key)]`) to enforce something (like auth checks) across many endpoints without repeating `Depends()` in every single function signature.
- **`Depends` vs class-based dependencies**: a dependency can also be a callable class (with `__call__`), which is useful when the dependency itself needs configurable parameters (e.g., a reusable `RateLimiter(times=5, seconds=60)` dependency class).

---

## 7. Async / Await in FastAPI

- FastAPI supports both `async def` and regular `def` path operations, and it's important to know how each is actually executed:
  - **`async def` path operations** run directly on the main event loop. This is efficient **only if everything inside is genuinely non-blocking** (e.g., using an async DB driver, `httpx.AsyncClient` for HTTP calls) — if you accidentally call a blocking/synchronous function (like a normal `requests.get()` or a blocking DB driver) inside an `async def` function, you **block the entire event loop**, stalling every other concurrent request being served by that worker, which is a common and serious performance bug.
  - **Regular `def` path operations** are automatically run by FastAPI in a separate thread pool (via Starlette), so blocking code inside them doesn't block the main event loop — this is actually the *safer* default if you're calling synchronous/blocking libraries and don't want to (or can't easily) convert everything to async.
- **Rule of thumb**: use `async def` when everything you call inside is `async`-compatible (async DB drivers like `asyncpg`, `motor` for MongoDB, `httpx.AsyncClient`); use plain `def` if you're relying on traditional synchronous libraries (e.g., the classic synchronous `requests` library, or a synchronous ORM session) so FastAPI can safely offload it to a thread.
- **`await`**: used inside `async def` functions to pause execution until an awaited coroutine completes, *without* blocking the event loop — other requests can be processed by the event loop during that wait.
- **Event loop**: the core mechanism of asynchronous programming in Python (`asyncio`) — a single-threaded loop that manages and switches between many concurrent tasks/coroutines, running whichever one is ready to make progress next, and parking ones that are waiting on I/O.
- **`asyncio.gather()`**: lets you run multiple independent async calls concurrently and wait for all of them to finish (e.g., fetching data from three different services in parallel rather than sequentially) — a common performance pattern inside async FastAPI endpoints that call multiple downstream services.

---

## 8. Middleware

- **Middleware** is code that runs on *every* request before it reaches your path operation, and/or on every response before it's sent back to the client — used for cross-cutting concerns that shouldn't be repeated in every endpoint.
```python
@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    response.headers["X-Process-Time"] = str(time.time() - start_time)
    return response
```
- Common uses: logging every request, adding custom headers, measuring request timing, global error catching, CORS handling.
- **CORS (Cross-Origin Resource Sharing) middleware**: a very common built-in middleware (`CORSMiddleware`) needed whenever your API is called from a browser-based frontend on a different domain/port — without it, browsers block the frontend's requests due to the same-origin policy. You configure allowed origins, methods, headers, and credentials explicitly.
- **Middleware execution order**: middleware wraps around the request/response cycle like layers of an onion — the first middleware added is the outermost layer (runs first on the way in, last on the way out).
- Middleware vs Dependencies: middleware runs for **every** request/response regardless of route (good for truly global concerns); dependencies are more granular and can be applied selectively per-route/router, and can participate in FastAPI's DI system (e.g., returning a value used by the endpoint) in ways middleware cannot as cleanly.

---

## 9. Background Tasks

- FastAPI's `BackgroundTasks` lets you schedule work to run **after** the response has already been sent to the client — useful for tasks that shouldn't make the client wait (sending a confirmation email after signup, writing a log entry, triggering a non-critical notification).
```python
from fastapi import BackgroundTasks

def write_log(message: str):
    with open("log.txt", "a") as f:
        f.write(message)

@app.post("/send-notification/")
async def send_notification(email: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(write_log, f"notification sent to {email}")
    return {"message": "Notification sent"}
```
- **Important limitation**: `BackgroundTasks` run in the **same process** as your app, after the response — they are *not* a substitute for a real task queue. For anything long-running, resource-intensive, or requiring retries/durability (e.g., video processing, sending bulk emails, anything that must survive a server restart), use a proper distributed task queue (e.g., **Celery**, **RQ**, or an ASGI-native option like **ARQ**, often backed by Redis or RabbitMQ) instead — this connects directly back to the HLD "Message Queues / Asynchronous Processing" concepts, just applied at the application-framework level.

---

## 10. Security & Authentication

### 10.1 OAuth2 & Password Flow
- FastAPI has built-in utilities (`fastapi.security`) implementing standard OAuth2 flows, most commonly the **Password flow with Bearer tokens** for a typical username/password login API:
  1. Client submits username/password to a `/token` endpoint (using `OAuth2PasswordRequestForm`).
  2. Server verifies credentials, generates a **JWT** (see glossary below) containing the user's identity, and returns it as an access token.
  3. Client includes `Authorization: Bearer <token>` on subsequent requests.
  4. A dependency (`OAuth2PasswordBearer`) extracts and validates the token on protected routes, raising `401 Unauthorized` automatically if missing/invalid, and injects the decoded user info into the endpoint via `Depends()`.
- This maps directly onto the HLD Security concepts (Step 10) — FastAPI's security utilities are essentially a convenient, type-hinted way to implement those token/OAuth2 patterns without hand-rolling all the boilerplate.

### 10.2 Password Hashing
- Never store plaintext passwords — use a library like `passlib` (with `bcrypt`) to hash passwords before storing, and verify by hashing the login attempt and comparing hashes, not by decrypting anything (hashing is one-way).

### 10.3 CORS, HTTPS, and Scopes
- Combine `CORSMiddleware` (Section 8) with proper HTTPS termination (usually handled by a reverse proxy like Nginx or a cloud load balancer in front of Uvicorn, not by Uvicorn itself in production) for a secure deployment.
- **OAuth2 scopes** can be declared on `Depends()` calls to restrict specific endpoints to tokens carrying specific permissions (e.g., `Security(get_current_user, scopes=["items:write"])`), enabling fine-grained authorization beyond simple "logged in or not."

---

## 11. Database Integration

- FastAPI is database-agnostic — it doesn't ship its own ORM, but integrates cleanly with:
  - **SQLAlchemy** (traditional, synchronous by default, or with its newer **async** support using `asyncpg`/`aiomysql` drivers) — the most common choice for relational DBs.
  - **SQLModel**: a library (by the same author as FastAPI) that combines SQLAlchemy and Pydantic into a single model definition, reducing duplication between your DB schema and your API schema.
  - **Motor**: the standard async driver for **MongoDB** when using `async def` endpoints.
  - **Tortoise ORM**, **Databases** (a lightweight async query library), and others.
- **Session-per-request pattern**: the standard pattern is a `Depends()`-based dependency (using `yield`, see Section 6) that creates a DB session at the start of a request and closes it after — ensuring sessions aren't leaked or shared unsafely across requests.
- **Async vs sync DB access ties back to Section 7**: if you use a synchronous DB driver/ORM session inside an `async def` endpoint without offloading it, you block the event loop — either use `def` endpoints (auto-threaded) with a sync ORM, or fully commit to async DB drivers throughout.
- **Migrations**: schema changes are typically managed with **Alembic** (SQLAlchemy's companion migration tool), which tracks incremental schema changes as versioned scripts, similar in spirit to how the HLD notes describe safe data migration strategies.

---

## 12. Testing FastAPI Applications

- FastAPI provides a `TestClient` (built on `httpx`) that lets you make requests directly against your app in tests without running a real server:
```python
from fastapi.testclient import TestClient
client = TestClient(app)

def test_read_item():
    response = client.get("/items/1")
    assert response.status_code == 200
    assert response.json() == {"item_id": 1}
```
- **Overriding dependencies in tests**: `app.dependency_overrides[get_db] = get_test_db` lets you swap real dependencies (like a production DB connection) for test doubles (like an in-memory/test database or a mock) cleanly, without touching your actual endpoint code — a direct, practical application of the Dependency Inversion Principle from the LLD notes.
- **Async tests**: for testing `async def` endpoints/dependencies directly (not just via `TestClient`, which handles the async bridging for you), use `pytest-asyncio` along with an async HTTP client (`httpx.AsyncClient`).
- **Fixtures**: `pytest` fixtures are commonly used to set up reusable test resources (a test DB session, a test client, seeded test data) shared across multiple test functions.

---

## 13. Deployment

- **Development**: `uvicorn main:app --reload` — the `--reload` flag auto-restarts the server on code changes; never use `--reload` in production (it adds overhead and is meant purely for dev convenience).
- **Production process model**: run multiple worker processes to utilize multiple CPU cores (a single Python process, even async, is still limited by the Global Interpreter Lock for CPU-bound work and can only use one core) — typically via Gunicorn managing several Uvicorn workers, or Uvicorn's own `--workers N` flag, or higher-level tools like `uvicorn[standard]` and process managers such as `supervisord`/`systemd`.
- **Containerization**: FastAPI apps are commonly packaged into a **Docker** container (see the HLD notes on containerization) for consistent deployment — official base images exist, and a typical Dockerfile installs dependencies, copies the app, and runs it via Uvicorn/Gunicorn as the container's entrypoint.
- **Reverse proxy**: in production, a reverse proxy (Nginx, Traefik, or a cloud load balancer) usually sits in front of your Uvicorn/Gunicorn workers to handle TLS termination, serve static files, and load-balance across multiple app instances/containers — connecting directly to the HLD "Reverse Proxy" and "Load Balancer" concepts.
- **Health checks**: expose a simple `/health` endpoint that container orchestrators (like Kubernetes) can poll to know if an instance is alive/ready — ties to the HLD Monitoring section's idea of proactively detecting unhealthy instances so they can be restarted/removed from rotation automatically.
- **Environment-based configuration**: use Pydantic's `BaseSettings` (in the separate `pydantic-settings` package for Pydantic v2) to load configuration (DB URLs, secret keys, feature flags) from environment variables in a typed, validated way, rather than scattering `os.environ.get(...)` calls throughout the codebase.

---

## 14. Glossary of Key Technical Terms

| Term | Meaning |
|---|---|
| **ASGI** | Asynchronous Server Gateway Interface — the async-capable standard interface between Python web servers and applications; what FastAPI/Starlette run on. |
| **WSGI** | Web Server Gateway Interface — the older, synchronous-only standard used by frameworks like classic Flask/Django. |
| **Starlette** | The lightweight ASGI toolkit/framework FastAPI is built on top of, providing routing, middleware, and the core request/response machinery. |
| **Pydantic** | A data validation/parsing library using Python type hints; FastAPI uses it to define and validate request/response schemas. |
| **Uvicorn** | A fast ASGI server used to actually run a FastAPI application and handle raw HTTP connections. |
| **Gunicorn** | A production-grade process manager, often used to run/manage multiple Uvicorn worker processes. |
| **Path operation** | FastAPI's term for a function bound to a specific HTTP method + URL path (e.g., `@app.get("/items/")`). |
| **Path parameter** | A dynamic segment of the URL path itself (e.g., `{item_id}` in `/items/{item_id}`). |
| **Query parameter** | Extra data supplied in the URL after `?` (e.g., `?limit=10`), mapped to non-path function arguments. |
| **Request body** | Data sent in the body of a request (typically JSON for APIs), parsed into a Pydantic model. |
| **Response model** | A Pydantic model used to define/filter the shape of data returned by an endpoint, regardless of what the function internally returns. |
| **Dependency Injection (DI)** | A pattern where reusable logic (`Depends()`) is declared once and automatically supplied to any endpoint/function that needs it, rather than manually instantiated everywhere. |
| **`yield`-based dependency** | A dependency function using `yield` instead of `return`, allowing setup code before and teardown/cleanup code after the request is handled. |
| **Middleware** | Code that runs around every request/response cycle globally (e.g., logging, CORS, timing headers). |
| **CORS** | Cross-Origin Resource Sharing — the browser security mechanism/HTTP headers that control whether a frontend on one domain is allowed to call an API on another domain. |
| **Background task** | Work scheduled to run after a response is sent, within the same process — not a substitute for a real task queue for heavy/long work. |
| **OpenAPI** | The specification format (formerly "Swagger") describing an API's endpoints, schemas, and parameters — FastAPI auto-generates this from your code. |
| **Swagger UI / ReDoc** | Interactive, browsable API documentation UIs auto-served by FastAPI at `/docs` and `/redoc`, generated from the OpenAPI schema. |
| **JWT (JSON Web Token)** | A self-contained, signed token format commonly used to represent an authenticated user's identity/claims in API requests (see HLD Security notes for full detail). |
| **OAuth2** | An authorization framework/protocol FastAPI's security utilities implement helpers for, commonly used for token-based login flows. |
| **Event loop** | The core async runtime construct (from Python's `asyncio`) that manages switching between many concurrent coroutines/tasks on a single thread. |
| **Coroutine** | A function defined with `async def` that can be paused (`await`) and resumed, without blocking the thread it runs on. |
| **Blocking call** | A synchronous operation that occupies the thread/event loop until it finishes — dangerous inside `async def` functions if not offloaded, since it stalls all other concurrent requests. |
| **ORM (Object-Relational Mapper)** | A library (e.g., SQLAlchemy) that lets you interact with a relational database using Python objects/classes instead of raw SQL strings. |
| **Migration** | A versioned, incremental change to a database schema, typically managed by a tool like Alembic. |
| **Pytest fixture** | A reusable setup/teardown unit in the `pytest` testing framework, used to provide shared test resources (DB sessions, clients, test data). |
| **Pydantic `BaseSettings`** | A Pydantic-based pattern for loading and validating application configuration from environment variables in a typed way. |
| **Uvicorn worker / multi-worker deployment** | Running multiple separate OS processes (each with its own event loop) so a FastAPI app can use multiple CPU cores, since a single async process is still limited to one core for CPU-bound work. |
