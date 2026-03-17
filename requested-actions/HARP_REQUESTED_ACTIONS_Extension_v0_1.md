# HARP Extension: Requested Actions — v0.1

**Namespace:** `org.harp.requestedActions`
**Status:** Draft
**Version:** 0.1

---

## 1. Overview

This extension allows an enforcer to embed a set of **requested UI actions** within a HARP artifact. The approver application renders these actions as buttons (or equivalent controls) in place of the default Approve / Reject pair.

Every action maps to a strict HARP Core decision (`allow` or `deny`). Captions and styles are **presentation hints only** — they influence the UI but never change the underlying decision semantics.

---

## 2. Extension Key

```
org.harp.requestedActions
```

Placed inside the artifact plaintext's `extensions` object, which is defined in the [HARP Core Artifact Schema](../core/harp-core-artifact.schema.json).

---

## 3. Normative Model

```json
{
  "org.harp.requestedActions": {
    "version": 1,
    "actions": [
      {
        "id": "approve",
        "caption": "Approve",
        "style": "primary",
        "decision": "allow"
      },
      {
        "id": "reject",
        "caption": "Reject",
        "style": "danger",
        "decision": "deny"
      }
    ]
  }
}
```

### 3.1 Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | integer | Yes | Extension schema version. Must be `1`. |
| `actions` | array | Yes | Ordered list of action items. |
| `actions[].id` | string | Yes | Unique identifier within this extension instance (case-insensitive). |
| `actions[].caption` | string | Yes | Human-readable button label. Non-empty, trimmed. |
| `actions[].style` | string | No | Visual hint: `"primary"`, `"secondary"`, `"danger"`. Defaults to `"secondary"`. Unknown values normalize to `"secondary"`. |
| `actions[].decision` | string | Yes | HARP Core decision: `"allow"` or `"deny"`. |

---

## 4. Validation Rules

1. `version` MUST equal `1`. Any other value → extension is **ignored** (fallback to defaults).
2. `actions` array MUST contain 1–5 items.
3. Action `id` values MUST be unique (case-insensitive comparison).
4. `caption` MUST be a non-empty string after trimming whitespace.
5. `decision` MUST be exactly `"allow"` or `"deny"` (case-sensitive).
6. `style` is optional. Unknown values normalize to `"secondary"` — they MUST NOT cause validation failure.

If **any** validation rule fails, the entire extension MUST be ignored and the approver MUST fall back to the default Approve / Reject buttons.

---

## 5. Fallback Behavior

Approvers MUST implement the following fallback logic:

| Condition | Behavior |
|-----------|----------|
| `extensions` field absent | Show default Approve + Reject |
| `org.harp.requestedActions` key absent | Show default Approve + Reject |
| Extension present but fails validation | Log warning, show default Approve + Reject |
| Decryption failure (ciphertext unavailable) | Show default Approve + Reject |
| Unknown `version` value | Show default Approve + Reject |

**Default actions** (used in all fallback cases):
```json
[
  { "id": "approve", "caption": "Approve", "style": "primary", "decision": "allow" },
  { "id": "reject", "caption": "Reject", "style": "danger", "decision": "deny" }
]
```

---

## 6. Canonicalization & Hash Binding

The `extensions` object is part of the artifact plaintext. Under the HARP canonicalization process (JCS / RFC 8785 → SHA-256), the extension data is **automatically included** in the artifact hash.

This means:
- Any modification to the extension data changes the artifact hash.
- The signed decision covers the _exact_ set of requested actions that were presented to the approver.
- No additional canonicalization steps are required beyond the standard HARP artifact hashing.

---

## 7. Security Properties

1. **Non-authoritative presentation**: Captions and styles are hints. The underlying decision (`allow`/`deny`) is what controls enforcement.
2. **Hash-bound**: Extensions are inside the encrypted, hashed artifact — tampering invalidates the hash.
3. **Zero-knowledge gateway**: Extension data is inside the encrypted ciphertext. The gateway never sees it.
4. **No decision escalation**: An action mapped to `deny` cannot become `allow` through the extension. The enforcer's original mapping is final.

---

## 8. Interoperability Matrix

| Enforcer | Mobile App | Result |
|----------|-----------|--------|
| Old (no extensions) | Old (hardcoded buttons) | Works — existing behavior |
| Old (no extensions) | New (dynamic buttons) | Works — mobile falls back to default buttons |
| New (with extensions) | Old (hardcoded buttons) | Works — old mobile ignores unknown JSON key |
| New (with extensions) | New (dynamic buttons) | Works — mobile renders dynamic buttons |

---

## 9. Examples

### 9.1 Default Approve / Reject

```json
{
  "org.harp.requestedActions": {
    "version": 1,
    "actions": [
      { "id": "approve", "caption": "Approve", "style": "primary", "decision": "allow" },
      { "id": "reject", "caption": "Reject", "style": "danger", "decision": "deny" }
    ]
  }
}
```

### 9.2 Custom Captions

```json
{
  "org.harp.requestedActions": {
    "version": 1,
    "actions": [
      { "id": "allow-once", "caption": "Allow Once", "style": "primary", "decision": "allow" },
      { "id": "block", "caption": "Block Command", "style": "danger", "decision": "deny" }
    ]
  }
}
```

### 9.3 Multiple Options with Same Decision

```json
{
  "org.harp.requestedActions": {
    "version": 1,
    "actions": [
      { "id": "proceed", "caption": "Proceed", "style": "primary", "decision": "allow" },
      { "id": "allow-dry-run", "caption": "Dry-Run Only", "style": "secondary", "decision": "allow" },
      { "id": "deny-action", "caption": "Deny", "style": "danger", "decision": "deny" }
    ]
  }
}
```
