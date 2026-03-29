# Chapter 1: The Data Model Foundation

> **Key takeaway:** FHIR resources are the "tables" of healthcare, `@medplum/fhirtypes` gives you compile-time safety, and Medplum's type schema system powers runtime validation. Understanding this layer is what lets you move fast without breaking data.

---

## 1.1 What Is a FHIR Resource?

If you come from a relational database background, think of it this way:

| Relational DB Concept | FHIR Equivalent | Example |
|-----------------------|-----------------|---------|
| Table | Resource Type | `Patient`, `Observation`, `ServiceRequest` |
| Row | Resource Instance | A specific patient with `id: "homer-simpson"` |
| Column | Element / Field | `Patient.birthDate`, `Patient.name` |
| Foreign Key | Reference | `Observation.subject` → `Patient/homer-simpson` |
| JOIN | `_include` / `_revinclude` / Chained Search | Fetch Observations with their Patients |
| CHECK constraint | FHIRPath Constraint / Profile | `birthDate <= today()` |

Every resource has a standard shape defined by the FHIR specification. Medplum implements FHIR R4.

### The Universal Fields

Every single FHIR resource has these fields:

```typescript
{
  resourceType: 'Patient',           // Which "table" this belongs to
  id: 'homer-simpson',               // Server-assigned unique ID
  meta: {
    versionId: '3',                  // Incremented on every update
    lastUpdated: '2024-01-15T...',   // Timestamp of last change
    profile: ['http://...'],         // Profiles this resource conforms to
    tag: [...],                      // Arbitrary tags for categorization
    source: 'https://ehr.example/'   // Where this data originated
  }
}
```

---

## 1.2 TypeScript Types: `@medplum/fhirtypes`

Medplum generates TypeScript interfaces for every FHIR resource type. This is your primary tool for writing correct code.

```typescript
import type { Patient, Observation, ServiceRequest, Reference } from '@medplum/fhirtypes';

// The compiler catches mistakes at build time
const patient: Patient = {
  resourceType: 'Patient',
  name: [{ given: ['Homer'], family: 'Simpson' }],
  birthDate: '1956-05-12',
  gender: 'male',
  // gender: 'apache-helicopter'  // TS Error: not assignable to type 'male' | 'female' | ...
};
```

### Key Type Patterns

**`Reference<T>`** — the FHIR "foreign key":
```typescript
// Reference is generic — it knows what it points to
const observation: Observation = {
  resourceType: 'Observation',
  status: 'final',
  code: { text: 'Heart Rate' },
  subject: { reference: 'Patient/homer-simpson' } as Reference<Patient>,
};
```

**`WithId<T>`** — returned by the server after creation:
```typescript
import type { WithId } from '@medplum/core';

// Before creation: Patient (id is optional)
const draft: Patient = { resourceType: 'Patient' };

// After creation: WithId<Patient> (id is guaranteed)
const created: WithId<Patient> = await medplum.createResource(draft);
console.log(created.id); // Always defined — no need for null checks
```

**`CodeableConcept` and `Coding`** — structured medical codes:
```typescript
import type { CodeableConcept, Coding } from '@medplum/fhirtypes';

// A CodeableConcept can carry multiple codings from different systems
const bloodPressureCode: CodeableConcept = {
  coding: [
    {
      system: 'http://loinc.org',         // The code system (like a namespace)
      code: '85354-9',                     // The code itself
      display: 'Blood pressure panel',     // Human-readable label
    },
  ],
  text: 'Blood Pressure',                  // Fallback display text
};
```

---

## 1.3 The Internal Type Schema System

At runtime, Medplum uses an internal schema representation to validate resources, power search, and generate UIs. This lives in `packages/core/src/typeschema/`.

```
┌─────────────────────────────────────────────────────────┐
│                  StructureDefinition                     │
│  (FHIR's "CREATE TABLE" equivalent — defines the shape)  │
└────────────────────────┬────────────────────────────────┘
                         │ parsed by
                         ▼
┌─────────────────────────────────────────────────────────┐
│               InternalTypeSchema                         │
│                                                          │
│  name: "Patient"                                         │
│  elements: {                                             │
│    "name":      { min: 0, max: Infinity, type: [...] }   │
│    "birthDate": { min: 0, max: 1, type: [{ code: "date" }] } │
│    "gender":    { min: 0, max: 1, binding: { ... } }    │
│  }                                                       │
│  constraints: [                                          │
│    { key: "pat-1", expression: "..." }                   │
│  ]                                                       │
└─────────────────────────────────────────────────────────┘
```

### InternalSchemaElement — What Each Field Knows

From `packages/core/src/typeschema/types.ts`:

```typescript
interface InternalSchemaElement {
  description: string;
  path: string;            // e.g., "Patient.birthDate"
  min: number;             // Minimum cardinality (0 = optional, 1 = required)
  max: number;             // Maximum cardinality (1 = single, Infinity = array)
  isArray?: boolean;       // Shorthand for max > 1
  type: ElementType[];     // Allowed types (Reference, string, CodeableConcept, etc.)
  constraints?: Constraint[];  // FHIRPath expressions that must evaluate to true
  fixed?: TypedValue;      // Exact value required (like an enum with one option)
  pattern?: TypedValue;    // Partial value that must be present
  binding?: {              // Terminology binding (code must come from a ValueSet)
    strength: 'required' | 'extensible' | 'preferred' | 'example';
    valueSet: string;
  };
  slicing?: SlicingRules;  // Rules for constraining array elements (advanced)
}
```

