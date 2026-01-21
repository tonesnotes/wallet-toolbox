# Unified Group Permissions Architecture Refactor

## Problem Statement

Currently, the `WalletPermissionsManager` has two separate modes for permission handling:

1. **Regular Permissions**: Individual `ensure*` methods trigger one-off permission prompts via `requestPermissionFlow()`, resulting in multiple sequential user prompts.

2. **Group Permissions**: Only enforced during `waitForAuthentication()`, where the manifest is fetched and all missing permissions are requested at once.

This creates a poor user experience because:
- If an app doesn't call `waitForAuthentication()` first, users see individual permission prompts
- Even if `waitForAuthentication()` is called, any permissions not in the manifest still trigger individual prompts later
- The manifest is fetched fresh every time `waitForAuthentication()` is called (no caching)

## Goal

Unify the permission flow so that **any** `ensure*` call automatically checks for group permissions in the app's manifest and triggers a single grouped permission request for all missing permissions, rather than individual prompts.

---

## Current Architecture

### Affected Components

| Component | Location | Current Behavior |
|-----------|----------|------------------|
| `ensureProtocolPermission()` | Lines 890-1004 | Calls `requestPermissionFlow()` for individual prompt |
| `ensureBasketAccess()` | Lines 1010-1073 | Calls `requestPermissionFlow()` for individual prompt |
| `ensureCertificateAccess()` | Lines 1078-1161 | Calls `requestPermissionFlow()` for individual prompt |
| `ensureSpendingAuthorization()` | Lines 1166-1230 | Calls `requestPermissionFlow()` for individual prompt |
| `ensureLabelAccess()` | Lines 1236-1290 | Calls `requestPermissionFlow()` for individual prompt |
| `requestPermissionFlow()` | Lines 1296-1349 | Fires individual `onXXXRequested` events |
| `waitForAuthentication()` | Lines 3350-3465 | Fetches manifest, triggers `onGroupedPermissionRequested` |

### Current Flow

```
waitForAuthentication() ─────────────────────────────┐
        │                                            │
        ▼                                            │
   Fetch manifest.json                               │
        │                                            │
        ▼                                            │
   Filter already-granted permissions                │
        │                                            │
        ▼                                            │
   onGroupedPermissionRequested                      │
        │                                            │
        ▼                                            │
   (Later) ensure*() called ─────────────────────────┘
        │
        ▼
   requestPermissionFlow()
        │
        ▼
   onProtocolPermissionRequested (individual prompt)
```

---

## Proposed Architecture

### New Flow

```
ensure*() called
    │
    ▼
Check cache/token exists? ──YES──► Return true
    │
    NO
    │
    ▼
requestPermissionFlow()
    │
    ▼
maybeRequestGroupedPermissions()
    │
    ├── Config disabled? ─────────────────► null (continue individual flow)
    │
    ├── Fetch manifest (CACHED)
    │       │
    │       ├── No manifest/no group perms ► null (continue individual flow)
    │       │
    │       └── Has group perms
    │               │
    │               ▼
    │       filterAlreadyGrantedPermissions()
    │               │
    │               ├── All granted ───────► null (continue individual flow)
    │               │
    │               └── Some missing
    │                       │
    │                       ▼
    │               Trigger onGroupedPermissionRequested
    │                       │
    │                       ▼
    │               Wait for grant/deny
    │                       │
    │                       ▼
    │               checkSpecificPermissionAfterGroupFlow()
    │                       │
    │                       └── Return result
    │
    └── null ─────────────────────────────► Continue individual flow
                                                    │
                                                    ▼
                                            onProtocolPermissionRequested
                                            (or other individual event)
```

---

## Implementation Tasks

### Task 1: Add Manifest Cache Properties

**File**: `WalletPermissionsManager.ts`  
**Location**: After line ~470 (near existing `permissionCache`)  
**Priority**: High (blocking for other tasks)

Add new private properties:

```typescript
/** Cache for fetched manifests. Key is normalized originator. */
private manifestCache: Map<string, {
  groupPermissions: GroupedPermissions | null  // null = no group permissions found
  fetchedAt: number
}> = new Map()

/** Track in-progress manifest fetches to deduplicate concurrent requests. */
private manifestFetchInProgress: Map<string, Promise<GroupedPermissions | null>> = new Map()

/** How long a cached manifest remains valid (5 minutes). */
private static readonly MANIFEST_CACHE_TTL_MS = 5 * 60 * 1000
```

---

### Task 2: Create `fetchManifestGroupPermissions()` Method

**File**: `WalletPermissionsManager.ts`  
**Location**: Section 8 (Internal Helper Utilities), around line ~3490  
**Priority**: High (blocking for other tasks)

Extract manifest fetching logic into a reusable, cached method:

