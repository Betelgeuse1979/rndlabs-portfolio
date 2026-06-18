# Bank Statement Converter

## Business Problem

Accounting users needed a local way to convert scanned South African bank statements into structured transaction files for review and import into accounting systems. The original work started as a one-off Absa statement conversion script and grew into a multi-bank converter because statement layouts, OCR quality, running balances, and export formats varied by bank.

The main business risk was manual capture: transactions could be missed, duplicated, misclassified as debit or credit, or exported with row counts that did not match the extracted data. Password-protected PDFs, scanned image PDFs, and bank-specific layout differences also made a single simple parser unreliable.

## Actual Implementation

I built a Python statement conversion engine with both a Django web interface and a PySide6 desktop pilot. The conversion engine remains framework-independent under `statement_converter/`, while the web and desktop layers call into the same OCR, parser, validation, and export code.

The implementation includes:

- Multi-bank parser support for Absa, FNB, Standard Bank, Investec, and Nedbank.
- Auto-detection that runs available parsers and chooses the parser with the strongest extraction result, with penalties for noisy matches.
- OCR processing for scanned PDFs and image files using `pypdfium2`, Pillow, `pytesseract`, and the external Tesseract OCR executable.
- PDF rendering at configurable DPI, with the desktop workflow defaulting to 450 DPI.
- Password-protected PDF handling through `pypdfium2`, including safe messages for missing and incorrect passwords.
- Bank-specific parser logic for date formats, debit/credit classification, running balances, continuation lines, card references, merchant extraction, and OCR separator noise.
- pandas DataFrame normalization into a consistent internal transaction shape: `Date`, `Description`, `Debit`, `Credit`, `Balance`, `BankCharges`, `Merchant`, `CardNo`, and `ExtraDescription`.
- CSV export generation for Standard Converter CSV, Sage CSV, Xero statement import CSV, and `all` mode.
- Combined CSV output for batching multiple statements into one accountant import file.
- Raw OCR text and unmatched-line files for parser tuning and accountant review.

The source I inspected implements CSV exports. I did not find XLSX export generation in the current source or pilot release documentation, so this case study does not claim XLSX output.

## Desktop Workflow and Managed Folders

The desktop pilot is designed as a local-first Windows utility. On first run it creates managed application folders under:

```text
%APPDATA%\Bank Statement Converter\data\
```

The managed folders are:

- `input`: statement PDFs and supported image files waiting for conversion.
- `passwords`: optional text files containing passwords for encrypted PDFs.
- `output`: generated accountant export files.
- `processed`: successfully converted source files when Dry Run is disabled.
- `failed`: files that could not be converted when Dry Run is disabled.
- `logs`: conversion logs.

The desktop settings file stores folder paths and the combine-output preference only. It does not store statement data, OCR text, credentials, PDF passwords, or bank transaction data.

## ETL, Validation, and Reconciliation Checks

The conversion workflow is an ETL pipeline:

1. Extract statement text through OCR or PDF/image rendering.
2. Transform bank-specific OCR text into normalized transaction rows.
3. Validate the extracted rows against financial controls.
4. Load the results into CSV files for accounting review or import.

The validation layer performs:

- Opening balance detection from statement text.
- Closing balance detection from statement text, with fallback to the final extracted running balance when needed.
- Calculated closing balance verification using opening balance plus signed transaction totals.
- Running balance validation row by row.
- Duplicate transaction detection using date, description, debit, credit, and balance columns.
- Export consistency checks to confirm generated CSV row counts match the extracted transaction count.
- Warning, fail, pass, and not-available statuses so uncertain validations are visible rather than hidden.

The FNB sample documented in the project extracted 40 rows. The debit total matched the statement turnover of `14,541.49`, and the credit total matched the statement turnover of `4,830.30`.

## Password-Protected PDF Handling

The desktop runner passes a run-only password into the conversion engine. The password can come from the UI field or from text files in the managed `passwords` folder. The runner tries password candidates for PDF files and distinguishes between:

- Missing password: `PDF is password protected. Please enter the PDF password and try again.`
- Incorrect password: `PDF password was incorrect or the PDF could not be opened.`

Passwords are not written to `settings.json`, logs, output files, or debug text. Tests verify that correct passwords are passed to the converter and that wrong or blank passwords fail without exposing the secret.

## Dry Run and Retry Failed Files

Dry Run is enabled by default in the desktop pilot. In Dry Run mode the app:

- Scans the input folder.
- Attempts OCR, parsing, validation, and logging.
- Shows summary counts.
- Does not move source PDFs.
- Does not create final output CSV files.
- Does not modify the source statements.

Production mode starts when Dry Run is disabled. In production mode the app writes final outputs, moves successful source files to `processed`, and moves failed files to `failed`.

`Retry Failed Files` temporarily uses the `failed` folder as the input folder for one run. Successful retry files move from `failed` to `processed`; files that still fail remain in `failed`. The retry workflow does not change the configured input folder and respects Dry Run and combined-output settings.

