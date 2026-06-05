# Architecture

This is a **thinking template** adapted for an ABAP Cloud target
(SAP S/4HANA Cloud Public Edition / SAP BTP, ABAP Environment —
"Steampunk"). It is filled in progressively as stories enter
implementation. The stack is settled in
`docs/decisions/0007-abap-cloud-stack.md`.

## Discovery Before Shape

Before proposing implementation, identify:

- **Product surfaces:** Fiori Elements app (primary), OData consumer
  (secondary), ADT-driven unit test runner (developer surface).
- **Runtime stack:** ABAP Cloud, ABAP OO, CDS view entities, RAP,
  Released Objects API. Hosted on SAP BTP / S/4HANA Cloud Public
  Edition. See ADR 0007.
- **Core domains:** unknown at skeleton time. To be derived from
  the first accepted spec or change request.
- **Boundary inputs:** inbound OData / HTTP requests (Fiori Elements
  client), IDoc / webhooks from S/4HANA processes, scheduled jobs
  (RAP background processing), environment configuration
  (`/sap/opu/odata/...` endpoints, destination configuration).
- **Validation ladder:** ATC (released-objects check), ABAP Unit, abapGit
  serialization round-trip, optional integration test against a
  developer SAP system.

## Default Layering

For ABAP Cloud, the default layering maps to ABAP package structure and
class responsibilities:

```text
domain
  <- application
      <- infrastructure
          <- interface
              <- app surfaces
```

Mapping to ABAP package conventions:

| Layer | ABAP package convention | Example types |
| --- | --- | --- |
| domain | `zif_*` interfaces, value-object types | `zif_invoice_id`, value-object for status |
| application | `zcl_*_handler`, `zcl_*_service` | `zcl_invoice_creation_handler` |
| infrastructure | `zcl_*_repository_*`, `zcl_*_gateway_*` | CDS view entities, RAP behavior |
| interface | `zcl_*_consumer_*` | OData / HTTP inbound adapters |
| app surfaces | Fiori Elements app (out of this repo) | metadata extensions |

Folders are not created until a story needs them. abapGit handles
serialization on demand.

## Candidate Structure

abapGit-rooted folder layout (only the parts that match the layering):

```text
src/
  zif_*.abap                  -- domain interfaces
  zcl_*_vo.abap               -- value objects (immutable, no state)
  zcl_*_factory.abap          -- domain factories
  zcl_*_handler.abap          -- application handlers (use cases)
  zcl_*_service.abap          -- application services
  zcl_*_repository_*.abap     -- infrastructure (DB / API adapters)
  zcl_*_consumer_*.abap       -- interface adapters (OData, HTTP)
  ddl/                        -- CDS view entities
  bdef/                       -- RAP behavior definitions
  srvd/                       -- RAP service bindings
  meta/                       -- metadata extensions
  zcl_*.testclasses.abap      -- ABAP Unit local test classes
package.dev.yml / package.json -- abapGit manifest
```

## Dependency Rule

Inner layers must not depend on outer layers.

| Layer | May depend on | Must not depend on |
| --- | --- | --- |
| domain | released standard types and interfaces | unreleased objects, UI, ADT, framework glue |
| application | domain | unreleased objects, UI, transport framework internals |
| infrastructure | domain, application | interface controllers, Fiori surface assumptions |
| interface | all backend layers | UI state, browser-only assumptions |
| app surfaces (Fiori) | OData contract and released metadata | direct domain internals |

Additional rule for ABAP Cloud: **no layer may depend on an object
that is not flagged `Released: ✓` in the SAP API Information Hub.**
ATC enforces this.

## Parse-First Boundary Rule

Unknown data must be parsed at boundaries before it enters inner code.

ABAP Cloud boundaries:

- OData / HTTP request payloads — parsed into CDS-mapped structures or
  ABAP structures with explicit field types.
- IDoc / webhook payloads — parsed via released `*_READ_*` function
  modules or custom parsers that target released types.
- Environment variables / destination config — read via released
  destination APIs (`CL_HTTP_DESTINATION_*`); never read raw
  `GET PARAMETER` from `SY` untyped.
- DB rows from `SELECT` — typed via CDS view entity return types.
- Test fixtures — built with explicit constructor calls, never parsed
  from `JSON`/`XML` strings inside the test.

Target flow:

```text
unknown input
  -> parser / mapping
  -> typed ABAP structure or value object
  -> application handler
  -> domain object / value object
```

Inner code works with meaningful product types such as
`zcl_invoice_id` (a value object), `zcl_fiscal_year` (a value object),
or CDS-typed `invoice_status` (an ABAP type derived from a CDS
enum), rather than raw `string` or `i`.

## Command/Query Boundary

For RAP-based services, keep command/query separation clear:

- **Commands** (writes): `MODIFY` operations on RAP entities, owned
  by behavior handler / saver class. Own audit side effects.
- **Queries** (reads): `SELECT` against CDS view entities, or
  read-only RAP query. No side effects, no log writes.
- **Shared domain rules** live in the application/domain layer, not
  in the behavior handler.

If a story uses raw ABAP SQL outside RAP, the same split applies:
writes go through a single repository class; reads can be local to
the consumer.

## Observability Contract

A future server / RAP handler should emit one canonical log line per
request with:

- timestamp (UTC, ISO 8601)
- level (`DEBUG`, `INFO`, `WARNING`, `ERROR`)
- request_id (from `X-CorrelationID` header or generated)
- user_id when known (from session or OAuth subject)
- action (the use case or behavior operation)
- duration_ms
- status_code (HTTP or RAP operation status)
- message

Use `cl_bali_log` and the released logging APIs (or the
`/sap/opu/odata/IWFND/...` standard trace where it applies). Audit
logs are product records (e.g. for authorization-relevant changes)
and must be stored separately from application logs. Do not use one
as a substitute for the other.

## Out of Scope (Skeleton)

Until a `normal` or `high-risk` story is intaken, none of the above
folders or files exist. They are introduced only when a story needs
them — see `docs/decisions/0007-abap-cloud-stack.md` for the
follow-up backlog.
