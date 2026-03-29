# Chapter 6: Updating & Patching

> **Key takeaway:** Medplum gives you three update strategies — full replace (PUT), surgical patch (PATCH), and atomic batch — plus optimistic locking to prevent lost updates. Choosing the right strategy depends on whether you have the full resource, how concurrent your environment is, and whether operations must be atomic.

---

## 6.1 Update Decision Tree

```
  Need to modify a resource?
            │
  ┌─────────┴──────────────────────────────┐
  │                                         │
  Have the full resource?          Only know specific fields
  │                                to change?
  │                                         │
  updateResource (PUT)               patchResource (PATCH)
  Replaces entire resource           Surgically modifies fields
  │                                         │
  │                                         │
  Concurrent editors?              Concurrent editors?
  │                                         │
  Use If-Match header              Include 'test' operation
  (optimistic locking)             for versionId check
  │                                         │
  │                                         │
  Multiple resources must           Multiple resources must
  change atomically?                change atomically?
  │                                         │
  Transaction Bundle                Transaction Bundle
  type: 'transaction'               with PATCH entries
```

---

## 6.2 Full Update: `updateResource`

Replaces the entire resource. You must send ALL fields, not just the changed ones.

### Signature

```typescript
updateResource<T extends Resource>(
  resource: T,               // Must include resourceType and id
  options?: MedplumRequestOptions
): Promise<WithId<T>>
```

### Example: Update Patient Demographics

```typescript
// Step 1: Read the current resource
const patient = await medplum.readResource('Patient', 'homer-simpson');

// Step 2: Modify what you need
patient.telecom = [
  ...(patient.telecom ?? []),
  { system: 'email', value: 'homer.j.simpson@springfield.net', use: 'work' },
];

// Step 3: Write it back
const updated = await medplum.updateResource(patient);
console.log(updated.meta?.versionId); // Incremented
```

### What Happens If Nothing Changed

If the resource body is identical to the current version, the server returns `304 Not Modified` and no new version is created. The client returns the original resource.

---

## 6.3 JSON Patch: `patchResource`

