# Chapter 7: Validation & Data Integrity

> **Key takeaway:** Medplum validates data at four layers — TypeScript types (compile time), client-side schema validation, the server validation pipeline, and access policies. Think of these as your CHECK constraints, NOT NULL constraints, foreign key checks, and row-level security — all working together to prevent bad data from reaching the database.

---

## 7.1 The Validation Layer Cake

```
  ┌───────────────────────────────────────────────────────────────┐
  │  Layer 1: TypeScript Types (Compile Time)                     │
  │  @medplum/fhirtypes                                           │
  │  Catches: wrong field names, invalid enum values, type errors │
  │  Analogy: IDE autocomplete that prevents typos                │
  ├───────────────────────────────────────────────────────────────┤
  │  Layer 2: Client-Side Validation (Runtime, Optional)          │
  │  validateResource() from @medplum/core                        │
  │  Catches: cardinality violations, regex failures,             │
  │           FHIRPath constraint failures                        │
  │  Analogy: Form validation before submit                       │
  ├───────────────────────────────────────────────────────────────┤
  │  Layer 3: Server Validation Pipeline (Runtime, Enforced)      │
  │  packages/server/src/fhir/repo.ts                             │
  │  Catches: everything from Layer 2, PLUS:                      │
  │           profile conformance, terminology bindings,          │
  │           reference existence, uniqueness                     │
  │  Analogy: Database CHECK constraints + FK constraints         │
  ├───────────────────────────────────────────────────────────────┤
  │  Layer 4: Access Policies & Write Constraints                 │
  │  AccessPolicy resource + FHIRPath write constraints           │
  │  Catches: unauthorized operations, forbidden state changes,   │
  │           field-level restrictions                            │
  │  Analogy: Row-level security + column-level permissions       │
  └───────────────────────────────────────────────────────────────┘
```

---

## 7.2 Layer 1: TypeScript Types

The first line of defense. Type errors are caught at build time.

```typescript
import type { Patient, Observation } from '@medplum/fhirtypes';

// ✅ Correct — TypeScript is happy
const patient: Patient = {
  resourceType: 'Patient',
  gender: 'male',                    // Must be 'male' | 'female' | 'other' | 'unknown'
  birthDate: '1990-03-15',           // string (FHIR date format)
  name: [{ given: ['Homer'], family: 'Simpson' }],
};

// ❌ Compile error — 'superhero' is not a valid gender
const bad: Patient = {
  resourceType: 'Patient',
  gender: 'superhero',              // TS2322: Type '"superhero"' is not assignable
};

// ❌ Compile error — status is required on Observation
const obs: Observation = {
  resourceType: 'Observation',
  // Missing 'status' and 'code' — both required
};
```

### Limitations of Type-Only Validation

TypeScript catches structural issues but not semantic ones:
- It won't enforce `birthDate <= today()` (FHIRPath constraint)
- It won't validate that a LOINC code is real (terminology binding)
- It won't check that a referenced resource exists (referential integrity)

---

## 7.3 Layer 2: Client-Side Validation

### Using `validateResource` on the Client

```typescript
import { validateResource } from '@medplum/core';

const patient: Patient = {
  resourceType: 'Patient',
  name: [{ given: ['Homer'], family: 'Simpson' }],
};

try {
  validateResource(patient);
  console.log('Resource is valid');
} catch (err) {
  // OperationOutcomeError with details about what's wrong
  console.error(err.outcome.issue);
}
```

### What Client-Side Validation Checks

From `packages/core/src/typeschema/validation.ts`:

```
  validateResource(resource)
           │
           ├── 1. Null/undefined check
           │      Scans for null values in the resource tree
           │      (null is almost never valid in FHIR)
           │
           ├── 2. Cardinality check
           │      min/max bounds for each element
           │      e.g., Observation.status has min:1 (required)
           │
           ├── 3. Type validation
           │      Regex patterns for primitives:
           │      ┌──────────┬────────────────────────┐
           │      │ date     │ YYYY(-MM(-DD)?)?        │
           │      │ dateTime │ Full ISO 8601           │
           │      │ id       │ [A-Za-z0-9\-.]{1,64}   │
           │      │ code     │ No leading/trailing ws  │
           │      │ uri      │ Non-whitespace          │
           │      │ string   │ Max 1MB, not all ws     │
           │      └──────────┴────────────────────────┘
           │
           ├── 4. Fixed/Pattern value check
           │      Some elements must match exact values
           │      (fixed) or contain specific substructures (pattern)
           │
           ├── 5. FHIRPath constraint evaluation
           │      Executes constraint expressions like:
           │      "name.exists() or telecom.exists()"
           │
           ├── 6. Reference type check
           │      Validates reference targets match allowed types
           │      e.g., Observation.subject can only point to Patient,
           │      Group, Device, or Location
           │
           ├── 7. Additional properties check
           │      Detects unknown fields not in the schema
           │
           └── 8. Required binding check
                  Collects coded values with binding.strength='required'
                  for later terminology validation
```

