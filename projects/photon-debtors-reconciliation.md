# Photon Debtors Reconciliation

## Problem Solved

Debtor and payment reconciliation can become time-consuming when source data arrives from multiple systems, bank statements, exports, or manually maintained spreadsheets. This project supports structured reconciliation by cleaning data, matching records, identifying exceptions, and preparing review outputs.

## Business Context

Operational finance teams need reliable debtor visibility and exception reporting. The value of this work is in turning scattered source data into a repeatable reconciliation process that supports faster review and better decision-making.

## Tech Stack

- Python
- pandas
- CSV and Excel workflows
- ETL and reconciliation logic
- Reporting exports
- Validation checks

## Key Features

- Imports debtor, payment, and statement-style data from structured files
- Cleans and normalizes inconsistent fields
- Matches records using deterministic reconciliation rules
- Flags unmatched, duplicate, missing, or suspicious records
- Produces review-friendly outputs for finance users
- Supports repeatable reconciliation runs

## Data / Automation Work Involved

- Built ETL steps to load, clean, transform, and compare finance data
- Used pandas to normalize identifiers, dates, amounts, and references
- Applied validation checks before producing reconciliation outputs
- Separated matched records from exceptions that require human review
- Reduced manual spreadsheet comparison work

## Testing and Quality Notes

- Tested matching logic with representative sample data
- Checked totals, row counts, duplicate handling, and exception categories
- Designed outputs to support review rather than hide uncertainty
- Excluded real debtor data, payment records, account details, and client-specific exports

## What This Demonstrates To Employers

- Strong fit for data engineering and finance operations automation
- Practical ETL, validation, reconciliation, and reporting experience
- Ability to work with messy operational data while preserving reviewability
- Business-outcome focus: faster reconciliation, fewer manual errors, clearer exceptions

## Repository Status

Private/internal. Public materials avoid sensitive debtor records, financial details, and company-specific source data.

