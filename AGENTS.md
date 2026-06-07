# AGENTS.md — BearBean Android

Context for AI assistants working in this repo. This file is the single source of truth for
the BearBean Android client. It was generated from the live BearBean backend (`main.py`).

---

## 1. What this project is

BearBean is a privacy-first, **fully local** family AI assistant for the Bimm household
(Chris, Jithmi, daughters Isla and Rowen). A local Ollama model (`qwen3.5:9b`) runs inference
on a home GPU server; facts live in a local ChromaDB vector store; a FastAPI backend exposes a
small HTTP API. No cloud LLM APIs.

**This repo is the Android client** — a thin native app that calls the backend over HTTP and
renders results. It holds no model and no database. All intelligence stays server-side.

### MVP scope (build this, nothing more)

Two screens, native Kotlin + Jetpack Compose:

1. **Chat** — send a message to `POST /chat`, display BearBean's reply. The backend parses
   `remember that …` and `forget that …` itself, so those work through the same chat box; no
   special UI needed.
2. **Facts browser** — list facts from `GET /facts`, filter by category and family member,
   delete via `DELETE /forget/{id}`.

**Out of scope:** manual sync triggers, settings beyond connection config, push/daily-briefing,
streaming, local persistence, and the OpenAI-compatible `/v1/*` endpoints (those serve Open
WebUI, not this app). Do not scope-creep into these.

### Tone

The backend already styles replies: polite but not effusive, direct, no exclamation points, no
emojis, complete sentences. The app displays replies verbatim — never re-process or "friendly
up" the text.

---

## 2. Connection & base URL

| Context                 | Base URL                       | Auth                                  |
|-------------------------|--------------------------------|---------------------------------------|
| **Tailscale (primary)** | `http://100.72.81.7:8765`      | none — tailnet membership is the auth |
| Home LAN (fallback)     | `http://192.168.86.200:8765`   | none                                  |
| Cloudflare tunnel       | `https://ai.bimm.dev`          | Cloudflare Access service token       |

**Default the app to the Tailscale address (`http://100.72.81.7:8765`).** Tailscale reaches
the server from anywhere — home or away — so the app needs no Cloudflare token and works off
the home wifi. Each device must have the Tailscale app installed and signed into the tailnet.
The LAN IP and the Cloudflare tunnel are fallbacks only.

