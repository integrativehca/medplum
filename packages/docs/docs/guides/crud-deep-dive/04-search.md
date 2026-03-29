# Chapter 4: Search — The Query Engine

> **Key takeaway:** FHIR search is not arbitrary SQL — it uses predefined search parameters. Medplum gives you four search methods, powerful modifiers, chained searches (FHIR's version of JOINs), and pagination strategies. Mastering search is the difference between making one API call and making twenty.

---

## 4.1 Search Methods Overview

```
                    What do you need from the search?
                              │
          ┌───────────┬───────┴───────┬──────────────┐
          │           │               │              │
     Full Bundle   Just the      First match    All results
     (metadata +   resources     only?          (paginated)?
      entries)?    as an array?       │              │
          │           │               │              │
      search()   searchResources() searchOne()  searchResourcePages()
          │           │               │              │
    Bundle<T>    ResourceArray<T>  T | undefined  AsyncGenerator<T[]>
```

### The Four Search Methods

```typescript
// 1. Full Bundle — includes total, links, metadata
const bundle = await medplum.search('Patient', 'name=Simpson');
console.log(bundle.total);                    // Total matching count
console.log(bundle.entry?.[0]?.resource);     // First resource
console.log(bundle.link);                     // Pagination links

// 2. Resources array — most common for lists
const patients = await medplum.searchResources('Patient', 'name=Simpson');
console.log(patients.length);                 // Array of Patient resources
console.log(patients.bundle.total);           // Still have bundle metadata

// 3. First match only — for lookups
const patient = await medplum.searchOne('Patient', 'identifier=MRN-001');
// Returns Patient | undefined (auto-sets _count=1)

// 4. Paginated iteration — for large datasets
for await (const page of medplum.searchResourcePages('Patient', { _count: 100 })) {
  for (const patient of page) {
    await processPatient(patient);
  }
}
```

---

## 4.2 Search Parameters: FHIR's Column Indexes

Unlike SQL where you can query any column, FHIR defines specific search parameters for each resource type. Think of them as pre-built indexes.

### Common Patient Search Parameters

| Parameter | Type | Maps To | Example |
|-----------|------|---------|---------|
| `name` | string | `Patient.name` | `name=Simpson` |
| `family` | string | `Patient.name.family` | `family=Simpson` |
| `given` | string | `Patient.name.given` | `given=Homer` |
| `birthdate` | date | `Patient.birthDate` | `birthdate=1956-05-12` |
| `gender` | token | `Patient.gender` | `gender=male` |
| `identifier` | token | `Patient.identifier` | `identifier=MRN\|123` |
| `address-city` | string | `Patient.address.city` | `address-city=Springfield` |
| `general-practitioner` | reference | `Patient.generalPractitioner` | `general-practitioner=Practitioner/123` |
| `organization` | reference | `Patient.managingOrganization` | `organization=Organization/456` |

### Search Parameter Types

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    SEARCH PARAMETER TYPES                       │
  ├──────────┬──────────────────────────────────────────────────────┤
  │ string   │ Case-insensitive prefix match                       │
  │          │ "Sim" matches "Simpson", "Simmons"                   │
  │          │ Modifiers: :contains, :exact                         │
  ├──────────┼──────────────────────────────────────────────────────┤
  │ token    │ Case-sensitive exact match on code + optional system │
  │          │ "http://loinc.org|8867-4" matches exact LOINC code   │
  │          │ Modifiers: :not, :text, :in, :not-in, :above, :below│
  ├──────────┼──────────────────────────────────────────────────────┤
  │ reference│ Points to another resource                           │
  │          │ "Patient/123" or just "123"                          │
  ├──────────┼──────────────────────────────────────────────────────┤
  │ date     │ Dates with comparison operators                      │
  │          │ "ge2024-01-01" = on or after Jan 1, 2024             │
  │          │ Prefixes: eq, ne, gt, lt, ge, le, sa, eb             │
  ├──────────┼──────────────────────────────────────────────────────┤
  │ number   │ Numeric values with comparison operators             │
  │          │ Same prefixes as date                                │
  ├──────────┼──────────────────────────────────────────────────────┤
  │ quantity │ Number + unit + system                               │
  │          │ "gt5.4|http://unitsofmeasure.org|mg"                 │
  └──────────┴──────────────────────────────────────────────────────┘
```

---

## 4.3 Query Syntax: AND, OR, and Operators

### AND Logic (Multiple Parameters)

```typescript
// Find male patients named Simpson
await medplum.searchResources('Patient', {
  name: 'Simpson',
  gender: 'male',
});
// Equivalent URL: GET /Patient?name=Simpson&gender=male
```

### OR Logic (Comma-Separated Values)

```typescript
// Find tasks that are completed OR cancelled
await medplum.searchResources('Task', {
  status: 'completed,cancelled',
});
// Equivalent URL: GET /Task?status=completed,cancelled
```

### Comparison Operators (Date, Number, Quantity)

```typescript
// Patients born after 1990
await medplum.searchResources('Patient', { birthdate: 'gt1990-01-01' });

// Observations with value between 40 and 60
await medplum.searchResources('Observation', [
  ['value-quantity', 'gt40'],
  ['value-quantity', 'lt60'],
]);

// Appointments in the next 7 days
const nextWeek = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString();
await medplum.searchResources('Appointment', {
  date: `le${nextWeek}`,
});
```

---

## 4.4 Modifiers: Fine-Tuning String and Token Searches

```typescript
// :contains — substring match (string parameters)
await medplum.searchResources('Patient', { 'name:contains': 'omer' });
// Matches: Homer, Gomer, etc.

// :exact — case-sensitive exact match
await medplum.searchResources('Patient', { 'name:exact': 'Homer' });

// :not — exclude values (token parameters)
await medplum.searchResources('Task', { 'status:not': 'completed' });

// :missing — filter by presence/absence
await medplum.searchResources('Patient', { 'birthdate:missing': 'true' });
// Find patients WITHOUT a birth date

// :text — full-text search on coded fields
await medplum.searchResources('Observation', { 'code:text': 'blood pressure' });

// :in — check against a ValueSet
await medplum.searchResources('Condition', {
  'code:in': 'http://hl7.org/fhir/ValueSet/condition-severity',
});
```

---

## 4.5 Special Parameters

### Sorting

```typescript
// Sort by family name (ascending)
await medplum.searchResources('Patient', { _sort: 'family' });

// Sort by date (descending — note the minus prefix)
await medplum.searchResources('Observation', { _sort: '-date' });

// Multi-field sort
await medplum.searchResources('Patient', { _sort: 'family,-birthdate' });
```

### Selecting Fields

```typescript
// Only return id, name, and birthDate (reduces payload)
await medplum.searchResources('Patient', {
  name: 'Simpson',
  _elements: 'id,name,birthDate',
});

// Summary mode (FHIR-defined subset of fields)
await medplum.search('Patient', { _summary: 'true' });

// Count only (no resources returned)
await medplum.search('Patient', { name: 'Simpson', _summary: 'count' });
```

### Getting Total Count

```typescript
// Accurate total (may be slower for large datasets)
const result = await medplum.search('Patient', {
  name: 'Simpson',
  _total: 'accurate',
});
console.log(`Found ${result.total} patients`);

// Estimate total (faster)
const estimated = await medplum.search('Patient', {
  name: 'Simpson',
  _total: 'estimate',
});
```

### Filtering by Metadata

```typescript
// By last updated time
await medplum.searchResources('Patient', { _lastUpdated: 'gt2024-01-01' });

// By tag
await medplum.searchResources('Patient', { _tag: 'http://example.com|vip' });

// By profile conformance
await medplum.searchResources('Patient', {
  _profile: 'http://hl7.org/fhir/us/core/StructureDefinition/us-core-patient',
});

// By source system
await medplum.searchResources('Patient', { _source: 'https://ehr.example.com' });
```

---

## 4.6 Chained Search: FHIR's JOIN

Chained search lets you filter resources based on properties of *referenced* resources — like a JOIN in SQL.

### Forward Chaining

```
  SQL equivalent:
    SELECT o.* FROM Observation o
    JOIN Patient p ON o.subject = p.id
    WHERE p.name LIKE 'Homer%'

  FHIR chained search:
    GET /Observation?subject:Patient.name=Homer
```

```typescript
// Find observations for patients named Homer
await medplum.searchResources('Observation', {
  'subject:Patient.name': 'Homer',
});

// Find encounters at a specific organization
await medplum.searchResources('Encounter', {
  'service-provider.name': 'Springfield General',
});

// Multi-level chain: Observations → Encounter → Organization
await medplum.searchResources('Observation', {
  'encounter:Encounter.service-provider.name': 'Kaiser',
});
```

### Reverse Chaining (`_has`)

"Find resources that are *referenced by* other resources matching a condition."

```
  SQL equivalent:
    SELECT p.* FROM Patient p
    WHERE EXISTS (
      SELECT 1 FROM Observation o
      WHERE o.subject = p.id
        AND o.code = '8867-4'
        AND o.value > 150
    )

  FHIR reverse chain:
    GET /Patient?_has:Observation:subject:code=8867-4
                &_has:Observation:subject:value-quantity=gt150
```

```typescript
// Find patients who have high heart rate observations
await medplum.searchResources('Patient', {
  '_has:Observation:subject:code': '8867-4',
  '_has:Observation:subject:value-quantity': 'gt150',
});

// Find practitioners who have upcoming appointments
await medplum.searchResources('Practitioner', {
  '_has:Appointment:actor:date': `ge${new Date().toISOString()}`,
});
```

---

## 4.7 The `_filter` Parameter: Complex Query Logic

For queries that go beyond AND/OR on individual parameters.

```typescript
// Complex filter: Male patients named Simpson or Flanders
await medplum.searchResources('Patient', {
  _filter: 'gender eq "male" and (name co "Simpson" or name co "Flanders")',
});

// Observations with specific value ranges
await medplum.searchResources('Observation', {
  _filter: 'value-quantity gt 100 and value-quantity lt 200',
});
```

### Filter Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `eq` | Equals | `gender eq "male"` |
| `ne` | Not equals | `status ne "cancelled"` |
| `co` | Contains | `name co "simp"` |
| `gt`, `lt`, `ge`, `le` | Comparison | `birthdate gt "1990-01-01"` |
| `sa` | Starts after | `date sa "2024-01-01"` |
| `eb` | Ends before | `date eb "2024-12-31"` |
| `pr` | Exists (present) | `email pr true` |
| `in` | In ValueSet | `code in "http://..."` |

---

## 4.8 Pagination Strategies

### Offset-Based Pagination

```typescript
// Page 1 (items 0-19)
await medplum.searchResources('Patient', { _count: '20', _offset: '0' });

// Page 2 (items 20-39)
await medplum.searchResources('Patient', { _count: '20', _offset: '20' });
```

**Limitation:** Offset-based pagination is limited to 10,000 results. For larger datasets, use cursor-based pagination.

### Cursor-Based Pagination

More efficient for large datasets. The cursor token is returned in the Bundle's `link` array.

```typescript
// First page
const firstPage = await medplum.search('Patient', {
  _count: '100',
  _sort: '_lastUpdated',  // Required for cursor-based pagination
});

// Get the cursor for the next page
const nextLink = firstPage.link?.find((l) => l.relation === 'next');
if (nextLink) {
  const nextPage = await medplum.search('Patient', nextLink.url);
}
```

### Async Generator: `searchResourcePages`

The easiest way to iterate through all results:

```typescript
// Automatically follows pagination links
let total = 0;
for await (const page of medplum.searchResourcePages('Patient', { _count: 100 })) {
  for (const patient of page) {
    total++;
    await processPatient(patient);
  }
}
console.log(`Processed ${total} patients`);
```

---

## 4.9 The Search Infrastructure: How It Works Internally

```
  medplum.searchResources('Observation', { subject: 'Patient/123', code: '8867-4' })
           │
           ▼
  ┌──────────────────────────────────┐
  │  parseSearchRequest()            │   packages/core/src/search/search.ts
  │  URL → SearchRequest object      │
  │  {                               │
  │    resourceType: 'Observation',  │
  │    filters: [                    │
  │      { code: 'subject',         │
  │        operator: 'EQUALS',      │
  │        value: 'Patient/123' },  │
  │      { code: 'code',            │
  │        operator: 'EQUALS',      │
  │        value: '8867-4' }        │
  │    ]                             │
  │  }                               │
  └──────────┬───────────────────────┘
             │
             ▼
  ┌──────────────────────────────────┐
  │  Server: Build SQL Query         │   packages/server/src/fhir/search.ts
  │                                  │
  │  Lookup tables consulted:        │
  │  ┌────────────────────────────┐  │
  │  │ Observation table          │  │   Main resource table
  │  │ Observation_References     │  │   Reference lookup table
  │  │ Observation_Tokens         │  │   Token (code) lookup table
  │  └────────────────────────────┘  │
  └──────────┬───────────────────────┘
             │
             ▼
  ┌──────────────────────────────────┐
  │  PostgreSQL executes query       │
  │  Returns matching resource IDs   │
  └──────────┬───────────────────────┘
             │
             ▼
  ┌──────────────────────────────────┐
  │  Build FHIR Bundle response      │
  │  - entry[] with resources        │
  │  - link[] for pagination         │
  │  - total count (if requested)    │
  └──────────────────────────────────┘
```

### Server-Side Lookup Tables

The server maintains denormalized lookup tables for efficient search. From `packages/server/src/fhir/lookups/`:

| Table | Indexes | Used For |
|-------|---------|----------|
| `HumanName` | Given name, family name | `name`, `given`, `family` searches |
| `Address` | City, state, postal code, country | `address-*` searches |
| `Reference` | Target resource type + ID | Reference parameter searches |
| `Coding` | System + code | Token parameter searches |

These are automatically populated when resources are written — you don't need to manage them.

---

## 4.10 Business Use Cases

### Lab Dashboard: Find Pending Results

```typescript
// Find all active service requests for a patient that need results
const pendingOrders = await medplum.searchResources('ServiceRequest', {
  subject: `Patient/${patientId}`,
  status: 'active',
  _sort: '-authored-on',
  _count: '50',
});
```

### Patient Search Bar

```typescript
// Flexible patient search by name, MRN, or phone
async function searchPatients(query: string) {
  // Try as identifier first (MRN lookup)
  const byMrn = await medplum.searchResources('Patient', {
    identifier: query,
    _count: '10',
  });
  if (byMrn.length > 0) return byMrn;

  // Fall back to name search
  return medplum.searchResources('Patient', {
    name: query,
    _count: '10',
    _sort: 'family',
  });
}
```

### Activity Feed: Recent Changes Across Resource Types

```typescript
// Get recently updated observations and conditions for a patient
const [recentObs, recentConditions] = await Promise.all([
  medplum.searchResources('Observation', {
    subject: `Patient/${patientId}`,
    _sort: '-_lastUpdated',
    _count: '10',
  }),
  medplum.searchResources('Condition', {
    subject: `Patient/${patientId}`,
    _sort: '-_lastUpdated',
    _count: '10',
  }),
]);
```

---

## Summary: Search Quick Reference

| Feature | Syntax | Example |
|---------|--------|---------|
| Basic search | `param=value` | `name=Simpson` |
| AND | Multiple params | `name=Simpson&gender=male` |
| OR | Comma-separated | `status=active,draft` |
| Comparison | Prefix on value | `birthdate=gt1990-01-01` |
| Contains | `:contains` modifier | `name:contains=omer` |
| Exact match | `:exact` modifier | `name:exact=Homer` |
| Exclude | `:not` modifier | `status:not=cancelled` |
| Missing field | `:missing` modifier | `birthdate:missing=true` |
| Forward chain | `ref:Type.param` | `subject:Patient.name=Homer` |
| Reverse chain | `_has:Type:ref:param` | `_has:Observation:subject:code=8867-4` |
| Include refs | `_include` | `_include=Observation:subject` |
| Reverse include | `_revinclude` | `_revinclude=Observation:subject` |
| Sort | `_sort` | `_sort=-date,name` |
| Pagination | `_count` + `_offset` | `_count=20&_offset=40` |
| Total count | `_total` | `_total=accurate` |
| Complex filter | `_filter` | `_filter=name co "sim" and gender eq "male"` |

**Next:** [Chapter 5 — References & Relationships →](./05-references-and-relationships.md)