### Validation with Profiles

```typescript
import { validateResource, loadDataType } from '@medplum/core';

// Load a custom profile
const profileSd = await medplum.readResource('StructureDefinition', 'my-profile-id');
loadDataType(profileSd);

// Validate against the profile
validateResource(patient, { profile: profileSd });
```

---

## 7.4 Layer 3: Server Validation Pipeline

This is the enforced gate. No data reaches the database without passing through this pipeline.

### Full Server Validation Flow

From `packages/server/src/fhir/repo.ts`:

```
  POST /Patient (or PUT /Patient/123)
           │
           ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 1. checkResourcePermissions()                                │
  │    Does the user's AccessPolicy allow this operation?        │
  │    → 403 Forbidden if not                                    │
  └────────────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 2. preCommitValidation()                                     │
  │    Execute pre-commit Bots (if configured)                   │
  │    Bot can:                                                  │
  │    - Modify the resource (add defaults, normalize data)      │
  │    - Throw an error to reject the write                      │
  └────────────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 3. rewriteAttachments()                                      │
  │    Convert inline attachments to Binary references           │
  └────────────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 4. replaceConditionalReferences()                            │
  │    Resolve "Patient?identifier=MRN|001" → "Patient/abc-123" │
  └────────────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 5. restoreReadonlyFields()                                   │
  │    Protect server-managed fields: id, meta.versionId,        │
  │    meta.lastUpdated, meta.author                             │
  └────────────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 6. validateResource() — MAIN VALIDATION                      │
  │                                                              │
  │    Strict Mode:                                              │
  │    ├── Validate against base StructureDefinition             │
  │    ├── Validate against all meta.profile[] profiles          │
  │    └── Validate terminology bindings (if enabled)            │
  │                                                              │
  │    Non-Strict Mode:                                          │
  │    ├── JSON Schema validation (always enforced)              │
  │    └── StructureDefinition validation (logged as warning)    │
  └────────────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 7. validateResourceReferences() (if checkReferencesOnWrite)  │
  │    For every reference in the resource:                      │
  │    - Does the target resource exist?                         │
  │    - Does the user have access to it?                        │
  │    → 400 if reference is invalid                             │
  └────────────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ 8. Write to database + Index search parameters               │
  └─────────────────────────────────────────────────────────────┘
```

### Strict vs. Non-Strict Mode

| Feature | Strict Mode | Non-Strict Mode |
|---------|------------|-----------------|
| Base FHIR validation | Error → reject | Warning → log |
| Profile validation | Error → reject | Not enforced |
| Terminology validation | Optional (if enabled) | Not enforced |
| JSON Schema validation | Always enforced | Always enforced |
| Best for | Production, compliance | Migration, prototyping |

---

## 7.5 Profiles: Custom "Table Schemas"

Profiles are like subclasses of FHIR resources — they add constraints on top of the base definition.

### Example: A Profile That Requires Birth Date

```
  Base Patient                    Patient Profile
  ┌───────────────────────┐       ┌───────────────────────┐
  │ name:     0..*        │       │ name:     1..* (MS)    │  ← At least one name
  │ birthDate: 0..1       │  ───► │ birthDate: 1..1 (MS)   │  ← Now required
  │ gender:   0..1        │       │ gender:   0..1         │
  │ identifier: 0..*      │       │ identifier: 1..* (MS)  │  ← At least one identifier
  └───────────────────────┘       └───────────────────────┘
```

### Creating a Profile with FHIR Shorthand (FSH)

```
Profile: MyClinicPatient
Parent: Patient
Id: my-clinic-patient
Title: "My Clinic Patient Profile"
Description: "Patient must have name, birthdate, and MRN"

* name 1..* MS
* birthDate 1..1 MS
* identifier 1..* MS
* identifier ^slicing.discriminator.type = #pattern
* identifier ^slicing.discriminator.path = "system"
* identifier ^slicing.rules = #open
* identifier contains mrn 1..1 MS
* identifier[mrn].system = "https://my-clinic.example/mrn"
```

### Using a Profile

```typescript
// Tag the resource with the profile
const patient: Patient = {
  resourceType: 'Patient',
  meta: {
    profile: ['https://my-clinic.example/StructureDefinition/my-clinic-patient'],
  },
  name: [{ given: ['Homer'], family: 'Simpson' }],
  birthDate: '1956-05-12',
  identifier: [
    { system: 'https://my-clinic.example/mrn', value: 'MRN-001' },
  ],
};

// On write, the server validates against both:
// 1. Base Patient StructureDefinition
// 2. my-clinic-patient profile
const created = await medplum.createResource(patient);
```

