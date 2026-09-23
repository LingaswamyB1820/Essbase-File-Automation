# Essbase Automation -- OU Validation & File Management

## 📌 Project Overview

This project automates the **Essbase OU (Organizational Unit) validation
and file-management process** using **Azure Blob Storage, Azure
Databricks, PySpark, Pandas, Azure Logic Apps, and Databricks Secrets**.

The solution reads an `Essbase.xlsx` source file from Azure Blob
Storage, processes selected Excel sheets, combines the OU and Department
data, validates OU values using regex/business rules, and separates
valid and invalid records.

The pipeline then:

-   Saves valid OU records as CSV
-   Generates an OU-to-Department mapping TXT file
-   Archives valid output files
-   Logs invalid OU records
-   Sends an automated notification through Azure Logic Apps when
    invalid data is detected
-   Archives the original source Excel file
-   Applies a rolling **13-week retention policy** to historical files

------------------------------------------------------------------------

## 🏗️ High-Level Architecture

![Essbase Automation Architecture](docs/architecture.png)

------------------------------------------------------------------------

## 🔄 End-to-End Data Flow

``` text
Azure Blob Storage
        |
        v
   Essbase.xlsx
        |
        v
Azure Databricks
        |
        +--> Read selected Excel sheets
        |
        +--> Select OU & Department columns
        |
        +--> Remove empty rows
        |
        +--> Combine all sheets
        |
        v
     OU Validation
     /           \
    /             \
 Valid OU       Invalid OU
    |               |
    v               v
Valid CSV        Invalid CSV Log
    |               |
    v               v
Mapping TXT      Azure Logic App
    |               |
    v               v
Archive CSV     Notification
    |
    v
Source File Archive
    |
    v
13-Week Retention Cleanup
```

------------------------------------------------------------------------

# 1. Source -- Azure Blob Storage

The source Excel file is stored in Azure Blob Storage.

### Storage Structure

``` text
Storage Account
└── Container: files
    └── Essbase Automation
        └── Source File
            └── Essbase.xlsx
```

The automation downloads the source file from the Blob Storage container
before processing.

------------------------------------------------------------------------

# 2. Databricks Processing

Azure Databricks is used as the main processing layer.

### 2.1 Read Excel

The process:

1.  Downloads `Essbase.xlsx` from Azure Blob Storage.
2.  Reads only the required sheets.
3.  Processes the following sheets:

``` text
FTE
GSC
Othr Cmp
Internal WFC
External WFC
Revenue
IOI
```

------------------------------------------------------------------------

## 2.2 Process Sheets

For each selected Excel sheet:

-   Read only columns **A and B**
-   Column A → `OU`
-   Column B → `Department`
-   Remove empty rows
-   Standardize the required structure
-   Combine data from all selected sheets

Example:

  OU      Department
  ------- ------------
  06569   Finance
  06569   Technology
  06164   Operations

------------------------------------------------------------------------

# 3. Combine Data

Data from all selected Excel sheets is combined into a single DataFrame.

Example:

``` python
combined_df = pd.concat(all_sheet_data, ignore_index=True)
```

The resulting dataset contains:

``` text
OU
Department
```

This combined DataFrame is then passed to the OU validation process.

------------------------------------------------------------------------

# 4. OU Validation

The OU validation layer checks whether the OU value follows the expected
format.

Validation can be implemented using:

-   Regular expressions
-   Business rules
-   Required length
-   Numeric format
-   Leading-zero preservation
-   Null/empty checks

Example:

``` python
OU Pattern:
^\d{5}$
```

This example validates a five-digit OU such as:

``` text
06569
06164
```

Leading zeros must be preserved.

------------------------------------------------------------------------

## Validation Output

The processing creates two DataFrames:

``` text
valid_ou_df
invalid_ou_df
```

### Valid OU

Records that satisfy the required validation rules are stored in:

``` text
valid_ou_df
```

### Invalid OU

Records that fail validation are stored in:

``` text
invalid_ou_df
```

