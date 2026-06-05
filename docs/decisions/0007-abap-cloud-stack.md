# 0007 SAP Public Cloud ABAP Cloud Stack

Date: 2026-06-05

## Status

Accepted

## Context

The project targets **SAP S/4HANA Cloud Public Edition** and **SAP BTP,
ABAP Environment (Steampunk)**. These runtimes enforce a strict
released-objects contract: only APIs, classes, function modules, and DDIC
artifacts that SAP has marked `Released: ✓` (or a higher release status) may
be used by customer code. Several classic ABAP techniques — Dynpro
dialog programming, `SMARTFORMS`, classic ALV function modules, direct
read/write of classic DDIC tables — are either unreleased, deprecated, or
block release transport.

The project is also **cross-module**: it must work for FI, MM, SD, PP, HR,
BW, and any other module that ships in S/4HANA Cloud. No module-specific
assumptions can be baked into the stack.

The first intake (lane `tiny`, this decision) is a skeleton with no
application code, no schema, no CRUD, and no auth. The stack decision is
still required so that future `normal` / `high-risk` stories do not
silently drift into unreleased territory.

## Decision

1. **Language:** ABAP Cloud (ABAP for SAP Cloud) on release contracts that
   support the Steampunk / S/4HANA Cloud Public Edition APIs.
2. **Programming model:** ABAP OO + ABAP Clean Code. No procedural
   `PERFORM` chains, no `INCLUDE` programs, no classic dynpro.
3. **Data access:** CDS view entities (DDL source) as the stable contract
   for reads; ABAP SQL is permitted only against released objects.
4. **Service exposure (when needed):** OData via the ABAP RESTful
   Application Programming Model (RAP) — `behavior definition`,
   `metadata extension`, and `service binding` artifacts.
5. **Forms / output (when needed):** Adobe Forms via
   `FP_ADS_FORM_*`/`FP_TEST*` released APIs, or SAP Cloud Print.
6. **Tooling:** ABAP Development Tools (ADT) for day-to-day development;
   `abapGit` for serialization into this repository.
7. **Quality gates:** ATC (ABAP Test Cockpit) with the SAP-released-objects
   check enabled; ABAP Unit for unit tests; integration tests only via
   released test seams.

## Alternatives Considered

1. **Classic ABAP (Dynpro + ALV + SMARTFORMS) on a NetWeaver AS.**
   Rejected: not portable to S/4HANA Cloud Public Edition, which the
   project targets. Would require parallel codebases.
2. **Side-by-side extensibility on SAP BTP with CAP / Java / Node.js.**
   Rejected for this skeleton: the project is named SAP_ABAP and the
   initial scope is ABAP-native. CAP remains an option for non-ABAP
   extensions, to be revisited if a story requires it.
3. **Key-user extensibility (in-app, no custom ABAP).**
   Rejected for this skeleton: the project needs a developer-controlled
   workspace, and the cross-module scope implies logic that goes beyond
   what key-user fields and simple in-app rules can express.
4. **Mixed classic + cloud, gated by SY-SAPRL or system variable.**
   Rejected: introduces a non-deterministic split, breaks the
   released-objects invariant, and is an anti-pattern in Cloud ALM.

## Consequences

Positive:

- A single, deterministic code path that runs on S/4HANA Cloud Public
  Edition, BTP ABAP Environment, and (with care) on-premise S/4HANA
  2023+.
- ATC + released-objects gating catches drift before transport.
- RAP + CDS gives a stable, OData-friendly contract for any future Fiori
  or third-party client.
- Cross-module scope is honest: we are not pretending to ship FI/MM/SD
  domains on day one.

Tradeoffs:

- Any classic ABAP developer joining the project must re-learn the
  released-objects contract. This is a known ramp-up cost.
- Some legacy patterns (deep Dynpro flows, classic ALV) are unavailable.
  Equivalent UX is built with Fiori Elements + RAP.
- RAP + CDS has a higher upfront cost than a throwaway ALV report, but
  pays off the moment the first integration or test is added.
- No transport pipeline exists yet — every story that needs to ship
  against a real system will create that pipeline as part of its work.

## Follow-Up

- [ ] Add an abapGit manifest (`package.dev` / `package.json`) and a
      folder layout in a follow-up `tiny` story.
- [ ] Define a `tiny` smoke story: a single `zcl_` interface + class
      pair that compiles under ATC with the released-objects check.
- [ ] Capture the released-objects list (or the link to the SAP API
      Information Hub query) in `docs/GLOSSARY.md`.
- [ ] When the first real spec lands, re-evaluate the choice of
      `behavior definition` vs. `unmanaged query` for early
      service exposure.