Modify specific fields without sending the entire resource. Uses [RFC 6902 JSON Patch](https://datatracker.ietf.org/doc/html/rfc6902).

### Signature

```typescript
patchResource<RT extends ResourceType>(
  resourceType: RT,
  id: string,
  operations: PatchOperation[],
  options?: MedplumRequestOptions
): Promise<WithId<ExtractResource<RT>>>
```

### Patch Operations

| Operation | What It Does | Example |
|-----------|-------------|---------|
| `add` | Add a value (or append to array) | Add a phone number |
| `remove` | Remove a value | Remove an address |
| `replace` | Replace an existing value | Change status |
| `copy` | Copy a value from one path to another | Copy name to alias |
| `move` | Move a value from one path to another | Restructure data |
| `test` | Assert a value equals expected | Optimistic locking check |

### Example: Change Task Status

```typescript
const updated = await medplum.patchResource('Task', 'task-123', [
  // Safety: verify current version before patching
  { op: 'test', path: '/meta/versionId', value: '5' },
  // Change the status
  { op: 'replace', path: '/status', value: 'completed' },
  // Add a note
  {
    op: 'add',
    path: '/note/-',   // "-" means append to array
    value: { text: 'Completed by Dr. Smith', time: new Date().toISOString() },
  },
]);
```

### Example: Add a Tag to a Resource

```typescript
await medplum.patchResource('Patient', 'homer', [
  {
    op: 'add',
    path: '/meta/tag/-',
    value: { system: 'http://example.com/tags', code: 'vip' },
  },
]);
```

### Example: Remove a Telecom Entry

```typescript
// Remove the second telecom entry (index 1)
await medplum.patchResource('Patient', 'homer', [
  { op: 'remove', path: '/telecom/1' },
]);
```

### When to Use PATCH vs PUT

| | PUT (`updateResource`) | PATCH (`patchResource`) |
|--|---|---|
| **Payload** | Entire resource | Only the changes |
| **Bandwidth** | Higher | Lower |
| **Merge conflicts** | Overwrites all fields | Only touches specific paths |
| **Risk** | Can accidentally clear fields you didn't send | Can fail if path doesn't exist |
| **Best for** | Form submissions (you have the whole object) | Status updates, adding to arrays |

---

## 6.4 Optimistic Locking: Preventing Lost Updates

### The Lost Update Problem

```
  Time ──────────────────────────────────────────────────────►

  User A: Read Patient (v1)
                     User B: Read Patient (v1)
  User A: Update name → Write (v2) ✓
                     User B: Update phone → Write (v3)
                     ⚠️ User B's write OVERWRITES User A's name change!
```

### Solution: `If-Match` Header

```typescript
// Read current version
const patient = await medplum.readResource('Patient', 'homer');

// Make changes
patient.name = [{ given: ['Homer', 'Jay'], family: 'Simpson' }];

// Write with version check
try {
  const updated = await medplum.updateResource(patient, {
    headers: {
      'If-Match': `W/"${patient.meta?.versionId}"`,
    },
  });
} catch (err) {
  if (err instanceof OperationOutcomeError) {
    // 412 Precondition Failed — someone else updated first
    // Solution: re-read, re-apply changes, retry
    console.error('Concurrent update detected — please retry');
  }
}
```

### Optimistic Locking with PATCH

```typescript
await medplum.patchResource('Task', 'task-123', [
  // This 'test' operation fails if the versionId doesn't match
  { op: 'test', path: '/meta/versionId', value: currentVersionId },
  { op: 'replace', path: '/status', value: 'in-progress' },
]);
// If someone else updated between your read and your patch,
// the test fails and the entire patch is rejected
```

---

## 6.5 Upsert: Create-or-Update in One Call

```typescript
// If a Patient matching the identifier exists, update it.
// If not, create it.
const patient = await medplum.upsertResource(
  {
    resourceType: 'Patient',
    identifier: [{ system: 'http://example.com/mrn', value: 'MRN-001' }],
    name: [{ given: ['Homer'], family: 'Simpson' }],
    birthDate: '1956-05-12',
  },
  'identifier=http://example.com/mrn|MRN-001'
);
```

### Business Use Case: External Data Sync

```typescript
// Sync practitioners from an HR system — upsert by NPI
async function syncPractitioners(hrRecords: HRRecord[]) {
  for (const record of hrRecords) {
    await medplum.upsertResource(
      {
        resourceType: 'Practitioner',
        identifier: [
          { system: 'http://hl7.org/fhir/sid/us-npi', value: record.npi },
        ],
        name: [{ given: [record.firstName], family: record.lastName }],
        telecom: [{ system: 'email', value: record.email }],
      },
      `identifier=http://hl7.org/fhir/sid/us-npi|${record.npi}`
    );
  }
}
```

---

## 6.6 Batch and Transaction Updates

### Atomic Multi-Resource Update (Transaction)

```typescript
// Transfer a patient from one care team to another — atomically
const patient = await medplum.readResource('Patient', 'homer');
const oldTask = await medplum.readResource('Task', 'task-old');

const transaction: Bundle = {
  resourceType: 'Bundle',
  type: 'transaction',
  entry: [
    // Update patient's general practitioner
    {
      resource: {
        ...patient,
        generalPractitioner: [{ reference: 'Practitioner/dr-new' }],
      },
      request: {
        method: 'PUT',
        url: `Patient/${patient.id}`,
        ifMatch: `W/"${patient.meta?.versionId}"`,
      },
    },
    // Cancel the old task
    {
      resource: { ...oldTask, status: 'cancelled' },
      request: {
        method: 'PUT',
        url: `Task/${oldTask.id}`,
        ifMatch: `W/"${oldTask.meta?.versionId}"`,
      },
    },
    // Create a new task for the new practitioner
    {
      resource: {
        resourceType: 'Task',
        status: 'requested',
        intent: 'order',
        for: createReference(patient),
        owner: { reference: 'Practitioner/dr-new' },
        description: 'Review transferred patient',
      },
      request: { method: 'POST', url: 'Task' },
    },
  ],
};

