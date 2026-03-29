---
name: generate-api-layer
description: >-
  Generates or updates typed API service modules and companion type files from
  src/api/spec.json (method, URL, body, params, response) grouped by service
  name. Merges into existing files when present without duplicating operations or
  types, and fills in missing fields on existing definitions. Use when the user
  asks to scaffold API clients, sync services from the API spec, add endpoints
  to src/api, or applies this workflow for nabel-beauty.
---

# Generate API layer from spec

## When this applies

The canonical spec is **`src/api/spec.json`**. Read that file only—do not ask for a path or accept alternates. Implement or update TypeScript under `src/api/` to match the project’s existing patterns.

## Before coding

1. Read `src/api/spec.json` and [reference.md](reference.md) for schema and merge rules.
2. Read the live implementations to stay aligned:
   - `src/api/services/base.service.ts` — use `apiRequest<TData>(config)`; responses are `ApiEnvelope`-wrapped; unwrapped `data` is `TData`.
   - `src/api/types/base.type.ts` — reuse `HttpMethod`, `ApiRequestConfig`, `RequestParams`.

If `src/api/spec.json` is missing, create it as `{ "services": {} }` when bootstrapping, or tell the user the file is required before generation.

## Output layout

For each service key in the spec (e.g. `auth`):

| Artifact | Path |
|----------|------|
| Types | `src/api/types/<stem>.type.ts` |
| Service functions | `src/api/services/<stem>.service.ts` |

Use a **stable file stem** per service (lowercase; hyphenate multi-word names). Match an existing stem if that service already has files.

## Implementation rules

1. **Imports**: From service files, import `apiRequest` from `./base.service` (same folder). Import operation types from `../types/<stem>.type` (or `@/api/types/<stem>.type` if the project already prefers `@/` for api—match surrounding files in `src/api`).

2. **Naming**:
   - One exported async function per operation: `camelCase` from `operation.name`.
   - Types: `PascalCase` from operation name plus suffix, e.g. `LoginBody`, `LoginResponse`, or `AuthLoginBody` / `AuthLoginResponse` when avoiding collisions across services in shared type files—prefer **per-service type files** so `LoginBody` inside `auth.type.ts` is enough.

3. **Service function body**:
   - Call `apiRequest<ResponseType>({ url: <operation.url>, method, data, params, signal })`.
   - Include `signal` in the public function signature when appropriate: `options?: { signal?: AbortSignal }` merged into config.
   - Do not reimplement fetch or envelope parsing.

4. **Types**:
   - Model `body` / `params` / `response` from the spec as TypeScript types (interfaces or type aliases). Use `unknown` or existing project primitives only when the spec is vague; otherwise mirror the spec structure.

## If files do not exist

Create `<stem>.type.ts` and `<stem>.service.ts`, add the new types and functions, then add barrel lines to `src/api/types/index.ts` and `src/api/services/index.ts`.

## If files exist

1. Parse the file (exports, interfaces, functions).
2. For each spec operation:
   - If a function with the same **name** already exists: compare **method**, **url**, and type usage. If everything matches the spec, **skip** that operation.
   - If the function exists but **omit** `signal`, **params**, or typings that the spec includes, **add** the missing parameters and types without renaming unrelated exports.
   - If types exist but the spec adds **properties** to `body`, `params`, or `response`, **extend** those types with the new fields (do not delete fields the app may rely on).
3. If the spec duplicates an endpoint under a **different** operation name, treat as different functions; if method+url match **and** names differ, avoid two functions with identical behavior—prefer clarifying with the user or collapsing to one exported name consistent with the newer spec.

## Consistency

- Follow `strict` TypeScript and existing formatting in `src/api`.
- Do not add noise comments; optional brief WHY only if the merge logic is non-obvious.
- Do not run dev servers or production builds unless the user explicitly asks.

## Checklist

- [ ] `src/api/spec.json` read and services enumerated
- [ ] Per-service types and service files created or merged
- [ ] No duplicate operations for the same method+url+name combination
- [ ] Barrel `index.ts` files export new modules
- [ ] Imports and `apiRequest` usage match `base.service.ts`
