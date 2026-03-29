# Chapter 8: Putting It All Together

> **Key takeaway:** This chapter connects the dots — real end-to-end workflows, a decision framework for choosing the right tool, performance patterns, and a symbol reference. This is the chapter you come back to when you're building a feature and need to decide "how should I implement this?"

---

## 8.1 End-to-End Workflow: Lab Order Lifecycle

This example walks through an entire lab order from creation to results, showing which Medplum APIs to use at each step.

```
  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
  │ 1. Place Order │ ──► │ 2. Collect     │ ──► │ 3. Process    │
  │ ServiceRequest │     │ Specimen       │     │ Results       │
  │ status: active │     │ Specimen       │     │ Observation[] │
  └───────────────┘     └───────────────┘     └───────┬───────┘
                                                       │
  ┌───────────────┐     ┌───────────────┐              │
  │ 5. Notify     │ ◄── │ 4. Finalize   │ ◄────────────┘
  │ Subscription  │     │ DiagReport     │
  │ → Bot         │     │ status: final  │
  └───────────────┘     └───────────────┘
```

### Step 1: Place the Order (Transaction)

```typescript
import { MedplumClient, createReference } from '@medplum/core';
import type { Bundle, ServiceRequest, Patient, Practitioner } from '@medplum/fhirtypes';

async function placeLabOrder(
  medplum: MedplumClient,
  patientId: string,
  practitionerId: string,
  testCode: string,
  testDisplay: string
): Promise<ServiceRequest> {
  // Use a transaction to create the order + task atomically
  const transaction: Bundle = {
    resourceType: 'Bundle',
    type: 'transaction',
    entry: [
      {
        fullUrl: 'urn:uuid:order-1',
        resource: {
          resourceType: 'ServiceRequest',
          status: 'active',
          intent: 'order',
          subject: { reference: `Patient/${patientId}` },
          requester: { reference: `Practitioner/${practitionerId}` },
          code: {
            coding: [{ system: 'http://loinc.org', code: testCode, display: testDisplay }],
          },
          authoredOn: new Date().toISOString(),
        },
        request: { method: 'POST', url: 'ServiceRequest' },
      },
      {
        resource: {
          resourceType: 'Task',
          status: 'requested',
          intent: 'order',
          focus: { reference: 'urn:uuid:order-1' },
          for: { reference: `Patient/${patientId}` },
          description: `Collect specimen for ${testDisplay}`,
        },
        request: { method: 'POST', url: 'Task' },
      },
    ],
  };

  const result = await medplum.executeBatch(transaction);
  return result.entry?.[0]?.resource as ServiceRequest;
}
```

### Step 2: Record Specimen Collection (Create + Patch)

```typescript
async function recordSpecimenCollection(
  medplum: MedplumClient,
  serviceRequestId: string,
  patientId: string
) {
  // Create the specimen
  const specimen = await medplum.createResource({
    resourceType: 'Specimen',
    status: 'available',
    type: { coding: [{ system: 'http://snomed.info/sct', code: '119297000', display: 'Blood' }] },
    subject: { reference: `Patient/${patientId}` },
    request: [{ reference: `ServiceRequest/${serviceRequestId}` }],
    collection: {
      collectedDateTime: new Date().toISOString(),
    },
  });

  // Update the task status
  const task = await medplum.searchOne('Task', {
    focus: `ServiceRequest/${serviceRequestId}`,
  });

  if (task) {
    await medplum.patchResource('Task', task.id, [
      { op: 'test', path: '/meta/versionId', value: task.meta?.versionId },
      { op: 'replace', path: '/status', value: 'in-progress' },
    ]);
  }

  return specimen;
}
```

### Step 3: Record Results (Batch Create)

```typescript
async function recordLabResults(
  medplum: MedplumClient,
  serviceRequestId: string,
  patientId: string,
  results: Array<{ code: string; display: string; value: number; unit: string }>
) {
  // Create all observations in a batch
  const batch: Bundle = {
    resourceType: 'Bundle',
    type: 'batch',
    entry: results.map((r) => ({
      resource: {
        resourceType: 'Observation',
        status: 'final',
        code: { coding: [{ system: 'http://loinc.org', code: r.code, display: r.display }] },
        subject: { reference: `Patient/${patientId}` },
        basedOn: [{ reference: `ServiceRequest/${serviceRequestId}` }],
        valueQuantity: { value: r.value, unit: r.unit, system: 'http://unitsofmeasure.org' },
        effectiveDateTime: new Date().toISOString(),
      },
      request: { method: 'POST', url: 'Observation' },
    })),
  };

  return medplum.executeBatch(batch);
}
```