------------------------------------------------------------------------

# 5. Valid OU Flow

## 5.1 Save Valid CSV

Valid records are written to:

``` text
Essbase Automation/
└── Regex Valid/
    └── Regex Valid.csv
```

A dated version can also be generated for archival purposes:

``` text
Regex Valid_YYYYMMDD.csv
```

------------------------------------------------------------------------

## 5.2 Create Mapping TXT

A mapping file is generated from the valid OU data.

Example:

``` text
OU Department
06569 Finance
06569 Technology
06164 Operations
```

The mapping file uses a **tab-separated format**.

Important requirements:

-   Tab separated
-   Preserve leading zeros
-   Maintain OU-to-Department relationship

Example:

``` text
06569    Finance
06569    Technology
06164    Operations
```

Output:

``` text
OU Department Mapping.txt
```

------------------------------------------------------------------------

## 5.3 Archive Valid CSV

After successful processing, the valid CSV is moved/copied into the
archive location.

Example:

``` text
Essbase Automation/
└── Regex Valid/
    └── Regex Valid_YYYYMMDD.csv
```

The original working CSV can then be removed according to the
file-management rules.

------------------------------------------------------------------------

# 6. Invalid OU Flow

If invalid OU records are detected, the pipeline creates an invalid-data
log.

## 6.1 Save Invalid CSV

Example location:

``` text
Essbase Automation/
└── Regex Invalid Log/
    └── Regex Invalid_YYYYMMDD.csv
```

The file contains the OU records that failed validation.

------------------------------------------------------------------------

## 6.2 Azure Logic App Notification

When invalid OU data is detected, Azure Logic Apps sends an automated
notification.

The notification can contain:

-   Message
-   Processing details
-   Invalid OU information
-   JSON payload containing invalid records

Example conceptual payload:

``` json
{
  "status": "Invalid OU Found",
  "file": "Essbase.xlsx",
  "invalid_count": 5,
  "details": [
    {
      "ou": "ABC12",
      "department": "Finance"
    }
  ]
}
```

This allows business users/support teams to quickly identify and correct
invalid OU values.

------------------------------------------------------------------------

# 7. Source File Archival

After processing is completed, the original `Essbase.xlsx` file is
archived.

Example:

``` text
Essbase Automation/
└── Source File Log/
    └── Essbase_YYYY_MM_DD.xlsx
```

The original source file is removed from the active `Source File` folder
after successful archival.

This prevents the same input file from being processed repeatedly.

------------------------------------------------------------------------

# 8. Retention Policy

A rolling **13-week retention policy** is applied to historical files.

The cleanup process removes files older than 13 weeks from the relevant
log/archive folders.

### Folders covered

``` text
Essbase Automation/
├── Regex Valid Log/
├── Source File Log/
└── Regex Invalid Log/
```

Conceptually:

``` text
Current Date
     |
     v
Keep last 13 weeks
     |
     v
Delete older files
```

This helps control storage usage while retaining sufficient historical
processing information.

------------------------------------------------------------------------

# 9. Technology Stack

  Technology           Purpose
  -------------------- -----------------------------------
  Azure Blob Storage   Source and output file storage
  Azure Databricks     Data processing and orchestration
  PySpark              Distributed data processing
  Python               Automation and validation logic
  Pandas               Excel/DataFrame processing
  Regex                OU format validation
  Azure Logic Apps     Invalid-data notifications
  Databricks Secrets   Secure credential management
  Excel                Source input
  CSV                  Validation output/logging
  TXT                  OU-to-Department mapping

------------------------------------------------------------------------

# 10. Suggested Project Structure

``` text
essbase-automation/
│
├── README.md
│
├── docs/
│   └── architecture.png
│
├── notebooks/
│   ├── 01_read_essbase_excel.py
│   ├── 02_process_sheets.py
│   ├── 03_combine_data.py
│   ├── 04_validate_ou.py
│   ├── 05_generate_outputs.py
│   └── 06_archive_cleanup.py
│
├── src/
│   ├── excel_reader.py
│   ├── ou_validator.py
│   ├── file_manager.py
│   └── notification.py
│
├── config/
│   └── config.example.json
│
├── tests/
│   ├── test_ou_validator.py
│   └── test_file_manager.py
│
├── sample_data/
│   └── sample_essbase.xlsx
│
└── requirements.txt
```

