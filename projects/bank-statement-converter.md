# Bank Statement Converter

## Problem Solved

Bank statement data is often locked in PDFs or inconsistent exports, making reconciliation and reporting slow and error-prone. This project converts statement files into clean, structured data that can be reviewed, exported, and used in downstream finance workflows.

## Business Context

Finance and operations teams frequently need reliable transaction data for reconciliation, debtor tracking, payment review, and reporting. Manual capture introduces avoidable errors and consumes time that should be spent on exception handling and analysis.

## Tech Stack

- Python
- pandas
- PDF processing
- OCR-assisted extraction where needed
- CSV and Excel-ready exports
- Validation and normalization logic

## Key Features

- Parses bank statement source files into structured transaction rows
- Normalizes dates, descriptions, amounts, balances, and reference fields
- Produces CSV outputs suitable for finance review and reconciliation
- Handles inconsistent statement formatting where possible
- Flags suspicious or incomplete extracted values for review
- Keeps conversion logic separate from sensitive source documents

## Data / Automation Work Involved

- Extracted semi-structured statement data from PDF or text-based sources
- Cleaned and normalized transaction-level records with pandas
- Applied validation checks for totals, balances, date ordering, and missing fields
- Produced repeatable outputs for finance and reconciliation processes
- Reduced manual copy-paste work and improved consistency of downstream data

## Testing and Quality Notes

- Tested against representative statement samples and edge cases
- Checked output row counts, amount parsing, balance continuity, and date formats
- Added review points for OCR uncertainty and extraction anomalies
- Avoided storing or publishing real financial data in the repository

## What This Demonstrates To Employers

- Strong Python data extraction and transformation skills
- Practical pandas experience with messy financial data
- Understanding of OCR limitations and validation requirements
- Ability to automate high-value operational finance workflows safely

## Repository Status

Demo/private. Public-facing materials exclude real bank statements, account details, financial records, and client-specific formats.