### Step 4: Finalize the Report (Create + Update)

```typescript
async function finalizeReport(
  medplum: MedplumClient,
  serviceRequestId: string,
  patientId: string,
  observationIds: string[]
) {
  // Create the diagnostic report with references to all observations
  const report = await medplum.createResource({
    resourceType: 'DiagnosticReport',
    status: 'final',
    code: { coding: [{ system: 'http://loinc.org', code: '58410-2', display: 'CBC panel' }] },
    subject: { reference: `Patient/${patientId}` },
    basedOn: [{ reference: `ServiceRequest/${serviceRequestId}` }],
    result: observationIds.map((id) => ({ reference: `Observation/${id}` })),
    effectiveDateTime: new Date().toISOString(),
    issued: new Date().toISOString(),
  });

  // Mark the service request as completed
  await medplum.patchResource('ServiceRequest', serviceRequestId, [
    { op: 'replace', path: '/status', value: 'completed' },
  ]);

  return report;
}
```

### Step 5: Automated Notification (Subscription + Bot)

```typescript
// This Subscription triggers a Bot when a DiagnosticReport is finalized
const subscription = await medplum.createResource({
  resourceType: 'Subscription',
  status: 'active',
  criteria: 'DiagnosticReport?status=final',
  channel: {
    type: 'rest-hook',
    endpoint: `Bot/${notificationBotId}`,
  },
});

// The Bot (separate file):
export async function handler(medplum: MedplumClient, event: BotEvent) {
  const report = event.input as DiagnosticReport;
  const patient = await medplum.readReference(report.subject!);

  // Send notification
  await medplum.createResource({
    resourceType: 'Communication',
    status: 'completed',
    subject: report.subject,
    payload: [{
      contentString: `Lab results are ready: ${report.code?.text}`,
    }],
    recipient: [report.subject!],
  });
}
```

---

## 8.2 Decision Framework: Which API to Use

### "I need to create data"

| Situation | Use | Why |
|-----------|-----|-----|
| Simple new resource | `createResource` | Simplest path |
| Avoid duplicates (external data import) | `createResourceIfNoneExist` | Idempotent on identifier |
| Create or update by identifier | `upsertResource` | Sync external data |
| Multiple independent creates | `executeBatch` (batch) | One HTTP call, partial failure OK |
| Multiple dependent creates | `executeBatch` (transaction) | Atomic, with internal references |
| Upload a file | `createBinary` / `createAttachment` | Handles streaming, content types |

### "I need to read data"

| Situation | Use | Why |
|-----------|-----|-----|
| Know the ID | `readResource` | Direct, cached |
| Have a Reference object | `readReference` | Follows the link |
| Need a list | `searchResources` | Returns array |
| Need first match | `searchOne` | Auto `_count=1` |
| Need metadata (total count, links) | `search` | Returns full Bundle |
| Need all pages | `searchResourcePages` | Async generator |
| Need historical versions | `readHistory` | Audit trail |
| Need related resources too | `search` + `_include` | Single request |
| Need specific fields from related | `graphql` | Minimal payload |
| Need everything about a patient | `readPatientEverything` | One call |

### "I need to update data"

| Situation | Use | Why |
|-----------|-----|-----|
| Have the full resource | `updateResource` | PUT replaces entire resource |
| Only changing a few fields | `patchResource` | Surgical, no read needed |
| Concurrent editing risk | `If-Match` header or `test` op | Optimistic locking |
| Create or update | `upsertResource` | Single call |
| Multiple atomic changes | `executeBatch` (transaction) | All-or-nothing |

### "I need to validate data"

| Situation | Use | Why |
|-----------|-----|-----|
| Before form submit | `validateResource` (client) | Immediate feedback |
| Test without writing | `$validate` operation | Full server validation |
| Enforce custom rules | Access policy write constraints | Declarative, no code |
| Complex business logic | Pre-commit bot | Full programming power |
| Enforce data shape | Profile (StructureDefinition) | Reusable across resources |

