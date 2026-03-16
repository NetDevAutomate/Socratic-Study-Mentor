# FastAPI + HTMX + Jinja2 Best Practices (2026)

Research compiled from official FastAPI docs, HTMX docs, Alpine.js docs, fasthx library (704 stars),
aiosqlite PyPI, and community patterns. Targeted at the Socratic Study Mentor web application.

---

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [HTMX Patterns](#2-htmx-patterns)
3. [HTMX + Alpine.js Together](#3-htmx--alpinejs-together)
4. [FastAPI Dependency Injection](#4-fastapi-dependency-injection)
5. [Dual API: JSON + HTMX Endpoints](#5-dual-api-json--htmx-endpoints)
6. [Jinja2 Template Inheritance](#6-jinja2-template-inheritance)
7. [Authentication: HTTP Basic Auth](#7-authentication-http-basic-auth)
8. [Static Files + PWA Service Worker](#8-static-files--pwa-service-worker)
9. [WebSocket Support](#9-websocket-support)
10. [Performance](#10-performance)

---

## 1. Project Structure

**Source**: FastAPI official docs ("Bigger Applications - Multiple Files"), community consensus.

The recommended layout separates routes, templates, static files, and business logic:

```
study_web/
    __init__.py
    main.py                    # FastAPI app factory, mount statics, include routers
    config.py                  # Settings via pydantic-settings (BaseSettings)
    deps.py                    # Shared dependencies: db, auth, config, templates
    models.py                  # Pydantic models for request/response validation
    db.py                      # Database connection management (aiosqlite)
    routers/
        __init__.py
        flashcards.py          # /cards/* routes
        artefacts.py           # /artefacts/* routes (audio/video/PDF)
        dashboard.py           # /dashboard/* routes
        settings.py            # /settings/* routes
        feedback.py            # /feedback/* routes
        api.py                 # /api/* JSON-only endpoints
        ws.py                  # WebSocket endpoints
    templates/
        base.html              # Master layout with nav, HTMX/Alpine script tags
        partials/              # HTMX fragment templates (no <html>/<body>)
            _card.html
            _card_flip.html
            _artefact_item.html
            _progress_chart.html
            _search_results.html
            _modal.html
            _toast.html
        pages/                 # Full page templates (extend base.html)
            flashcards.html
            artefacts.html
            dashboard.html
            settings.html
            feedback.html
        components/            # Reusable Jinja2 macros
            _nav.html
            _pagination.html
            _form_field.html
    static/
        css/
            style.css
        js/
            app.js             # Alpine.js stores, HTMX config
        img/
        manifest.json          # PWA manifest
        sw.js                  # Service worker
    tests/
        conftest.py
        test_flashcards.py
        test_artefacts.py
        test_dashboard.py
```

### Key principles

- **One router per page/feature** -- each router is an `APIRouter` with its own prefix and tags.
- **Templates split into pages/ and partials/** -- pages extend `base.html`; partials are bare HTML
  fragments returned by HTMX endpoints.
- **deps.py centralises all `Depends()` callables** -- avoids circular imports.
- **config.py uses `pydantic-settings`** -- loads from env vars / `.env` file.

### main.py skeleton

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

from study_web.config import settings
from study_web.db import init_db, close_db
from study_web.routers import flashcards, artefacts, dashboard, settings as settings_router, feedback, api, ws


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: initialise DB connection
    app.state.db = await init_db(settings.db_path)
    yield
    # Shutdown: close DB connection
    await close_db(app.state.db)


app = FastAPI(title="Study Mentor Web", lifespan=lifespan)

# Static files
app.mount("/static", StaticFiles(directory="study_web/static"), name="static")

# Include routers
app.include_router(flashcards.router, prefix="/cards", tags=["flashcards"])
app.include_router(artefacts.router, prefix="/artefacts", tags=["artefacts"])
app.include_router(dashboard.router, prefix="/dashboard", tags=["dashboard"])
app.include_router(settings_router.router, prefix="/settings", tags=["settings"])
app.include_router(feedback.router, prefix="/feedback", tags=["feedback"])
app.include_router(api.router, prefix="/api", tags=["api"])
app.include_router(ws.router, prefix="/ws", tags=["websocket"])
```

---

## 2. HTMX Patterns

**Source**: HTMX official docs v2.x, htmx.org/examples.

### 2a. Partial page updates (the core pattern)

Every HTMX endpoint returns an HTML **fragment**, not a full page. The fragment replaces
a targeted DOM element.

```html
<!-- Button triggers GET, response replaces #card-container -->
<button hx-get="/cards/next"
        hx-target="#card-container"
        hx-swap="outerHTML">
    Next Card
</button>

<div id="card-container">
    {% include "partials/_card.html" %}
</div>
```

FastAPI endpoint returns the partial:

```python
@router.get("/next", response_class=HTMLResponse)
async def next_card(request: Request, db=Depends(get_db)):
    card = await get_random_card(db)
    return templates.TemplateResponse(
        request=request,
        name="partials/_card.html",
        context={"card": card},
    )
```

### 2b. Form submission

```html
<form hx-post="/feedback/submit"
      hx-target="#feedback-result"
      hx-swap="innerHTML"
      hx-indicator="#spinner">
    <textarea name="message" required></textarea>
    <button type="submit">Send</button>
    <span id="spinner" class="htmx-indicator">Sending...</span>
</form>
<div id="feedback-result"></div>
```

Server returns a success fragment or validation errors as HTML.

### 2c. Infinite scroll (artefact list)

The last element in the list triggers loading the next page when scrolled into view:

```html
<!-- Each page of results -->
{% for item in artefacts %}
<div class="artefact-item">{{ item.title }}</div>
{% endfor %}

<!-- Sentinel: triggers next page load when revealed -->
{% if has_more %}
<div hx-get="/artefacts/list?page={{ next_page }}"
     hx-trigger="revealed"
     hx-swap="afterend"
     hx-indicator="#load-more-spinner">
    <span id="load-more-spinner" class="htmx-indicator">Loading...</span>
</div>
{% endif %}
```

**Key**: use `hx-trigger="revealed"` on the last element. For `overflow-y: scroll`
containers, use `hx-trigger="intersect once"` instead.

### 2d. Live search

```html
<input type="search" name="q"
       placeholder="Search artefacts..."
       hx-post="/artefacts/search"
       hx-trigger="input changed delay:500ms, keyup[key=='Enter']"
       hx-target="#search-results"
       hx-indicator=".search-indicator">

<span class="search-indicator htmx-indicator">Searching...</span>
<div id="search-results"></div>
```

**Key patterns**:
- `delay:500ms` debounces input to avoid flooding the server.
- `changed` modifier prevents re-sending on arrow keys / same value.
- Comma-separated triggers allow Enter key as immediate submit.

### 2e. Modal dialogs

```html
<!-- Trigger button -->
<button hx-get="/artefacts/42/details"
        hx-target="body"
        hx-swap="beforeend">
    View Details
</button>
```

Server returns the modal fragment appended to `<body>`:

```html
<!-- partials/_modal.html -->
<div id="modal-overlay" class="modal-overlay"
     x-data="{ open: true }"
     x-show="open"
     x-transition:enter="transition ease-out duration-200"
     x-transition:enter-start="opacity-0"
     x-transition:enter-end="opacity-100"
     x-transition:leave="transition ease-in duration-150"
     x-transition:leave-start="opacity-100"
     x-transition:leave-end="opacity-0"
     @keydown.escape.window="open = false; setTimeout(() => $el.remove(), 200)">

    <div class="modal-underlay" @click="open = false; setTimeout(() => $el.remove(), 200)"></div>

    <div class="modal-content"
         x-show="open"
         x-transition:enter="transition ease-out duration-200"
         x-transition:enter-start="opacity-0 scale-95"
         x-transition:enter-end="opacity-100 scale-100">
        <h2>{{ artefact.title }}</h2>
        <p>{{ artefact.description }}</p>
        <button @click="open = false; setTimeout(() => $el.remove(), 200)"
                class="btn">Close</button>
    </div>
</div>
```

**Key**: Alpine.js handles the animation/close; HTMX handles fetching the modal content
from the server. This is the recommended split of concerns.

### 2f. Out-of-Band (OOB) swaps

Update multiple parts of the page from a single response:

```html
<!-- Primary response swapped into hx-target as normal -->
<div id="card-container">
    {% include "partials/_card.html" %}
</div>

<!-- OOB: also update the progress counter -->
<span id="progress-count" hx-swap-oob="true">
    {{ completed }} / {{ total }}
</span>

<!-- OOB: also update the streak display -->
<span id="streak-display" hx-swap-oob="true">
    Streak: {{ streak }}
</span>
```

**Use case**: After answering a flashcard, update the card AND the progress counter
AND the streak display in one response. No extra requests needed.

### 2g. History / URL updates

```html
<a hx-get="/cards/deck/python-basics"
   hx-target="#main-content"
   hx-push-url="true">
    Python Basics
</a>
```

**Critical rule**: If you push a URL, that URL MUST return a full page when accessed
directly (e.g., user refreshes or shares the link). Detect HTMX requests via the
`HX-Request` header and return partial vs full page accordingly.

### 2h. CSRF protection

Set CSRF token globally via `hx-headers` on the `<body>`:

```html
<body hx-headers='{"X-CSRF-Token": "{{ csrf_token }}"}'>
```

---

## 3. HTMX + Alpine.js Together

**Source**: HTMX docs "Scripting" section, Alpine.js docs, community patterns.

### The division of responsibility

| Concern | Tool | Why |
|---------|------|-----|
| Fetching data from server | HTMX | Server returns HTML fragments |
| Toggling UI state (dropdowns, tabs) | Alpine.js | No server round-trip needed |
| Animations & transitions | Alpine.js | `x-transition` is declarative CSS |
| Card flip animation | Alpine.js | Pure client-side state toggle |
| Form validation feedback | HTMX | Server validates, returns error HTML |
| Pomodoro timer display | Alpine.js | Client-side countdown, synced via WS |

### Script loading order

```html
<!-- base.html <head> -->
<script src="https://unpkg.com/htmx.org@2.0.4"
        integrity="sha384-..."
        crossorigin="anonymous"></script>
<script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
```

HTMX loads first (synchronous). Alpine loads deferred. They coexist without conflict
because Alpine operates on `x-*` attributes and HTMX on `hx-*` attributes.

### Reinitializing Alpine after HTMX swaps

When HTMX swaps new HTML into the DOM, Alpine needs to initialize any new `x-data`
elements. Alpine.js v3 handles this automatically via MutationObserver -- it detects
new `x-data` attributes added to the DOM. **No manual initialization needed.**

However, if you need to run custom JS after an HTMX swap:

```javascript
// app.js
document.addEventListener('htmx:afterSwap', (event) => {
    // Alpine auto-initializes x-data elements, but you can
    // run additional setup here if needed
    console.log('HTMX swapped:', event.detail.target.id);
});
```

### Flashcard flip animation (Alpine.js only, no server needed)

```html
<!-- partials/_card.html -->
<div x-data="{ flipped: false }"
     class="card-container perspective-1000"
     id="card-{{ card.id }}">

    <div class="card-inner transition-transform duration-500"
         :class="flipped ? 'rotate-y-180' : ''">

        <!-- Front face -->
        <div class="card-face card-front" x-show="!flipped">
            <p>{{ card.question }}</p>
            <button @click="flipped = true" class="btn">Show Answer</button>
        </div>

        <!-- Back face -->
        <div class="card-face card-back" x-show="flipped" x-transition>
            <p>{{ card.answer }}</p>
            <div class="rating-buttons">
                <button hx-post="/cards/{{ card.id }}/rate"
                        hx-vals='{"rating": 1}'
                        hx-target="#card-container"
                        hx-swap="outerHTML">
                    Hard
                </button>
                <button hx-post="/cards/{{ card.id }}/rate"
                        hx-vals='{"rating": 3}'
                        hx-target="#card-container"
                        hx-swap="outerHTML">
                    Good
                </button>
                <button hx-post="/cards/{{ card.id }}/rate"
                        hx-vals='{"rating": 5}'
                        hx-target="#card-container"
                        hx-swap="outerHTML">
                    Easy
                </button>
            </div>
        </div>
    </div>
</div>
```

CSS for the 3D flip:

```css
.perspective-1000 { perspective: 1000px; }
.card-inner { transform-style: preserve-3d; position: relative; }
.rotate-y-180 { transform: rotateY(180deg); }
.card-face { backface-visibility: hidden; }
.card-back { transform: rotateY(180deg); position: absolute; inset: 0; }
```

**Pattern**: Alpine handles the flip (client-side). When the user rates the card,
HTMX sends the rating to the server and swaps in the next card. Clean separation.

### Alpine.js stores for shared client state

```javascript
// app.js -- Alpine global store for study session state
document.addEventListener('alpine:init', () => {
    Alpine.store('session', {
        cardsReviewed: 0,
        streak: 0,
        pomodoroMinutes: 25,
        pomodoroRunning: false,

        incrementReviewed() {
            this.cardsReviewed++;
        },
        resetStreak() {
            this.streak = 0;
        }
    });
});
```

Access from any component: `$store.session.cardsReviewed`.

---

## 4. FastAPI Dependency Injection

**Source**: FastAPI official docs (Dependencies, Dependencies with yield, Lifespan Events).

### 4a. Database connection via lifespan + dependency

```python
# db.py
import aiosqlite
from pathlib import Path


async def init_db(db_path: str | Path) -> aiosqlite.Connection:
    """Create and return an aiosqlite connection. Called at startup."""
    db = await aiosqlite.connect(str(db_path))
    db.row_factory = aiosqlite.Row
    await db.execute("PRAGMA journal_mode=WAL")
    await db.execute("PRAGMA foreign_keys=ON")
    return db


async def close_db(db: aiosqlite.Connection) -> None:
    """Close the database connection. Called at shutdown."""
    await db.close()
```

```python
# deps.py
from typing import Annotated
from fastapi import Depends, Request
from fastapi.templating import Jinja2Templates
import aiosqlite

from study_web.config import Settings, get_settings


templates = Jinja2Templates(directory="study_web/templates")


def get_db(request: Request) -> aiosqlite.Connection:
    """Retrieve the shared DB connection from app state."""
    return request.app.state.db


def get_templates() -> Jinja2Templates:
    """Return the shared templates instance."""
    return templates


# Type aliases for clean signatures
DB = Annotated[aiosqlite.Connection, Depends(get_db)]
Templates = Annotated[Jinja2Templates, Depends(get_templates)]
Config = Annotated[Settings, Depends(get_settings)]
```

Usage in routes:

```python
# routers/flashcards.py
from fastapi import APIRouter, Request
from fastapi.responses import HTMLResponse
from study_web.deps import DB, Templates

router = APIRouter()


@router.get("/", response_class=HTMLResponse)
async def flashcard_page(request: Request, db: DB, tmpl: Templates):
    decks = await db.execute_fetchall("SELECT * FROM decks")
    return tmpl.TemplateResponse(
        request=request,
        name="pages/flashcards.html",
        context={"decks": decks},
    )
```

### 4b. Dependencies with yield (for per-request resources)

If you need a per-request transaction or cursor:

```python
# deps.py
from typing import AsyncGenerator

async def get_cursor(request: Request) -> AsyncGenerator[aiosqlite.Cursor, None]:
    """Per-request cursor with automatic cleanup."""
    db = request.app.state.db
    cursor = await db.cursor()
    try:
        yield cursor
    finally:
        await cursor.close()
```

### 4c. Config via pydantic-settings

```python
# config.py
from functools import lru_cache
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    db_path: str = "study_data.db"
    admin_username: str = "admin"
    admin_password_hash: str = ""
    secret_key: str = "change-me-in-production"
    debug: bool = False

    model_config = {"env_prefix": "STUDY_", "env_file": ".env"}


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

---

## 5. Dual API: JSON + HTMX Endpoints

**Source**: HTMX docs (Request Headers), FastAPI docs, community patterns.

HTMX sends the header `HX-Request: true` with every request. Use this to decide
whether to return HTML or JSON.

### Strategy A: Separate routers (recommended for this project)

```python
# routers/flashcards.py -- HTMX/HTML endpoints
router = APIRouter()

@router.get("/{card_id}", response_class=HTMLResponse)
async def get_card_html(request: Request, card_id: int, db: DB, tmpl: Templates):
    card = await fetch_card(db, card_id)
    return tmpl.TemplateResponse(
        request=request, name="partials/_card.html", context={"card": card}
    )


# routers/api.py -- JSON endpoints
api_router = APIRouter()

@api_router.get("/cards/{card_id}")
async def get_card_json(card_id: int, db: DB) -> CardResponse:
    card = await fetch_card(db, card_id)
    return CardResponse.model_validate(card)
```

### Strategy B: Single endpoint, content negotiation

```python
@router.get("/{card_id}")
async def get_card(request: Request, card_id: int, db: DB, tmpl: Templates):
    card = await fetch_card(db, card_id)

    if request.headers.get("HX-Request") == "true":
        return tmpl.TemplateResponse(
            request=request, name="partials/_card.html", context={"card": card}
        )
    return CardResponse.model_validate(card)
```

### Strategy C: fasthx library (decorator-based)

The [fasthx](https://github.com/volfpeter/fasthx) library (704 stars, actively maintained)
provides decorators that automatically handle HTMX vs JSON responses:

```python
from fasthx import Jinja
from fasthx.jinja import JinjaContext

jinja = Jinja(templates)

@router.get("/{card_id}")
@jinja.hx("partials/_card.html")  # Returns HTML for HTMX requests
async def get_card(card_id: int, db: DB) -> Card:
    return await fetch_card(db, card_id)  # Returns JSON for non-HTMX
```

**Recommendation**: Use Strategy A (separate routers) for clarity. Your HTMX routes
return HTML fragments; your API routes return JSON. No ambiguity, easy to test independently.

### Full page vs partial detection

For `hx-push-url` pages that must work on direct access AND HTMX navigation:

```python
@router.get("/deck/{deck_id}", response_class=HTMLResponse)
async def deck_page(request: Request, deck_id: int, db: DB, tmpl: Templates):
    deck = await fetch_deck(db, deck_id)
    cards = await fetch_cards(db, deck_id)
    context = {"deck": deck, "cards": cards}

    if request.headers.get("HX-Request") == "true":
        # HTMX navigation: return just the content fragment
        return tmpl.TemplateResponse(
            request=request, name="partials/_deck_content.html", context=context
        )
    # Direct access: return full page
    return tmpl.TemplateResponse(
        request=request, name="pages/deck.html", context=context
    )
```

---

## 6. Jinja2 Template Inheritance

**Source**: FastAPI templates docs, Jinja2 official docs, community patterns.

### base.html (master layout)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Study Mentor{% endblock %}</title>

    <!-- PWA -->
    <link rel="manifest" href="/static/manifest.json">
    <meta name="theme-color" content="#1a1a2e">

    <!-- CSS -->
    <link rel="stylesheet" href="/static/css/style.css">

    <!-- HTMX -->
    <script src="https://unpkg.com/htmx.org@2.0.4"></script>
    <!-- Alpine.js -->
    <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.14.8/dist/cdn.min.js"></script>

    {% block head_extra %}{% endblock %}
</head>
<body hx-headers='{"X-CSRF-Token": "{{ csrf_token }}"}'
      x-data
      class="{% block body_class %}{% endblock %}">

    <!-- Navigation -->
    {% include "components/_nav.html" %}

    <!-- Toast notifications (OOB target) -->
    <div id="toast-container" class="toast-container"></div>

    <!-- Main content area -->
    <main id="main-content">
        {% block content %}{% endblock %}
    </main>

    <!-- App JS (Alpine stores, HTMX config) -->
    <script src="/static/js/app.js"></script>

    {% block scripts_extra %}{% endblock %}
</body>
</html>
```

### Page template (extends base)

```html
<!-- pages/flashcards.html -->
{% extends "base.html" %}

{% block title %}Flashcards - Study Mentor{% endblock %}

{% block content %}
<div class="flashcard-page" x-data="{ deckFilter: 'all' }">
    <h1>Flashcard Review</h1>

    <!-- Deck selector -->
    <select x-model="deckFilter"
            hx-get="/cards/deck-cards"
            hx-target="#card-container"
            hx-include="[name='deck']"
            name="deck">
        <option value="all">All Decks</option>
        {% for deck in decks %}
        <option value="{{ deck.id }}">{{ deck.name }}</option>
        {% endfor %}
    </select>

    <!-- Card display area (replaced by HTMX) -->
    <div id="card-container">
        {% include "partials/_card.html" %}
    </div>

    <!-- Progress bar -->
    <div id="progress-bar">
        {% include "partials/_progress_chart.html" %}
    </div>
</div>
{% endblock %}
```

### Partial template (bare fragment, NO extends)

```html
<!-- partials/_card.html -->
<div id="card-{{ card.id }}" class="card"
     x-data="{ flipped: false }">
    <div class="card-question" x-show="!flipped">
        <p>{{ card.question }}</p>
        <button @click="flipped = true" class="btn-flip">Reveal</button>
    </div>
    <div class="card-answer" x-show="flipped" x-transition>
        <p>{{ card.answer }}</p>
        <div class="rating-row">
            <button hx-post="/cards/{{ card.id }}/rate"
                    hx-vals='{"rating": 1}'
                    hx-target="#card-container"
                    hx-swap="outerHTML">Again</button>
            <button hx-post="/cards/{{ card.id }}/rate"
                    hx-vals='{"rating": 3}'
                    hx-target="#card-container"
                    hx-swap="outerHTML">Good</button>
            <button hx-post="/cards/{{ card.id }}/rate"
                    hx-vals='{"rating": 5}'
                    hx-target="#card-container"
                    hx-swap="outerHTML">Easy</button>
        </div>
    </div>
</div>
```

### Jinja2 macros for reusable components

```html
<!-- components/_form_field.html -->
{% macro form_field(name, label, type="text", value="", error="") %}
<div class="form-group {% if error %}has-error{% endif %}">
    <label for="{{ name }}">{{ label }}</label>
    <input type="{{ type }}" id="{{ name }}" name="{{ name }}"
           value="{{ value }}" class="form-control">
    {% if error %}
    <span class="error-text">{{ error }}</span>
    {% endif %}
</div>
{% endmacro %}
```

Usage:

```html
{% from "components/_form_field.html" import form_field %}
{{ form_field("email", "Email Address", type="email", error=errors.get("email")) }}
```

---

## 7. Authentication: HTTP Basic Auth

**Source**: FastAPI advanced security docs.

### Implementation with timing-attack-safe comparison

```python
# deps.py
import secrets
from typing import Annotated

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBasic, HTTPBasicCredentials

from study_web.config import get_settings

security = HTTPBasic()


def verify_credentials(
    credentials: Annotated[HTTPBasicCredentials, Depends(security)],
) -> str:
    """Verify HTTP Basic credentials. Returns username if valid."""
    settings = get_settings()

    # Timing-attack-safe comparison
    username_correct = secrets.compare_digest(
        credentials.username.encode("utf-8"),
        settings.admin_username.encode("utf-8"),
    )
    password_correct = secrets.compare_digest(
        credentials.password.encode("utf-8"),
        settings.admin_password.encode("utf-8"),
    )

    if not (username_correct and password_correct):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
            headers={"WWW-Authenticate": "Basic"},
        )
    return credentials.username


# Type alias
AuthUser = Annotated[str, Depends(verify_credentials)]
```

### Apply globally or per-router

```python
# Global: protect all routes
app = FastAPI(dependencies=[Depends(verify_credentials)])

# Per-router: protect only settings and feedback
settings_router = APIRouter(dependencies=[Depends(verify_credentials)])

# Per-endpoint:
@router.get("/admin")
async def admin_page(user: AuthUser):
    ...
```

### Skip auth for static files and health checks

Static files mounted via `app.mount()` bypass FastAPI dependencies.
For health checks, add them before the auth middleware:

```python
@app.get("/health")
async def health():
    return {"status": "ok"}

# Then apply auth to everything else via router-level dependencies
```

---

## 8. Static Files + PWA Service Worker

**Source**: FastAPI static files docs, MDN PWA docs.

### Static file serving

```python
from fastapi.staticfiles import StaticFiles

# Mount BEFORE routers so /static/* is handled first
app.mount("/static", StaticFiles(directory="study_web/static"), name="static")
```

In templates, reference static files via:

```html
<link rel="stylesheet" href="{{ url_for('static', path='css/style.css') }}">
```

### PWA Manifest

```json
{
    "name": "Socratic Study Mentor",
    "short_name": "StudyMentor",
    "start_url": "/",
    "display": "standalone",
    "background_color": "#1a1a2e",
    "theme_color": "#1a1a2e",
    "icons": [
        { "src": "/static/img/icon-192.png", "sizes": "192x192", "type": "image/png" },
        { "src": "/static/img/icon-512.png", "sizes": "512x512", "type": "image/png" }
    ]
}
```

### Service Worker (sw.js)

```javascript
const CACHE_NAME = 'study-mentor-v1';
const PRECACHE_URLS = [
    '/',
    '/static/css/style.css',
    '/static/js/app.js',
    '/static/manifest.json',
];

// Install: precache shell assets
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME).then((cache) => cache.addAll(PRECACHE_URLS))
    );
    self.skipWaiting();
});

// Activate: clean old caches
self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then((names) =>
            Promise.all(
                names.filter((n) => n !== CACHE_NAME).map((n) => caches.delete(n))
            )
        )
    );
    self.clients.claim();
});

// Fetch: network-first for HTML (HTMX fragments), cache-first for static assets
self.addEventListener('fetch', (event) => {
    const url = new URL(event.request.url);

    if (url.pathname.startsWith('/static/')) {
        // Static assets: cache-first
        event.respondWith(
            caches.match(event.request).then((cached) =>
                cached || fetch(event.request).then((response) => {
                    const clone = response.clone();
                    caches.open(CACHE_NAME).then((cache) => cache.put(event.request, clone));
                    return response;
                })
            )
        );
    } else {
        // HTML/HTMX requests: network-first, fallback to cache
        event.respondWith(
            fetch(event.request)
                .then((response) => {
                    // Only cache GET requests for full pages
                    if (event.request.method === 'GET' && !event.request.headers.get('HX-Request')) {
                        const clone = response.clone();
                        caches.open(CACHE_NAME).then((cache) => cache.put(event.request, clone));
                    }
                    return response;
                })
                .catch(() => caches.match(event.request))
        );
    }
});
```

**Key decisions**:
- Cache-first for static assets (CSS, JS, images) -- fast, versioned via cache name.
- Network-first for HTML pages -- always fresh, fallback to cached version offline.
- Do NOT cache HTMX fragment responses in the service worker -- they are partial HTML
  and would break if served as full pages. Check `HX-Request` header to skip them.

### Register the service worker

```html
<!-- base.html, before closing </body> -->
<script>
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/static/sw.js', { scope: '/' });
}
</script>
```

Note: The service worker file is in `/static/sw.js` but registered with `scope: '/'`.
You need to serve it with the `Service-Worker-Allowed: /` header, or serve it from
the root URL:

```python
# main.py -- serve sw.js from root for proper scope
from fastapi.responses import FileResponse

@app.get("/sw.js")
async def service_worker():
    return FileResponse(
        "study_web/static/sw.js",
        media_type="application/javascript",
        headers={"Service-Worker-Allowed": "/"},
    )
```

---

## 9. WebSocket Support

**Source**: FastAPI WebSocket docs, HTMX WebSocket extension docs.

### FastAPI WebSocket for Pomodoro timer sync

```python
# routers/ws.py
from fastapi import APIRouter, WebSocket, WebSocketDisconnect, Depends
import asyncio
import json

router = APIRouter()


class PomodoroManager:
    """Manages connected WebSocket clients for Pomodoro sync."""

    def __init__(self):
        self.connections: list[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.connections.remove(websocket)

    async def broadcast(self, message: dict):
        disconnected = []
        for ws in self.connections:
            try:
                await ws.send_json(message)
            except Exception:
                disconnected.append(ws)
        for ws in disconnected:
            self.connections.remove(ws)


pomodoro_manager = PomodoroManager()


@router.websocket("/pomodoro")
async def pomodoro_ws(websocket: WebSocket):
    await pomodoro_manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_json()
            action = data.get("action")

            if action == "start":
                await pomodoro_manager.broadcast({
                    "type": "timer_started",
                    "minutes": data.get("minutes", 25),
                })
            elif action == "pause":
                await pomodoro_manager.broadcast({"type": "timer_paused"})
            elif action == "reset":
                await pomodoro_manager.broadcast({"type": "timer_reset"})

    except WebSocketDisconnect:
        pomodoro_manager.disconnect(websocket)
```

### Client-side: Alpine.js + native WebSocket (not HTMX ws extension)

For the Pomodoro timer, use Alpine.js with native WebSocket rather than HTMX's ws
extension. The timer is fundamentally a client-side countdown that syncs state -- it
does not need HTML fragment responses.

```html
<div x-data="pomodoroTimer()" class="pomodoro-widget">
    <div class="timer-display" x-text="formattedTime"></div>

    <button @click="start()" x-show="!running" class="btn">Start</button>
    <button @click="pause()" x-show="running" class="btn">Pause</button>
    <button @click="reset()" class="btn-secondary">Reset</button>

    <!-- Settings -->
    <select x-model.number="duration" @change="reset()">
        <option value="25">25 min</option>
        <option value="15">15 min</option>
        <option value="5">5 min break</option>
    </select>
</div>

<script>
function pomodoroTimer() {
    return {
        duration: 25,
        seconds: 25 * 60,
        running: false,
        interval: null,
        ws: null,

        init() {
            this.ws = new WebSocket(`ws://${location.host}/ws/pomodoro`);
            this.ws.onmessage = (event) => {
                const msg = JSON.parse(event.data);
                if (msg.type === 'timer_started') {
                    this.duration = msg.minutes;
                    this.seconds = msg.minutes * 60;
                    this.startCountdown();
                } else if (msg.type === 'timer_paused') {
                    this.pauseCountdown();
                } else if (msg.type === 'timer_reset') {
                    this.resetCountdown();
                }
            };
        },

        get formattedTime() {
            const m = Math.floor(this.seconds / 60);
            const s = this.seconds % 60;
            return `${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`;
        },

        start() {
            this.ws.send(JSON.stringify({ action: 'start', minutes: this.duration }));
        },

        pause() {
            this.ws.send(JSON.stringify({ action: 'pause' }));
        },

        reset() {
            this.ws.send(JSON.stringify({ action: 'reset' }));
        },

        startCountdown() {
            this.running = true;
            this.interval = setInterval(() => {
                if (this.seconds > 0) {
                    this.seconds--;
                } else {
                    this.pauseCountdown();
                    // Notify user timer complete
                    if (Notification.permission === 'granted') {
                        new Notification('Pomodoro Complete!');
                    }
                }
            }, 1000);
        },

        pauseCountdown() {
            this.running = false;
            clearInterval(this.interval);
        },

        resetCountdown() {
            this.pauseCountdown();
            this.seconds = this.duration * 60;
        },

        destroy() {
            clearInterval(this.interval);
            this.ws?.close();
        }
    };
}
</script>
```

### When to use HTMX ws extension vs native WebSocket

| Use Case | Approach | Reason |
|-----------|----------|--------|
| Pomodoro timer sync | Native WS + Alpine | Timer is client-side state, not HTML fragments |
| Live notifications | HTMX SSE extension | Server pushes HTML fragments for toast messages |
| Real-time dashboard | HTMX SSE extension | Server pushes updated chart/stat fragments |
| Chat (if needed) | HTMX ws extension | Messages are HTML fragments from server |

### Server-Sent Events (SSE) for live dashboard updates

SSE is simpler than WebSockets when the server just pushes updates (no client-to-server):

```python
# routers/dashboard.py
from fastapi.responses import StreamingResponse
import asyncio


@router.get("/stream")
async def dashboard_stream(db: DB):
    async def event_generator():
        while True:
            stats = await get_current_stats(db)
            html = f'<div id="live-stats" hx-swap-oob="true">{stats.html}</div>'
            yield f"event: update\ndata: {html}\n\n"
            await asyncio.sleep(30)

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
    )
```

Client:

```html
<div hx-ext="sse" sse-connect="/dashboard/stream">
    <div id="live-stats" sse-swap="update">
        Loading stats...
    </div>
</div>
```

---

## 10. Performance

**Source**: aiosqlite docs, FastAPI deployment docs, uvicorn docs.

### aiosqlite: async wrapper around sync sqlite3

aiosqlite (v0.22.1, MIT, production-stable) provides an async interface to sqlite3
using a single shared thread per connection. It does NOT use a thread pool -- there
is one background thread that serializes all SQL operations.

```python
# Pattern: single connection, shared across all requests
async def init_db(path: str) -> aiosqlite.Connection:
    db = await aiosqlite.connect(path)
    db.row_factory = aiosqlite.Row
    await db.execute("PRAGMA journal_mode=WAL")   # Write-Ahead Logging
    await db.execute("PRAGMA synchronous=NORMAL")  # Faster writes, safe with WAL
    await db.execute("PRAGMA cache_size=-64000")    # 64MB cache
    await db.execute("PRAGMA busy_timeout=5000")    # 5s wait on locks
    return db
```

### Why NOT connection pooling for SQLite

SQLite is a file-based database. Connection pooling is a pattern for network databases
(PostgreSQL, MySQL) where establishing a connection is expensive. For SQLite:

- **Single writer**: SQLite allows only one writer at a time. Multiple connections
  just queue behind the write lock.
- **WAL mode**: Allows concurrent reads with a single writer. One connection is sufficient.
- **aiosqlite uses one thread**: Additional connections add threads without benefit.

**Recommendation**: Use a single `aiosqlite.Connection` stored in `app.state.db`.
This is the simplest correct pattern for SQLite with FastAPI.

If you later need higher write throughput, switch to PostgreSQL + asyncpg + connection pool.

### uvicorn workers

```bash
# Development
uvicorn study_web.main:app --reload --host 0.0.0.0 --port 8000

# Production: single worker (SQLite constraint)
uvicorn study_web.main:app --host 0.0.0.0 --port 8000 --workers 1
```

**Critical**: Do NOT use multiple uvicorn workers with SQLite. Each worker would
open its own database connection, and SQLite's single-writer constraint means they
would constantly conflict. Use `--workers 1`.

If you need multiple workers, use:
- Multiple workers + PostgreSQL (proper connection pool), OR
- A reverse proxy (nginx/caddy) in front of a single uvicorn process.

### Async vs sync endpoints

FastAPI runs `async def` endpoints on the main event loop and `def` endpoints in a
thread pool. Since aiosqlite is async, use `async def` for all database-accessing routes:

```python
# CORRECT: async endpoint with aiosqlite
@router.get("/cards")
async def list_cards(db: DB):
    rows = await db.execute_fetchall("SELECT * FROM cards LIMIT 50")
    return rows

# WRONG: sync endpoint blocks the event loop
@router.get("/cards")
def list_cards(db: DB):
    # This would need sync sqlite3, not aiosqlite
    ...
```

### Response caching for HTMX fragments

Use ETags or Cache-Control headers for infrequently-changing content:

```python
from fastapi.responses import HTMLResponse

@router.get("/artefacts/{artefact_id}/viewer", response_class=HTMLResponse)
async def artefact_viewer(request: Request, artefact_id: int, db: DB, tmpl: Templates):
    artefact = await fetch_artefact(db, artefact_id)

    response = tmpl.TemplateResponse(
        request=request,
        name="partials/_artefact_item.html",
        context={"artefact": artefact},
    )
    # Cache static artefact descriptions for 5 minutes
    response.headers["Cache-Control"] = "private, max-age=300"
    return response
```

### HTMX-specific performance tips

1. **Use `hx-indicator`** to show loading spinners -- prevents double-clicks.
2. **Use `hx-disabled-elt="this"`** to disable buttons during requests.
3. **Use `hx-swap="outerHTML transition:true"`** for View Transitions API
   (smooth page transitions in supporting browsers).
4. **Preload on hover** with the HTMX preload extension:
   ```html
   <body hx-ext="preload">
       <a href="/cards" preload="mouseover">Flashcards</a>
   </body>
   ```

---

## Quick Reference: Complete Dependency Graph

```
main.py (app factory)
    |-- lifespan: init_db() -> app.state.db
    |-- mount: /static -> StaticFiles
    |-- include: routers/*.py
         |-- each router uses:
              |-- DB (from deps.py -> app.state.db)
              |-- Templates (from deps.py -> Jinja2Templates)
              |-- Config (from deps.py -> pydantic-settings)
              |-- AuthUser (from deps.py -> HTTPBasic verify)
```

## Key Libraries (pyproject.toml)

```toml
[project]
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.32",
    "jinja2>=3.1",
    "aiosqlite>=0.22",
    "pydantic-settings>=2.7",
    "python-multipart>=0.0.18",   # Required for form data
]

[dependency-groups]
dev = [
    "httpx>=0.28",     # For TestClient async tests
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
    "ruff>=0.8",
]
```

---

## Sources

| Source | Authority | Topics Covered |
|--------|-----------|----------------|
| [FastAPI Official Docs](https://fastapi.tiangolo.com) | Official | Project structure, DI, templates, auth, WebSocket, middleware |
| [HTMX Official Docs v2](https://htmx.org/docs/) | Official | All HTMX patterns, OOB swaps, history, CSRF, scripting |
| [HTMX Examples](https://htmx.org/examples/) | Official | Infinite scroll, active search, modals, click-to-edit |
| [Alpine.js Docs](https://alpinejs.dev) | Official | x-show, x-transition, x-data, stores |
| [fasthx](https://github.com/volfpeter/fasthx) | Community (704 stars) | FastAPI+HTMX decorator patterns |
| [aiosqlite](https://pypi.org/project/aiosqlite/) | Community (production-stable) | Async SQLite patterns |
| [MDN PWA Guide](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps) | Official | Service worker caching strategies |