const result = await medplum.executeBatch(transaction);
// All three operations succeed or ALL roll back
```

### PATCH Inside a Transaction

```typescript
const transaction: Bundle = {
  resourceType: 'Bundle',
  type: 'transaction',
  entry: [
    {
      resource: {
        resourceType: 'Binary',
        contentType: 'application/json-patch+json',
        // PATCH operations must be base64-encoded in transaction bundles
        data: btoa(JSON.stringify([
          { op: 'replace', path: '/status', value: 'completed' },
        ])),
      },
      request: {
        method: 'PATCH',
        url: 'Task/task-123',
      },
    },
  ],
};
```

---

## 6.7 Soft Delete and Restore

### Soft Delete

```typescript
await medplum.deleteResource('Patient', 'homer');
// Creates a new version with "deleted" marker
// Resource won't appear in searches
// History is preserved
```

### Hard Delete (Admin Only)

```typescript
// Permanently removes ALL versions — irreversible
await medplum.post(medplum.fhirUrl('Patient', 'homer', '$expunge'));
```

### Restore a Soft-Deleted Resource

```typescript
async function restoreResource(resourceType: ResourceType, id: string) {
  const history = await medplum.readHistory(resourceType, id);

  // Find the most recent non-deleted version
  const lastGoodVersion = history.entry?.find(
    (entry) => entry.response?.status === '200' && entry.resource
  )?.resource;

  if (!lastGoodVersion) {
    throw new Error('No restorable version found');
  }

  // Update with the good version — this un-deletes it
  return medplum.updateResource(lastGoodVersion);
}
```

---

## 6.8 Server-Side Update Pipeline

```
  medplum.updateResource(resource)
           │
           ▼
  ┌─────────────────────┐
  │  PUT /Type/id        │
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  If-Match check      │    Compare versionId
  │  (if header present) │    → 412 Precondition Failed if mismatch
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Access Policy check │    Can user update this resource type?
  │  + Write Constraints │    FHIRPath: %before.status != 'final'
  └──────────┬──────────┘    (prevent editing finalized resources)
             │
             ▼
  ┌─────────────────────┐
  │  Pre-commit Bots     │    Can modify resource or reject
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Restore readonly    │    Server restores fields user can't change:
  │  fields              │    id, meta.versionId, meta.lastUpdated,
  └──────────┬──────────┘    meta.author (from access policy)
             │
             ▼
  ┌─────────────────────┐
  │  Validate resource   │    Full validation pipeline
  └──────────┬──────────┘    (see Chapter 7)
             │
             ▼
  ┌─────────────────────┐
  │  isNotModified?      │    If resource is identical to current version
  │                      │    → Return 304, no new version created
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Write new version   │    New row in version table
  │  Update indexes      │    Update lookup tables
  │  Fire subscriptions  │    Trigger matching subscriptions
  └─────────────────────┘
```

---

## 6.9 Business Use Case: Order Status Workflow

```typescript
// A lab order moves through a state machine:
//   draft → active → completed
//                  → cancelled

async function advanceOrderStatus(
  orderId: string,
  newStatus: 'active' | 'completed' | 'cancelled'
) {
  const order = await medplum.readResource('ServiceRequest', orderId);

  // Validate the state transition
  const validTransitions: Record<string, string[]> = {
    draft: ['active', 'cancelled'],
    active: ['completed', 'cancelled'],
  };

  const allowed = validTransitions[order.status] ?? [];
  if (!allowed.includes(newStatus)) {
    throw new Error(
      `Cannot transition from '${order.status}' to '${newStatus}'`
    );
  }

  // Use PATCH with optimistic locking
  return medplum.patchResource('ServiceRequest', orderId, [
    { op: 'test', path: '/meta/versionId', value: order.meta?.versionId },
    { op: 'replace', path: '/status', value: newStatus },
    ...(newStatus === 'completed'
      ? [{ op: 'add', path: '/extension/-', value: {
            url: 'http://example.com/completed-at',
            valueDateTime: new Date().toISOString(),
          }}]
      : []),
  ]);
}
```

---

## Summary: Update Method Quick Reference

| Method | HTTP | Use When | Payload | Idempotent? |
|--------|------|----------|---------|-------------|
| `updateResource` | PUT | You have the full resource | Entire resource | Yes |
| `patchResource` | PATCH | Only changing specific fields | JSON Patch operations | No (use `test` op for safety) |
| `upsertResource` | PUT (conditional) | Create-or-update by search criteria | Entire resource | Yes |
| `deleteResource` | DELETE | Soft-deleting | None | Yes |
| `executeBatch` (transaction) | POST | Multiple atomic changes | Bundle | Depends on entries |

**Next:** [Chapter 7 — Validation & Data Integrity →](./07-validation-and-integrity.md)
