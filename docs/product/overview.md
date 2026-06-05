# SAP Public Cloud — Cross-Module ABAP Cloud Skeleton

> Status: **Skeleton (lane: tiny)** — initial project setup only.
> No domain schema, no CRUD, no auth, no data migration, no provider
> integration in this intake.

## Scope

This repo is a cross-module ABAP Cloud workspace for **SAP S/4HANA Cloud Public
Edition** (and SAP BTP, ABAP Environment — "Steampunk"). It is intentionally
generic: domain files (FI, MM, SD, PP, HR, BW, …) are **not** created up front.
Per `docs/product/README.md`, empty structure is healthier than fake product
truth. Domains are added only when a real spec or change request lands.

## Target Runtime

| Layer | Choice |
| --- | --- |
| Platform | SAP S/4HANA Cloud Public Edition / SAP BTP, ABAP Environment |
| Language | ABAP Cloud (ABAP for SAP Cloud, release contract ≥ 7.58 / 2023) |
| Programming model | ABAP OO, ABAP Clean Code, Released Objects API only |
| Data access | CDS View Entities (DDL source), ABAP SQL |
| Service exposure | OData via RAP (ABAP RESTful Application Programming Model) — only when a real story needs it |
| Transport | ADT (ABAP Development Tools) in Eclipse / VS Code; `abapGit` for offline-friendly serialization |
| Quality | ATC (ABAP Test Cockpit), unit tests with ABAP Unit, integration via test seams |

## Hard Constraints (released-objects only)

The following are **not allowed** in this project because they are not
released for SAP Public Cloud / Steampunk:

- Dynpro / Screen Painter (classic dialog programming).
- ALV with classic function modules (`REUSE_ALV_*`). Use SAP GUI ALV
  **only** in released contexts, or use Fiori Elements instead.
- `SMARTFORMS`, `SAPscript` (use Adobe Forms / SAP Cloud Print if needed).
- Direct access to classic DDIC tables — use **CDS view entities** as the
  stable contract.
- Unreleased SAP objects (any class/function/table not flagged
  `Released: ✓` in the SAP API Information Hub).
- Modifying SAP standard repository objects (no "repair" / "enhancement" of
  SAP-internal code; use BAdIs, user exits, or key-user extensibility
  exposed by SAP).

Violations are caught by ATC checks against the SAP-released-objects list.

## Out of Scope for This Skeleton

- No CDS view entities yet.
- No OData service exposure.
- No RAP behavior definition.
- No BAdI / user-exit implementation.
- No unit tests (no test classes to compile).
- No transport requests / release branches.
- No CI/CD pipeline against an actual SAP system.

These are introduced only when a `normal` or `high-risk` story is intaken.

## Entry Points (Future)

When a story enters implementation, the first files to land will be:

1. `package.dev` / `package.json` (abapGit manifest) — package layout
   declaration.
2. `zcl_*` global ABAP classes following ABAP Clean Code
   (`zif_*` interfaces, value objects, factory methods).
3. CDS view entities under `src/` (or any abapGit-rooted folder).
4. Behavior definitions and metadata extensions for RAP services.

No `z*` include or report programs will be authored.

## Update Rule

When behavior changes:

1. Update the affected domain file (e.g. `docs/product/<domain>.md`) or
   create a new one from a real spec — never from speculation.
2. Update or create a story packet under `docs/stories/`.
3. Update durable proof with `scripts/bin/harness-cli story update`.
4. Record a decision in `docs/decisions/` when the contract changes.