```typescript
/**
 * Fetches and caches the manifest.json groupPermissions for an originator.
 * 
 * Features:
 * - Returns cached result if within TTL
 * - Deduplicates concurrent requests to same originator
 * - Caches "no group permissions" result to avoid repeated failed fetches
 * 
 * @param originator Normalized originator string
 * @returns GroupedPermissions if manifest has them, null otherwise
 */
private async fetchManifestGroupPermissions(originator: string): Promise<GroupedPermissions | null> {
  // 1. Check cache first
  const cached = this.manifestCache.get(originator)
  if (cached && Date.now() - cached.fetchedAt < WalletPermissionsManager.MANIFEST_CACHE_TTL_MS) {
    return cached.groupPermissions
  }

  // 2. Check if fetch is already in progress (deduplicate concurrent requests)
  const inProgress = this.manifestFetchInProgress.get(originator)
  if (inProgress) {
    return inProgress
  }

  // 3. Fetch and cache
  const fetchPromise = (async (): Promise<GroupedPermissions | null> => {
    try {
      const proto = originator.startsWith('localhost:') ? 'http' : 'https'
      const response = await fetch(`${proto}://${originator}/manifest.json`)
      if (response.ok) {
        const manifest = await response.json()
        const groupPermissions: GroupedPermissions | null = manifest?.babbage?.groupPermissions || null
        this.manifestCache.set(originator, { groupPermissions, fetchedAt: Date.now() })
        return groupPermissions
      }
    } catch (e) {
      // Ignore fetch/parse errors - treat as "no manifest"
    }
    // Cache the negative result to avoid repeated failed fetches
    this.manifestCache.set(originator, { groupPermissions: null, fetchedAt: Date.now() })
    return null
  })()

  this.manifestFetchInProgress.set(originator, fetchPromise)
  try {
    return await fetchPromise
  } finally {
    this.manifestFetchInProgress.delete(originator)
  }
}
```

---

### Task 3: Create `filterAlreadyGrantedPermissions()` Method

**File**: `WalletPermissionsManager.ts`  
**Location**: Before `requestPermissionFlow()`, around line ~1290  
**Priority**: High

Extract the filtering logic currently duplicated in `waitForAuthentication()`:

```typescript
/**
 * Filters a GroupedPermissions object to only include permissions not yet granted.
 * 
 * @param originator Normalized originator string
 * @param groupPermissions The full set of permissions from manifest
 * @param displayOriginator Optional display originator for lookups
 * @returns A new GroupedPermissions containing only the missing permissions
 */
private async filterAlreadyGrantedPermissions(
  originator: string,
  groupPermissions: GroupedPermissions,
  displayOriginator?: string
): Promise<GroupedPermissions> {
  const permissionsToRequest: GroupedPermissions = {
    description: groupPermissions.description,
    protocolPermissions: [],
    basketAccess: [],
    certificateAccess: []
  }

  // Check spending authorization
  if (groupPermissions.spendingAuthorization) {
    const hasAuth = await this.hasSpendingAuthorization({
      originator,
      satoshis: groupPermissions.spendingAuthorization.amount
    })
    if (!hasAuth) {
      permissionsToRequest.spendingAuthorization = groupPermissions.spendingAuthorization
    }
  }

  // Check protocol permissions
  for (const p of groupPermissions.protocolPermissions || []) {
    const hasPerm = await this.hasProtocolPermission({
      originator,
      privileged: false, // Privilege is never allowed in group permissions for security
      protocolID: p.protocolID,
      counterparty: p.counterparty || 'self'
    })
    if (!hasPerm) {
      permissionsToRequest.protocolPermissions!.push(p)
    }
  }

  // Check basket access
  for (const b of groupPermissions.basketAccess || []) {
    const hasAccess = await this.hasBasketAccess({
      originator,
      basket: b.basket
    })
    if (!hasAccess) {
      permissionsToRequest.basketAccess!.push(b)
    }
  }

  // Check certificate access
  for (const c of groupPermissions.certificateAccess || []) {
    const hasAccess = await this.hasCertificateAccess({
      originator,
      privileged: false, // Privilege is never allowed in group permissions for security
      verifier: c.verifierPublicKey,
      certType: c.type,
      fields: c.fields
    })
    if (!hasAccess) {
      permissionsToRequest.certificateAccess!.push(c)
    }
  }

  return permissionsToRequest
}

/**
 * Helper to check if a GroupedPermissions object has any permissions to request.
 */
private hasAnyPermissionsToRequest(permissions: GroupedPermissions): boolean {
  return !!(
    permissions.spendingAuthorization ||
    (permissions.protocolPermissions?.length ?? 0) > 0 ||
    (permissions.basketAccess?.length ?? 0) > 0 ||
    (permissions.certificateAccess?.length ?? 0) > 0
  )
}
```

---

### Task 4: Create `maybeRequestGroupedPermissions()` Method

**File**: `WalletPermissionsManager.ts`  
**Location**: Before `requestPermissionFlow()`, around line ~1290  
**Priority**: High

This is the core integration point:

```typescript
/**
 * Attempts to handle a permission request via the grouped permissions flow.
 * 
 * Called by requestPermissionFlow() before falling back to individual prompts.
 * 
 * @param currentRequest The specific permission being requested
 * @returns 
 *   - true: Permission was granted via group flow
 *   - false: Permission was explicitly denied
 *   - null: Group flow not applicable, caller should continue with individual flow
 */
private async maybeRequestGroupedPermissions(currentRequest: PermissionRequest): Promise<boolean | null> {
  // Skip if group permissions are disabled in config
  if (!this.config.seekGroupedPermission) {
    return null
  }

  const originator = currentRequest.originator

  // Fetch manifest (cached)
  const groupPermissions = await this.fetchManifestGroupPermissions(originator)
  
  if (!groupPermissions) {
    // No manifest or no group permissions defined - use individual flow
    return null
  }

  // Filter to only missing permissions
  const permissionsToRequest = await this.filterAlreadyGrantedPermissions(
    originator,
    groupPermissions,
    currentRequest.displayOriginator
  )

  if (!this.hasAnyPermissionsToRequest(permissionsToRequest)) {
    // All manifest permissions already granted - use individual flow for this request
    // (The requested permission might not be in the manifest)
    return null
  }

  // Trigger the grouped permission flow
  const key = `group:${originator}`
  
  if (this.activeRequests.has(key)) {
    // Another call is already waiting on group permissions - piggyback on it
    await new Promise<boolean>((resolve, reject) => {
      this.activeRequests.get(key)!.pending.push({ resolve, reject })
    })
  } else {
    // This is the first call - create and trigger the group request
    try {
      await new Promise<boolean>(async (resolve, reject) => {
        this.activeRequests.set(key, {
          request: {
            originator,
            permissions: permissionsToRequest,
            displayOriginator: currentRequest.displayOriginator
          },
          pending: [{ resolve, reject }]
        })

        await this.callEvent('onGroupedPermissionRequested', {
          requestID: key,
          originator,
          permissions: permissionsToRequest
        })
      })
    } catch (e) {
      // Permission was denied
      throw e
    }
  }

  // After group flow completes, verify the specific permission was granted
  return await this.checkSpecificPermissionAfterGroupFlow(currentRequest)
}