---

## 8.3 Performance Patterns

### Pattern 1: Auto-Batching for Parallel Fetches

```typescript
const medplum = new MedplumClient({ autoBatchTime: 100 });

// Component A fetches patient
const patientPromise = medplum.readResource('Patient', 'homer');
// Component B fetches practitioner (same tick)
const practitionerPromise = medplum.readResource('Practitioner', 'dr-hibbert');
// Component C fetches organization (same tick)
const orgPromise = medplum.readResource('Organization', 'springfield-general');

// These three GET requests are bundled into ONE batch request
const [patient, practitioner, org] = await Promise.all([
  patientPromise, practitionerPromise, orgPromise,
]);
```

### Pattern 2: Avoid N+1 with `_include`

```typescript
// ❌ N+1 problem: 1 search + N reads
const observations = await medplum.searchResources('Observation', {
  subject: `Patient/${patientId}`,
});
// Then for each observation, reading the performer... 💀
for (const obs of observations) {
  const performer = await medplum.readReference(obs.performer?.[0]!);
}

// ✅ Single request with _include
const bundle = await medplum.search('Observation', {
  subject: `Patient/${patientId}`,
  _include: 'Observation:performer',
});
```

### Pattern 3: Use `_elements` to Reduce Payload

```typescript
// ❌ Fetching full Patient resources just to display a name list
const patients = await medplum.searchResources('Patient', { _count: '100' });

// ✅ Only fetch the fields you need
const patients = await medplum.searchResources('Patient', {
  _count: '100',
  _elements: 'id,name,birthDate',
});
```

### Pattern 4: Cursor Pagination for Large Exports

```typescript
// ❌ Offset pagination breaks at 10,000
await medplum.searchResources('Observation', { _offset: '15000' }); // Error!

// ✅ Cursor pagination for unlimited results
for await (const page of medplum.searchResourcePages('Observation', {
  _count: 1000,
  _sort: '_lastUpdated',
})) {
  await bulkProcess(page);
}
```

### Pattern 5: Transaction for Related Writes

```typescript
// ❌ Non-atomic: if step 2 fails, step 1 is already committed
await medplum.createResource(patient);
await medplum.createResource(encounter);  // What if this fails?

// ✅ Atomic: both succeed or both roll back
await medplum.executeBatch({
  resourceType: 'Bundle',
  type: 'transaction',
  entry: [
    { fullUrl: 'urn:uuid:p1', resource: patient, request: { method: 'POST', url: 'Patient' } },
    { resource: { ...encounter, subject: { reference: 'urn:uuid:p1' } },
      request: { method: 'POST', url: 'Encounter' } },
  ],
});
```

---

## 8.4 Code Symbol Reference

Quick lookup for the key symbols you'll import and use.

### From `@medplum/core`

| Symbol | Type | Purpose |
|--------|------|---------|
| `MedplumClient` | Class | Main client for all FHIR operations |
| `createReference(resource)` | Function | Resource → Reference with display |
| `getReferenceString(ref)` | Function | Reference or Resource → `"Type/id"` |
| `resolveId(ref)` | Function | Reference or Resource → `"id"` |
| `parseReference(ref)` | Function | Reference → `[ResourceType, id]` |
| `getDisplayString(resource)` | Function | Resource → human-readable name |
| `validateResource(resource)` | Function | Client-side FHIR validation |
| `OperationOutcomeError` | Class | Error thrown on validation/server failures |
| `isOperationOutcome(x)` | Function | Type guard for OperationOutcome |
| `WithId<T>` | Type | `T & { id: string }` — resource with guaranteed ID |
| `ReadablePromise<T>` | Type | Promise that can be sync-read from cache |
| `PatchOperation` | Type | JSON Patch operation (`op`, `path`, `value`) |
| `SearchRequest` | Interface | Parsed search query representation |
| `parseSearchRequest(url)` | Function | URL → SearchRequest |
| `formatSearchQuery(request)` | Function | SearchRequest → URL query string |

### From `@medplum/fhirtypes`