The Cloudflare tunnel (`ai.bimm.dev`) is the only path that needs auth: a **Cloudflare Access
service token** sent as `CF-Access-Client-Id` / `CF-Access-Client-Secret` headers (the
interactive Google login can't be used by a native app). Not needed on Tailscale or LAN.

---

## 3. Backend API reference

JSON over HTTP. Only the endpoints below are used by the MVP.

### `POST /chat` — core conversation endpoint
**Non-streaming**: the backend calls Ollama with `stream: false` and a 120 s timeout, then
returns the whole reply. **Set the HTTP client read timeout to >120 s (e.g. 130 s)** and show a
loading state — this is the single most important client setting. The default 10 s timeout
fails on every real reply.

Request:
```json
{ "message": "string, required, min length 1",
  "user": "string, optional — e.g. \"chris\"; tags who is asking",
  "fact_limit": 15 }
```
`fact_limit` (int 0–50, default 15): how many stored facts the backend may retrieve. Leave at
default; `0` disables retrieval.

Response `200`:
```json
{ "reply": "string — display verbatim",
  "facts_used": ["fact_id", "..."],
  "model": "qwen3.5:9b" }
```
`model` is `"bearbean-intent"` when handled by the remember/forget parser instead of the LLM
(optionally badge those). Built-in intents (just send the text, no special handling):
`remember that <fact>` / `remember: <fact>` stores; `forget that <desc>` / `forget: <desc>`
deletes the best confident match. Errors: `502` if Ollama fails.

### `GET /facts` — list facts (facts browser)
Query params (all optional): `category` (exact match, from the controlled vocab),
`family_member` (e.g. `isla`, `rowen`, `jithmi`, `chris`, `all`), `limit` (int 1–1000, default
100).

Response `200`:
```json
{ "count": 1,
  "facts": [
    { "id": "cal_abc123",
      "text": "Isla dentist appointment",
      "metadata": { "category": "appointment", "family_member": "isla",
                    "source": "calendar", "event_start": "2026-06-10T15:00:00" } } ] }
```
`metadata` is a **free-form map — do not assume any key is present.** Decode as
`Map<String, String?>`, not a fixed shape. Common keys: `category`, `family_member`, `source`,
`tags` (optional), `event_start` (calendar facts only). ID prefixes: `cal_*` = calendar,
`fact_*` = chat/email/manual.

### `DELETE /forget/{fact_id}` — delete a fact
Response `200`: `{ "id": "...", "deleted": true }`. `404` if the id is gone (already deleted —
just drop it from the list). Confirm with the user first; deletion is immediate and permanent.

### `GET /categories` — controlled vocabulary (filter dropdown)
Response: `{ "categories": ["appointment","contact","general","household","medical","person","preference","school"] }`.
Fetch once and cache.

### `GET /health` — connectivity check
Response: `{ "status": "ok", "facts_stored": 42 }`. Use on startup to verify base URL/auth
before the user hits a slow `/chat`.

### Not used by the MVP (reference only)
`POST /remember` (structured fact creation), `POST /recall` (semantic search), `POST
/sync/calendar`, `POST /sync/email`, `GET /v1/models`, `POST /v1/chat/completions`.

---

## 4. Android technical design

### Stack
Kotlin · Jetpack Compose + Material 3 · min SDK 26 (adjust to household devices) · MVVM with
unidirectional data flow (ViewModel exposes immutable `StateFlow` state) · coroutines · Retrofit
+ OkHttp + kotlinx.serialization · Navigation-Compose · manual DI for the MVP (add Hilt later
only if it grows).

### Package layout
```
com.bimm.bearbean
├── BearBeanApp.kt              // Application; builds singleton dependencies
├── MainActivity.kt             // hosts the Compose NavHost
├── data
│   ├── BearBeanApi.kt          // Retrofit interface
│   ├── BearBeanRepository.kt   // wraps the API, returns Result
│   ├── NetworkModule.kt        // OkHttp + Retrofit, timeouts, auth interceptor
│   └── model                   // ChatModels, FactModels, HealthModels
├── ui
│   ├── chat                    // ChatScreen, ChatViewModel
│   ├── facts                   // FactsScreen, FactsViewModel
│   ├── nav                     // BearBeanNavHost: "chat" + "facts" destinations
│   └── theme
└── config
    └── AppConfig.kt            // base URL + service token from BuildConfig
```

### Networking layer (`NetworkModule`)
- **Read timeout > 120 s** (`readTimeout(130, SECONDS)`). Critical — see `/chat` above.
- **Auth interceptor:** add `CF-Access-Client-Id`/`CF-Access-Client-Secret` only when a
  Cloudflare token is configured (tunnel base URL). Skip on Tailscale/LAN.
- **kotlinx.serialization** with `ignoreUnknownKeys = true`; never hard-fail on unknown fields.

Retrofit interface:
```kotlin
interface BearBeanApi {
    @GET("health") suspend fun health(): HealthResponse
    @POST("chat") suspend fun chat(@Body req: ChatRequest): ChatResponse
    @GET("facts") suspend fun facts(
        @Query("category") category: String? = null,
        @Query("family_member") familyMember: String? = null,
        @Query("limit") limit: Int = 100,
    ): FactsResponse
    @DELETE("forget/{id}") suspend fun forget(@Path("id") id: String): DeleteResponse
    @GET("categories") suspend fun categories(): CategoriesResponse
}
```

Models (match the API exactly):
```kotlin
@Serializable data class ChatRequest(
    val message: String, val user: String? = null,
    @SerialName("fact_limit") val factLimit: Int = 15)
@Serializable data class ChatResponse(
    val reply: String, @SerialName("facts_used") val factsUsed: List<String>, val model: String)
@Serializable data class FactsResponse(val count: Int, val facts: List<Fact>)
@Serializable data class Fact(
    val id: String, val text: String,
    val metadata: Map<String, String?> = emptyMap()) // free-form; tolerate missing keys
@Serializable data class DeleteResponse(val id: String, val deleted: Boolean)
@Serializable data class HealthResponse(
    val status: String, @SerialName("facts_stored") val factsStored: Int)
@Serializable data class CategoriesResponse(val categories: List<String>)
```

### ChatScreen
Scrolling message list (user + BearBean bubbles) in `ChatViewModel` state. Input row: text
field + send button. On send: optimistically append the user message, show an obvious
"thinking…" indicator (replies take up to ~2 min), disable the send button while in flight,
then append the reply or an error row. Keep the list scrollable during the call. Optionally
badge replies where `model == "bearbean-intent"`. Chat history is **in-memory only** for the
MVP — the backend is stateless per request, so each message stands alone.

### FactsScreen
On load: fetch `GET /categories` (dropdown) and `GET /facts`. Filter controls: category
dropdown + family-member filter; re-query on change. Each row shows the fact text plus
category/family-member chips (guard against missing metadata keys). Delete: swipe or overflow
action → confirmation dialog → `DELETE /forget/{id}` → remove from list. Pull-to-refresh and an
empty state ("No facts stored yet").

### Config & secrets
Mirror the backend's gitignored-secrets discipline. Put connection config in `local.properties`
(gitignored by the Android template) and expose via `BuildConfig`:
```properties
# local.properties (NOT committed)
bearbean.baseUrl=http://100.72.81.7:8765/   # Tailscale — works home or away
bearbean.cfClientId=                         # only if baseUrl is https://ai.bimm.dev
bearbean.cfClientSecret=
```
In `build.gradle.kts`, read these and emit `buildConfigField`s; `AppConfig` reads them. The
base URL must end with `/` (Retrofit requirement).

> **Cleartext HTTP:** the Tailscale address is `http://`, so add a
> `network_security_config.xml` permitting cleartext to `100.72.81.7` (and `192.168.86.200` if
> keeping the LAN fallback), or modern Android blocks the request. Alternatively enable
> Tailscale HTTPS/MagicDNS and use an `https://` hostname to avoid the exception entirely.

### State, errors, edge cases
Wrap repository calls in `runCatching` and surface failures as UI error states, not crashes.
Distinguish **timeout** (slow model) from **connection failure** (wrong URL / off tailnet) —
different fixes. `502` from `/chat` = "BearBean's model is unavailable" (Ollama failed
server-side). `404` from `/forget` = already gone; drop silently.

### Deferred (not MVP)
Streaming, local chat-history persistence (Room), a runtime base-URL/auth settings screen,
manual sync triggers, daily-briefing view, push notifications.

### Build & run
Open this repo **locally** in Android Studio (not over SSH). Gradle sync, run on an emulator or
a household device. With the Tailscale base URL the app reaches `100.72.81.7:8765` from
anywhere on the tailnet — verify with `GET /health` first. An emulator needs Tailscale running
on the host machine, or use the LAN IP while testing on home wifi.