### Business Use Case: Enforce US Core Compliance

```typescript
// US Core Patient requires: identifier, name, gender
const usCorePat: Patient = {
  resourceType: 'Patient',
  meta: {
    profile: ['http://hl7.org/fhir/us/core/StructureDefinition/us-core-patient'],
  },
  identifier: [{ system: 'http://hospital.example/mrn', value: '12345' }],
  name: [{ given: ['Jane'], family: 'Doe' }],
  gender: 'female',
};
```

---

## 7.6 FHIRPath Constraints: The CHECK Constraint Equivalent

FHIRPath constraints are expressions that must evaluate to `true` for a resource to be valid. They're defined in StructureDefinitions and profiles.

### How Constraints Are Evaluated

From `packages/core/src/typeschema/validation.ts`:

```typescript
// Constraint structure
interface Constraint {
  key: string;          // e.g., "obs-7"
  severity: 'error' | 'warning';
  expression: string;   // FHIRPath expression
  description: string;  // Human-readable description
}

// Example constraints from the FHIR spec:

// obs-7: "If Observation.code is the same as an Observation.component.code
//         then the value element associated with the code SHALL NOT be present"
// expression: "value.empty() or component.code.where(coding.intersect(%resource.code.coding).exists()).empty()"

// obs-6: "dataAbsentReason SHALL only be present if Observation.value[x] is not present"
// expression: "dataAbsentReason.empty() or value.empty()"
```

### Available Variables in Constraints

| Variable | What It Is |
|----------|-----------|
| `%context` | The current element being validated |
| `%resource` | The resource being validated |
| `%rootResource` | The top-level resource (for contained resources) |
| `%ucum` | The UCUM code system URL |

---

## 7.7 Layer 4: Access Policies & Write Constraints

Access policies control WHO can do WHAT to WHICH resources.

### Basic CRUD Permissions

```json
{
  "resourceType": "AccessPolicy",
  "name": "Nurse Access",
  "resource": [
    {
      "resourceType": "Patient",
      "interaction": ["read", "search"]
    },
    {
      "resourceType": "Observation",
      "interaction": ["create", "read", "search"]
    },
    {
      "resourceType": "Task",
      "interaction": ["create", "read", "update", "search"]
    }
  ]
}
```

### Read-Only Fields

```json
{
  "resourceType": "AccessPolicy",
  "name": "Intake Clerk",
  "resource": [
    {
      "resourceType": "Patient",
      "readonlyFields": ["deceased", "managingOrganization"]
    }
  ]
}
```

### Hidden Fields

```json
{
  "resourceType": "AccessPolicy",
  "name": "Limited View",
  "resource": [
    {
      "resourceType": "Patient",
      "hiddenFields": ["identifier", "address", "telecom"]
    }
  ]
}
```

### Write Constraints: FHIRPath-Based Business Rules

Write constraints are expressions evaluated before a write is allowed. They have access to both the previous and new state.

```json
{
  "resourceType": "AccessPolicy",
  "name": "No Editing Final Reports",
  "resource": [
    {
      "resourceType": "DiagnosticReport",
      "writeConstraint": [
        {
          "language": "text/fhirpath",
          "expression": "%before.exists() implies %before.status != 'final'"
        }
      ]
    }
  ]
}
```

**Variables:**
- `%before` — the resource state before the update (undefined for creates)
- `%after` — the resource state after the update

### Business Use Cases for Write Constraints

```json
// Prevent reopening completed tasks
{
  "expression": "%before.exists() implies %before.status != 'completed'"
}

// Only allow status changes (not other field edits) on active orders
{
  "expression": "%before.status = 'active' implies (%before.code = %after.code and %before.subject = %after.subject)"
}

// Require a reason when cancelling
{
  "expression": "%after.status = 'cancelled' implies %after.statusReason.exists()"
}
```

### Criteria-Based Access: Row-Level Security

```json
{
  "resourceType": "AccessPolicy",
  "name": "California Patients Only",
  "resource": [
    {
      "resourceType": "Patient",
      "criteria": "Patient?address-state=CA"
    },
    {
      "resourceType": "Observation",
      "criteria": "Observation?subject._compartment=Patient?address-state=CA"
    }
  ]
}
```

---

## 7.8 Bot-Based Validation: Custom Business Logic

For validation rules that are too complex for FHIRPath or need external data:

