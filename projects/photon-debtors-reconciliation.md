# Photon Debtors Reconciliation

## Business Problem

Photon needed a reliable way to identify customers receiving service but not billing correctly, customers being billed without corresponding service evidence, and accounts requiring manual investigation. The existing operational picture depended on spreadsheets and separate source systems: accounting data, switch usernames, PPPoE secrets, active sessions, logs, technician notes, and manual comments.

The core problem was identity. A customer name, Sage account, PPPoE username, contract note, or active session can all refer to the same real-world service, but none of those fields is reliable enough on its own. The project therefore treats the physical service point as the primary identity:

```text
complex + unit number
```

All other data is imported as evidence attached to that service point. The system does not write back to MikroTik, Radius, Sage, or CRM systems. It imports evidence, identifies matches and exceptions, assigns risk, proposes reviewable changes, and generates reports for human review.

## Actual Implementation

I built the Phase 2A foundation as a Python reconciliation and audit platform using SQLAlchemy 2.x, Alembic, pandas, openpyxl, rapidfuzz, psycopg, and python-dotenv.

The local development default is SQLite:

```text
sqlite:///photon_reconcile.db
```

The project is also designed for PostgreSQL through a SQLAlchemy URL:

```text
PHOTON_DATABASE_URL=postgresql+psycopg://user:password@localhost/photon
```

The implementation includes:

- SQLAlchemy models for complexes, units, service points, customer identities, occupants, PPPoE accounts, Sage accounts, contract records, technician notes, reconciliation runs, evidence records, source API objects, match candidates, compliance snapshots, risk assessments, proposed changes, audit events, and verification tasks.
- Alembic migration `0001_phase_2a_foundation.py`, which creates the Phase 2A schema.
- PostgreSQL-oriented JSON handling using SQLAlchemy JSON with a PostgreSQL JSONB variant.
- Soft-delete and timestamp mixins for most domain records.
- UUID string primary keys for cross-source traceability.
- CLI commands for database initialization, accounting Excel import, MikroTik secrets import, active-session import, PPPoE log import, read-only Sage sync, Sage-vs-spreadsheet comparison, reconciliation, and report generation.

## Multi-Source ETL Workflow

The reconciliation process is an ETL pipeline:

1. Extract evidence from accounting spreadsheets, MikroTik exports, PPPoE logs, and optional Sage API reads.
2. Normalize customer names, complex aliases, unit numbers, and PPPoE usernames.
3. Load parsed evidence into SQLAlchemy models while preserving raw source data.
4. Match Sage/accounting evidence to PPPoE evidence by service point and name similarity.
5. Score confidence and risk.
6. Create compliance snapshots, proposed changes, audit events, verification tasks, and Excel reports.

The CLI supports these source import paths:

```bash
python -m photon_reconcile.cli import-accounting "Active Customers on Switch vs Sage April 2026.xlsx"
python -m photon_reconcile.cli import-mikrotik-secrets secrets.txt
python -m photon_reconcile.cli import-active active.txt
python -m photon_reconcile.cli import-logs pppoe.log
python -m photon_reconcile.cli sync-sage --dry-run --output-json sage_dump.json
python -m photon_reconcile.cli run-reconciliation
python -m photon_reconcile.cli generate-reports --output-dir reports
```

## Accounting and Sage Evidence

The accounting Excel importer reads the `Sage vs Radius Switch` sheet with pandas and treats the spreadsheet as evidence, not as authority. It imports:

- Sage/customer display names from the `Customer` column.
- Switch usernames from the `Switch` column.
- Monthly charge values from `Total Selling`.
- Manual comments and operational notes from `Comment`, `Cancellation Date`, `Description`, and `Action`.
- Source row IDs and raw row data for traceability.

For each row, the importer attempts to parse a service point from customer strings such as:

```text
WH/Ade09/Lindiwe
```

This becomes a canonical service point key:

```text
westinghouse:adelaar:9
```

The read-only Sage connector is designed to replace manual spreadsheet copying over time. It supports two Sage Accounting API surfaces:

- Sage Business Cloud Accounting API v3.1 at `https://api.accounting.sage.com/v3.1`.
- South African Sage One Accounting API v2.0 at `https://accounting.sageone.co.za/api/2.0.0`.

