# Chapter 2: Creating Resources

> **Key takeaway:** Medplum gives you five creation strategies — simple create, conditional create, upsert, batch create, and transaction create. Choosing the right one prevents duplicate data and race conditions.

---

## 2.1 The Creation Decision Tree

```
                    Need to create a resource?
                              │
                    ┌─────────┴──────────┐
                    │                     │
              Single resource?      Multiple resources?
                    │                     │
          ┌─────────┴──────────┐    ┌─────┴──────┐
          │                    │    │             │
   Is idempotency        Just create   Must ALL    Can some
   important?            it             succeed?   fail?
          │                    │    │             │
    ┌─────┴─────┐         createResource  │        │
    │           │              │    │             │
  Know the   Only have       │  Transaction   Batch
  search     a search        │  (atomic)    (independent)
  criteria?  query?          │    │             │
    │           │              │  executeBatch  executeBatch
    │     createResourceIfNoneExist │  type:'transaction'  type:'batch'
    │           │              │    │             │
  upsertResource               │    │             │
    │                          │    │             │
    └──────────────────────────┴────┴─────────────┘
```

---

## 2.2 Simple Create: `createResource`

The most common operation. You provide the resource data; Medplum assigns the `id` and `meta`.

### Signature

```typescript
// From packages/core/src/client.ts
createResource<T extends Resource>(
  resource: T,
  options?: MedplumRequestOptions
): Promise<WithId<T>>
```

### Basic Example

```typescript
import { MedplumClient } from '@medplum/core';
import type { Patient } from '@medplum/fhirtypes';

const medplum = new MedplumClient();

const patient = await medplum.createResource<Patient>({
  resourceType: 'Patient',
  name: [{ given: ['Homer', 'Jay'], family: 'Simpson' }],
  birthDate: '1956-05-12',
  gender: 'male',
  telecom: [
    { system: 'phone', value: '555-1234', use: 'home' },
    { system: 'email', value: 'homer@springfield.net' },
  ],
});

// patient.id is now guaranteed to exist (WithId<Patient>)
console.log(`Created patient: ${patient.id}`);
```

### Business Use Case: Patient Registration

```typescript
import { createReference } from '@medplum/core';
import type { Patient, Practitioner, Encounter } from '@medplum/fhirtypes';

// Step 1: Create the patient
const patient = await medplum.createResource<Patient>({
  resourceType: 'Patient',
  name: [{ given: ['Jane'], family: 'Doe' }],
  birthDate: '1990-03-15',
  gender: 'female',
  identifier: [
    {
      system: 'https://my-clinic.example/mrn',
      value: 'MRN-20240115-001',
    },
  ],
});

// Step 2: Create the first encounter, referencing the patient
const encounter = await medplum.createResource<Encounter>({
  resourceType: 'Encounter',
  status: 'in-progress',
  class: { code: 'AMB', display: 'ambulatory' },
  subject: createReference(patient),        // ← Reference created from the patient
  participant: [
    {
      individual: { reference: 'Practitioner/dr-smith' },  // ← Known practitioner
    },
  ],
});
```

### What `createReference` Does Under the Hood

From `packages/core/src/utils.ts`:

```typescript
// createReference(patient) produces:
{
  reference: 'Patient/homer-simpson',   // "ResourceType/id"
  display: 'Homer Jay Simpson'          // Auto-generated from resource display
}
```

---

## 2.3 Conditional Create: `createResourceIfNoneExist`

Prevents duplicates by searching first. If a matching resource exists, it returns the existing one instead of creating a duplicate.

### Signature

```typescript
createResourceIfNoneExist<T extends Resource>(
  resource: T,
  query: string,                    // Search query (without resourceType or ?)
  options?: MedplumRequestOptions
): Promise<WithId<T>>
```

### How It Works

```
   createResourceIfNoneExist(resource, query)
                    │
                    ▼
         POST /ResourceType
         Header: If-None-Exist: {query}
                    │
           ┌────────┴────────┐
           │                  │
     No match found     Match found
           │                  │
     Create new         Return existing
     resource           resource
           │                  │
     201 Created       200 OK (or 304)
```

### Example: Avoid Duplicate Organizations

```typescript
import type { Organization } from '@medplum/fhirtypes';

// If an Organization with this NPI already exists, return it.
// Otherwise, create it.
const org = await medplum.createResourceIfNoneExist<Organization>(
  {
    resourceType: 'Organization',
    name: 'Springfield General Hospital',
    identifier: [
      {
        system: 'http://hl7.org/fhir/sid/us-npi',
        value: '1234567890',
      },
    ],
  },
  'identifier=http://hl7.org/fhir/sid/us-npi|1234567890'
);
```

### When to Use This

