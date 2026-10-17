# Transport — Project Architecture

## Purpose

Transport is a self-hosted, layered web retrieval platform for interactive browsing, scripts, crawlers, and AI tools. It provides consistent network access, shared per-site scheduling, optional authenticated browser state, content extraction, and multiple client interfaces.

The initial implementation focuses on public HTTP(S) retrieval. Later capabilities build on the same core without requiring separate network microservices.

## Design principles

- **One modular Python service:** internal layers communicate through Python calls, not localhost HTTP. Expose REST, MCP, CLI, Python, and browser-facing adapters as appropriate.
- **Layered, not rigidly pipelined:** callers can fetch raw bytes without rendering or extraction. Higher-level operations reuse lower-level components.
- **Central coordination:** all managed automated retrieval passes through shared scheduling, routing, policy, and diagnostics.
- **Interactive and programmatic frontends:** separate client-facing interfaces, common underlying capabilities where technically possible.
- **Single-user initially, profile-aware throughout:** default profile today; isolation and multiple profiles possible later.
- **Direct systemd deployment:** avoid Docker unless a dependency genuinely warrants it.
- **Incremental usefulness:** every milestone should yield a working tool.

## Logical architecture

```text
Interactive clients                  Programmatic clients
Chrome extension / reader UI         REST / MCP / CLI / Python
               \                     /
                Client adapters / API
                         |
           Content services (optional)
     article / extract / resolve / search
           archive / watch / jobs
                         |
                Request coordinator
      policies / cache / deduplication
       per-site scheduling / priorities
                         |
                Retrieval engines
         HTTPX / Chromium / other handlers
                         |
                Network egress
        direct / HTTP proxy / SOCKS / VPN
```

Cross-cutting components: browser profile manager, session management, observability, storage, configuration, and security. A conventional HTTP/SOCKS proxy may be a separate adapter; ordinary HTTPS CONNECT forwarding cannot transparently replace encrypted responses with browser-rendered pages.

## Core retrieval contract

Define typed `FetchRequest` and `FetchResponse` objects independent of HTTP transport and presentation.

Request fields eventually include URL, method, headers/body, profile, region/egress, retrieval mode, priority, timeout, cache policy, and output handling. Response includes original/final URL, status, headers, body or stored-object reference, redirect history, timestamps, selected engine/route, queue wait, elapsed time, and errors.

Preserve raw retrieved content; transformation produces new objects. Keep API wire formats separate from internal objects. Support small in-memory responses and bounded streaming to disk or callers, without requiring a separate process.

## Scheduling and concurrency

Use one FastAPI/Uvicorn worker and one asyncio event loop initially. HTTPX asynchronous I/O handles many concurrent requests; no process or thread per site. Maintain lazily created logical site queues/admission state.

- Enforce site-level minimum request spacing and maximum active requests, plus global resource limits.
- Prioritize interactive requests while preventing starvation and large batches from monopolizing queues.
- Derive site keys from registrable domains by default, with hostname-specific overrides.
- Honor Retry-After and use bounded retries/backoff for transient failures.
- Distinguish top-level browser navigation from browser subresources; pacing every asset like a crawler request would make interactive browsing unusable.
- Enforce a site-wide ceiling across profiles and egress routes; optionally track more specific limits.
- Remove canceled queued requests; avoid duplicate concurrent fetches when safe.
- Later distinguish a durable batch/job scheduler from the per-request admission controller.

An in-memory scheduler is sufficient for the first single-process milestone. Multiple workers/processes would require coordinated shared scheduling state.

## Routing and site policies

Central policy resolution combines global defaults, registrable-domain rules, hostname overrides, and explicit request options. Policies cover rate limits, preferred retrieval engine, headers, proxy/region, timeouts, cache, and extraction strategy.

The HTTP engine should be agnostic to egress route. Start with direct and configured HTTP/SOCKS proxies; VPN/geographic routing can be added without changing client APIs. Keep authenticated sessions on stable egress routes where possible.

## Browser profiles and authentication (later)

A persistent logical `default` profile represents browser identity and related state. Three operating modes:

1. **Isolated:** unauthenticated/public retrieval without shared browser state.
2. **Cookies:** synchronize approved-domain cookies from an interactive laptop Chrome extension into a protected central store for HTTPX and server Chromium. Preserve HttpOnly, SameSite, expiry, partitioning and other attributes; prefer one-way laptop-to-server synchronization initially.
3. **Browser:** use a persistent server-side Chromium profile through Playwright; allow remote interactive access for login, CAPTCHA, or site-specific challenges.