/**
 * After a group permission flow completes, check if the specific requested permission was granted.
 * 
 * This handles the case where the user granted some but not all permissions.
 */
private async checkSpecificPermissionAfterGroupFlow(request: PermissionRequest): Promise<boolean> {
  switch (request.type) {
    case 'protocol':
      return await this.hasProtocolPermission({
        originator: request.originator,
        privileged: request.privileged ?? false,
        protocolID: request.protocolID!,
        counterparty: request.counterparty ?? 'self'
      })
    case 'basket':
      return await this.hasBasketAccess({
        originator: request.originator,
        basket: request.basket!
      })
    case 'certificate':
      return await this.hasCertificateAccess({
        originator: request.originator,
        privileged: request.privileged ?? false,
        verifier: request.certificate!.verifier,
        certType: request.certificate!.certType,
        fields: request.certificate!.fields
      })
    case 'spending':
      return await this.hasSpendingAuthorization({
        originator: request.originator,
        satoshis: request.spending!.satoshis
      })
    default:
      return false
  }
}
```

---

### Task 5: Modify `requestPermissionFlow()`

**File**: `WalletPermissionsManager.ts`  
**Location**: Lines 1296-1349  
**Priority**: High

Update to try grouped permissions first:

```typescript
private async requestPermissionFlow(r: PermissionRequest): Promise<boolean> {
  const normalizedOriginator = this.normalizeOriginator(r.originator) || r.originator
  const preparedRequest: PermissionRequest = {
    ...r,
    originator: normalizedOriginator,
    displayOriginator: r.displayOriginator ?? r.originator
  }

  // ========== NEW: Try group permissions first ==========
  const groupResult = await this.maybeRequestGroupedPermissions(preparedRequest)
  if (groupResult !== null) {
    // Group flow handled it (either granted or denied)
    if (!groupResult) {
      throw new Error('Permission denied.')
    }
    return groupResult
  }
  // ========== END NEW ==========

  // Continue with existing individual permission flow...
  const key = this.buildRequestKey(preparedRequest)

  // If there's already a queue for the same resource, we piggyback on it
  const existingQueue = this.activeRequests.get(key)
  if (existingQueue && existingQueue.pending.length > 0) {
    return new Promise<boolean>((resolve, reject) => {
      existingQueue.pending.push({ resolve, reject })
    })
  }

  // Otherwise, create a new queue with a single entry
  return new Promise<boolean>(async (resolve, reject) => {
    this.activeRequests.set(key, {
      request: preparedRequest,
      pending: [{ resolve, reject }]
    })

    // Fire the relevant onXXXRequested event
    switch (preparedRequest.type) {
      case 'protocol':
        await this.callEvent('onProtocolPermissionRequested', {
          ...preparedRequest,
          requestID: key
        })
        break
      case 'basket':
        await this.callEvent('onBasketAccessRequested', {
          ...preparedRequest,
          requestID: key
        })
        break
      case 'certificate':
        await this.callEvent('onCertificateAccessRequested', {
          ...preparedRequest,
          requestID: key
        })
        break
      case 'spending':
        await this.callEvent('onSpendingAuthorizationRequested', {
          ...preparedRequest,
          requestID: key
        })
        break
    }
  })
}
```

---

### Task 6: Simplify `waitForAuthentication()`

**File**: `WalletPermissionsManager.ts`  
**Location**: Lines 3350-3465  
**Priority**: Medium

Refactor to use the new shared methods:

```typescript
public async waitForAuthentication(
  ...args: Parameters<WalletInterface['waitForAuthentication']>
): ReturnType<WalletInterface['waitForAuthentication']> {
  let [_, originator] = args
  
  if (this.config.seekGroupedPermission && originator) {
    const { normalized: normalizedOriginator } = this.prepareOriginator(originator)
    originator = normalizedOriginator

    // Use the shared manifest fetching logic (now cached)
    const groupPermissions = await this.fetchManifestGroupPermissions(originator)

    if (groupPermissions) {
      // Use the shared filtering logic
      const permissionsToRequest = await this.filterAlreadyGrantedPermissions(
        originator,
        groupPermissions
      )

      if (this.hasAnyPermissionsToRequest(permissionsToRequest)) {
        const key = `group:${originator}`
        
        if (this.activeRequests.has(key)) {
          // Piggyback on existing request
          await new Promise<boolean>((resolve, reject) => {
            this.activeRequests.get(key)!.pending.push({ resolve, reject })
          })
        } else {
          // Create new group request
          try {
            await new Promise<boolean>(async (resolve, reject) => {
              this.activeRequests.set(key, {
                request: { originator, permissions: permissionsToRequest },
                pending: [{ resolve, reject }]
              })

              await this.callEvent('onGroupedPermissionRequested', {
                requestID: key,
                originator,
                permissions: permissionsToRequest
              })
            })
          } catch (e) {
            throw e
          }
        }
      }
    }
  }

  return this.underlying.waitForAuthentication(...args)
}
```

---

### Task 7: Add Public Method to Clear Manifest Cache (Optional)

**File**: `WalletPermissionsManager.ts`  
**Location**: Section 6 (Public List/Has/Revoke Methods)  
**Priority**: Low

For debugging and testing:

```typescript
/**
 * Clears the cached manifest for a specific originator or all originators.
 * Useful for testing or when an app updates its manifest.
 * 
 * @param originator Optional originator to clear. If omitted, clears all.
 */