The connector intentionally exposes only GET/list operations. It does not create, update, delete, allocate, post, suspend, or rename anything in Sage. Raw Sage API objects are preserved in `source_api_objects` with object type, object ID, raw JSON, fetched timestamp, company ID, redacted source URL, and reconciliation run ID.

## PPPoE and MikroTik Evidence

The MikroTik importers handle:

- `/ppp secret print detail` exports.
- `/ppp active print detail` exports.
- PPPoE log exports.

The importers parse key-value records and log lines, normalize PPPoE usernames, and create `PppoeAccount` records where usernames are present. They also create `EvidenceRecord` rows for the raw source text, parsed fields, confidence, warnings, source filename, source row ID, and linked service point when one can be resolved.

Username normalization handles cases such as:

- Standard usernames with domains.
- Generic `@photon` usernames that do not identify a unit.
- Reversed unit/complex patterns such as `benjamin@7egret`.
- Domain aliases such as `flamingo` mapping to the canonical `flam` complex code.

## Normalization and Validation Rules

The normalization layer converts inconsistent operational labels into comparable identifiers:

- Complex aliases such as `ade`, `adelaar`, `flamingo`, `fla`, `gannet`, and `egret`.
- Unit numbers with leading zeros removed.
- Sage customer labels into service point keys and occupant hints.
- PPPoE usernames into normalized username, local part, domain, complex, unit, and canonical service point key.
- Customer names into normalized strings for comparison.

Confidence scores are assigned during parsing:

- Sage-style customer strings receive high confidence when both complex and unit are parsed.
- PPPoE usernames receive high confidence when a canonical service point key can be derived.
- Generic `@photon` usernames receive warnings because they do not identify a unit.
- Missing or malformed fields produce warnings rather than being silently accepted.

Sage configuration validation is explicit. Tests cover missing OAuth credentials, missing Sage One Basic/API-key credentials, `.env` loading, and redaction of secrets from debug summaries and URLs.

## Matching, Confidence, and Duplicate Detection

Reconciliation runs over each service point and compares linked Sage/accounting records with linked PPPoE records.

Match candidate scoring is implemented as:

- Sage-only evidence: score `45`, reason `SERVICE_POINT_FROM_SAGE_ONLY`.
- PPPoE-only evidence: score `35`, reason `SERVICE_POINT_FROM_SWITCH_ONLY`.
- Sage plus PPPoE evidence: starts at `50` for `COMPLEX_UNIT_MATCH`.
- Name similarity of 90 or higher adds `30` and reason `HIGH_NAME_SIMILARITY`.
- Name similarity of 70 or higher adds `15` and reason `MEDIUM_NAME_SIMILARITY`.
- Active PPPoE evidence adds `10` and reason `ACTIVE_EVIDENCE`.
- Final confidence is capped at `100`.

Confidence labels are:

- `HIGH` for scores of 80 or higher.
- `MEDIUM` for scores of 55 to 79.
- `LOW` below 55.

Duplicate detection is included in the reconciliation engine. A service point is flagged as duplicate when more than two Sage/PPPoE assignments are attached to the same service point. Duplicate service point assignment contributes to risk scoring and exception reporting.

The Sage-vs-spreadsheet comparison also reports spreadsheet duplicate/gap conditions and counts which rows are missing from API evidence.

## Risk Scoring and Exception Identification

Risk scoring is additive and produces `INFO`, `LOW`, `MEDIUM`, or `HIGH` levels.

Implemented risk rules include:

- Active PPPoE account without a Sage record.
- Active service where contract status is not confirmed.
- Duplicate service point assignment.
- Sage status note containing cancellation or inactive language while PPPoE is active.
- Generic `@photon` username.
- Sage and PPPoE present but match confidence below 80.
- Sage record without switch account.
- PPPoE account without active session and without Sage record.

The reconciliation engine creates:

- `MatchCandidate` rows with confidence score, confidence label, reason codes, and evidence IDs.
- `ComplianceStateSnapshot` rows with operational state, compliance state, reason codes, and evidence IDs.
- `RiskAssessment` rows with risk score, risk level, reason codes, and evidence IDs.
- `ProposedChange` rows for reviewable actions such as requesting a site visit, marking orphaned accounts, or standardizing generic usernames.
- `VerificationTask` rows for medium/high-risk service points that require field review.
- `AuditEvent` rows when proposed changes are detected.

Compliance states include active, dormant, orphaned switch account, active username mismatch, legacy account, and manual review required.

