# Chapter 3: Reading & Retrieving Data

> **Key takeaway:** Medplum provides six read strategies — by ID, by reference, by version, by history, by canonical URL, and by graph — plus a caching layer that prevents redundant network calls. Pick the right one for your access pattern.

---

## 3.1 Read Strategy Map

```
                  How do you identify the resource?
                              │
        ┌─────────┬───────────┼───────────┬──────────────┐
        │         │           │           │              │
    Have the   Have a     Have a      Need all      Need related
    type + id  Reference  canonical   versions?     resources?
        │      object     URL?           │              │
        │         │           │           │              │
  readResource  readReference readCanonical readHistory  readResourceGraph
        │         │           │           │              │
        │         │           │           │              │
        └─────────┴───────────┴───────────┴──────────────┘
                              │
                    All return ReadablePromise
                    (can be read synchronously from cache)
```

---

## 3.2 Read by ID: `readResource`

The most direct way to fetch a single resource.

### Signature

```typescript
readResource<RT extends ResourceType>(
  resourceType: RT,
  id: string,
  options?: MedplumRequestOptions
): ReadablePromise<WithId<ExtractResource<RT>>>
```

### Example

```typescript
const patient = await medplum.readResource('Patient', 'homer-simpson');
console.log(patient.name?.[0]?.family); // "Simpson"
```

### What Is `ReadablePromise`?

A `ReadablePromise` is a Medplum extension of `Promise` that can be read synchronously if the data is already cached:

```typescript
// Async (normal usage) — always works
const patient = await medplum.readResource('Patient', '123');

// Sync read from cache — works if cached, throws if pending
const promise = medplum.readResource('Patient', '123');
if (promise.isOk()) {
  const patient = promise.read(); // No await needed — returns from cache
}
```

This pattern powers Medplum's React components, which can render cached data immediately without suspense waterfalls.

---

## 3.3 Read by Reference: `readReference`

When you have a `Reference<T>` object (common when following links between resources).

### Signature

```typescript
readReference<T extends Resource>(
  reference: Reference<T>,
  options?: MedplumRequestOptions
): ReadablePromise<WithId<T>>
```

### Example: Following a Reference Chain

```typescript
import type { Observation, Patient, Practitioner } from '@medplum/fhirtypes';

// You have an Observation and want the linked Patient and Practitioner
const observation = await medplum.readResource('Observation', 'obs-123');

// Follow the reference to get the full Patient resource
const patient = await medplum.readReference(observation.subject as Reference<Patient>);

// Follow the reference to get who performed the observation
const performer = await medplum.readReference(
  observation.performer?.[0] as Reference<Practitioner>
);
```

### Business Use Case: Display a Lab Result Card

```typescript
async function getLabResultDisplay(observationId: string) {
  const obs = await medplum.readResource('Observation', observationId);
  const patient = await medplum.readReference(obs.subject as Reference<Patient>);

  return {
    patientName: patient.name?.[0]?.family,
    testName: obs.code?.text,
    value: obs.valueQuantity?.value,
    unit: obs.valueQuantity?.unit,
    status: obs.status,
  };
}
```

---

## 3.4 Version History: `readHistory` and `readVersion`

FHIR stores every version of a resource. This is like an automatic audit trail.

### Read Full History

```typescript
const history = await medplum.readHistory('Patient', 'homer-simpson');

// history is a Bundle containing all versions, newest first
for (const entry of history.entry ?? []) {
  console.log(
    `Version ${entry.resource?.meta?.versionId}` +
    ` at ${entry.resource?.meta?.lastUpdated}`
  );
}
```

### Read a Specific Version

```typescript
const oldVersion = await medplum.readVersion('Patient', 'homer-simpson', '2');
console.log(oldVersion.meta?.lastUpdated); // Timestamp of version 2
```

### Paginated History

```typescript
// Get the most recent 10 versions
const recentHistory = await medplum.readHistory('Patient', 'homer-simpson', {
  count: 10,
  offset: 0,
});
```

### Business Use Case: Audit Log for Patient Demographics

```typescript
async function getPatientChangeLog(patientId: string) {
  const history = await medplum.readHistory('Patient', patientId);

  return (history.entry ?? []).map((entry) => ({
    version: entry.resource?.meta?.versionId,
    updatedAt: entry.resource?.meta?.lastUpdated,
    updatedBy: entry.resource?.meta?.author?.display,
    // Compare fields between versions to show what changed
    name: entry.resource?.name?.[0],
    status: entry.response?.status,  // "200" for versions, "410" for deleted
  }));
}
```

---

## 3.5 Read by Canonical URL: `readCanonical`

For resources identified by URL rather than ID (common for StructureDefinitions, ValueSets, and other "definition" resources).

```typescript
const profile = await medplum.readCanonical(
  'StructureDefinition',
  'http://hl7.org/fhir/us/core/StructureDefinition/us-core-patient'
);
```

