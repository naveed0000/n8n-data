# Complete Error Handling Reference for LLM API POST Requests

**Scope:** Every failure mode that can occur before, during, and after an HTTP POST call to an LLM completion/chat endpoint, across a typical AI workflow (Node.js, Python, Java, Go, cURL, n8n). Vendor-agnostic by default, with provider-specific notes for **OpenAI, Anthropic, Google Gemini, xAI Grok, DeepSeek, Mistral, and Cohere** where their behavior diverges.

**How to use this document:** Each numbered section below is a self-contained error-category table using this schema:

`Category | Error Name | HTTP Status | Error Code | Retryable | Cause | Detection | Recovery | Example Response`

Skip to the appendix (after Section 20) for the error hierarchy, retry matrix, status code reference, decision tree, best practices, common mistakes, and a pre-production checklist.

---

## Table of Contents

1. [Client-Side Validation Errors](#1-client-side-validation-errors)
2. [Authentication Errors](#2-authentication-errors)
3. [Authorization Errors](#3-authorization-errors)
4. [HTTP Errors](#4-http-errors)
5. [Server Errors](#5-server-errors)
6. [Network Errors](#6-network-errors)
7. [Rate Limiting Errors](#7-rate-limiting-errors)
8. [Model Errors](#8-model-errors)
9. [Token Errors](#9-token-errors)
10. [Safety Errors](#10-safety-errors)
11. [Content Errors](#11-content-errors)
12. [Structured Output Errors](#12-structured-output-errors)
13. [Streaming Errors](#13-streaming-errors)
14. [Function / Tool Calling Errors](#14-function--tool-calling-errors)
15. [File Upload Errors](#15-file-upload-errors)
16. [SDK-Specific Errors](#16-sdk-specific-errors)
17. [Retry Strategy](#17-retry-strategy)
18. [Logging Recommendations](#18-logging-recommendations)
19. [Recovery Strategy](#19-recovery-strategy)
20. [n8n-Specific Errors](#20-n8n-specific-errors)
- [Appendix A: Complete Error Hierarchy](#appendix-a-complete-error-hierarchy)
- [Appendix B: Retry Matrix](#appendix-b-retry-matrix)
- [Appendix C: Status Code Reference](#appendix-c-status-code-reference)
- [Appendix D: Decision Tree](#appendix-d-decision-tree-for-handling-errors)
- [Appendix E: Best Practices](#appendix-e-best-practices)
- [Appendix F: Common Mistakes](#appendix-f-common-mistakes)
- [Appendix G: Pre-Production Checklist](#appendix-g-pre-production-checklist)

---

## 1. Client-Side Validation Errors

These occur before or as the request is serialized — either caught locally, or rejected immediately by the API's schema validator (typically `400`).

| Category | Error Name | HTTP Status | Error Code | Retryable | Cause | Detection | Recovery | Example Response |
|---|---|---|---|---|---|---|---|---|
| Validation | Missing/empty request body | 400 | `invalid_request_error` | No | Body is `null`, `""`, or omitted | Pre-flight check body length before send | Fail fast, raise config error | `{"error":{"type":"invalid_request_error","message":"Request body is empty"}}` |
| Validation | Invalid / malformed JSON | 400 | `invalid_request_error` / `parse_error` | No | Trailing commas, unescaped quotes, truncated payload | `JSON.parse` throws `SyntaxError` before send | Validate with a schema/linter before send | `{"error":{"message":"Invalid JSON payload received"}}` |
| Validation | Invalid UTF-8 / unsupported encoding | 400 | `invalid_request_error` | No | Payload has invalid byte sequences or non-UTF-8 charset | Encode/decode check, `Content-Type: charset=utf-8` | Force UTF-8 encoding at serialization | `{"error":{"message":"'utf-8' codec can't decode byte"}}` |
| Validation | Missing required field (`model`, `messages`) | 400 | `invalid_request_error` | No | Required top-level key absent | JSON schema validation against API spec | Fail fast with clear field name | `{"error":{"message":"Missing required parameter: 'model'"}}` |
| Validation | Invalid field type (string vs number vs array) | 400 | `invalid_request_error` | No | e.g. `temperature: "high"` instead of float | Type-check request object before send | Coerce or reject with typed error | `{"error":{"message":"'temperature' must be a number"}}` |
| Validation | Null / empty string where not allowed | 400 | `invalid_request_error` | No | Required string field sent as `null` or `""` | Non-null assertion in request builder | Strip nulls, enforce required-field defaults | `{"error":{"message":"content: none is not an allowed value"}}` |
| Validation | Invalid enum value (`role`, `finish_reason` filters) | 400 | `invalid_request_error` | No | `role: "human"` instead of `"user"` | Enum whitelist check client-side | Map to accepted enum before send | `{"error":{"message":"'human' is not one of ['system','user','assistant','tool']"}}` |
| Validation | Missing / empty `messages` array | 400 | `invalid_request_error` | No | `messages: []` or omitted entirely | Length check `> 0` before send | Fail fast; require at least one message | `{"error":{"message":"'messages' field is required and cannot be empty"}}` |
| Validation | Missing `content` in a message | 400 | `invalid_request_error` | No | Message object has no `content` and no `tool_calls` | Per-message schema validation | Fail fast, point to message index | `{"error":{"message":"messages[1].content: field required"}}` |
| Validation | Invalid model name/string | 400 / 404 | `model_not_found` | No | Typo, deprecated alias, wrong provider prefix | Compare against provider's model list/enum | Validate against a cached model registry | `{"error":{"message":"The model 'gpt-5-turbo-x' does not exist"}}` |
| Validation | Invalid `temperature` / `top_p` (out of range) | 400 | `invalid_request_error` | No | Value outside provider's accepted range (e.g. temp > 2) | Range check client-side | Clamp or reject with range hint | `{"error":{"message":"temperature must be between 0 and 2"}}` |
| Validation | Invalid `max_tokens` (negative, > context) | 400 | `invalid_request_error` | No | Requested output exceeds model's max output/context | Compare against known model limits | Reject or auto-clamp to model max | `{"error":{"message":"max_tokens is too large for this model"}}` |
| Validation | Invalid `response_format` / JSON schema | 400 | `invalid_request_error` | No | Malformed JSON Schema passed to structured-output mode | Validate schema with a JSON-Schema validator (ajv, jsonschema) | Fail fast before send | `{"error":{"message":"Invalid schema for response_format"}}` |
| Validation | Invalid MIME type for multimodal input | 400 | `invalid_request_error` | No | Unsupported image/audio/doc MIME type in content block | Whitelist check (`image/png`, `image/jpeg`, `application/pdf`, etc.) | Convert/reject unsupported types upfront | `{"error":{"message":"Unsupported image MIME type: image/tiff"}}` |
| Validation | Invalid tool / function declaration | 400 | `invalid_request_error` | No | Tool schema missing `name`, invalid JSON Schema in `parameters` | Validate each tool definition independently | Fail fast, list offending tool | `{"error":{"message":"tools[0].function.parameters is not valid JSON Schema"}}` |
| Validation | Invalid safety settings object | 400 | `invalid_request_error` | No | Unrecognized category/threshold enum (Gemini-style `safety_settings`) | Enum validation against provider docs | Fail fast | `{"error":{"message":"Invalid safety category: HARM_CATEGORY_X"}}` |

---

## 2. Authentication Errors

Failures proving *who* is calling — always `401` (with `403` used by a couple of providers for revoked/expired keys).

| Category | Error Name | HTTP Status | Error Code | Retryable | Cause | Detection | Recovery | Example Response |
|---|---|---|---|---|---|---|---|---|
| Auth | Missing API key | 401 | `authentication_error` | No | No `Authorization` / `x-api-key` header sent | Check header presence before send | Fail fast in config validation | `{"error":{"type":"authentication_error","message":"No API key provided"}}` |
| Auth | Missing `Authorization` header entirely | 401 | `missing_auth` | No | Header key typo (`Authorizaton`) or dropped by proxy | Log outgoing headers in non-prod | Assert header set in HTTP client wrapper | `{"error":{"message":"Authorization header is required"}}` |
| Auth | Invalid API key | 401 | `invalid_api_key` | No | Wrong key, copy-paste error, extra whitespace | Compare key format/prefix (`sk-`, `sk-ant-`) client-side | Rotate/re-issue key, verify env var loaded | `{"error":{"message":"Incorrect API key provided"}}` |
| Auth | Expired API key | 401 | `invalid_api_key` | No | Key past its TTL (some enterprise/OAuth setups) | Track key issue/expiry dates | Refresh key via key-management flow | `{"error":{"message":"API key has expired"}}` |
| Auth | Revoked API key | 401 | `invalid_api_key` | No | Key manually revoked in provider console | Alert on sudden 401 spike for a previously-working key | Re-issue new key, rotate secret store | `{"error":{"message":"This API key has been revoked"}}` |
| Auth | Invalid Bearer token format | 401 | `invalid_request_error` | No | `Authorization: sk-xxx` instead of `Bearer sk-xxx` | Validate header format before send | Fix header template in HTTP client | `{"error":{"message":"Invalid Authorization header format"}}` |
| Auth | Invalid OAuth token | 401 | `invalid_token` | Sometimes (if refreshable) | Expired/malformed OAuth2 access token | Check token `exp` claim locally before use | Refresh via OAuth refresh-token flow, retry once | `{"error":{"message":"OAuth token is invalid or expired"}}` |
| Auth | Invalid service account credentials | 401 / 403 | `PERMISSION_DENIED` | No | Malformed service-account JSON, wrong scopes (GCP/Vertex) | Validate SA key file structure and scopes on load | Regenerate SA key, grant correct IAM roles | `{"error":{"status":"UNAUTHENTICATED","message":"Request had invalid authentication credentials"}}` |
| Auth | Organization / project mismatch | 401 / 403 | `invalid_organization` | No | Key belongs to a different org than the one referenced in headers | Compare `OpenAI-Organization` header vs. key's org | Correct org/project header or use matching key | `{"error":{"message":"You are not permitted to use this organization"}}` |

---

## 3. Authorization Errors

Caller is *known* but not *allowed* — typically `403`.

| Category | Error Name | HTTP Status | Error Code | Retryable | Cause | Detection | Recovery | Example Response |
|---|---|---|---|---|---|---|---|---|
| Authz | No permission for this resource | 403 | `permission_denied` | No | Key/role lacks scope for the endpoint or model | Map required scopes per endpoint | Request elevated access from account admin | `{"error":{"message":"You do not have permission to perform this action"}}` |
| Authz | Project disabled | 403 | `project_disabled` | No | Project deactivated in provider console | Alert on 403 with this specific code | Reactivate project or migrate to active one | `{"error":{"message":"This project has been disabled"}}` |
| Authz | Billing disabled / no payment method | 403 | `billing_not_active` | No | Free-tier exhausted, card declined, account suspended for non-payment | Monitor billing-status webhook/API if provider offers one | Update payment method, alert finance owner | `{"error":{"message":"You exceeded your current quota, please check your plan and billing details"}}` |
| Authz | Model access denied | 403 | `model_not_authorized` | No | Account not allow-listed for a gated model (e.g. new frontier model) | Compare requested model against account's allowed-model list | Request access / fall back to an authorized model | `{"error":{"message":"Your account does not have access to model 'claude-opus-4-8'"}}` |
| Authz | Region / geography not allowed | 403 | `unsupported_country_region_territory` | No | Request originates from an embargoed or unsupported region | Check provider's supported-region list | Route through supported region, or fail with clear message | `{"error":{"message":"Country, region, or territory not supported"}}` |
| Authz | Account suspended | 403 | `account_suspended` | No | ToS violation, fraud flag, chargeback | Alert immediately, page account owner | Contact provider support | `{"error":{"message":"Your account has been suspended"}}` |
| Authz | API/product disabled for account | 403 | `api_disabled` | No | Feature flag off for this account tier (e.g. batch API, fine-tuning) | Check plan/tier capabilities at startup | Upgrade plan or use alternate endpoint | `{"error":{"message":"This API is not enabled for your account"}}` |

---

## 4. HTTP Errors

Standard status-code failures that aren't already covered above.

| Category | Error Name | HTTP Status | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|---|
| HTTP | Bad Request | 400 | No | Malformed payload (see Section 1) | Status code check | Fix payload, do not retry unmodified |
| HTTP | Unauthorized | 401 | No | Bad/missing credentials (see Section 2) | Status code check | Fix credentials |
| HTTP | Forbidden | 403 | No | Insufficient permission (see Section 3) | Status code check | Fix authorization/plan |
| HTTP | Not Found | 404 | No | Wrong endpoint path, invalid model ID in URL | Status code check + URL audit | Verify endpoint/model, don't retry blindly |
| HTTP | Method Not Allowed | 405 | No | GET used on a POST-only endpoint | Status code check | Fix HTTP verb |
| HTTP | Not Acceptable | 406 | No | `Accept` header requests unsupported response type | Status code check | Set correct `Accept: application/json` |
| HTTP | Request Timeout | 408 | Yes | Client too slow sending the full request | Client-side timeout + status check | Retry with backoff, check network |
| HTTP | Conflict | 409 | Sometimes | Concurrent modification (e.g. same idempotency key, different body) | Status code check | Use a fresh idempotency key or reconcile state |
| HTTP | Gone | 410 | No | Endpoint/model permanently retired | Status code check | Migrate to current endpoint/model |
| HTTP | Length Required | 411 | No | Missing `Content-Length` header | Status code check | Ensure HTTP client sets it automatically |
| HTTP | Payload Too Large | 413 | No | Request body (prompt + files) exceeds size limit | Pre-flight payload size check | Chunk/compress input, use file upload API instead of inline base64 |
| HTTP | URI Too Long | 414 | No | Oversized query string (rare for POST, common if params in URL) | Pre-flight URL length check | Move params into the body |
| HTTP | Unsupported Media Type | 415 | No | `Content-Type` isn't `application/json` | Status code check | Set correct `Content-Type` header |
| HTTP | Range Not Satisfiable | 416 | No | Invalid `Range` header on a file-download endpoint | Status code check | Fix/remove range header |
| HTTP | Unprocessable Entity | 422 | No | Semantically invalid but syntactically valid JSON (e.g. contradictory params) | Status code check | Fix conflicting parameters |
| HTTP | Too Early | 425 | Yes (after delay) | Server rejects request sent via `Early-Data` (TLS 1.3 0-RTT) | Status code check | Retry as a normal (non-0-RTT) request |
| HTTP | Upgrade Required | 426 | No | Client using a deprecated protocol version | Status code check | Upgrade HTTP client/TLS version |
| HTTP | Precondition Required | 428 | No | Server requires conditional headers (`If-Match`) that are missing | Status code check | Add required precondition header |
| HTTP | Too Many Requests | 429 | Yes | Rate limit / quota exceeded (see Section 7) | Status code check + `Retry-After` header | Exponential backoff honoring `Retry-After` |
| HTTP | Request Header Fields Too Large | 431 | No | Oversized headers (huge auth token, custom headers) | Status code check | Trim custom headers, shorten tokens |

---

## 5. Server Errors

Failures on the provider's side — generally safe to retry with backoff (except `501`).

| Category | Error Name | HTTP Status | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|---|
| Server | Internal Server Error | 500 | Yes | Unhandled exception in provider's backend | Status code check | Retry with exponential backoff (max 3-5 attempts) |
| Server | Not Implemented | 501 | No | Endpoint/feature not implemented for this route | Status code check | Do not retry; use a supported feature/endpoint |
| Server | Bad Gateway | 502 | Yes | Upstream service (load balancer to inference cluster) failure | Status code check | Retry with backoff; check provider status page |
| Server | Service Unavailable | 503 | Yes | Planned maintenance or overload | Status code + `Retry-After` if present | Backoff and retry; queue for later if sustained |
| Server | Gateway Timeout | 504 | Yes | Upstream took too long (often large-context or overloaded model) | Status code check + latency logging | Retry with backoff, consider raising client timeout |
| Server | Insufficient Storage | 507 | Yes (after delay) | Provider-side storage exhaustion (rare, batch/file APIs) | Status code check | Retry later; alert if persistent |
| Server | Loop Detected | 508 | No | Infinite redirect/proxy loop server-side | Status code check | Alert immediately, this indicates infra misconfiguration, not transient load |

> **Provider note:** Anthropic uses a dedicated `overloaded_error` (surfaced as `529` or `503` depending on integration) distinct from generic `500`s, treat it like `503` for retry purposes. OpenAI and Gemini fold overload conditions into `500`/`503`.

---

## 6. Network Errors

These never reach the HTTP layer, they surface as client/socket exceptions.

| Category | Error Name | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|
| Network | DNS lookup failed | Yes (after delay) | `ENOTFOUND`, hostname unresolvable, DNS outage, typo in base URL | Catch `ENOTFOUND`/`EAI_AGAIN` | Verify base URL, retry with backoff, check local DNS resolver |
| Network | Connection refused | Yes | `ECONNREFUSED`, nothing listening (wrong port, firewall) | Catch `ECONNREFUSED` | Verify endpoint/port, retry with backoff |
| Network | Connection reset | Yes | `ECONNRESET`, peer abruptly closed the connection | Catch `ECONNRESET` | Retry with backoff; check for load balancer idle timeouts |
| Network | Connection aborted | Yes | Local or intermediate proxy aborted the request | Catch `ECONNABORTED` | Retry with backoff |
| Network | Socket hang up | Yes | Server closed socket before responding (Node.js specific) | Catch `socket hang up` error message | Retry with backoff, increase `keepAlive` timeout |
| Network | Connection timed out | Yes | `ETIMEDOUT`, no response within socket timeout window | Client-side timeout | Increase timeout for long-context requests, retry with backoff |
| Network | Host unreachable | Yes (after delay) | `EHOSTUNREACH`, routing failure between client and host | Catch `EHOSTUNREACH` | Check network path/VPN, retry with backoff |
| Network | Network unreachable | Yes (after delay) | `ENETUNREACH`, local network/interface down | Catch `ENETUNREACH` | Check local connectivity before retrying |
| Network | TLS handshake failed | No (usually config) | Protocol/cipher mismatch, expired cert, clock skew | Catch `TLS handshake` / `EPROTO` errors | Fix TLS config, sync system clock, update CA bundle |
| Network | SSL certificate error | No (usually config) | Expired, self-signed, or untrusted cert in the chain | Catch `CERT_HAS_EXPIRED`, `UNABLE_TO_VERIFY_LEAF_SIGNATURE` | Update CA certs; never blanket-disable cert verification in prod |
| Network | Proxy error | Yes | Corporate/VPN proxy rejecting or mangling the request | Catch `5xx` from proxy layer, inspect proxy headers | Verify proxy allowlist for the API domain |
| Network | VPN interference | Yes | VPN routing breaks TLS SNI or drops packets | Compare behavior with/without VPN | Add API domain to VPN split-tunnel allowlist |

---

## 7. Rate Limiting Errors

All rate-limit failures return `429` with (ideally) a `Retry-After` header.

| Category | Error Name | HTTP Status | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|---|
| Rate Limit | RPM exceeded | 429 | Yes | Too many requests per minute for the account/model tier | `429` + `Retry-After` or `x-ratelimit-remaining-requests: 0` | Exponential backoff + request queue/throttle |
| Rate Limit | TPM exceeded | 429 | Yes | Too many tokens (input+output) per minute | `x-ratelimit-remaining-tokens: 0` header | Backoff, reduce batch size, spread load |
| Rate Limit | RPD exceeded | 429 | Yes (next day) or No if hard cap | Daily request cap hit | `x-ratelimit-reset-requests` header | Queue for next reset window, alert if blocking production |
| Rate Limit | Quota exhausted | 429 | No (until renewed) | Prepaid/monthly credit fully consumed | Explicit `insufficient_quota` code | Top up balance/raise limit, alert billing owner |
| Rate Limit | Daily quota exceeded | 429 | Next reset | Free-tier or org-level daily cap | Error code + reset header | Queue, notify, or fall back to a different key/project |
| Rate Limit | Monthly quota exceeded | 429 | Next billing cycle | Hard monthly spend cap reached | Explicit error code | Escalate to account owner, raise cap |
| Rate Limit | Burst limit exceeded | 429 | Yes | Too many requests in a very short window (sub-minute) | Rapid `429`s despite RPM headroom | Add jitter + smaller concurrency, use a token-bucket limiter |
| Rate Limit | Concurrent request limit exceeded | 429 | Yes | Too many simultaneous in-flight requests | `429` with concurrency-specific code | Cap client-side concurrency (semaphore), queue overflow |

**`Retry-After` header:** Servers may return seconds (`Retry-After: 30`) or an HTTP date. Always prefer this value over a fixed backoff when present, it reflects the server's actual reset window.

**Exponential backoff:** delay grows geometrically per attempt: `delay = base * 2^attempt` (e.g. 1s, 2s, 4s, 8s...), capped at a sane ceiling (30-60s) to avoid unbounded waits.

**Jitter:** add randomness to the computed delay (`delay = random(0, computed_delay)` or `computed_delay +/- 20%`) so that many clients retrying simultaneously don't re-collide in a synchronized retry storm ("thundering herd").

**Adaptive retry:** dynamically adjusts concurrency/backoff based on the *live* rate of `429`s/`5xx`s observed (similar to TCP congestion control) rather than a fixed schedule, reduces throughput proactively before hitting hard limits, and ramps back up as errors subside.

> **Provider note:** OpenAI, Anthropic, and Mistral all expose `x-ratelimit-*` response headers on every call (even successful ones) so you can pre-emptively throttle before hitting `429`. Gemini/Vertex primarily surfaces limits via `429 RESOURCE_EXHAUSTED` without granular remaining-count headers. Cohere and DeepSeek follow the OpenAI-style header convention.

---

## 8. Model Errors

| Category | Error Name | HTTP Status | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|---|
| Model | Model not found | 404 | No | Typo, wrong provider prefix, model doesn't exist | Status + `model_not_found` code | Validate against live model-list endpoint |
| Model | Model deprecated | 400 / 410 | No | Model retired/sunset by provider | Deprecation error code, provider changelog | Migrate to the successor model proactively before sunset date |
| Model | Model unavailable | 503 | Yes | Temporary provider-side outage for that specific model | Status code + model-specific error | Backoff retry; fallback model if SLA-critical |
| Model | Model overloaded | 429 / 503 | Yes | Demand exceeds provisioned capacity (common for new/flagship models) | `overloaded_error` code (Anthropic) or `503` | Backoff, fallback to alternate model/region |
| Model | Model warming up / cold start | 503 | Yes | Serverless/dedicated deployment scaling up from zero | Short-lived `503`s at low traffic times | Retry after a few seconds; consider provisioned throughput for latency-sensitive apps |
| Model | Model disabled | 403 | No | Provider disabled the model account-wide (safety/legal reasons) | `403` + explicit code | Switch model, check provider announcements |
| Model | Invalid model version/snapshot | 400 / 404 | No | Referencing a pinned snapshot that's been retired (e.g. `-0314`) | 404 on specific dated alias | Pin to a currently-supported snapshot, monitor deprecation notices |
| Model | Unsupported model capability | 400 | No | Requesting vision/tools/JSON-mode on a model that doesn't support it | `400` referencing the unsupported feature | Check model capability matrix before enabling a feature flag |

---

## 9. Token Errors

| Category | Error Name | HTTP Status | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|---|
| Tokens | Input too large | 400 | No | Prompt alone exceeds the model's context window | Pre-count tokens client-side before send | Truncate/summarize/chunk input |
| Tokens | Output too large / `max_tokens` too high | 400 | No | Requested `max_tokens` + prompt tokens > context limit | Compare `prompt_tokens + max_tokens` vs. model limit | Lower `max_tokens` or shorten prompt |
| Tokens | Context window exceeded | 400 | No | Total conversation history exceeds model's max context | Running token counter across the conversation | Trim oldest turns, summarize history, use a larger-context model |
| Tokens | Prompt too long | 400 | No | Provider-specific hard cap below the theoretical context window | Same as above | Same as above |
| Tokens | Completion exceeds limit | (`finish_reason: length`) | No | Model stopped because it hit `max_tokens`, not a true error | Check `finish_reason`/`stop_reason` field in response | Increase `max_tokens` or ask for a continuation |
| Tokens | Total tokens exceeded (input+output) | 400 | No | Combined usage exceeds a per-request or per-plan cap | Sum `prompt_tokens + completion_tokens` against plan limits | Reduce scope of request, split into multiple calls |

**How token counting works:** LLMs don't count characters or words, text is split into *tokens* by the model's tokenizer (e.g. OpenAI's `tiktoken`/BPE, Anthropic's tokenizer, SentencePiece for many open models). Roughly 1 token equals about 4 characters or three-quarters of an English word, but this varies a lot by language and content type (code, non-Latin scripts, and emoji tokenize less efficiently). Every request consumes **input tokens** (system + history + prompt + tool schemas + any injected images/documents, which are also tokenized) and **output tokens** (the generation). Always pre-count with the provider's official tokenizer library rather than approximating, approximations that run slightly high cause needless truncation, and approximations that run low cause hard `400`s at the worst possible moment (e.g. mid-batch job).

---

## 10. Safety Errors

| Category | Error Name | HTTP Status | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|---|
| Safety | Blocked by safety filters (general) | 400 | No | Content trips the provider's moderation/safety classifier | `finish_reason: "content_filter"` / `block_reason` field | Do not blindly retry identical input; revise prompt or route to human review |
| Safety | Harmful content | 400 | No | Request or output classified as dangerous (weapons, self-harm, etc.) | Safety category flag in response | Do not retry; log for policy review |
| Safety | Violence | 400 | No | Graphic violence detected in input/output | Category-specific safety flag (Gemini exposes per-category scores) | Revise prompt scope; escalate if pattern of abuse |
| Safety | Hate speech | 400 | No | Discriminatory/hateful content detected | Category-specific safety flag | Do not retry unmodified |
| Safety | Sexual content | 400 | No | Sexual/explicit content detected | Category-specific safety flag | Do not retry unmodified |
| Safety | Malware generation request | 400 | No | Prompt requests malicious code/exploit generation | Safety classifier flag | Do not retry; this is a policy violation, not a transient error |
| Safety | Prompt injection detected | 400 | No | Provider detects an attempt to override system instructions (common in tool-use/RAG pipelines) | Explicit injection-detection flag (where offered) | Sanitize retrieved/external content before including in prompt |
| Safety | Jailbreak attempt detected | 400 | No | Classifier flags an attempt to bypass safety training | Safety flag / refusal in output | Do not retry with obfuscated variants; review use case legitimacy |

> **Provider note:** Gemini returns granular `safetyRatings` per category with probability scores and a `blockReason`. OpenAI and Anthropic return a coarser `content_filter`/refusal signal, often as a normal `200` response where the model *itself* refuses in natural language, rather than a hard API-level block, so safety handling must check **both** the HTTP-level block and the model's own refusal text. Mistral and Cohere expose configurable moderation via a separate endpoint rather than inline blocking.

---

## 11. Content Errors

Failures in the *shape* of an otherwise-successful (`200`) response.

| Category | Error Name | HTTP Status | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|---|
| Content | Empty response | 200 | Yes (once) | Model returned zero content (rare, often transient) | `content`/`text` field is empty string | Retry once; if persistent, treat as a model-side incident |
| Content | Null response | 200 | Yes (once) | `content` field is `null` instead of a string/array | Explicit null check (not just falsy check) | Retry once, then fallback |
| Content | Missing candidates (Gemini) | 200 | Yes (once) | `candidates` array empty, usually a safety block under the hood | Check `promptFeedback.blockReason` | Inspect block reason before retrying |
| Content | Missing text in message | 200 | No | Model returned only a tool call / refusal with no text field expected downstream | Schema check on response before use | Handle tool-call-only responses as a distinct branch, not an error |
| Content | Invalid `finish_reason` / `stop_reason` | 200 | Depends | Unexpected value (e.g. `"content_filter"`, `"tool_calls"` when text was expected) | Whitelist expected finish reasons per call type | Branch handling logic on finish reason explicitly |
| Content | Blocked completion (post-hoc) | 200 | No | Output generated then withheld by a secondary safety pass | `finish_reason: "content_filter"` on an otherwise `200` | Do not retry unmodified; log and route to review if unexpected |

---

## 12. Structured Output Errors

Errors specific to JSON mode / schema-constrained generation.

| Category | Error Name | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|
| Structured Output | Invalid/broken JSON returned | Yes (1-2x) | Model produced non-parseable JSON despite JSON mode (rare with native structured output, more common with prompt-only JSON) | Wrap `JSON.parse` in try/catch | Retry with a "repair" instruction, or use native structured-output/function-calling mode instead of prompted JSON |
| Structured Output | Missing required property | Yes (1-2x) | Model omitted a `required` field from your schema | Validate response against the JSON Schema (ajv/jsonschema) | Retry with schema violation fed back to the model, or use strict mode if the provider supports it |
| Structured Output | Additional/unexpected properties | Sometimes | Model added extra fields beyond schema (if `additionalProperties: false` not enforced) | Schema validation | Set `additionalProperties: false` and/or `strict: true`; strip extras defensively |
| Structured Output | Invalid schema (client-side) | No | Your JSON Schema itself is malformed or uses unsupported keywords | Validate schema before sending, not just the response | Fix schema; check provider's supported JSON-Schema subset |
| Structured Output | Enum mismatch | Yes (1x) | Model returned a value outside the declared `enum` | Post-response enum check | Retry with explicit reminder, or map to nearest valid value |
| Structured Output | Array validation failed (min/maxItems, item type) | Yes (1x) | Model returned wrong array length/type | Schema validation | Retry with corrective feedback |
| Structured Output | Object validation failed (nested schema) | Yes (1x) | Nested object doesn't match sub-schema | Recursive schema validation | Retry with corrective feedback pinpointing the failing path |
| Structured Output | Recursive schema not supported | No | Provider rejects `$ref`-based recursive schemas | `400` on schema submission | Flatten schema or use provider-supported recursion depth |
| Structured Output | Schema too large/complex | No | Schema exceeds provider's size/depth/property-count limit | `400` with schema-size error | Simplify schema, split into multiple calls |

---

## 13. Streaming Errors

Errors specific to Server-Sent Events (SSE) / chunked streaming responses.

| Category | Error Name | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|
| Streaming | Stream interrupted mid-response | Yes | Network drop, server restart, load-balancer timeout during a long generation | `stream.on('error')` before a terminal `[DONE]`/`stop` event | Resume via context (if provider supports it) or restart the request with received-so-far text as a hint |
| Streaming | Partial response (truncated, no error event) | Yes | Connection closed without a clean terminal event | Track whether a terminal `finish_reason` chunk was received | Treat as incomplete; retry the full request |
| Streaming | SSE disconnected | Yes | Idle timeout, proxy buffering issue, intermediate CDN dropping the connection | `EventSource.onerror` / stream `close` without completion | Reconnect with backoff; disable proxy buffering (`X-Accel-Buffering: no`) |
| Streaming | Client closed connection early | No (by definition) | User navigated away / client aborted | `AbortController` signal fired | Clean up server-side resources; not a failure to retry |
| Streaming | Timeout during stream (no new chunks) | Yes | Model stalls mid-generation (rare) or network stalls | Per-chunk idle timeout (reset on each received chunk) | Abort and retry the full request after N seconds of silence |

---

## 14. Function / Tool Calling Errors

| Category | Error Name | HTTP Status | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|---|
| Tool Calling | Invalid tool definition sent | 400 | No | Malformed `tools` array in the request | `400` referencing tool index | Validate tool schemas before send (see Section 1) |
| Tool Calling | Missing tool (model calls undeclared tool) | 200 (app-level error) | No | Model hallucinated a tool name not in your `tools` list (rare with strict mode) | Compare `tool_calls[].name` against declared tools | Reject and re-prompt, or ignore and treat as text |
| Tool Calling | Invalid function/tool name mismatch | 200 (app-level) | No | Case mismatch or stale tool registry on your side | Name lookup fails at dispatch time | Keep tool registry in sync with what's sent per request |
| Tool Calling | Invalid arguments (schema violation) | 200 (app-level) | Yes (1x) | `tool_calls[].function.arguments` doesn't match declared parameter schema | Validate arguments against the tool's JSON Schema before executing | Retry with the model, feeding back the validation error |
| Tool Calling | JSON parse failure on arguments | 200 (app-level) | Yes (1x) | `arguments` string isn't valid JSON (older/non-strict tool-calling modes) | try/catch around `JSON.parse` | Retry generation, or use strict/native tool-calling mode which guarantees valid JSON |
| Tool Calling | Tool execution timeout | (your infra) | Yes | Your own tool/function handler hangs or is slow | Timeout wrapper around tool execution | Return a timeout error back to the model as a tool result so it can adapt |
| Tool Calling | Tool execution failed | (your infra) | Depends | Your tool handler throws (DB down, 3rd-party API failure, etc.) | try/catch around tool execution | Return structured error as the tool result; let the model decide next step, or fail the turn |

---

## 15. File Upload Errors

Applicable when the API supports file/image/document/audio uploads (inline base64 or a dedicated Files API).

| Category | Error Name | HTTP Status | Retryable | Cause | Detection | Recovery |
|---|---|---|---|---|---|---|
| File Upload | File too large | 413 | No | File exceeds per-file or per-request size cap | Pre-flight file size check | Compress, downsample, or split; use Files API instead of inline base64 for large files |
| File Upload | Unsupported file type | 400 | No | Extension/MIME not in the provider's supported list | Whitelist check pre-upload | Convert to a supported format before upload |
| File Upload | Corrupted file | 400 | No | Truncated upload, bad encoding, damaged source file | Checksum/integrity check before upload | Re-export/re-download source file, retry upload |
| File Upload | Upload timeout | 408 / network | Yes | Large file over a slow connection | Client-side timeout on upload call | Retry with resumable upload if supported, or increase timeout |
| File Upload | Missing file (referenced but not attached) | 400 | No | `file_id` referenced in a message but the upload never completed | Verify upload response before referencing `file_id` | Confirm upload success before using the file in a completion call |
| File Upload | Invalid MIME type declared vs. actual | 400 | No | Declared `Content-Type` doesn't match actual file bytes | Server-side sniffing (provider) / client-side magic-byte check | Set correct `Content-Type` matching actual content |

---

## 16. SDK-Specific Errors

Common exceptions surfaced by each SDK/tool on top of the raw HTTP/network errors above.

| SDK / Tool | Common Exception Types | Notes |
|---|---|---|
| **Node.js (`https`/`fetch`)** | `TypeError: fetch failed`, `AbortError`, `FetchError`, raw `Error` with `.code` (`ECONNRESET`, `ETIMEDOUT`) | Native `fetch` wraps network errors in a generic `TypeError`; inspect `.cause` for the real `errno`/`code`. |
| **Python (`requests`/`httpx`)** | `requests.exceptions.ConnectionError`, `Timeout`, `HTTPError`, `JSONDecodeError`; `httpx.ConnectError`, `httpx.ReadTimeout`, `httpx.HTTPStatusError` | `response.raise_for_status()` converts `4xx`/`5xx` into `HTTPError`/`HTTPStatusError`; must still parse `.response.json()` for the provider's error body. |
| **Provider Python SDKs (OpenAI, Anthropic)** | `APIConnectionError`, `APITimeoutError`, `AuthenticationError`, `PermissionDeniedError`, `NotFoundError`, `RateLimitError`, `InternalServerError`, `BadRequestError`, `APIStatusError` | Typed exception hierarchy mapped 1:1 to status codes, catch the most specific type first. |
| **Java (`HttpClient` / provider SDK)** | `java.net.UnknownHostException`, `java.net.ConnectException`, `java.net.SocketTimeoutException`, `javax.net.ssl.SSLHandshakeException`, SDK-specific `ApiException` | Checked exceptions require explicit `throws`/try-catch at every call site, easy to accidentally swallow. |
| **Go (`net/http` / provider SDK)** | `*url.Error` wrapping `context.DeadlineExceeded`, `net.DNSError`, `*net.OpError` (`ECONNREFUSED`/`ECONNRESET`), SDK typed `*apierror.Error` | Idiomatic Go returns `(resp, err)`, always check `err != nil` *and* `resp.StatusCode` separately; a non-nil response can still carry an error status. |
| **cURL** | Exit codes: `6` (couldn't resolve host), `7` (couldn't connect), `28` (timeout), `35` (SSL connect error), `52` (empty reply), `56` (recv failure) | Script wrappers should check `$?` against these codes, not just parse stdout. |
| **Axios (JS)** | `AxiosError` with `.code` (`ECONNABORTED`, `ERR_NETWORK`), `.response` (present for HTTP errors), `.request` (present but no `.response` = network-level failure) | Classic gotcha: `error.response` undefined means the request never got a server reply, treat as network error, not an API error. |
| **Fetch API (browser/Node 18+)** | Rejects only on network failure; **does not** reject on `4xx`/`5xx`! | Must manually check `response.ok`/`response.status`, a `500` response resolves the promise successfully. |
| **n8n HTTP Request Node** | `NodeApiError`, `NodeOperationError`, generic "Problem in node" with embedded HTTP status | Enable "Continue on Fail" + check `$json.error`/`$json.statusCode` in a downstream IF node rather than letting the workflow hard-stop. |

---

## 17. Retry Strategy

| Error Class | Should Retry? | Max Retries | Base Interval | Backoff | Jitter | Circuit Breaker |
|---|---|---|---|---|---|---|
| Network errors (DNS, reset, timeout) | Yes | 3-5 | 500ms-1s | Exponential (x2) | Yes, +/-20% | Open after 5 consecutive failures |
| `408` Request Timeout | Yes | 2-3 | 1s | Exponential | Yes | Count toward breaker |
| `429` Rate Limited | Yes | 5+ (bounded by budget, not count) | Honor `Retry-After` | Exponential, capped ~60s | Yes | Reduce concurrency, don't hard-open breaker |
| `500` Internal Server Error | Yes | 3 | 1s | Exponential | Yes | Count toward breaker |
| `502`/`503`/`504` | Yes | 3-5 | 1-2s | Exponential | Yes | Open breaker if sustained >30s |
| `400` Bad Request | No | 0 | - | - | - | N/A, fix the payload |
| `401`/`403` Auth/Authz | No | 0 (except one silent OAuth-refresh retry) | - | - | - | Alert, don't spin |
| `404` Not Found | No | 0 | - | - | - | Alert, likely a config error |
| Safety-blocked content | No | 0 (unmodified) | - | - | - | Route to human review instead |
| Structured-output validation failure | Yes (with correction) | 1-2 | Immediate | None needed | No | N/A |
| Streaming interruption | Yes | 2-3 | Immediate-1s | Linear | No | N/A |

**Circuit breaker recommendation:** trip to "open" after a threshold of consecutive/rate-based failures (e.g. 5 failures in 10 requests), stop sending real traffic for a cool-down window (e.g. 30s), then allow a small number of "half-open" probe requests before fully closing again. This protects both your system (no wasted retries) and the provider (no pile-on during an incident).

---

## 18. Logging Recommendations

Log the following for **every** request, success or failure:

| Field | Why |
|---|---|
| Timestamp | Correlate with provider status pages / other logs |
| Request ID (`x-request-id` or provider-issued ID) | Required for provider support tickets |
| Model | Track per-model error rates and cost |
| Endpoint / route | Distinguish chat vs. embeddings vs. files, etc. |
| HTTP Status Code | Primary triage signal |
| Error Code (provider-specific string) | Finer-grained than status code alone |
| Error Message | Human-readable detail for debugging |
| Request Body (**sanitized**, redact PII, secrets, and full prompt if sensitive) | Reproduce issues without leaking data into logs |
| Response Body (or truncated summary) | Diagnose content-shape issues |
| Latency (ms) | Detect degradation before it becomes hard failures |
| Retry Count / attempt number | Understand true failure rate vs. retry-masked failure rate |
| Token usage (`prompt_tokens`, `completion_tokens`) | Cost tracking, context-limit debugging |

Never log raw API keys, full user PII, or unredacted file contents, hash or truncate as needed.

---

## 19. Recovery Strategy

- **Retry** — for transient, idempotent failures (network blips, `429`, `5xx`). Always with backoff + jitter.
- **Fail Fast** — for deterministic client errors (`400`, `401`, `403`, `404`) where retrying an unmodified request will only reproduce the same failure.
- **Ignore** — for non-critical, best-effort calls (e.g. optional telemetry enrichment) where a fallback default is acceptable.
- **Queue** — buffer requests during rate-limit or outage windows and replay them once capacity/service returns, rather than dropping work.
- **Dead Letter Queue (DLQ)** — after exhausting retries, move the failed job to a DLQ for offline inspection instead of silently dropping it.
- **Human Review** — for safety-blocked content, repeated structured-output failures, or anything a machine can't safely auto-correct.
- **Alert** — page/notify on-call for auth failures, billing issues, sustained `5xx`/`429` rates, or circuit-breaker trips, these indicate systemic, not per-request, problems.
- **Fallback Model** — degrade gracefully to a secondary model/provider when the primary is overloaded or deprecated, if your use case tolerates quality/format differences.
- **Cached Response** — for read-heavy or idempotent prompts, serve a recent cached completion during an outage rather than failing the user-facing request entirely.

---

## 20. n8n-Specific Errors

| Node / Component | Error | Cause | Detection | Recovery |
|---|---|---|---|---|
| HTTP Request Node | "Problem in node" (wraps HTTP/network error) | Any of the HTTP/network errors above | Check `$json.error`, `statusCode` with "Continue on Fail" enabled | Route failures to an IF/Switch node; implement retry via "Retry On Fail" node setting |
| AI Agent Node | Tool execution error / max iterations reached | Sub-tool call failed repeatedly, or agent loop didn't converge | Node execution log, agent's intermediate steps | Lower `maxIterations`, add explicit fallback tool, review prompt |
| Code Node | `ReferenceError`/`TypeError` in custom JS/Python | Bug in inline code, unexpected input shape | Node execution error panel | Wrap logic in try/catch, validate `items` shape before use |
| Split Out Node | "No output data" / field not found | Referenced field path doesn't exist on incoming items | Check field name against actual JSON structure | Add a Set/IF node upstream to guarantee field presence |
| Loop Over Items (Split in Batches) | Infinite loop / workflow timeout | Loop-continuation logic never terminates | Execution duration monitoring | Add explicit exit condition and a max-iteration guard |
| Merge Node | Mismatched item counts between branches | Upstream branches produce unequal item counts for "combine" mode | Compare item counts pre-merge | Use "Append" mode or normalize counts before merging |
| Wait Node | Workflow resumes with stale data | Long wait + upstream data changed/expired (e.g. expired auth token) | Check token/data freshness on resume | Re-fetch/re-auth immediately after a Wait node, don't reuse pre-wait context blindly |
| Execute Workflow Node | Sub-workflow error not propagated | Sub-workflow's own error handling swallows the failure | Check sub-workflow's error output explicitly | Explicitly return error status from sub-workflow, don't rely on implicit propagation |
| Expression Evaluation | `[object Object]` output / expression error | Referencing undefined path, wrong node name in expression | Red-underlined expression field in editor, execution error | Test expressions with a Set node first, use optional chaining (`?.`) |
| Binary Data | "Binary data not found" | Referencing binary property that wasn't produced upstream | Check "Binary" tab of upstream node's output | Confirm upstream node actually outputs binary (e.g. correct HTTP Request "Response Format") |
| JSON Parsing | "Unexpected token" in Function/Code node | LLM/API returned non-JSON text where JSON was expected | try/catch around `JSON.parse` in Code node | Validate/repair before parsing; treat parse failure as its own branch, not a workflow crash |
| Timeout | Workflow/node execution timeout | Long-running LLM call exceeds n8n's configured timeout | Workflow execution log shows timeout status | Raise node-level timeout for LLM calls; move very long jobs to an async/webhook pattern |
| Workflow Memory | "JavaScript heap out of memory" | Very large payloads (big files, huge arrays) held in memory across nodes | Instance-level memory alerts, crash logs | Stream/paginate large data, offload large binaries to external storage instead of passing inline |
| Execution Cancellation | Manual/timeout cancellation mid-run | User or system cancels a long-running execution | Execution status = "Canceled" | Design idempotent steps so a re-run doesn't duplicate side effects (emails sent, records created) |

---

## Appendix A: Complete Error Hierarchy

```
LLM API Request Error
│
├── Pre-Flight (never reaches the network)
│   ├── Client-Side Validation Errors (Section 1)
│   └── Local config errors (missing env var, malformed base URL)
│
├── Network Layer (never reaches the HTTP layer)
│   └── Network Errors (Section 6): DNS, connection, TLS/SSL, proxy/VPN
│
├── HTTP Response Received
│   ├── 4xx Client Errors
│   │   ├── 400 Bad Request        → Validation (Section 1) / Structured Output (Section 12)
│   │   ├── 401 Unauthorized       → Authentication (Section 2)
│   │   ├── 403 Forbidden          → Authorization (Section 3) / Safety (Section 10)
│   │   ├── 404 Not Found          → Model Errors (Section 8)
│   │   ├── 405/406/411/414/415/416/426/431 → Protocol-level HTTP Errors (Section 4)
│   │   ├── 408 Request Timeout    → HTTP Errors (Section 4)
│   │   ├── 409 Conflict           → HTTP Errors (Section 4)
│   │   ├── 413 Payload Too Large  → HTTP Errors (Section 4) / File Upload (Section 15)
│   │   ├── 422 Unprocessable      → HTTP Errors (Section 4)
│   │   ├── 425/428 Precondition/Early → HTTP Errors (Section 4)
│   │   └── 429 Too Many Requests  → Rate Limiting (Section 7)
│   │
│   ├── 5xx Server Errors
│   │   └── 500/501/502/503/504/507/508 → Server Errors (Section 5) / Model Errors (Section 8)
│   │
│   └── 2xx "Successful" but Semantically Broken
│       ├── Content Errors (Section 11): empty/null/missing candidates
│       ├── Token Errors (Section 9): finish_reason = length
│       ├── Safety Errors (Section 10): finish_reason = content_filter
│       ├── Structured Output Errors (Section 12): schema-invalid JSON
│       ├── Tool Calling Errors (Section 14): bad arguments, hallucinated tool
│       └── Streaming Errors (Section 13): interrupted before terminal event
│
├── Application / Orchestration Layer
│   ├── SDK Errors (Section 16): typed exceptions per language
│   └── n8n / Workflow Errors (Section 20): node-level failures
│
└── Cross-Cutting Concerns (apply to any node above)
    ├── Retry Strategy (Section 17)
    ├── Logging (Section 18)
    └── Recovery Strategy (Section 19)
```

---

## Appendix B: Retry Matrix

| HTTP Status | Retry? | Strategy | Notes |
|---|---|---|---|
| Network error (no status) | Yes | Exponential backoff + jitter, 3-5 attempts | Circuit-break after repeated failures |
| 400 | No | Fix request | Never retry unmodified |
| 401 | No* | *One retry allowed only after a token refresh | Otherwise fail fast |
| 403 | No | Escalate to human/admin | Not transient |
| 404 | No | Fix endpoint/model reference | Config error |
| 405/406/411/414/415/416/426/431 | No | Fix request construction | Client bug |
| 408 | Yes | Backoff, 2-3 attempts | Consider raising client timeout |
| 409 | Maybe | Reconcile state, retry with fresh idempotency key | Not blind retry |
| 410 | No | Migrate to current resource | Permanent |
| 413 | No | Reduce payload size | Not transient |
| 422 | No | Fix conflicting parameters | Client bug |
| 425 | Yes | Retry without 0-RTT | Rare in practice |
| 428 | No | Add required header | Config fix |
| 429 | Yes | Honor `Retry-After`, exponential backoff + jitter | Also throttle proactively via rate-limit headers |
| 500 | Yes | Backoff, 3 attempts | Alert if sustained |
| 501 | No | Not supported here | Use different endpoint/feature |
| 502/503/504 | Yes | Backoff, 3-5 attempts | Circuit-break if sustained >30s |
| 507/508 | Yes/No | 507 retry after delay; 508 alert, don't retry | Rare, infra-level |

---

## Appendix C: Status Code Reference

| Code | Meaning | Typical LLM-API Cause |
|---|---|---|
| 200 | OK | Successful call — still validate content shape (Section 11) |
| 400 | Bad Request | Invalid payload, schema, or parameters |
| 401 | Unauthorized | Missing/invalid/expired credentials |
| 403 | Forbidden | Valid credentials, insufficient permission/billing/region |
| 404 | Not Found | Wrong endpoint or unknown model |
| 405 | Method Not Allowed | Wrong HTTP verb |
| 406 | Not Acceptable | Unsupported `Accept` header |
| 408 | Request Timeout | Client too slow sending request |
| 409 | Conflict | Idempotency key reuse with different payload |
| 410 | Gone | Endpoint/model permanently retired |
| 411 | Length Required | Missing `Content-Length` |
| 413 | Payload Too Large | Prompt/files exceed size limit |
| 414 | URI Too Long | Oversized query string |
| 415 | Unsupported Media Type | Wrong `Content-Type` |
| 416 | Range Not Satisfiable | Bad `Range` header |
| 422 | Unprocessable Entity | Semantically invalid request |
| 425 | Too Early | 0-RTT rejection |
| 426 | Upgrade Required | Deprecated protocol |
| 428 | Precondition Required | Missing conditional header |
| 429 | Too Many Requests | Rate limit / quota exceeded |
| 431 | Header Fields Too Large | Oversized headers |
| 500 | Internal Server Error | Unhandled provider-side exception |
| 501 | Not Implemented | Feature not supported on this route |
| 502 | Bad Gateway | Upstream failure |
| 503 | Service Unavailable | Overload / maintenance / model unavailable |
| 504 | Gateway Timeout | Upstream too slow |
| 507 | Insufficient Storage | Provider storage exhaustion |
| 508 | Loop Detected | Infra-level proxy loop |
| 529 | Overloaded (Anthropic-specific) | Anthropic capacity exceeded |

---

## Appendix D: Decision Tree for Handling Errors

```
Request fails
│
├─ Did the request even leave the client?
│   ├─ No → Client-side validation/config error
│   │        → Fail fast, fix payload, do NOT retry
│   │
│   └─ Yes → Continue
│
├─ Was there an HTTP response at all?
│   ├─ No (network/timeout error)
│   │        → Is it likely transient? (DNS blip, reset, timeout)
│   │            ├─ Yes → Retry with exponential backoff + jitter (cap 3-5 attempts)
│   │            └─ No (TLS/cert/config) → Fix configuration, alert, do NOT retry
│   │
│   └─ Yes → Check status code
│       │
│       ├─ 2xx → Validate response SHAPE before treating as success
│       │        ├─ Empty/null content → Retry once, then fallback
│       │        ├─ finish_reason = content_filter → Do NOT retry; route to review
│       │        ├─ finish_reason = length → Increase max_tokens or continue generation
│       │        ├─ Structured-output schema violation → Retry w/ correction (1-2x)
│       │        └─ Well-formed → Success, proceed
│       │
│       ├─ 4xx (client error)
│       │        ├─ 401/403 → Fix credentials/permissions; alert; do NOT blind-retry
│       │        ├─ 404 → Fix endpoint/model reference; alert
│       │        ├─ 429 → Honor Retry-After; exponential backoff + jitter; throttle
│       │        └─ Other 4xx → Fix payload; do NOT retry unmodified
│       │
│       └─ 5xx (server error)
│                └─ Retry with exponential backoff + jitter (cap 3-5 attempts)
│                    ├─ Still failing after max retries? → Circuit-break, fallback model/cache, alert
│                    └─ Succeeded on retry → Log retry count, proceed
│
└─ After all handling: log full context (Section 18) regardless of outcome
```

---

## Appendix E: Best Practices

1. **Validate before you send.** Catch schema, type, and range errors client-side — every request you don't send is a request that can't fail server-side.
2. **Use typed SDKs where available.** OpenAI/Anthropic/etc. SDKs give you a typed exception hierarchy instead of parsing raw status codes and strings.
3. **Never trust `response.ok` alone with `fetch`.** It doesn't reject on `4xx`/`5xx` — always check `.ok`/`.status` explicitly.
4. **Distinguish "no response" from "bad response."** Axios's `error.request` without `error.response`, and Go's non-nil `err` with a non-nil `resp`, are both classic sources of misclassified errors.
5. **Always set a request timeout.** An LLM call with no client-side timeout can hang indefinitely on a stalled connection.
6. **Honor `Retry-After` over your own backoff schedule** whenever the server provides it.
7. **Add jitter to every backoff.** Uncoordinated synchronized retries from many clients cause secondary outages ("thundering herd").
8. **Treat `200` as necessary, not sufficient.** Always validate `finish_reason`, content presence, and (if applicable) schema conformance before treating a response as usable.
9. **Pre-count tokens with the real tokenizer**, not a character-count heuristic, especially for non-English or code-heavy prompts.
10. **Use native structured-output/function-calling modes** instead of "please respond in JSON" prompting — it collapses an entire category of parse errors.
11. **Sanitize untrusted content before it enters a prompt** (RAG chunks, tool outputs, user-pasted text) to reduce prompt-injection risk.
12. **Circuit-break, don't retry forever.** Uncapped retries against a genuinely down dependency waste resources and delay failover.
13. **Log request IDs on every call**, success or failure — they're the fastest path to provider support resolving an incident.
14. **Redact secrets and PII from logs**, but keep enough of the sanitized payload to reproduce the bug.
15. **Build a fallback path** (secondary model, cached response, or graceful degradation) for anything user-facing and latency-sensitive.
16. **Version-pin models deliberately**, and track provider deprecation calendars so migrations are planned, not emergency fire drills.
17. **Test your error handling, not just your happy path** — simulate `429`s, timeouts, and malformed responses in CI/staging.

---

## Appendix F: Common Mistakes

- **Treating every error the same way** (blanket retry-3-times-then-fail) instead of branching on whether the error is actually retryable.
- **Retrying non-idempotent side effects** (e.g. a tool call that sends an email) without deduplication, causing duplicate actions.
- **Not checking `finish_reason`** and silently accepting truncated or safety-blocked output as if it were complete.
- **Assuming `fetch`/`axios` throws on HTTP errors** the same way it throws on network errors — it doesn't, by default.
- **Hardcoding model names** without a fallback path, so a single deprecation breaks production overnight.
- **No client-side timeout**, leading to connections that hang indefinitely and exhaust connection pools.
- **Retrying immediately without backoff**, which turns a brief provider hiccup into a self-inflicted rate-limit spiral.
- **No jitter**, causing synchronized retry storms across many concurrent workers/instances.
- **Logging full raw request/response bodies** including API keys, user PII, or entire documents — a compliance and security liability.
- **Parsing model output with regex instead of a schema validator**, which breaks silently on minor format drift.
- **Ignoring `x-ratelimit-*` headers** and only reacting after a hard `429`, instead of throttling proactively.
- **Not distinguishing a model *refusal* (a normal `200` where the model itself declines) from an API-level safety block** — these need different handling logic.
- **Swallowing errors in a `try/catch` with no logging**, making production incidents invisible until a user complains.
- **Using a single global retry count across a whole batch job** instead of per-request tracking, masking which specific items actually failed.
- **Not testing the failure paths at all** — many teams only load-test the happy path before shipping.

---

## Appendix G: Pre-Production Checklist

- [ ] All request fields are validated client-side against the provider's schema before sending
- [ ] API keys/secrets are loaded from a secrets manager, never hardcoded, and never logged
- [ ] A request timeout is set on every outbound call (including streaming idle timeouts)
- [ ] Retry logic exists and is scoped **only** to retryable error classes (network, 429, 5xx)
- [ ] Retries use exponential backoff **with jitter**, and respect `Retry-After` when present
- [ ] A circuit breaker (or equivalent) exists to stop retry storms during sustained outages
- [ ] Every response is validated for content shape (`finish_reason`, non-empty content) before use, even on `200`
- [ ] Structured-output/tool-call responses are validated against their JSON Schema, with a corrective retry path
- [ ] Token usage is pre-counted with the provider's actual tokenizer, with headroom before context limits
- [ ] Safety-blocked responses are routed to a distinct handling path (not retried unmodified)
- [ ] A fallback model or cached-response path exists for user-facing, latency-sensitive flows
- [ ] Structured logs capture timestamp, request ID, model, status/error code, latency, and retry count for every call
- [ ] Logs redact API keys, PII, and full file contents
- [ ] Alerts are configured for: auth failures, billing/quota errors, sustained 5xx/429 rates, and circuit-breaker trips
- [ ] Failed jobs (after retry exhaustion) land in a dead-letter queue rather than being silently dropped
- [ ] Model deprecation dates are tracked and migration is planned ahead of forced sunsets
- [ ] Error handling has been tested against simulated 429s, timeouts, malformed JSON, and dropped streams — not just the happy path
- [ ] Idempotency keys (where supported) are used for any retried request with side effects