## Reporting

The report generator creates Excel workbooks using pandas and openpyxl:

- `technical_reconciliation.xlsx`
- `compliance_review.xlsx`
- `orphaned_accounts.xlsx`
- `proposed_username_standardization.xlsx`

Reports include service point, Sage customer, PPPoE username, active status, confidence score, confidence label, risk score, risk level, reason codes, evidence IDs, approval state, and human review notes columns where relevant.

The Sage connector comparison also generates:

- `sage_vs_spreadsheet_gap_report.xlsx`
- `sage_connector_validation.md`

The recorded Sage connector validation report showed:

- 89 spreadsheet customer rows.
- 0 Sage API objects available in the workspace at that time.
- 89 spreadsheet-only records.
- 88 manual spreadsheet-only rows.
- Manual-only fields still required: `Switch`, `Comment`, `Description`, `Action`, and `Cancellation Date`.

The conclusion from that validation was that the spreadsheet could not yet be retired because no Sage API evidence had been synced and manual operational evidence still needed a replacement workflow.

## Technical Challenges

- Service identity could not rely on customer name alone. The project uses physical service point identity as the anchor and attaches accounting, PPPoE, and technician evidence to it.
- PPPoE usernames are inconsistent. The normalizers handle generic domains, reversed unit/complex strings, missing `@` symbols, aliases, and incomplete domain information.
- Accounting spreadsheets mix structured data with human comments and operational judgement. The importer preserves those fields as evidence rather than flattening them into a false source of truth.
- Sage API coverage varies by region/product. The connector supports both Sage Business Cloud Accounting v3.1 and South African Sage One Accounting v2.0 while keeping all operations read-only.
- Risk scoring needs to identify review priorities without automating changes. The system creates proposed changes and verification tasks, not direct updates to MikroTik or Sage.
- Reports need to be reviewable by non-developers, so generated Excel files include reason codes, evidence IDs, and human review note columns.

## Testing Performed

The project contains 18 tests covering:

- Sage customer parsing, including leading-zero unit normalization.
- Lowercase and malformed spacing in customer identifiers.
- PPPoE username normalization for generic `@photon` usernames.
- Reversed unit/complex username patterns.
- Domain alias handling.
- Risk scoring for active PPPoE accounts without Sage records.
- Duplicate service point risk contribution.
- Cancelled or inactive Sage notes while service remains active.
- Sage OAuth credential validation.
- Sage One South Africa API-key/basic-auth validation.
- `.env` loading for Sage configuration.
- Secret redaction in URLs and safe summaries.
- Sage dry-run behavior that does not persist API objects.
- Mapping Sage API objects into raw source objects, Sage accounts, customer identities, and evidence records.
- Spreadsheet comparison logic that counts missing API records.
- Sage One URL construction with API key and company ID.
- Sage Business Cloud v3.1 URL construction without token leakage.
- Safe redaction of Sage One username/password values.

## Measurable Outcomes

- Built an 18-table SQLAlchemy/Alembic foundation for PPPoE, Sage/accounting, evidence, reconciliation, risk, audit, and reporting data.
- Implemented multi-source ETL for accounting Excel, MikroTik PPPoE secrets, active sessions, logs, and read-only Sage API evidence.
- Generated four reconciliation review workbooks plus Sage gap/validation reports.
- Added 18 tests covering normalization, risk scoring, Sage connector safety, API evidence mapping, and spreadsheet comparison.
- Produced a Sage connector validation report showing 89 spreadsheet rows, 88 rows with manual-only operational evidence, and a clear decision not to retire the spreadsheet until Sage API evidence and manual comments/actions have a replacement path.

## What This Demonstrates

This project demonstrates Python data engineering, SQLAlchemy modeling, Alembic migrations, PostgreSQL-oriented design, pandas ETL, source evidence preservation, PPPoE subscriber reconciliation, accounting-system imports, matching logic, confidence scoring, risk scoring, duplicate detection, audit events, exception reporting, and Excel report generation.

It also demonstrates a safety-focused automation approach: the system imports and reconciles evidence, but proposed changes remain reviewable and no write operations are performed against MikroTik, Radius, Sage, or CRM systems.

## Repository Status

Private/internal. The public portfolio excludes real debtor records, PPPoE exports, Sage credentials, API dumps, accounting workbooks, generated reports containing sensitive rows, and operational customer details.