> Keep real credentials, connection strings, secrets, and production
> configuration outside GitHub. Use Databricks Secrets, Azure Key Vault,
> environment variables, or another approved secret-management
> mechanism.

------------------------------------------------------------------------

# 11. OU Validation -- Example

A simple Python validation example:

``` python
import re

def validate_ou(ou):
    if ou is None:
        return False

    ou = str(ou).strip()

    return bool(re.fullmatch(r"\d{5}", ou))
```

Example:

``` python
validate_ou("06569")
# True

validate_ou("ABC12")
# False
```

### Important

When reading Excel data, avoid converting OU values into numeric types
because:

``` text
06569
```

could become:

``` text
6569
```

The automation must preserve the original leading zero.

------------------------------------------------------------------------

# 12. Error Handling

The pipeline should handle common failure scenarios such as:

### Source file missing

``` text
Essbase.xlsx not found
```

### Empty Excel sheet

``` text
No usable records found
```

### Invalid OU

``` text
Invalid OU detected
```

### File write failure

``` text
Unable to create output file
```

### Azure connection failure

``` text
Unable to access Blob Storage
```

Failures should be logged and, where appropriate, surfaced through the
notification mechanism.

------------------------------------------------------------------------

# 13. Security

Security considerations include:

-   Do not hard-code Azure credentials
-   Use Databricks Secrets for sensitive configuration
-   Apply least-privilege access to Blob Storage
-   Restrict access to production containers
-   Avoid committing credentials to GitHub
-   Use service principals/managed identities where appropriate
-   Keep production configuration separate from source code

------------------------------------------------------------------------

# 14. Business Benefits

The automation provides:

-   ✅ Automated OU validation
-   ✅ Reduced manual Excel processing
-   ✅ Consistent validation rules
-   ✅ Automatic valid/invalid data separation
-   ✅ Automated mapping-file generation
-   ✅ Automated file archival
-   ✅ Automated invalid-data notification
-   ✅ Controlled historical retention
-   ✅ Improved auditability
-   ✅ Reduced duplicate processing
-   ✅ Scalable Databricks-based processing

------------------------------------------------------------------------

# 15. End-to-End Summary

``` text
1. Read Essbase.xlsx from Azure Blob Storage
                 ↓
2. Select required Excel sheets
                 ↓
3. Read OU & Department columns
                 ↓
4. Remove empty records
                 ↓
5. Combine all sheets
                 ↓
6. Validate OU using Regex + Business Rules
                 ↓
        ┌────────┴────────┐
        ↓                 ↓
   Valid OU           Invalid OU
        ↓                 ↓
   Save CSV          Save Invalid Log
        ↓                 ↓
 Create Mapping       Logic App
      TXT             Notification
        ↓
 Archive Valid CSV
        ↓
 Archive Source Excel
        ↓
 Apply 13-Week Retention
```

------------------------------------------------------------------------

## 🚀 Future Enhancements

Potential enhancements include:

-   Parameterized Databricks jobs
-   Automated unit testing
-   Data-quality metrics
-   Azure Monitor integration
-   Pipeline execution monitoring
-   Detailed audit tables
-   Retry mechanisms
-   Schema validation
-   Centralized logging
-   CI/CD using GitHub Actions or Azure DevOps
-   Delta Lake-based audit history
-   Power BI monitoring dashboard

------------------------------------------------------------------------

## 👨‍💻 Project Focus

This project demonstrates a practical **Azure + Databricks data
engineering automation pattern** combining:

**Cloud Storage → Data Processing → Data Quality → File Management →
Notification → Retention**

It is designed to reduce manual operational effort while providing
consistent validation, traceability, and automated exception handling.
