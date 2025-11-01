## Architecture overview

- Codex ships with a configuration object that records the active model slug, its family metadata (context window, auto-compaction thresholds), and the model provider definition used to reach OpenAI.

  Loading configuration merges built-in providers (such as `openai`) with any user overrides, picks the provider by ID (defaulting to `openai`), and resolves the effective model (default `gpt-5-codex` on non-Windows hosts).



- The bundled `openai` provider targets the Responses API (`/v1/responses`), injects Codex’s version header, and requires OpenAI authentication instead of a plain environment variable key, which is why `gpt-5-codex` conversations run through the Responses pipeline by default.



## Authentication workflow

- Credentials live in `$CODEX_HOME/auth.json`, written either by piping an API key into `codex login --with-api-key` or by completing the ChatGPT OAuth-based login helper; the docs describe copying this file onto headless machines when needed.


- `CodexAuth` abstracts both API-key mode and ChatGPT OAuth tokens, supporting token refresh via `https://auth.openai.com/oauth/token` and returning either the stored API key or the latest access token for Bearer authentication.


- `load_auth` prefers a `CODEX_API_KEY` override (when enabled) and otherwise reads `auth.json`, choosing API-key mode if a key is present or ChatGPT mode when only OAuth tokens exist.


- `AuthManager` caches the resolved `CodexAuth`, exposes clones to callers, reloads on demand, and can refresh tokens when a request fails with `401` (triggered automatically in the client on unauthorized responses).


- CLI surfaces decide whether the `CODEX_API_KEY` environment fallback is allowed; for example, the `codex exec` entry point passes `enable_codex_api_key_env = true` when creating the shared manager.



## Provider configuration and HTTP client setup

- `ModelProviderInfo::create_request_builder` combines any environment-based API key with the `CodexAuth` snapshot, chooses the correct base URL (ChatGPT vs. api.openai.com), appends query parameters, and applies static plus environment-sourced HTTP headers (including optional organization/project headers).


- When no provider-specific key is set, Codex relies entirely on the `CodexAuth` bearer token. If a key is required but missing, the helper raises a structured env-var error so the CLI can surface a friendly prompt.


- The shared `reqwest` client adds Codex’s `originator` header, a sanitized multi-platform User-Agent string, and disables proxies when running inside Codex’s sandbox.



## Request construction for `gpt-5-codex`

- Each conversation spawns a `ModelClient` using the active configuration, the provider definition, the auth manager, and the OpenTelemetry event manager.


- Because the `openai` provider speaks the Responses API, `ModelClient::stream` delegates to `stream_responses`, which composes the combined instructions (base + AGENTS.md + prompt), tool list, reasoning settings, and optional text controls (verbosity or JSON schema). It also activates an Azure-specific workaround when needed.


- The serialized payload includes `model`, `instructions`, `input` history, automatic tool selection, optional prompt cache key (set to the conversation ID so Codex can reuse context), reasoning parameters, and text controls for GPT‑5 verbosity and schema enforcement.


- Before sending, Codex adds `OpenAI-Beta: responses=experimental`, forwards `conversation_id`/`session_id` headers, requests `text/event-stream`, and, when using ChatGPT auth, sets the `chatgpt-account-id` header gathered from the decoded token.


- `ModelProviderInfo::get_full_url` ensures ChatGPT-authenticated sessions hit `https://chatgpt.com/backend-api/codex/responses`, while API-key sessions use `https://api.openai.com/v1/responses`, so downstream routing matches the credential type.



## Streaming, retries, and error handling

- The client wraps each attempt in telemetry, extracts rate-limit headers, and streams the SSE feed; successful responses spawn `process_sse`, which enforces idle timeouts, reports token usage, and emits `ResponseEvent::Completed` once the model signals completion.


- Retry behavior is driven by provider-level `request_max_retries`/`stream_max_retries`; HTTP 5xx/429 errors trigger exponential backoff (respecting `Retry-After`), while 429 bodies tagged `usage_limit_reached` surface as structured `UsageLimitReachedError`s that include the latest plan type and reset hints from the error payload.


- Unauthorized responses cause an immediate token refresh through `AuthManager`, allowing transparent recovery from expired ChatGPT tokens before the next retry attempt.


- Rate-limit telemetry is parsed from custom `x-codex-*` headers and forwarded as dedicated events, enabling the UI to visualize the user’s consumption.



## Guidance for implementing the flow elsewhere

- Mirror Codex’s separation of concerns: persist credentials in a structured file, wrap them with a thread-safe auth manager that can refresh tokens, and expose an abstraction that always yields a Bearer token regardless of auth mode.


- Model providers should declare whether they need OpenAI auth or a standalone key, specify base URLs, and let callers override headers/query parameters, matching Codex’s TOML schema for extensibility.


- Build requests via a helper that injects headers, selects the correct host for ChatGPT tokens, and appends provider-specific metadata. Structure payloads after `ResponsesApiRequest`, including conversation caching, reasoning controls, and tool declarations so your agent can use the full toolchain exposed to `gpt-5-codex`.


- Adopt Codex’s streaming loop as a template: wrap SSE parsing with timeout-aware logging, translate server-side usage metrics into your own telemetry, and centralize retry logic that distinguishes fatal quota errors from transient transport failures.



By following these patterns—credential management, provider abstraction, payload construction, and resilient streaming—you can recreate Codex’s end-to-end authentication and request handling for `gpt-5-codex` in other projects while remaining compatible with OpenAI’s evolving Responses API.