```typescript
// Bot: validate-service-request.ts
import { BotEvent, MedplumClient } from '@medplum/core';
import type { ServiceRequest } from '@medplum/fhirtypes';

export async function handler(medplum: MedplumClient, event: BotEvent): Promise<any> {
  const serviceRequest = event.input as ServiceRequest;

  // Custom validation: verify the ordering provider is licensed
  const practitioner = await medplum.readReference(serviceRequest.requester!);

  if (!practitioner.qualification?.some((q) =>
    q.code?.coding?.some((c) => c.system === 'http://example.com/licenses' && c.code === 'active')
  )) {
    throw new Error('Ordering provider does not have an active license');
  }

  // Custom validation: check insurance eligibility (external API)
  const patient = await medplum.readReference(serviceRequest.subject!);
  const coverage = await medplum.searchOne('Coverage', {
    beneficiary: `Patient/${patient.id}`,
    status: 'active',
  });

  if (!coverage) {
    throw new Error('Patient does not have active insurance coverage');
  }

  // Return the (possibly modified) resource
  return serviceRequest;
}
```

### Configuring a Pre-Commit Bot

The bot runs BEFORE validation and can modify the resource:

```typescript
// Attach as a pre-commit subscription
const subscription = await medplum.createResource({
  resourceType: 'Subscription',
  status: 'active',
  criteria: 'ServiceRequest',
  channel: {
    type: 'rest-hook',
    endpoint: 'Bot/validate-service-request-bot-id',
  },
  extension: [
    {
      url: 'https://medplum.com/fhir/StructureDefinition/subscription-supported-interaction',
      valueCode: 'create',
    },
  ],
});
```

---

## 7.9 Terminology Validation

Ensures coded values come from the correct code systems.

### Binding Strengths

| Strength | Server Behavior | When to Use |
|----------|----------------|-------------|
| `required` | Rejects invalid codes (if `validateTerminology: true`) | Status fields, administrative codes |
| `extensible` | Warning only | Clinical codes with known sets |
| `preferred` | No enforcement | Suggested but flexible |
| `example` | No enforcement | Illustrative only |

### How Terminology Validation Works

```
  Resource with coded field
  (e.g., Observation.code with LOINC binding)
           │
           ▼
  ┌─────────────────────┐
  │  binding.strength    │
  │  == 'required'?      │
  └──────────┬──────────┘
        ┌────┴────┐
       Yes        No → Skip
        │
        ▼
  ┌─────────────────────┐
  │  Look up ValueSet    │   Fetch the ValueSet by URL
  │  by binding URL      │   from the database
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │  Check if code is    │   Is the coding.system + coding.code
  │  in the ValueSet     │   a member of this ValueSet?
  └──────────┬──────────┘
        ┌────┴────┐
       Yes        No
        │          │
      Pass     Create OperationOutcome issue
               "Code 'xyz' not found in ValueSet"
```

---

## 7.10 `$validate` Operation: Test Before You Write

Validate a resource without actually creating it:

```typescript
const outcome = await medplum.validateResource({
  resourceType: 'Patient',
  name: [{ given: ['Homer'], family: 'Simpson' }],
  birthDate: '1956-05-12',
});

// outcome is an OperationOutcome
if (outcome.issue?.some((i) => i.severity === 'error')) {
  console.error('Validation failed:', outcome.issue);
} else {
  console.log('Resource is valid');
}
```

### Business Use Case: Form Validation Before Submit

```typescript
async function validatePatientForm(formData: Partial<Patient>): Promise<string[]> {
  const resource: Patient = {
    resourceType: 'Patient',
    ...formData,
    meta: {
      profile: ['https://my-clinic.example/StructureDefinition/my-clinic-patient'],
    },
  };

  const outcome = await medplum.validateResource(resource);

  return (outcome.issue ?? [])
    .filter((i) => i.severity === 'error')
    .map((i) => `${i.expression?.join('.')}: ${i.diagnostics}`);
}
```

---

## Summary: Validation Quick Reference

| Layer | Where | What It Catches | When |
|-------|-------|-----------------|------|
| TypeScript types | Build time | Wrong fields, bad enums, missing required fields | `tsc` / IDE |
| `validateResource()` (client) | Client runtime | Cardinality, regex, FHIRPath constraints | Before submit |
| Server validation pipeline | Server runtime | All of the above + profiles + terminology + references | On every write |
| Access policies | Server runtime | Unauthorized operations, forbidden fields, state transitions | On every operation |
| Pre-commit bots | Server runtime | Custom business logic, external checks | On create/update |
| `$validate` operation | On demand | Full server validation without writing | Testing / forms |

**Next:** [Chapter 8 — Putting It All Together →](./08-putting-it-all-together.md)