### Business Use Case: Why This Matters

When you build a feature like "create a lab order," you don't need to manually validate that `ServiceRequest.status` is one of `draft | active | completed | ...`. The type schema knows:

1. **At compile time:** TypeScript interfaces prevent typos
2. **At runtime (client):** `validateResource()` checks against the schema
3. **At runtime (server):** The repo validation pipeline enforces it before writing to the database

---

## 1.4 Common Resource Types and When to Use Them

```
┌──────────────────────────────────────────────────────────────────┐
│                    CORE CLINICAL RESOURCES                        │
├──────────────────┬───────────────────────────────────────────────┤
│ Patient          │ The person receiving care                      │
│ Practitioner     │ A healthcare provider (doctor, nurse, etc.)    │
│ Organization     │ A hospital, clinic, lab, payer                 │
│ Encounter        │ A visit or interaction                         │
│ Observation      │ A measurement (vitals, lab results, etc.)      │
│ Condition        │ A diagnosis or problem                         │
│ MedicationRequest│ A prescription                                 │
│ ServiceRequest   │ An order for a service (lab test, imaging)     │
│ DiagnosticReport │ Results of a diagnostic study                  │
│ DocumentReference│ A pointer to a document (PDF, image, etc.)     │
├──────────────────┼───────────────────────────────────────────────┤
│                    WORKFLOW RESOURCES                              │
├──────────────────┼───────────────────────────────────────────────┤
│ Task             │ A unit of work to be done                      │
│ Communication    │ A message between participants                 │
│ Appointment      │ A scheduled meeting                            │
│ CarePlan         │ A plan of care for a patient                   │
├──────────────────┼───────────────────────────────────────────────┤
│                    INFRASTRUCTURE RESOURCES                        │
├──────────────────┼───────────────────────────────────────────────┤
│ Bundle           │ A collection of resources (batch, transaction) │
│ Subscription     │ Event-driven webhook trigger                   │
│ Bot              │ Serverless function (Medplum-specific)         │
│ AccessPolicy     │ Permissions and access control                 │
│ StructureDefinition │ Schema/profile definition                   │
│ ValueSet         │ A set of allowed codes                         │
└──────────────────┴───────────────────────────────────────────────┘
```

### Mapping Features to Resources

| Feature Requirement | Primary Resources | Why |
|---------------------|-------------------|-----|
| "Track patient demographics" | `Patient` | Core identity resource |
| "Record vital signs" | `Observation` → `Patient` | Observations link to patients via `subject` |
| "Order a lab test" | `ServiceRequest` → `Patient`, `Practitioner` | Order with subject and requester |
| "Store lab results" | `Observation`, `DiagnosticReport` → `ServiceRequest` | Results reference back to orders |
| "Manage appointments" | `Appointment` → `Patient`, `Practitioner` | Multi-participant scheduling |
| "Track care team tasks" | `Task` → `Patient`, `Practitioner` | Assignable, trackable work items |
| "Send a message" | `Communication` → `Patient`, `Practitioner` | Threaded messaging |
| "React to data changes" | `Subscription` → `Bot` | Event-driven automation |
| "Restrict data access" | `AccessPolicy` | RBAC and field-level control |

---

## 1.5 Loading Schemas in Your Application

### Client-Side Schema Loading

```typescript
import { MedplumClient } from '@medplum/core';

const medplum = new MedplumClient();

// Load the schema for a resource type (fetches StructureDefinition + SearchParameters)
await medplum.requestSchema('Patient');

// Load a custom profile schema
await medplum.requestProfileSchema('http://hl7.org/fhir/us/core/StructureDefinition/us-core-patient');
```

### Server-Side Schema Loading

The server loads all base FHIR schemas at startup from `@medplum/definitions`. When a resource declares `meta.profile`, the server fetches and caches the profile's StructureDefinition for validation.

---

## 1.6 Quick Reference: Key Imports

```typescript
// Types for FHIR resources
import type {
  Patient, Observation, ServiceRequest, DiagnosticReport,
  Bundle, Reference, CodeableConcept, Coding,
  OperationOutcome
} from '@medplum/fhirtypes';

// Client and utilities
import {
  MedplumClient,
  createReference,       // Resource → Reference
  getReferenceString,    // Resource | Reference → "Type/id"
  resolveId,             // Reference | Resource → id string
  parseReference,        // Reference → [ResourceType, id]
  getDisplayString,      // Resource → human-readable string
} from '@medplum/core';

// Type helper
import type { WithId } from '@medplum/core';
```

---

## Summary

| Concept | What It Is | Where It Lives |
|---------|-----------|----------------|
| FHIR Resource | The fundamental data unit (like a table row) | FHIR R4 specification |
| `@medplum/fhirtypes` | TypeScript interfaces for all resources | `packages/fhirtypes/` |
| `InternalTypeSchema` | Runtime schema representation | `packages/core/src/typeschema/types.ts` |
| `StructureDefinition` | FHIR's "CREATE TABLE" — defines resource shape | Loaded from `@medplum/definitions` |
| `Reference<T>` | Type-safe "foreign key" between resources | `@medplum/fhirtypes` |
| `WithId<T>` | Resource guaranteed to have an `id` | `@medplum/core` |

**Next:** [Chapter 2 — Creating Resources →](./02-creating-resources.md)
