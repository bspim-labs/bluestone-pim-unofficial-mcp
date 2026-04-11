# MCP Design Decisions

This document explains the reasoning behind the design choices in this server. It is a record of decisions already made, not a guide for making new ones. For patterns to follow when adding tools, see [mcp-patterns.md](mcp-patterns.md).

---

## session_init as a separate tool

The beta notice could have been prepended to every tool response automatically. Instead it lives in a dedicated `session_init` tool that the model must call first.

**Why:** Prepending a notice to data tool responses breaks separation of concerns — the tool is returning data and a user notice in the same call. An earlier implementation used an `isFirstToolCall` flag that was shared across all tools. In Vercel's serverless environment, each request creates a new `McpServer` instance, so the flag reset unpredictably. A dedicated tool with a strong "ALWAYS call this FIRST" description removes the ambiguity entirely.

---

## betaPrefix removed from data tools

An earlier version used a `betaPrefix()` function that prepended the beta notice to the first data tool call as a fallback in case `session_init` was skipped.

**Why it was removed:** It merged two concerns in one response. The model cannot cleanly separate "here is the notice to show the user" from "here is the data to interpret." Combined with the serverless reset problem above, it could fire at unexpected times mid-conversation. The `session_init` tool handles this job cleanly on its own.

---

## Nested variants in list_products_in_category

Products are returned with VARIANT products nested under their GROUP parent rather than as a flat list with `variantParentId` references.

**Why:** A flat list of mixed GROUPs, VARIANTs, and SINGLEs gives the model no structural signal. Without nesting, models tended to list VARIANTs as standalone top-level products. Nesting makes the hierarchy explicit in the data so the model reproduces it correctly without needing display instructions alone to carry that weight.

---

## Plain-text summary line before JSON

All read tool responses begin with a human-readable summary ("Found 12 products in Electronics.") before the JSON payload.

**Why:** Models receiving only JSON often paste it verbatim into chat. The summary line already states the key facts, so the model paraphrases rather than dumps. The JSON remains present for the model to query when the user asks for specifics on a particular item.

---

## Plain-text-only responses for mutations

Write operations (create, update, delete) return a simple confirmation string rather than a JSON object.

**Why:** There is no structured data for the model to reason over after a mutation. A `{ success: true, message: "..." }` wrapper adds no value and increases the chance the model renders it as a code block in chat. A plain string like `Product "X" created. ID: abc123` is sufficient.

---

## Stateless bearer tokens (no session store)

The Vercel HTTP deployment has no database. Bearer tokens issued at `/token` are AES-256-GCM encrypted blobs containing the user's Bluestone credentials directly, not references to stored sessions.

**Why:** Vercel serverless functions have no persistent memory between invocations. The alternatives — a database or Redis — add infrastructure for a personal/beta project. Encoding state in the token itself means the server only needs the `SIGNING_SECRET` env var to verify and decrypt. Nothing is stored.

The trade-off: token revocation is not possible. A token is valid until it expires. Acceptable at this scale.

---

## Two OAuth flows on a single /authorize endpoint

Claude Desktop and RFC 7591-compliant clients (Cursor, VS Code, ChatGPT) both hit `/authorize`, but they go through different flows.

**Why:** Claude Desktop predates dynamic client registration. It encodes `mapiClientId:papiKey` directly in the `client_id` field and skips the browser form entirely. Newer clients register first at `/register`, then open `/authorize` in a browser where the user enters Bluestone credentials. A single endpoint handles both by inspecting the `client_id` format: if it contains `:`, it is the legacy flow; otherwise it is the dynamic registration flow that renders the HTML form.

---

## Pagination via Bluestone PAPI's itemsOnPage + pageNo

`list_products_in_category` passes `itemsOnPage` and `pageNo` to the Bluestone API rather than fetching all results and slicing client-side.

**Why:** Categories can contain hundreds of products. Fetching the full set on every call is slow, produces large payloads, and risks filling the model's context window. Server-side pagination keeps responses bounded. The `hasMore` and `totalPages` fields in the response tell the model when to offer to fetch the next page.

The tool exposes `limit` and `page` as input parameters (cleaner names for the model) and maps them to Bluestone's `itemsOnPage` and `pageNo` internally.

**Cross-page GROUP/VARIANT splits:** With page-based pagination, a GROUP product and some of its VARIANTs may land on different pages. Orphaned VARIANTs (those whose parent GROUP is not in the current page's results) are included as standalone items rather than silently dropped — their `type` field still identifies them correctly.
