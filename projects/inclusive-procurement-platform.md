# Inclusive Procurement Platform

## Business Problem

The project addressed KPI 2 Tier 2 spend verification for an internal procurement compliance process. The source data arrived through Excel workbooks such as Tier 2 spend verification files and buyer-specific procurement tracking registers. Those workbooks contained supplier names, contractor details, Vote/RFC/PO references, invoice references, claimed spend, approved or declined spend, and local supplier evidence.

The main risk was that spreadsheet review could not reliably prove why a spend row qualified, why it needed review, or where the source evidence came from. Fuzzy supplier matches, expired B-BBEE certificates, missing PO or invoice references, labour/TES/admin-fee descriptions, and mismatched procurement identifiers all needed to be flagged instead of silently accepted.

## Actual Implementation

I built an early secure prototype in Python using FastAPI, SQLAlchemy 2.x, Alembic, Pydantic, openpyxl, Jinja2 templates, and CSV exports. I designed it as a PostgreSQL-backed system with a SQLite pilot mode for Windows UAT, so the same application could support local demonstration while keeping the production data model oriented toward PostgreSQL.

The implementation includes:

- FastAPI API routes and a server-rendered internal UI under `/ui`.
- SQLAlchemy models for suppliers, contractors, contracts, Tier 2 transactions, supplier documents, buyers, procurement registers, reconciliation results, KPI commitments, local supplier allocations, import jobs, and audit events.
- Alembic migrations for schema evolution across the transaction workflow, import jobs, procurement registers, reconciliation result detail, and KPI commitment capture.
- ETL pipelines that import, validate, normalize, reconcile, and report on procurement data from multiple Excel workbook formats.
- A Tier 2 Spend Verification `.xlsx` importer that reads all workbook sheets, detects header rows within the first 20 rows, accepts multiple column-name variants, parses Decimal amounts and dates, and preserves source workbook, sheet, and row metadata.
- A buyer-specific Procurement Tracking importer that stores procurement register entries with Vote, RFC, PO, contractor, supplier, value, project, package, operation, workbook, sheet, and source row traceability.
- Import job persistence with SHA-256 file hashing, running/completed/failed status, rows seen/imported/skipped counts, needs-review counts, suspected-exclusion counts, and duplicate protection by workbook hash, sheet, and row.
- KPI 2 transaction review services for imported, needs-review, approved, declined, and excluded states.
- CSV exports for KPI 2 summary, review queue, supplier compliance, supplier matching, procurement reconciliation, reconciliation exceptions, reconciliation summary, and KPI commitments.
- A Windows pilot package with `start_app.bat`, `reset_demo_db.bat`, safe synthetic seed data, and a sanitized zip builder that excludes real workbooks, databases, uploads, logs, reports, and environment files.

## Data Validation and Reconciliation

The importer never approves spend automatically. Imported rows enter the database as `imported` or `needs_review`, and approval must go through the review service.

Validation and review flags include:

- Missing PO number.
- Missing invoice or reference.
- Missing local supplier.
- Unknown local supplier.
- Fuzzy supplier match.
- Missing B-BBEE status.
- Missing or expired B-BBEE certificate expiry.
- Suspected labour, TES, admin-fee, or admin-labour spend.
- Invalid transaction dates.
- Amount parsing issues.
- Approved or declined amounts greater than claimed amounts.
- Declined amount present.

Supplier matching uses normalized names and exact matching first. Fuzzy matching uses a similarity threshold of 0.82 and records the match score as review metadata only; fuzzy matches are never treated as approval evidence.

Procurement reconciliation compares imported Tier 2 spend against buyer-specific procurement register entries. The matching hierarchy is:

1. PO Number as the strongest match.
2. RFC Number as a secondary match.
3. Vote Number as review-only because multiple POs can exist under the same Vote.
4. Contractor or supplier name as review-only possible matches.

Reconciliation results store match status, confidence level, matched-by basis, review-required indicator, review notes, detection timestamp, and source workbook/sheet/row traceability for both the Tier 2 transaction and the procurement register entry.

## Technical Challenges

- Handling workbook formats that were not reliable database inputs: headers could move, monthly data could span multiple sheets, summary rows had to be skipped, and column names varied between files.
- Keeping imported workbook evidence separate from reviewed and accepted records so the database did not treat a spreadsheet as permanently authoritative.
- Preventing risky automation. The system deliberately blocks auto-approval for fuzzy supplier matches, unverified suppliers, expired B-BBEE evidence, excluded spend categories, labour/TES/admin-fee spend, and incomplete references.
- Designing reconciliation around real procurement identifiers. PO and RFC can support confirmed matches, while Vote-only and name-only matches remain review-bound.
- Preserving traceability from every imported row back to workbook, sheet, row number, and import job.
- Creating a local Windows UAT path that could run with synthetic data and without Docker, production secrets, real workbooks, or PostgreSQL.

## Testing Performed

The project contains 57 pytest test functions covering:

- Supplier name normalization.
- Tier 2 workbook import, blank-row skipping, invalid amount handling, missing contractor handling, duplicate import protection, and import job summary counts.
- Transaction validation for source traceability and amount constraints.
- KPI 2 verification rules for labour/TES exclusion, expired B-BBEE certificates, missing suppliers, duplicate warnings, and fuzzy-match review handling.
- Approval and decline workflows, including required decline reasons and audit event creation.
- Procurement register import, offset header detection, multi-sheet parsing, duplicate import skipping, missing identifier flags, reconciliation matching, mismatches, dashboard metrics, and CSV exports.
- KPI commitment validation, acceptance rules, allocation variance, audit records, API routes, UI routes, and CSV export content.
- Server-rendered UI routes, upload form import, review queue display, supplier and buyer CRUD, duplicate supplier blocking, and audit page rendering.

## Measurable Outcomes

- Converted workbook-based KPI 2 review into a database-backed import, review, reconciliation, audit, and reporting prototype.
- Built ETL coverage for multiple procurement workbook formats instead of relying on manual spreadsheet review.
- Added 57 pytest test functions across import, validation, reconciliation, reporting, API, and UI behavior.
- Produced seven CSV export paths for review and reporting outputs.
- Preserved source traceability for imported transaction and procurement register rows at workbook, sheet, row, and import-job level.
- Built a sanitized Windows pilot package path using SQLite and synthetic data, while keeping PostgreSQL defined for future deployment work.

## What This Demonstrates

This project demonstrates Python automation, FastAPI application development, SQLAlchemy data modeling, PostgreSQL-oriented database design, Excel ETL, validation, reconciliation, audit trails, CSV reporting, and workflow design for a compliance-heavy procurement process.

It also shows an engineering approach that favors controlled review over unsafe automation: imported data is parsed and flagged, but business approval remains explicit, auditable, and reversible through the application workflow.

## Repository Status

Private/internal. The public portfolio excludes real supplier data, procurement workbooks, local databases, generated reports, credentials, company-specific documents, and deployment secrets.