---

## 3.6 Graph Reads: `readResourceGraph` and `readPatientEverything`

### Patient Everything

Fetch all resources related to a patient in one call.

```typescript
const everything = await medplum.readPatientEverything('homer-simpson');

// Returns a Bundle containing the Patient + all related resources:
// Observations, Encounters, Conditions, MedicationRequests, etc.
for (const entry of everything.entry ?? []) {
  console.log(`${entry.resource?.resourceType}: ${entry.resource?.id}`);
}
```

### Graph Read

Fetch resources following a pre-defined GraphDefinition.

```typescript
const graph = await medplum.readResourceGraph(
  'Patient',
  'homer-simpson',
  'my-patient-graph'  // Name of a GraphDefinition resource
);
```

---

## 3.7 Caching: How Medplum Avoids Redundant Fetches

```
  medplum.readResource('Patient', '123')
                    │
                    ▼
          ┌─────────────────┐
          │ Check LRU Cache  │
          └────────┬────────┘
             ┌─────┴─────┐
             │            │
         Cache hit    Cache miss
         & fresh      or expired
             │            │
             ▼            ▼
        Return        HTTP GET /Patient/123
        immediately        │
                           ▼
                     Store in cache
                     (TTL = cacheTime)
                           │
                           ▼
                     Return resource
```

### Cache Configuration

```typescript
const medplum = new MedplumClient({
  resourceCacheSize: 1000,   // Max items in LRU cache (default: 1000)
  cacheTime: 60000,          // TTL in ms (default: 60000 in browser, 0 in Node.js)
});
```

### Cache Management Methods

```typescript
// Check cache without triggering a fetch
const cached = medplum.getCached('Patient', '123');

// Check cache by reference
const cachedRef = medplum.getCachedReference({ reference: 'Patient/123' });

// Force invalidation
medplum.invalidateUrl('Patient/123');     // Single resource
medplum.invalidateSearches('Patient');    // All Patient search results
medplum.invalidateAll();                  // Everything
```

### Write Operations Auto-Invalidate

When you create, update, or delete a resource, the client automatically:
1. Updates the cache entry for that resource
2. Invalidates all cached searches for that resource type

This means you never get stale search results after a write.

---

## 3.8 Batch Reads

Read multiple resources in a single HTTP request:

```typescript
const result = await medplum.executeBatch({
  resourceType: 'Bundle',
  type: 'batch',
  entry: [
    { request: { method: 'GET', url: 'Patient/homer-simpson' } },
    { request: { method: 'GET', url: 'Patient/marge-simpson' } },
    { request: { method: 'GET', url: 'Practitioner/dr-hibbert' } },
  ],
});

// Each entry has the response
const homer = result.entry?.[0]?.resource as Patient;
const marge = result.entry?.[1]?.resource as Patient;
const drHibbert = result.entry?.[2]?.resource as Practitioner;
```

Or use auto-batching for transparent grouping:

```typescript
const medplum = new MedplumClient({ autoBatchTime: 50 });

// These three reads become one HTTP request automatically
const [homer, marge, drHibbert] = await Promise.all([
  medplum.readResource('Patient', 'homer-simpson'),
  medplum.readResource('Patient', 'marge-simpson'),
  medplum.readResource('Practitioner', 'dr-hibbert'),
]);
```

---

## 3.9 Error Handling for Reads

```typescript
import { isOperationOutcome, OperationOutcomeError } from '@medplum/core';

try {
  const patient = await medplum.readResource('Patient', 'nonexistent-id');
} catch (err) {
  if (err instanceof OperationOutcomeError) {
    const outcome = err.outcome;
    // outcome.issue[0].code might be:
    //   'not-found'  → 404: Resource doesn't exist
    //   'deleted'    → 410: Resource was soft-deleted
    //   'forbidden'  → 403: No access to this resource
    console.error(`FHIR error: ${outcome.issue?.[0]?.diagnostics}`);
  }
}
```

---

## Summary: Read Method Quick Reference

| Method | Input | Returns | Cached? | Use When |
|--------|-------|---------|---------|----------|
| `readResource` | Type + ID | Single resource | Yes | You know the exact resource |
| `readReference` | `Reference<T>` | Single resource | Yes | Following links between resources |
| `readHistory` | Type + ID | Bundle of all versions | No | Audit trail, change log |
| `readVersion` | Type + ID + versionId | Single historical version | No | Viewing a past state |
| `readCanonical` | Type + canonical URL | Single resource | Yes | StructureDefinitions, ValueSets |
| `readPatientEverything` | Patient ID | Bundle of all related | No | Patient summary views |
| `readResourceGraph` | Type + ID + graph name | Bundle of linked resources | No | Complex relationship traversal |
| `getCached` | Type + ID | Resource or undefined | N/A | Sync read from cache only |

**Next:** [Chapter 4 — Search: The Query Engine →](./04-search.md)
