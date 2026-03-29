# Chapter 5: References & Relationships

> **Key takeaway:** FHIR doesn't have JOINs — it has References. Understanding how to create, follow, and query across references is the single most important skill for building connected healthcare features. Medplum gives you five strategies: direct reads, `_include`/`_revinclude`, chained search, GraphQL, and compartments.

---

## 5.1 The Reference Model

In relational databases, you JOIN tables via foreign keys. In FHIR, resources link to each other via `Reference` objects.

```
  ┌─────────────────────┐
  │    Patient           │
  │    id: "homer"       │◄──────────────────────────────────┐
  │    name: "Homer"     │                                   │
  └─────────────────────┘                                   │
         ▲       ▲                                          │
         │       │                                          │
    subject   subject                              managingOrganization
         │       │                                          │
  ┌──────┴──┐ ┌──┴─────────────┐                  ┌────────┴────────┐
  │Observation│ │ServiceRequest  │                  │  Organization   │
  │id: "obs-1"│ │id: "sr-1"     │                  │  id: "org-1"   │
  │code: HR   │ │code: CBC      │                  │  name: "Spfd"  │
  │subject:   │ │subject:       │  requester       └─────────────────┘
  │ Patient/  │ │ Patient/homer │◄──────────┐
  │ homer     │ │requester:     │           │
  │encounter: │ │ Practitioner/ │  ┌────────┴────────┐
  │ Encounter/│ │ dr-hibbert    │  │  Practitioner    │
  │ enc-1     │ └───────────────┘  │  id: "dr-hibbert"│
  └─────┬─────┘                    │  name: "Hibbert" │
        │                          └──────────────────┘
   encounter
        │
  ┌─────┴───────────┐
  │   Encounter      │
  │   id: "enc-1"    │
  │   subject:       │
  │   Patient/homer  │
  └──────────────────┘
```

### Anatomy of a Reference

```typescript
import type { Reference, Patient } from '@medplum/fhirtypes';

// A Reference has these fields:
const ref: Reference<Patient> = {
  reference: 'Patient/homer',      // Required: "ResourceType/id"
  display: 'Homer Simpson',        // Optional: human-readable text
  type: 'Patient',                 // Optional: explicit type hint
  identifier: {                    // Optional: logical identifier (for external refs)
    system: 'http://example.com/mrn',
    value: 'MRN-001',
  },
};
```

---

## 5.2 Reference Utility Functions

From `packages/core/src/utils.ts` — these are your everyday tools:

```typescript
import {
  createReference,
  getReferenceString,
  resolveId,
  parseReference,
  getDisplayString,
} from '@medplum/core';
import type { Patient, Reference } from '@medplum/fhirtypes';

const patient: Patient = {
  resourceType: 'Patient',
  id: 'homer',
  name: [{ given: ['Homer'], family: 'Simpson' }],
};

// createReference: Resource → Reference (with auto-generated display)
const ref = createReference(patient);
// { reference: 'Patient/homer', display: 'Homer Simpson' }

// getReferenceString: Resource or Reference → "Type/id" string
getReferenceString(patient);     // 'Patient/homer'
getReferenceString(ref);         // 'Patient/homer'

// resolveId: Extract just the ID
resolveId(ref);                  // 'homer'
resolveId(patient);              // 'homer'

// parseReference: Split into [ResourceType, id]
parseReference(ref);             // ['Patient', 'homer']

// getDisplayString: Human-friendly text
getDisplayString(patient);       // 'Homer Simpson'
```

---

## 5.3 Strategy 1: Direct Reference Reads

The simplest approach — follow one reference at a time.

```typescript
const observation = await medplum.readResource('Observation', 'obs-1');

// Follow the subject reference
const patient = await medplum.readReference(
  observation.subject as Reference<Patient>
);

// Follow the encounter reference
const encounter = await medplum.readReference(
  observation.encounter as Reference<Encounter>
);
```

### When to Use

- You need 1-2 related resources
- The resource is likely cached already
- You're in a detail view (not a list)

### Performance Tip: Parallel Reads

```typescript
// Read multiple references in parallel
const [patient, encounter, performer] = await Promise.all([
  medplum.readReference(obs.subject as Reference<Patient>),
  medplum.readReference(obs.encounter as Reference<Encounter>),
  medplum.readReference(obs.performer?.[0] as Reference<Practitioner>),
]);
```

---

## 5.4 Strategy 2: `_include` and `_revinclude`

Fetch related resources in a single search request. This is the closest thing to a SQL JOIN.