## Tesseract Detection and Setup Support

The app checks for Tesseract in three ways:

- `TESSERACT_CMD` environment variable.
- `tesseract.exe` available on `PATH`.
- Common Windows install paths:
  - `C:\Program Files\Tesseract-OCR\tesseract.exe`
  - `C:\Program Files (x86)\Tesseract-OCR\tesseract.exe`

The desktop UI and logs report whether Tesseract was found. The packaged app does not bundle Tesseract, but the Inno Setup installer checks for it after installation and can offer to install it through Winget:

```powershell
winget install --id UB-Mannheim.TesseractOCR --source winget --accept-package-agreements --accept-source-agreements
```

If Winget is unavailable or installation fails, the installer opens bundled Tesseract setup notes with manual instructions.

## Packaging and Pilot Deployment

The project includes Windows packaging support for:

- A PyInstaller one-folder desktop build at `dist\BankStatementConverter\BankStatementConverter.exe`.
- A portable one-file executable at `dist-single\BankStatementConverter.exe`.
- An optional Inno Setup installer at `dist\installer\BankStatementConverter-Setup.exe`.

The one-folder build is the troubleshooting fallback because dependencies are unpacked and easier to inspect. The one-file build is easier to share with testers but can start more slowly because PyInstaller extracts runtime files at launch.

The v0.2.0 pilot package includes:

- Installer and portable executable options.
- Release notes.
- Quick-start guide.
- SHA-256 checksums.
- Sample password instructions.
- Tesseract setup guidance.

The packaging scripts run desktop tests before building and verify the portable executable version output for the single-file build.

## Technical Challenges

- OCR quality differs by bank, scan quality, page layout, and statement type. The parsers include bank-specific handling for date formats, balances, debit/credit signs, continuation lines, card references, merchant fields, and noisy separators.
- Auto-detection has to avoid false positives. The parser scoring penalizes unmatched/noisy lines so a parser that extracts many weak rows does not automatically win.
- Running balance validation is useful only when enough numeric balances are available. The validation layer reports not-available or warning states instead of pretending every statement can be fully reconciled.
- Password-protected PDFs require safe failure handling without logging secrets or moving files incorrectly.
- Desktop batch processing must avoid data loss. Dry Run, unique output filenames, processed/failed folders, and retry behavior were implemented to keep source-file movement explicit.
- The packaged app must work on Windows machines where Python dependencies are bundled but Tesseract remains an external OCR dependency.

## Testing Performed

The project contains 43 unit tests covering:

- Desktop settings persistence with folder paths only.
- Managed AppData folder creation for input, output, processed, failed, logs, and passwords.
- Dry Run behavior that keeps source files in place and creates no final output files.
- Production-mode movement of successful files to `processed`.
- Movement of failed files to `failed` only when Dry Run is disabled.
- Non-combined output placement directly in the output folder.
- Filename collision handling with suffixes.
- Combined output generation.
- Retry Failed Files behavior, including successful retry, still-failing retry, Dry Run retry, and combined-output retry.
- Password-protected PDF behavior for blank, wrong, and correct passwords.
- Tests that passwords are passed to the converter but not written to logs.
- Tesseract detection through `TESSERACT_CMD`, PATH lookup, common Windows paths, and missing-dependency reporting.
- Validation checks for opening/closing balance reconciliation, incorrect totals, missing balances, duplicate transactions, running balance mismatch, empty data, and export row count mismatch.
- Parser tests for Standard Bank, Investec, Nedbank, and auto-detection behavior.

Manual pilot checks documented in the project include a packaged dry-run OCR smoke test using copied local test data outside the repository. The packaged app processed one file successfully, exported transactions, kept Dry Run enabled, and did not move the copied source file.

## Measurable Outcomes

- Expanded the original Absa-only script into a multi-bank converter covering Absa, FNB, Standard Bank, Investec, and Nedbank.
- Verified a real FNB sample with 40 extracted rows and debit/credit totals matching the statement turnover values documented in the project.
- Added 43 unit tests across parser behavior, validation, desktop batch processing, password handling, retry workflow, managed folders, and Tesseract detection.
- Built pilot packaging for Windows through PyInstaller one-folder, PyInstaller one-file, and Inno Setup installer paths.
- Implemented managed local folders and Dry Run safeguards so pilot users can test conversion without moving source statements or writing final outputs.

## What This Demonstrates

This project demonstrates Python data extraction, OCR processing, pandas-based ETL, bank-specific parsing, validation, duplicate detection, reconciliation-style balance checks, CSV export generation, desktop workflow design, packaging, and local technical operations support for accounting workflows.

It also shows careful handling of operational risk: passwords are not persisted, source files are not moved in Dry Run, failed files have a recovery workflow, OCR dependencies are detected explicitly, and generated outputs are validated against extracted transaction counts where possible.

## Repository Status

Private/demo. The public portfolio excludes real bank statements, OCR text, generated CSV outputs, logs, account details, passwords, client data, and packaged binaries.