| Symbol | Type | Purpose |
|--------|------|---------|
| `Patient`, `Observation`, etc. | Interface | FHIR resource types |
| `Reference<T>` | Interface | Type-safe resource reference |
| `Bundle` | Interface | Collection of resources (batch/transaction) |
| `CodeableConcept` | Interface | Coded value with display text |
| `Coding` | Interface | Single code from a code system |
| `Identifier` | Interface | Business identifier (MRN, NPI, etc.) |
| `OperationOutcome` | Interface | Validation/error result |
| `ResourceType` | Union type | All valid resource type strings |
| `Resource` | Interface | Base interface for all resources |

---

## 8.5 Testing with `@medplum/mock`

Write tests without a running server:

```typescript
import { MockClient } from '@medplum/mock';

describe('Lab Order Workflow', () => {
  let medplum: MockClient;

  beforeEach(async () => {
    medplum = new MockClient();
  });

  test('creates a service request with patient reference', async () => {
    const patient = await medplum.createResource({
      resourceType: 'Patient',
      name: [{ given: ['Test'], family: 'Patient' }],
    });

    const order = await medplum.createResource({
      resourceType: 'ServiceRequest',
      status: 'active',
      intent: 'order',
      subject: createReference(patient),
      code: {
        coding: [{ system: 'http://loinc.org', code: '58410-2', display: 'CBC' }],
      },
    });

    expect(order.id).toBeDefined();
    expect(order.subject?.reference).toBe(`Patient/${patient.id}`);
  });

  test('searches by reference', async () => {
    const patient = await medplum.createResource({
      resourceType: 'Patient',
      name: [{ given: ['Test'], family: 'Patient' }],
    });

    await medplum.createResource({
      resourceType: 'Observation',
      status: 'final',
      code: { text: 'Heart Rate' },
      subject: createReference(patient),
    });

    const observations = await medplum.searchResources('Observation', {
      subject: `Patient/${patient.id}`,
    });

    expect(observations.length).toBe(1);
  });
});
```

---

## 8.6 Where to Go Next

| Topic | Resource |
|-------|----------|
| React components for FHIR | `packages/react/` — ResourceForm, SearchControl, ResourceTable |
| Subscription workflows | `packages/docs/docs/subscriptions/` |
| Bot development | `packages/docs/docs/bots/` |
| Access control deep dive | `packages/docs/docs/access/access-policies.md` |
| GraphQL queries | `packages/docs/docs/graphql/` |
| US Core compliance | US Core profiles in `packages/docs/docs/fhir-datastore/profiles.md` |
| Example applications | `examples/` — 21 full example apps |
| FHIR specification | [hl7.org/fhir/R4](https://hl7.org/fhir/R4/) |
| Medplum API reference | `packages/docs/docs/api/` |

---

## Recap: The Mental Model

```
  ┌─────────────────────────────────────────────────────────────┐
  │                     Your Application                        │
  │                                                             │
  │  Feature Requirement ──► Map to FHIR Resources              │
  │                            │                                │
  │  "Track lab orders"  ──► ServiceRequest + Observation       │
  │  "Patient messaging" ──► Communication                      │
  │  "Appointment booking" ► Appointment                        │
  └────────────────────────────┬────────────────────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │   MedplumClient     │
                    │                     │
                    │  createResource     │  ← Chapter 2
                    │  readResource       │  ← Chapter 3
                    │  searchResources    │  ← Chapter 4
                    │  readReference      │  ← Chapter 5
                    │  updateResource     │  ← Chapter 6
                    │  patchResource      │  ← Chapter 6
                    │  executeBatch       │  ← Chapters 2, 6
                    │  validateResource   │  ← Chapter 7
                    └──────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    │   Medplum Server    │
                    │                     │
                    │  Validation Pipeline│  ← Chapter 7
                    │  Access Policies    │  ← Chapter 7
                    │  Search Indexing    │  ← Chapter 4
                    │  Subscriptions      │  ← Chapter 8
                    │  PostgreSQL         │
                    └─────────────────────┘
```

You now have the knowledge to:
1. **Model data** using FHIR resources and TypeScript types
2. **Create data** with the right strategy (simple, conditional, upsert, batch, transaction)
3. **Read data** efficiently (single reads, cached reads, batch reads)
4. **Search data** with parameters, modifiers, chaining, and pagination
5. **Link data** using references, includes, compartments, and GraphQL
6. **Update data** safely with PUT, PATCH, and optimistic locking
7. **Validate data** at every layer from compile time to server-side enforcement
8. **Build workflows** that combine all of the above into production features