public clearManifestCache(originator?: string): void {
  if (originator) {
    const normalized = this.normalizeOriginator(originator) || originator
    this.manifestCache.delete(normalized)
  } else {
    this.manifestCache.clear()
  }
}
```

---

## Edge Cases & Error Handling

### 1. Permission Not in Manifest

**Scenario**: App requests a permission that's not defined in its `manifest.json`.

**Behavior**: 
- `maybeRequestGroupedPermissions()` triggers group flow for manifest permissions
- After group flow, `checkSpecificPermissionAfterGroupFlow()` returns `false`
- `requestPermissionFlow()` continues to individual permission prompt

**No code change needed** - the design handles this naturally.

### 2. User Denies Grouped Permission

**Scenario**: User clicks "Deny" on the grouped permission prompt.

**Behavior**: 
- `denyGroupedPermission()` rejects all pending promises with error
- Error propagates up through `maybeRequestGroupedPermissions()`
- Caller receives the error

**No code change needed** - existing error handling applies.

### 3. User Grants Partial Permissions

**Scenario**: User grants some but not all permissions in the group.

**Behavior**:
- `grantGroupedPermission()` creates tokens only for granted permissions
- `checkSpecificPermissionAfterGroupFlow()` checks if specific permission was granted
- If not granted, falls through to individual prompt

**No code change needed** - the design handles this.

### 4. Manifest Fetch Fails

**Scenario**: Network error, 404, or invalid JSON when fetching manifest.

**Behavior**:
- `fetchManifestGroupPermissions()` catches error
- Caches `null` result to avoid repeated failures
- Returns `null`, causing fallback to individual flow

**No code change needed** - implemented in Task 2.

### 5. Concurrent Permission Requests

**Scenario**: Multiple `ensure*()` calls happen simultaneously.

**Behavior**:
- First call triggers group flow, creates entry in `activeRequests`
- Subsequent calls for same originator piggyback on existing request
- `manifestFetchInProgress` map prevents duplicate fetches

**No code change needed** - deduplication is built into the design.

---

## Testing Checklist

### Unit Tests

- [ ] `fetchManifestGroupPermissions()` returns cached result within TTL
- [ ] `fetchManifestGroupPermissions()` re-fetches after TTL expires
- [ ] `fetchManifestGroupPermissions()` deduplicates concurrent requests
- [ ] `fetchManifestGroupPermissions()` caches negative results (no manifest)
- [ ] `filterAlreadyGrantedPermissions()` correctly identifies missing permissions
- [ ] `maybeRequestGroupedPermissions()` returns `null` when config disabled
- [ ] `maybeRequestGroupedPermissions()` returns `null` when no manifest
- [ ] `maybeRequestGroupedPermissions()` triggers group event when permissions missing
- [ ] `checkSpecificPermissionAfterGroupFlow()` correctly checks each permission type

### Integration Tests

- [ ] App with manifest: First `ensure*()` triggers grouped prompt
- [ ] App with manifest: Second `ensure*()` uses cached manifest
- [ ] App without manifest: Falls back to individual prompts
- [ ] Partial grant: Ungranted permissions trigger individual prompts
- [ ] `waitForAuthentication()` still works as before
- [ ] Concurrent calls are properly deduplicated

### Manual Testing

- [ ] Verify grouped permission UI appears on first wallet operation
- [ ] Verify manifest is only fetched once per session (check network tab)
- [ ] Verify fallback to individual prompts when manifest missing
- [ ] Verify permission tokens are created correctly after group grant

---

## Migration Notes

### Breaking Changes

**None** - This is a backward-compatible enhancement.

### Behavioral Changes

1. Apps with `manifest.json` will now see grouped permission prompts on first `ensure*()` call, not just during `waitForAuthentication()`.

2. Manifest is cached for 5 minutes, so changes to `manifest.json` may take up to 5 minutes to take effect.

### Configuration

The existing `seekGroupedPermission` config option controls this behavior. Setting it to `false` disables the new functionality entirely.

---

## Summary

| Task | Description | Priority | Dependencies |
|------|-------------|----------|--------------|
| 1 | Add manifest cache properties | High | None |
| 2 | Create `fetchManifestGroupPermissions()` | High | Task 1 |
| 3 | Create `filterAlreadyGrantedPermissions()` | High | None |
| 4 | Create `maybeRequestGroupedPermissions()` | High | Tasks 2, 3 |
| 5 | Modify `requestPermissionFlow()` | High | Task 4 |
| 6 | Simplify `waitForAuthentication()` | Medium | Tasks 2, 3 |
| 7 | Add `clearManifestCache()` | Low | Task 1 |

**Estimated effort**: 4-6 hours for implementation, 2-4 hours for testing.

---
---

# Part 2: PACT (Protocol Access for Counterparty Trust)

## Problem Statement

Level 2 protocols require **counterparty-specific** permissions. When a user interacts with a **new person** through an application, they may need permissions for multiple Level 2 protocols:

- Payment protocol (send/receive money)
- Authentication protocol (verify identity)
- Off-message signatures (sign documents)
- HMAC verification (server authentication)
- And potentially many more...

**Current Behavior**: Each Level 2 protocol with a new counterparty triggers a separate permission prompt. If an app uses 5 protocols with a new contact, the user sees 5 sequential pop-ups.

**Result**: "Pop-up fatigue" where users click "Grant" without reading, defeating the security purpose.

## Goal

When a user first interacts with a **new counterparty** through an app, show a **single prompt** that:
1. Explains "You're interacting with a new person through this app"
2. Lists all the things this person will be able to do
3. Grants all required Level 2 protocol permissions at once

---

## Key Distinction: GroupedPermissions vs PACT

| Aspect | GroupedPermissions | PACT |
|--------|-------------------|--------|
| **Scope** | Per-originator | Per-originator + per-counterparty |
| **Trigger** | First wallet operation from app | First operation with a new counterparty |
| **Frequency** | Once per app | Once per counterparty per app |
| **Protocols** | Level 1 + baskets + certificates + spending | Level 2 only |
| **Counterparty** | `'self'` or specific | Dynamic (the new counterparty) |

These are **orthogonal** and can both apply:

```
User opens App X for the first time
    │
    ▼