| Scenario | Use `createResourceIfNoneExist`? |
|----------|----------------------------------|
| Importing data from an external system | Yes — external IDs prevent duplicates |
| User submitting a form that could be retried | Yes — idempotent on identifier |
| Creating a resource that's truly new every time (e.g., a new Observation) | No — use `createResource` |
| Upserting where you want to update if exists | No — use `upsertResource` |

---

## 2.4 Upsert: `upsertResource`

Update if it exists, create if it doesn't. This is a single atomic operation (conditional PUT).

### Signature

```typescript
upsertResource<T extends Resource>(
  resource: T,
  query: QueryTypes,               // Search parameters to find existing
  options?: MedplumRequestOptions
): Promise<WithId<T>>
```

### Example: Sync External Lab Catalog

```typescript
import type { ActivityDefinition } from '@medplum/fhirtypes';

// If a test with this code exists, update it. Otherwise, create it.
const labTest = await medplum.upsertResource<ActivityDefinition>(
  {
    resourceType: 'ActivityDefinition',
    status: 'active',
    name: 'CBC',
    title: 'Complete Blood Count',
    code: {
      coding: [{
        system: 'https://samplelab.com/tests',
        code: 'CBC-001',
      }],
    },
  },
  'identifier=https://samplelab.com/tests|CBC-001'
);
```

### Conditional Create vs. Upsert

| | `createResourceIfNoneExist` | `upsertResource` |
|--|---|---|
| **If match found** | Returns existing (no update) | Updates existing with new data |
| **If no match** | Creates new | Creates new |
| **HTTP method** | POST with `If-None-Exist` header | PUT with search params in URL |
| **Best for** | Ensuring existence without modifying | Syncing data from external sources |

---

## 2.5 Batch Create: Multiple Independent Creates

When you need to create many resources and they don't depend on each other.

### Example: Bulk Patient Import

```typescript
import type { Bundle, Patient } from '@medplum/fhirtypes';

const patients: Patient[] = [
  { resourceType: 'Patient', name: [{ family: 'Simpson', given: ['Homer'] }] },
  { resourceType: 'Patient', name: [{ family: 'Simpson', given: ['Marge'] }] },
  { resourceType: 'Patient', name: [{ family: 'Simpson', given: ['Bart'] }] },
];

const batch: Bundle = {
  resourceType: 'Bundle',
  type: 'batch',
  entry: patients.map((patient) => ({
    resource: patient,
    request: {
      method: 'POST',
      url: 'Patient',
    },
  })),
};

const result = await medplum.executeBatch(batch);

// Each entry in result has its own status — some may fail while others succeed
for (const entry of result.entry ?? []) {
  if (entry.response?.status?.startsWith('2')) {
    console.log(`Created: ${entry.response.location}`);
  } else {
    console.error(`Failed: ${entry.response?.outcome?.issue?.[0]?.diagnostics}`);
  }
}
```

---

## 2.6 Transaction Create: Atomic Multi-Resource Creation

When resources reference each other and ALL must succeed or ALL must fail.

### Example: Create Patient + Encounter + Observation Atomically

```typescript
import type { Bundle } from '@medplum/fhirtypes';

const transaction: Bundle = {
  resourceType: 'Bundle',
  type: 'transaction',
  entry: [
    // 1. Create the Patient
    {
      fullUrl: 'urn:uuid:patient-1',
      resource: {
        resourceType: 'Patient',
        name: [{ given: ['Jane'], family: 'Doe' }],
      },
      request: { method: 'POST', url: 'Patient' },
    },
    // 2. Create an Encounter referencing the Patient
    {
      fullUrl: 'urn:uuid:encounter-1',
      resource: {
        resourceType: 'Encounter',
        status: 'in-progress',
        class: { code: 'AMB' },
        subject: { reference: 'urn:uuid:patient-1' },  // ← temporary reference
      },
      request: { method: 'POST', url: 'Encounter' },
    },
    // 3. Create a vital sign Observation referencing both
    {
      resource: {
        resourceType: 'Observation',
        status: 'final',
        code: {
          coding: [{ system: 'http://loinc.org', code: '8867-4', display: 'Heart rate' }],
        },
        subject: { reference: 'urn:uuid:patient-1' },     // ← resolves to Patient
        encounter: { reference: 'urn:uuid:encounter-1' },  // ← resolves to Encounter
        valueQuantity: { value: 72, unit: 'beats/minute', system: 'http://unitsofmeasure.org', code: '/min' },
      },
      request: { method: 'POST', url: 'Observation' },
    },
  ],
};

const result = await medplum.executeBatch(transaction);
// If any entry fails, the entire transaction rolls back.
```

### How Internal References Work

```
  Before server processing:              After server processing:
  ┌─────────────────────────┐            ┌─────────────────────────┐
  │ fullUrl: urn:uuid:pat-1 │            │ id: abc-123             │
  │ resourceType: Patient   │ ────────►  │ resourceType: Patient   │
  └─────────────────────────┘            └─────────────────────────┘
                                                    ▲
  ┌─────────────────────────┐            ┌──────────┴──────────────┐
  │ subject:                │            │ subject:                │
  │   ref: urn:uuid:pat-1  │ ────────►  │   ref: Patient/abc-123  │
  │ resourceType: Encounter │            │ resourceType: Encounter │
  └─────────────────────────┘            └─────────────────────────┘
```