### `_include`: Fetch What I Reference (Forward)

```
  "Give me Observations AND the Patients they reference"

  ┌─────────────┐    _include     ┌──────────────┐
  │ Observation  │  ──────────►   │   Patient     │
  │ (searched)   │   subject      │ (included)    │
  └─────────────┘                 └──────────────┘
```

```typescript
// Search for observations and include the referenced patients
const bundle = await medplum.search('Observation', {
  code: '8867-4',
  _include: 'Observation:subject',
});

// The bundle contains BOTH Observations and Patients
for (const entry of bundle.entry ?? []) {
  if (entry.resource?.resourceType === 'Observation') {
    console.log('Observation:', entry.resource.id);
  } else if (entry.resource?.resourceType === 'Patient') {
    console.log('Included Patient:', entry.resource.id);
  }
  // entry.search?.mode tells you if it's 'match' or 'include'
}
```

### `_revinclude`: Fetch What References Me (Reverse)

```
  "Give me Patients AND all Observations that reference them"

  ┌──────────────┐   _revinclude   ┌─────────────┐
  │   Patient     │  ◄──────────   │ Observation  │
  │  (searched)   │    subject     │ (included)   │
  └──────────────┘                 └─────────────┘
```

```typescript
// Search for a patient and include all their observations
const bundle = await medplum.search('Patient', {
  _id: 'homer',
  _revinclude: 'Observation:subject',
});
```

### Multi-Level Includes with `:iterate`

```
  "Give me Patients, their Practitioners, and each Practitioner's Organization"

  ┌──────────┐  _include   ┌──────────────┐  _include:iterate  ┌──────────────┐
  │ Patient  │ ─────────►  │ Practitioner │  ────────────────►  │ Organization │
  │(searched)│  general-   │  (included)  │    organization     │  (included)  │
  └──────────┘ practitioner└──────────────┘                     └──────────────┘
```

```typescript
const bundle = await medplum.search('Patient', {
  _id: 'homer',
  _include: 'Patient:general-practitioner',
  '_include:iterate': 'Practitioner:organization',
});
```

### Extracting Resources from Include Results

```typescript
function extractByType<T extends Resource>(
  bundle: Bundle,
  resourceType: string
): T[] {
  return (bundle.entry ?? [])
    .filter((e) => e.resource?.resourceType === resourceType)
    .map((e) => e.resource as T);
}

const observations = extractByType<Observation>(bundle, 'Observation');
const patients = extractByType<Patient>(bundle, 'Patient');
```

---

## 5.5 Strategy 3: GraphQL for Structured Traversal

GraphQL lets you specify exactly which fields you want, and traverse references in a typed way.

```typescript
const result = await medplum.graphql(`
  {
    # Get a specific patient
    Patient(id: "homer") {
      id
      name { given family }
      birthDate

      # Reverse reference: find Observations pointing to this patient
      ObservationList(_reference: subject, _count: 5, _sort: "-date") {
        id
        code { text }
        valueQuantity { value unit }

        # Forward reference: resolve the encounter
        encounter {
          resource {
            ... on Encounter {
              id
              status
              class { code }
            }
          }
        }
      }
    }
  }
`);

// Access results directly
const patient = result.data.Patient;
const observations = result.data.Patient.ObservationList;
```

### GraphQL vs REST `_include`

```
  ┌──────────────────────┬───────────────────────────────────────┐
  │     Feature          │   REST _include    │   GraphQL        │
  ├──────────────────────┼────────────────────┼──────────────────┤
  │ Payload size         │ Full resources     │ Selected fields  │
  │ Field selection      │ No (all fields)    │ Yes              │
  │ Forward traversal    │ _include           │ resource { }     │
  │ Reverse traversal    │ _revinclude        │ List(_reference) │
  │ Multi-level          │ :iterate           │ Nested queries   │
  │ Search modifiers     │ Full support       │ Limited          │
  │ Chained search       │ Full support       │ Not supported    │
  │ Best for             │ Bulk data loading  │ UI data fetching │
  └──────────────────────┴────────────────────┴──────────────────┘
```

---

## 5.6 Strategy 4: Compartment Search

A "compartment" is a set of resources that are logically associated with a focal resource. The Patient compartment contains all resources that reference that patient.

```typescript
// Search within a patient's compartment
// Finds ALL Observations that reference Patient/homer (via subject, performer, etc.)
const observations = await medplum.searchResources('Observation', {
  _compartment: 'Patient/homer',
});

// This is similar to but broader than:
const sameResult = await medplum.searchResources('Observation', {
  subject: 'Patient/homer',
});

// The compartment version also catches Observations linked via performer, etc.
```