GroupedPermissions flow triggers
    │ "App X wants these general permissions..."
    ▼
User grants

... later ...

User sends message to Alice (new contact)
    │
    ▼
PACT flow triggers
    │ "You're interacting with Alice through App X.
    │  Alice will be able to: receive payments, verify your identity..."
    ▼
User grants

... later ...

User sends message to Bob (another new contact)
    │
    ▼
PACT flow triggers again for Bob
```

---

## Manifest Structure

Extend `manifest.json` with a new section for counterparty-specific protocols:

```json
{
  "babbage": {
    "groupPermissions": {
      "description": "App-level permissions",
      "protocolPermissions": [
        { "protocolID": [1, "app-data"], "description": "Store app data" }
      ],
      "basketAccess": [
        { "basket": "messages", "description": "Store messages" }
      ]
    },
    "counterpartyPermissions": {
      "description": "Permissions granted per person you interact with",
      "protocols": [
        { 
          "protocolID": [2, "3241645161d8"], 
          "description": "Send and receive payments" 
        },
        { 
          "protocolID": [2, "authrite"], 
          "description": "Authenticate messages" 
        },
        { 
          "protocolID": [2, "off-message signature"], 
          "description": "Sign documents for them" 
        },
        { 
          "protocolID": [2, "server hmac"], 
          "description": "Verify server requests" 
        }
      ]
    }
  }
}
```

### Validation Rules

1. `counterpartyPermissions.protocols[*].protocolID[0]` **must** be `2` (Level 2)
2. Level 0 and Level 1 protocols are rejected (they don't use counterparties meaningfully)
3. Privileged protocols are **not allowed** in PACT for security

---

## New TypeScript Interfaces

Add to `WalletPermissionsManager.ts`:

```typescript
/**
 * Describes Level 2 protocols that should be granted together when
 * the user first interacts with a new counterparty through the app.
 * 
 * This is the PACT (Protocol Access for Counterparty Trust) system.
 */
export interface CounterpartyPermissions {
  /** Human-readable description shown in the permission prompt */
  description?: string
  
  /** 
   * Level 2 protocols to grant for each new counterparty.
   * All protocolIDs MUST have security level 2.
   */
  protocols: Array<{
    protocolID: WalletProtocol  // Must be [2, 'protocolName']
    description: string
  }>
}

/**
 * Request object passed to UI when PACT permission is needed.
 */
export interface CounterpartyPermissionRequest {
  originator: string
  requestID: string
  /** The public key of the new counterparty */
  counterparty: PubKeyHex
  /** Human-readable name/label for the counterparty, if known */
  counterpartyLabel?: string
  /** The protocols being requested */
  permissions: CounterpartyPermissions
}

/**
 * Handler for PACT permission requests.
 */
export type CounterpartyPermissionEventHandler = (
  request: CounterpartyPermissionRequest
) => void | Promise<void>
```

---

## Updated Manifest Cache Structure

Modify Task 1 to include counterparty permissions:

```typescript
/** Cache for fetched manifests. Key is normalized originator. */
private manifestCache: Map<string, {
  groupPermissions: GroupedPermissions | null
  counterpartyPermissions: CounterpartyPermissions | null  // NEW
  fetchedAt: number
}> = new Map()

/** 
 * Cache of counterparties with established PACT for each originator.
 * Key: `${originator}:${counterparty}`
 * Value: timestamp when PACT was established
 */
private pactEstablishedCache: Map<string, number> = new Map()
```

---

## Updated Callbacks Interface

```typescript
export interface WalletPermissionsManagerCallbacks {
  onProtocolPermissionRequested?: PermissionEventHandler[]
  onBasketAccessRequested?: PermissionEventHandler[]
  onCertificateAccessRequested?: PermissionEventHandler[]
  onSpendingAuthorizationRequested?: PermissionEventHandler[]
  onGroupedPermissionRequested?: GroupedPermissionEventHandler[]
  onCounterpartyPermissionRequested?: CounterpartyPermissionEventHandler[]  // NEW
}
```

---

## Implementation Tasks (PACT-specific)

### Task 8: Update Manifest Fetching for PACT

**File**: `WalletPermissionsManager.ts`  
**Modify**: `fetchManifestGroupPermissions()` from Task 2  
**Priority**: High

Rename and extend to fetch both group and counterparty (PACT) permissions:

```typescript
interface ManifestPermissions {
  groupPermissions: GroupedPermissions | null
  counterpartyPermissions: CounterpartyPermissions | null
}

