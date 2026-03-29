# CRUD Deep Dive: Mastering Data Operations with Medplum

> **Audience:** Senior engineers seeking depth, and junior developers new to healthcare who want to build a strong foundation.
>
> **Goal:** Understand how to leverage the Medplum core library to map feature requirements to the right modules and build faster — without reinventing the wheel.

---

## How This Guide Is Organized

Each chapter builds on the previous, moving from foundational concepts to advanced patterns you'll use in production.

```
Chapter 1: The Data Model Foundation
    │   FHIR resource types, TypeScript types, and how Medplum models healthcare data
    ▼
Chapter 2: Creating Resources
    │   createResource, createResourceIfNoneExist, upsert, and write patterns
    ▼
Chapter 3: Reading & Retrieving Data
    │   readResource, readReference, history, version reads, caching
    ▼
Chapter 4: Search — The Query Engine
    │   Search parameters, modifiers, chaining, _include/_revinclude, pagination
    ▼
Chapter 5: References & Relationships
    │   How FHIR "joins" work: references, compartments, graphs, GraphQL
    ▼
Chapter 6: Updating & Patching
    │   updateResource, patchResource, optimistic locking, batch/transaction
    ▼
Chapter 7: Validation & Data Integrity
    │   Schema validation, FHIRPath constraints, profiles, terminology binding,
    │   access policies, write constraints, bot-based validation
    ▼
Chapter 8: Putting It All Together
        End-to-end workflows, performance patterns, and decision framework
```

---

## Chapters

| # | Chapter | What You'll Learn |
|---|---------|-------------------|
| 1 | [The Data Model Foundation](./01-data-model-foundation.md) | FHIR resources, TypeScript types, how `@medplum/fhirtypes` maps to real data |
| 2 | [Creating Resources](./02-creating-resources.md) | All the ways to write data — simple creates, conditional creates, upserts, and batch inserts |
| 3 | [Reading & Retrieving Data](./03-reading-data.md) | Reading by ID, by reference, version history, caching strategies |
| 4 | [Search — The Query Engine](./04-search.md) | Search parameters, operators, chaining, pagination, and the `_filter` parameter |
| 5 | [References & Relationships](./05-references-and-relationships.md) | How FHIR handles "joins" — references, `_include`, `_revinclude`, compartments, GraphQL |
| 6 | [Updating & Patching](./06-updating-and-patching.md) | Full updates, JSON Patch, optimistic locking, transactions |
| 7 | [Validation & Data Integrity](./07-validation-and-integrity.md) | Schema validation, profiles, constraints, access policies, bot-based rules |
| 8 | [Putting It All Together](./08-putting-it-all-together.md) | Real-world workflows, performance patterns, and a decision-making framework |

---

## Key Packages Referenced

| Package | Purpose | When You'll Use It |
|---------|---------|-------------------|
| `@medplum/core` | Client library, utilities, validation, search parsing | Every chapter |
| `@medplum/fhirtypes` | TypeScript type definitions for all FHIR resources | Every chapter |
| `@medplum/server` | Backend FHIR server (repo, validation pipeline, lookup tables) | Understanding server-side behavior |
| `@medplum/mock` | Mock FHIR server for testing | Writing tests |
| `@medplum/react` | React components for FHIR UIs | Building frontends |

---

## Prerequisites

- Basic TypeScript / JavaScript experience
- A running Medplum instance (local or cloud) — or use `@medplum/mock` for testing
- No prior healthcare / FHIR knowledge required (Chapter 1 covers the essentials)