### Business Use Case: Patient Portal

```typescript
// Get everything visible to a patient — their compartment scopes what they see
async function getPatientDashboard(patientId: string) {
  const [conditions, medications, appointments, tasks] = await Promise.all([
    medplum.searchResources('Condition', { _compartment: `Patient/${patientId}` }),
    medplum.searchResources('MedicationRequest', { _compartment: `Patient/${patientId}` }),
    medplum.searchResources('Appointment', {
      _compartment: `Patient/${patientId}`,
      date: `ge${new Date().toISOString()}`,
      _sort: 'date',
    }),
    medplum.searchResources('Task', {
      _compartment: `Patient/${patientId}`,
      status: 'requested,in-progress',
    }),
  ]);

  return { conditions, medications, appointments, tasks };
}
```

---

## 5.7 Strategy 5: Conditional References in Transactions

In transaction bundles, you can reference resources by search query instead of ID. The server resolves them at execution time.

```typescript
const transaction: Bundle = {
  resourceType: 'Bundle',
  type: 'transaction',
  entry: [
    {
      resource: {
        resourceType: 'Observation',
        status: 'final',
        code: { coding: [{ system: 'http://loinc.org', code: '8867-4' }] },
        // Reference by search query — resolved server-side
        subject: {
          reference: 'Patient?identifier=http://example.com/mrn|MRN-001',
        },
        // Reference by search query to a Practitioner
        performer: [{
          reference: 'Practitioner?identifier=http://hl7.org/fhir/sid/us-npi|1234567890',
        }],
        valueQuantity: { value: 72, unit: '/min' },
      },
      request: { method: 'POST', url: 'Observation' },
    },
  ],
};
```

This is powerful for data imports where you have external identifiers but not Medplum IDs.

---

## 5.8 Choosing the Right Strategy

```
  Need related resources?
          │
    ┌─────┴──────────────────────────────────┐
    │                                         │
  Single resource?                    List of resources?
    │                                         │
    ├── 1-2 refs → readReference        ┌─────┴──────┐
    │                                   │             │
    ├── Many refs → GraphQL        Need search    Need full
    │                              flexibility?   resources?
    └── All related →                   │             │
        readPatientEverything     Chained search   _include /
                                                 _revinclude
```

| Strategy | Strengths | Limitations | Best For |
|----------|-----------|-------------|----------|
| `readReference` | Simple, cached | N+1 problem for lists | Detail views |
| `_include` / `_revinclude` | Single request, full resources | Can't filter included resources | List views with related data |
| Chained search | Filter by referenced properties | One direction at a time | Filtered queries |
| GraphQL | Field selection, multi-level | Limited search modifiers | UI data fetching |
| Compartment | Broad relationship scope | Less precise than direct reference | Patient-centric views |

---

## 5.9 Real-World Pattern: Lab Results Dashboard

Combining multiple strategies for a lab results page:

```typescript
async function getLabResults(patientId: string) {
  // 1. Search for diagnostic reports, include the related observations
  const bundle = await medplum.search('DiagnosticReport', {
    subject: `Patient/${patientId}`,
    _sort: '-date',
    _count: '20',
    _include: 'DiagnosticReport:result',      // Include Observations
  });

  // 2. Separate the reports from the included observations
  const reports: DiagnosticReport[] = [];
  const observationMap = new Map<string, Observation>();

  for (const entry of bundle.entry ?? []) {
    if (entry.resource?.resourceType === 'DiagnosticReport') {
      reports.push(entry.resource as DiagnosticReport);
    } else if (entry.resource?.resourceType === 'Observation') {
      const obs = entry.resource as Observation;
      observationMap.set(`Observation/${obs.id}`, obs);
    }
  }

  // 3. Assemble the results: each report with its observations inlined
  return reports.map((report) => ({
    id: report.id,
    title: report.code?.text,
    date: report.effectiveDateTime,
    status: report.status,
    results: (report.result ?? []).map((ref) => {
      const obs = observationMap.get(ref.reference ?? '');
      return {
        name: obs?.code?.text,
        value: obs?.valueQuantity?.value,
        unit: obs?.valueQuantity?.unit,
        status: obs?.status,
      };
    }),
  }));
}
```

**Next:** [Chapter 6 — Updating & Patching →](./06-updating-and-patching.md)