/**
 * Fetches and caches the manifest.json permissions for an originator.
 * Returns both groupPermissions and counterpartyPermissions.
 */
private async fetchManifestPermissions(originator: string): Promise<ManifestPermissions> {
  // Check cache
  const cached = this.manifestCache.get(originator)
  if (cached && Date.now() - cached.fetchedAt < WalletPermissionsManager.MANIFEST_CACHE_TTL_MS) {
    return {
      groupPermissions: cached.groupPermissions,
      counterpartyPermissions: cached.counterpartyPermissions
    }
  }

  // Check in-progress
  const inProgress = this.manifestFetchInProgress.get(originator)
  if (inProgress) {
    return inProgress
  }

  // Fetch
  const fetchPromise = (async (): Promise<ManifestPermissions> => {
    try {
      const proto = originator.startsWith('localhost:') ? 'http' : 'https'
      const response = await fetch(`${proto}://${originator}/manifest.json`)
      if (response.ok) {
        const manifest = await response.json()
        const result: ManifestPermissions = {
          groupPermissions: manifest?.babbage?.groupPermissions || null,
          counterpartyPermissions: this.validateCounterpartyPermissions(
            manifest?.babbage?.counterpartyPermissions
          )
        }
        this.manifestCache.set(originator, { ...result, fetchedAt: Date.now() })
        return result
      }
    } catch (e) {
      // Ignore errors
    }
    const result: ManifestPermissions = { 
      groupPermissions: null, 
      counterpartyPermissions: null 
    }
    this.manifestCache.set(originator, { ...result, fetchedAt: Date.now() })
    return result
  })()

  this.manifestFetchInProgress.set(originator, fetchPromise)
  try {
    return await fetchPromise
  } finally {
    this.manifestFetchInProgress.delete(originator)
  }
}

/**
 * Validates counterparty permissions from manifest.
 * Ensures all protocols are Level 2.
 */
private validateCounterpartyPermissions(
  raw: any
): CounterpartyPermissions | null {
  if (!raw?.protocols?.length) return null
  
  const validProtocols = raw.protocols.filter((p: any) => {
    // Must be Level 2
    return Array.isArray(p.protocolID) && 
           p.protocolID[0] === 2 && 
           typeof p.protocolID[1] === 'string'
  })
  
  if (validProtocols.length === 0) return null
  
  return {
    description: raw.description,
    protocols: validProtocols
  }
}
```

---

### Task 9: Create PACT Establishment Check

**File**: `WalletPermissionsManager.ts`  
**Location**: Near other helper methods  
**Priority**: High

```typescript
/**
 * Checks if a PACT has been established with a counterparty for an originator.
 * A PACT is established when the user has previously granted permissions for this counterparty.
 * 
 * First checks the in-memory cache, then checks for existing permission tokens.
 */
private async hasPactEstablished(
  originator: string,
  counterparty: string
): Promise<boolean> {
  // Skip for special counterparties
  if (counterparty === 'self' || counterparty === 'anyone') {
    return true  // These don't need PACT
  }

  // Check cache first
  const cacheKey = `${originator}:${counterparty}`
  if (this.pactEstablishedCache.has(cacheKey)) {
    return true
  }

  // Check if we have ANY Level 2 token for this counterparty + originator
  // If yes, a PACT was established before
  const manifest = await this.fetchManifestPermissions(originator)
  if (!manifest.counterpartyPermissions?.protocols?.length) {
    // No PACT defined - consider established by default
    return true
  }

  // Check if the first protocol in the list has a token
  // (If one exists, they were granted together via PACT)
  const firstProtocol = manifest.counterpartyPermissions.protocols[0]
  const hasToken = await this.hasProtocolPermission({
    originator,
    privileged: false,
    protocolID: firstProtocol.protocolID,
    counterparty
  })

  if (hasToken) {
    this.pactEstablishedCache.set(cacheKey, Date.now())
    return true
  }

  return false
}

/**
 * Marks a PACT as established with a counterparty.
 */
private markPactEstablished(originator: string, counterparty: string): void {
  const cacheKey = `${originator}:${counterparty}`
  this.pactEstablishedCache.set(cacheKey, Date.now())
}
```

---

### Task 10: Create PACT Request Flow

**File**: `WalletPermissionsManager.ts`  
**Location**: Before `requestPermissionFlow()`  
**Priority**: High

```typescript
/**
 * Attempts to handle a Level 2 protocol permission via the PACT flow.
 * 
 * Called when ensureProtocolPermission is invoked with a Level 2 protocol
 * and a specific counterparty (not 'self' or 'anyone').
 * 
 * @param currentRequest The specific permission being requested
 * @returns 
 *   - true: Permission was granted via PACT
 *   - false: Permission was denied
 *   - null: PACT not applicable, caller should continue with other flows
 */