The server replaces all `urn:uuid:` references with the real IDs after creation. Medplum also topologically sorts entries so that referenced resources are created before the resources that reference them (see `reorderBundle` in `packages/core/src/bundle.ts`).

---

## 2.7 Conditional Create in Transactions

Combine `ifNoneExist` with transactions to avoid duplicates inside atomic operations.

```typescript
const transaction: Bundle = {
  resourceType: 'Bundle',
  type: 'transaction',
  entry: [
    {
      fullUrl: 'urn:uuid:org-1',
      resource: {
        resourceType: 'Organization',
        identifier: [{ system: 'http://example.com/orgs', value: 'springfield-general' }],
        name: 'Springfield General Hospital',
      },
      request: {
        method: 'POST',
        url: 'Organization',
        // Only create if no Organization with this identifier exists
        ifNoneExist: 'identifier=http://example.com/orgs|springfield-general',
      },
    },
    {
      resource: {
        resourceType: 'Patient',
        name: [{ given: ['Homer'], family: 'Simpson' }],
        managingOrganization: { reference: 'urn:uuid:org-1' },
      },
      request: { method: 'POST', url: 'Patient' },
    },
  ],
};
```

---

## 2.8 Auto-Batching: Client-Side Performance Optimization

The MedplumClient can automatically group multiple concurrent requests into a single batch.

```typescript
// Enable auto-batching (groups requests made within 100ms)
const medplum = new MedplumClient({
  autoBatchTime: 100,  // milliseconds
});

// These three creates are automatically combined into one batch request
const [patient, practitioner, org] = await Promise.all([
  medplum.createResource({ resourceType: 'Patient', name: [{ family: 'Doe' }] }),
  medplum.createResource({ resourceType: 'Practitioner', name: [{ family: 'Smith' }] }),
  medplum.createResource({ resourceType: 'Organization', name: 'Acme Health' }),
]);
// One HTTP request instead of three!
```

**Important:** Auto-batching only works when requests are made concurrently (via `Promise.all`). Sequential `await` calls will send individual requests.

---

## 2.9 Creating Binary and Media Resources

For files, images, and documents:

```typescript
// Upload a file as a Binary resource
const binary = await medplum.createBinary({
  data: myFile,                    // File | Blob | Uint8Array | string
  contentType: 'application/pdf',
  filename: 'lab-report.pdf',
});

// Or create an Attachment (wraps Binary for use in other resources)
const attachment = await medplum.createAttachment({
  data: myImage,
  contentType: 'image/jpeg',
  filename: 'wound-photo.jpg',
});

// Use the attachment in a DocumentReference
const docRef = await medplum.createResource({
  resourceType: 'DocumentReference',
  status: 'current',
  content: [{ attachment }],
  subject: createReference(patient),
});
```

---

## Summary: Creation Method Quick Reference

| Method | When to Use | Idempotent? | Atomic? |
|--------|------------|-------------|---------|
| `createResource` | Simple one-off creation | No | Single resource |
| `createResourceIfNoneExist` | Prevent duplicates (don't update existing) | Yes | Single resource |
| `upsertResource` | Create or update based on search | Yes | Single resource |
| `executeBatch` (batch) | Multiple independent operations | Depends on entries | No (partial failure OK) |
| `executeBatch` (transaction) | Multiple dependent operations | Depends on entries | Yes (all-or-nothing) |
| `createBinary` | File uploads | No | Single resource |

---

## Control Flow: What Happens When You Create a Resource

```
  medplum.createResource(resource)
           │
           ▼
  ┌─────────────────────┐
  │  POST /ResourceType  │    Client sends HTTP POST
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Check Permissions   │    AccessPolicy evaluated
  │  (AccessPolicy)      │    → Can this user create this type?
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Pre-commit Bots     │    Optional: Bots can modify or reject
  │  (if configured)     │    the resource before validation
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Validate Resource   │    Schema + FHIRPath constraints
  │  (StructureDefinition│    + Profile validation
  │   + Profiles)        │    + Terminology bindings
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Reference Validation│    Optional: Verify all referenced
  │  (if enabled)        │    resources exist and are accessible
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Write to Database   │    PostgreSQL: resource table +
  │  + Index             │    lookup tables (search indexes)
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Trigger             │    Subscriptions fire for matching
  │  Subscriptions       │    criteria → Bots, webhooks, etc.
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Return 201 Created  │    Response includes the full
  │  + Resource          │    resource with id, meta, etc.
  └─────────────────────┘
```

**Next:** [Chapter 3 — Reading & Retrieving Data →](./03-reading-data.md)