A profile is persistent identity/configuration; a session is a live client or browser context. Cookie sharing does not guarantee equivalent local storage, IndexedDB, passkeys, browser fingerprint, or browser-bound session state. Never concurrently mount/write the same Chromium profile from multiple processes or machines.

Authentication/challenge failures can pause affected site/profile jobs pending interactive intervention; verify access before resuming. Do not assume arbitrary CAPTCHA solving.

## Content services (later)

- **Article:** original HTML, cleaned semantic HTML, consistently styled standalone HTML, Markdown, structured JSON with title/author/date/canonical URL/images/captions. Evaluate Readability and Trafilatura.
- **Resolve:** redirects, canonical URL, content type, and appropriate handler (HTML, PDF, feeds, video, etc.).
- **Extract:** deterministic selectors and optional schema-driven structured extraction.
- **Search:** integrate SearXNG, including country/language/domain policies.
- **Archive/cache:** immutable fetched objects with provenance; filesystem for bodies and lightweight metadata/index storage. Respect HTTP caching and prevent accidental cross-profile authenticated cache leakage.
- **Watch:** periodic retrieval, normalized change detection, RSS/feed ingestion.
- **Jobs:** durable batch submission, progress, cancellation, partial results, restart recovery.

## Execution and deployment

- Python package with modular `api/`, `core/`, `retrieval/`, `content/`, and `storage/` directories.
- FastAPI for REST, HTTPX for retrieval, asyncio for I/O scheduling.
- One systemd-managed service on slagg, dedicated unprivileged user, virtual environment, one Uvicorn worker.
- Bounded response sizes, timeouts, connection counts, disk usage, and queue sizes.
- Stream large bodies asynchronously to temporary files, then atomically publish completed objects; separate CPU-heavy parsing/PDF tasks into a bounded process pool when warranted.
- Keep the service private/authenticated. Defend against SSRF, DNS rebinding, redirects to forbidden addresses, access to metadata/internal services, and credential leakage. No public open proxy.
- Diagnostics should include queue wait, route, redirects, status, cache outcome, retries, elapsed time, and failure category, without logging secrets.

## Milestones

### Stage 1 — Usable HTTP retrieval (first implementation)

- `POST /fetch` and `GET /health`, plus a simple CLI/Python client.
- HTTPX async retrieval with redirects, timeouts, bounded retries, headers, and direct/configured proxy routing.
- Central per-site admission scheduling, priority, global limits, and 429/503 backoff.
- Bounded response buffering and streaming to temporary files for large bodies.
- Basic diagnostics, configuration, tests, and systemd unit.
- Define profile and cache interfaces without implementing browser synchronization or sophisticated caching.

**Acceptance test:** concurrent retrieval of many URLs across several domains; per-site pacing and concurrency limits hold, other sites progress in parallel, interactive priority works, large downloads do not exhaust memory, and systemd restart succeeds.

### Stage 2 — Clean content
Article extraction to clean HTML, styled HTML, Markdown, and JSON; preserve metadata and meaningful images.

### Stage 3 — Browser retrieval
Persistent Playwright/Chromium profile, JS rendering, interactive remote login/challenge intervention, and HTTP fallback strategy.

### Stage 4 — Laptop integration
Chrome extension for scoped cookie synchronization and optionally a reader/browser integration. Keep remote-browser access as a stronger-coupling fallback.

### Stage 5 — MCP and search
Thin MCP adapter over existing Python operations; SearXNG search integration.

### Stage 6 — Persistent jobs and monitoring
Durable job queue, batch crawling, archive/cache, change detection, and additional content handlers.

Milestone order after Stage 1 is flexible.

## Open decisions

- Exact REST response format for binary content and stored download handles.
- Initial policy configuration syntax and site-key exceptions.
- Authentication for LAN clients (bind locally first; SSH tunnel or authenticated LAN endpoint).
- Cache freshness and storage limits.
- Browser integration details, once Chromium is introduced.

## Repository conventions

Maintain this `project.md` as the high-level requirements and architecture sketch. Keep implementation and operational details in separate documentation as the project develops. Commit working changes directly to the repository unless instructed otherwise.