private async maybeRequestPact(
  currentRequest: PermissionRequest
): Promise<boolean | null> {
  // Only applies to Level 2 protocol requests with specific counterparty
  if (currentRequest.type !== 'protocol') return null
  if (!currentRequest.protocolID || currentRequest.protocolID[0] !== 2) return null
  if (!currentRequest.counterparty) return null
  if (currentRequest.counterparty === 'self' || currentRequest.counterparty === 'anyone') {
    return null
  }

  const originator = currentRequest.originator
  const counterparty = currentRequest.counterparty

  // Check if PACT already established with this counterparty
  if (await this.hasPactEstablished(originator, counterparty)) {
    return null  // PACT already established - use normal flow
  }

  // Fetch manifest
  const manifest = await this.fetchManifestPermissions(originator)
  if (!manifest.counterpartyPermissions?.protocols?.length) {
    return null  // No PACT defined - use normal flow
  }

  // Filter to only missing permissions for this counterparty
  const protocolsToRequest = await this.filterMissingCounterpartyProtocols(
    originator,
    counterparty,
    manifest.counterpartyPermissions.protocols
  )

  if (protocolsToRequest.length === 0) {
    // All protocols already granted - mark PACT as established and continue
    this.markPactEstablished(originator, counterparty)
    return null
  }

  // Build the PACT request
  const permissionsToRequest: CounterpartyPermissions = {
    description: manifest.counterpartyPermissions.description,
    protocols: protocolsToRequest
  }

  // Trigger the PACT flow
  const key = `pact:${originator}:${counterparty}`

  if (this.activeRequests.has(key)) {
    // Piggyback on existing request
    await new Promise<boolean>((resolve, reject) => {
      this.activeRequests.get(key)!.pending.push({ resolve, reject })
    })
  } else {
    try {
      await new Promise<boolean>(async (resolve, reject) => {
        this.activeRequests.set(key, {
          request: {
            originator,
            counterparty,
            permissions: permissionsToRequest,
            displayOriginator: currentRequest.displayOriginator
          },
          pending: [{ resolve, reject }]
        })

        await this.callEvent('onCounterpartyPermissionRequested', {
          requestID: key,
          originator,
          counterparty,
          permissions: permissionsToRequest
        })
      })
    } catch (e) {
      throw e
    }
  }

  // Mark PACT as established
  this.markPactEstablished(originator, counterparty)

  // Check if the specific requested protocol was granted
  return await this.hasProtocolPermission({
    originator,
    privileged: false,
    protocolID: currentRequest.protocolID,
    counterparty
  })
}

/**
 * Filters counterparty protocols to only those not yet granted.
 */
private async filterMissingCounterpartyProtocols(
  originator: string,
  counterparty: string,
  protocols: Array<{ protocolID: WalletProtocol; description: string }>
): Promise<Array<{ protocolID: WalletProtocol; description: string }>> {
  const missing: Array<{ protocolID: WalletProtocol; description: string }> = []
  
  for (const p of protocols) {
    const hasPermission = await this.hasProtocolPermission({
      originator,
      privileged: false,
      protocolID: p.protocolID,
      counterparty
    })
    if (!hasPermission) {
      missing.push(p)
    }
  }
  
  return missing
}
```

---

### Task 11: Create Grant/Deny Methods for PACT

**File**: `WalletPermissionsManager.ts`  
**Location**: After existing grant/deny methods  
**Priority**: High

```typescript
/**
 * Grants a counterparty permission request (PACT).
 * Creates permission tokens for all granted protocols.
 * 
 * @param params.requestID The request ID from the event
 * @param params.granted Subset of protocols the user approved (or all)
 * @param params.expiry Optional expiry time for the tokens
 */
public async grantCounterpartyPermission(params: {
  requestID: string
  granted: Array<{ protocolID: WalletProtocol; description: string }>
  expiry?: number
}): Promise<void> {
  const matching = this.activeRequests.get(params.requestID)
  if (!matching) {
    throw new Error('Request ID not found.')
  }

  const originalRequest = matching.request as {
    originator: string
    counterparty: string
    permissions: CounterpartyPermissions
    displayOriginator?: string
  }

  const { originator, counterparty, permissions: requestedPermissions } = originalRequest

  // Validate granted protocols are subset of requested
  for (const g of params.granted) {
    const wasRequested = requestedPermissions.protocols.some(
      r => r.protocolID[0] === g.protocolID[0] && r.protocolID[1] === g.protocolID[1]
    )
    if (!wasRequested) {
      throw new Error(`Protocol ${g.protocolID[1]} was not in the original request.`)
    }
  }

  const expiry = params.expiry || 0

  // Create tokens for all granted protocols
  for (const p of params.granted) {
    const request: PermissionRequest = {
      type: 'protocol',
      originator,
      privileged: false,
      protocolID: p.protocolID,
      counterparty,
      reason: p.description
    }
    await this.createPermissionOnChain(request, expiry)
    this.markRecentGrant(request)
  }

  // Mark PACT as established
  this.markPactEstablished(originator, counterparty)

  // Resolve all pending promises
  for (const p of matching.pending) {
    p.resolve(true)
  }
  this.activeRequests.delete(params.requestID)
}

/**
 * Denies a counterparty permission request (PACT).
 */
public async denyCounterpartyPermission(requestID: string): Promise<void> {
  const matching = this.activeRequests.get(requestID)
  if (!matching) {
    throw new Error('Request ID not found.')
  }

  const err = new Error('Counterparty permission denied by user.')
  ;(err as any).code = 'ERR_COUNTERPARTY_PERMISSION_DENIED'
  
  for (const p of matching.pending) {
    p.reject(err)
  }
  this.activeRequests.delete(requestID)
}
```

---

### Task 12: Update `requestPermissionFlow()` for PACT

**File**: `WalletPermissionsManager.ts`  
**Modify**: `requestPermissionFlow()` from Task 5  
**Priority**: High

Update to check PACT **before** grouped permissions:

```typescript
private async requestPermissionFlow(r: PermissionRequest): Promise<boolean> {
  const normalizedOriginator = this.normalizeOriginator(r.originator) || r.originator
  const preparedRequest: PermissionRequest = {
    ...r,
    originator: normalizedOriginator,
    displayOriginator: r.displayOriginator ?? r.originator
  }

  // ========== PACT: Check counterparty permissions first ==========
  // This takes priority because it's more specific (per-counterparty)
  const pactResult = await this.maybeRequestPact(preparedRequest)
  if (pactResult !== null) {
    if (!pactResult) {
      throw new Error('Counterparty permission denied.')
    }
    return pactResult
  }
  // ========== END PACT ==========

  // ========== Try app-level group permissions ==========
  const groupResult = await this.maybeRequestGroupedPermissions(preparedRequest)
  if (groupResult !== null) {
    if (!groupResult) {
      throw new Error('Permission denied.')
    }
    return groupResult
  }
  // ========== END GROUP ==========

  // Continue with individual permission flow...
  const key = this.buildRequestKey(preparedRequest)
  // ... rest unchanged
}
```

---

## Complete Flow Diagram

```
ensureProtocolPermission([2, 'payment'], counterparty: '02abc...')
    │
    ▼
