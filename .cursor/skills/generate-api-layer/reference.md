# API spec reference

## Spec file

**Path:** `src/api/spec.json` (only; no user-supplied path).

**Format:** JSON. The agent reads this file directly when generating or updating the API layer.

## Shape

### Top level

| Field | Required | Description |
|-------|----------|-------------|
| `services` | Yes | Map of service key → service definition. |

### Service definition

| Field | Required | Description |
|-------|----------|-------------|
| `operations` | Yes | Array of endpoint definitions for this service. |

### Operation definition

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique within the service; drives function name (`camelCase`). |
| `method` | Yes | `GET` \| `POST` \| `PUT` \| `PATCH` \| `DELETE` \| `HEAD` \| `OPTIONS` |
| `url` | Yes | Path (e.g. `/users`) or absolute URL. Passed to existing `resolveUrl` via `apiRequest`. |
| `body` | No | Object describing JSON body fields (or `null` if none). Types become request/body types. |
| `params` | No | Same shape as `body` but for query parameters. |
| `response` | Yes | Object describing successful `data` payload (inside API envelope). |

Nested objects in `body`, `params`, or `response` define nested TypeScript types. Use consistent property names; for arrays in JSON, describe element shape in nested objects (the generator maps them to typed arrays).

### Optional fields (`body`, `params`, `response`)

Use a **type description string** as the JSON value for each leaf field (e.g. `"string"`, `"number"`, `"boolean"`). To mark that field as optional in generated TypeScript, append a space and **` <optional>`** (angle brackets included) to that string.

| Spec value | Generated meaning |
|------------|-------------------|
| `"string"` | required `string` |
| `"string <optional>"` | optional `string` (`property?: string`) |
| `"number <optional>"` | optional `number` |

Rules:

- Apply only to **leaf** entries whose value is a single type token (the usual string placeholders). Do not add `<optional>` to structural values such as nested objects or arrays; make individual properties inside those structures optional instead.
- Apply in **`body`**, **`params`**, and **`response`** the same way.
- When generating types, strip the ` <optional>` suffix, map the remainder to the TypeScript primitive or alias, and emit the property as optional (`?`).

### Example (JSON)

```json
{
  "services": {
    "auth": {
      "operations": [
        {
          "name": "login",
          "method": "POST",
          "url": "/auth/login",
          "body": {
            "email": "string",
            "password": "string",
            "rememberMe": "boolean <optional>"
          },
          "response": {
            "token": "string",
            "expiresAt": "string",
            "refreshToken": "string <optional>"
          }
        }
      ]
    }
  }
}
```

## File mapping

- Service key `auth` → `src/api/services/auth.service.ts` and `src/api/types/auth.type.ts`
- Normalize file stem: lowercase, hyphenate multi-word keys (`userProfile` → `user-profile` if you use multiple words in the key)

If the project already uses a different stem for the same domain, reuse the existing filename.

## Identity and merging

- **Operation identity**: pair `(method uppercase, normalized url string, operation name)`.
- **Skip** when an exported function with the same name already implements the same method and URL and types are equivalent.
- **Append** when the function or types are missing.
- **Augment** when a type or function exists but the spec adds fields or stricter shapes: add missing properties to interfaces; prefer extending interfaces over duplicating. If a conflict cannot be reconciled without breaking changes, stop and note it for the user.

## Barrel exports

After edits, ensure `src/api/services/index.ts` and `src/api/types/index.ts` each export the service module if not already:

`export * from "./<stem>.service";` and `export * from "./<stem>.type";`