Is counterparty 'self' or 'anyone'? ──YES──► Skip PACT
    │
    NO
    │
    ▼
requestPermissionFlow()
    │
    ▼
maybeRequestPact()    ◄── PACT CHECK
    │
    ├── Not Level 2 protocol? ────────────► null (continue)
    │
    ├── Is PACT established? ────────YES─► null (continue)
    │
    ├── Fetch manifest (cached)
    │       │
    │       ├── No counterpartyPermissions ► null (continue)
    │       │
    │       └── Has counterpartyPermissions
    │               │
    │               ▼
    │       Filter missing protocols
    │               │
    │               ├── All granted ───────► Mark PACT established, null (continue)
    │               │
    │               └── Some missing
    │                       │
    │                       ▼
    │               Trigger onCounterpartyPermissionRequested
    │                       │ "You're interacting with [counterparty].
    │                       │  They will be able to: ..."
    │                       ▼
    │               Wait for grant/deny
    │                       │
    │                       ▼
    │               Create tokens for all granted
    │                       │
    │                       ▼
    │               Mark PACT established
    │                       │
    │                       └── Return result
    │
    ▼
maybeRequestGroupedPermissions()    ◄── APP-LEVEL CHECK
    │
    └── ... (as before)
    │
    ▼
Individual permission flow    ◄── FALLBACK
```

---

## UI Considerations

The `onCounterpartyPermissionRequested` event provides data for a UI like:

```
┌─────────────────────────────────────────────────┐
│  New Contact in [App Name]                      │
├─────────────────────────────────────────────────┤
│                                                 │
│  You're about to interact with:                 │
│                                                 │
│  [Counterparty Identity / Public Key]           │
│                                                 │
│  This person will be able to:                   │
│                                                 │
│  ✓ Send and receive payments                    │
│  ✓ Authenticate messages                        │
│  ✓ Request document signatures                  │
│  ✓ Verify server requests                       │
│                                                 │
│  [Show Details ▼]                               │
│                                                 │
├─────────────────────────────────────────────────┤
│         [Deny]              [Allow]             │
└─────────────────────────────────────────────────┘
```

The "Show Details" expansion could show the raw protocol IDs and more technical descriptions.

---

## Testing Checklist (PACT)

### Unit Tests

- [ ] `validateCounterpartyPermissions()` rejects non-Level-2 protocols
- [ ] `hasPactEstablished()` returns true for 'self' and 'anyone'
- [ ] `hasPactEstablished()` checks cache first
- [ ] `hasPactEstablished()` falls back to token check
- [ ] `maybeRequestPact()` returns null for Level 1
- [ ] `maybeRequestPact()` returns null when PACT already established
- [ ] `filterMissingCounterpartyProtocols()` correctly identifies missing
- [ ] `grantCounterpartyPermission()` creates all tokens
- [ ] `grantCounterpartyPermission()` validates granted subset

### Integration Tests

- [ ] First interaction with new counterparty triggers PACT prompt
- [ ] Second interaction with same counterparty skips PACT (already established)
- [ ] Different counterparty triggers new PACT prompt
- [ ] App without `counterpartyPermissions` uses normal flow
- [ ] Partial grant still marks PACT as established
- [ ] Concurrent requests for same counterparty are deduplicated

---

## Summary Table (All Tasks)

| Task | Description | Priority | Dependencies | Category |
|------|-------------|----------|--------------|----------|
| 1 | Add manifest cache properties | High | None | Group |
| 2 | Create `fetchManifestPermissions()` | High | Task 1 | Group |
| 3 | Create `filterAlreadyGrantedPermissions()` | High | None | Group |
| 4 | Create `maybeRequestGroupedPermissions()` | High | Tasks 2, 3 | Group |
| 5 | Modify `requestPermissionFlow()` | High | Task 4 | Group |
| 6 | Simplify `waitForAuthentication()` | Medium | Tasks 2, 3 | Group |
| 7 | Add `clearManifestCache()` | Low | Task 1 | Group |
| 8 | Update manifest fetching for PACT | High | Task 2 | PACT |
| 9 | Create `hasPactEstablished()` check | High | Task 8 | PACT |
| 10 | Create `maybeRequestPact()` | High | Task 9 | PACT |
| 11 | Create grant/deny for PACT | High | Task 10 | PACT |
| 12 | Update `requestPermissionFlow()` for PACT | High | Tasks 5, 10 | PACT |

**Total Estimated Effort**: 
- Group Permissions: 4-6 hours implementation + 2-4 hours testing
- PACT: 4-6 hours implementation + 2-4 hours testing
- **Total: 8-12 hours implementation + 4-8 hours testing**