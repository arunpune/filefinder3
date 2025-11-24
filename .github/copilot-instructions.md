# FileFinder Project - AI Agent Instructions

## Project Overview
FileFinder is a cross-platform (Windows/Linux) file scanning and analysis tool that inventories files across drives, detects sensitive data patterns, and stores metadata in MySQL. The tool supports both file counting and deep data scanning modes with Excel file content analysis.

## Architecture & Components

### Main Components
- **`FileFinder_19/file_info_version_22.py`**: Primary scanning engine with interactive CLI (1100+ lines)
- **`FileFinder_19/file_info_mapfolders.py`**: Simplified file type aggregation scanner
- **`FileFinder_19/machine_info_migration_centre.py`**: Excel-to-MySQL migration utility for machine inventory
- **`PowerShell/`**: PowerShell-based scanning alternative
- **`SQLScripts/`**: Database schema and queries

### Data Flow
1. User selects scan mode (File Count vs File Data Scan) and OS (Windows/Linux)
2. Drive/path selection → file walk with extension/date filtering → sensitivity detection
3. MySQL upsert pattern: `f_machine_files_summary_count` → `d_file_details` → `xls_file_sheet` → `xls_file_sheet_row`
4. Audit trail written to `audit_info` table
5. Optional: Application logs inserted to `app_log_file` table

## Configuration Pattern

### Environment Variables (`.env` or DB-driven)
- Set `ENABLE_ENV_FROM_DB=true` to pull config from `env_info` MySQL table instead of `.env`
- Key configs: `D_FILE_DETAILS_FILE_EXTENSIONS`, `FILE_PATH_SCAN_SENSITIVE_PATTERNS`, `N_DAYS`, `ENABLE_EXCEL_FILE_DATA_SCAN`
- Database connection: `MYSQL_HOST`, `MYSQL_PORT`, `MYSQL_DATABASE`, `MYSQL_USERNAME`, `MYSQL_PASSWORD`

**Critical**: Configuration is fetched via `retrieve_env_values()` which branches on `enable_env_from_db` flag. Always check both `.env` and `env_info` table when debugging config issues.

## Platform-Specific Patterns

### Windows
- Uses `psutil.disk_partitions()` to enumerate drives, filters removable drives
- File ownership via `win32security.GetFileSecurity()` → `win32security.LookupAccountSid()`
- Network shares enumerated with `win32net.NetShareEnum()` → stored in `d_shared_folders`

### Linux
- Single root scan starting from `/` 
- File ownership via `pwd.getpwuid(os.stat().st_uid).pw_name`
- IP address: `subprocess.run(['hostname', '-I'])` vs Windows `socket.gethostbyname()`

## Database Schema & Relationships

### Table Hierarchy (Parent → Child)
```
f_machine_files_summary_count (PK: f_machine_files_summary_count_pk, UK: hostname)
├── d_file_details (FK: f_machine_files_summary_count_fk, UK: [fk, file_path])
│   └── xls_file_sheet (FK: d_file_details_fk, UK: [fk, sheet_name])
│       └── xls_file_sheet_row (FK: xls_file_sheet_fk, UK: [fk, sheet_name, row_no])
├── d_shared_folders (FK: f_machine_files_summary_count_fk, UK: [fk, shared_folder_name])
├── audit_info (FK: f_machine_files_summary_count_fk)
└── app_log_file (FK: f_machine_files_summary_count_fk)
```

**Standalone Tables:**
- `env_info` - Configuration key-value pairs (alternative to `.env`)
- `f_machine_files_count_sp` - Aggregated file extension counts per host (UK: [hostname, file_extension])
- `machine_info_migration_center` - Machine inventory from Migration Center Excel imports (UK: name)

### Key Schema Details
- **`f_machine_files_summary_count`**: Root table storing per-machine summary (total file counts by extension: `total_n_xls`, `total_n_pdf`, etc., plus drive info)
- **`d_file_details`**: Individual file metadata (3.7M+ rows typical) - includes `file_is_sensitive_data` boolean
- **`xls_file_sheet`**: Excel sheet-level metadata (row/col counts)
- **`xls_file_sheet_row`**: First N rows of Excel data (10 cols max, truncated to 255 chars)
- **`d_shared_folders`**: Windows network shares enumerated via `win32net.NetShareEnum()`
- **`audit_info`**: Scan execution tracking (start/end times, duration, activity status)

### Database Interaction

#### Upsert Pattern (used throughout)
```python
cursor.execute('''
    INSERT INTO table_name (...) VALUES (...)
    ON DUPLICATE KEY UPDATE field=VALUES(field), ...
''')
```
All tables include audit columns: `row_creation_date_time`, `row_created_by`, `row_modification_date_time`, `row_modification_by` populated with `FROM_UNIXTIME(start_time)` and `employee_username`.

#### Foreign Key Pattern
Subqueries retrieve parent PKs inline:
```python
(SELECT f_machine_files_summary_count_pk FROM f_machine_files_summary_count WHERE hostname = %s)
```

#### Stored Procedures
- **`GetFileCount_FileSize_Summary`**: Returns file count/size summary by hostname + grand total
- **`sp_InsertOrUpdateFileCounts`**: Aggregates `d_file_details` by extension into `f_machine_files_count_sp`

## Key Workflows

### Development Setup
```powershell
cd FileFinder_19
# Install dependencies (Poetry-managed)
poetry install
# Or use pip with requirements.txt
pip install -r requirements.txt
# For Linux: use requirements - Linux.txt
pip install -r "requirements - Linux.txt"
```

### Running Scans

#### Python Interactive Scan
```powershell
python file_info_version_22.py
# Interactive prompts for:
# 1. Employee username (for audit trail)
# 2. Scan type: "File Count" (summary only) or "File Data Scan" (full metadata)
# 3. OS type: "Windows" or "Linux"
# 4. Drive selection: "All Drive Scan" or "Specific Drive Scan"
```

#### PowerShell Alternative (Windows Only)
```powershell
# Edit PowerShell/PS__ArunV2_Final_8Feb2024.ps1 to set:
# - $f_machine_files_summary_count_fk (manual FK reference)
# - $ip_address, $hostname
# - $drives array (e.g., @("H:\"))
# - $csvFile output path
.\PowerShell\PS__ArunV2_Final_8Feb2024.ps1
# Outputs CSV file ready for manual MySQL import
```

**PowerShell Differences:**
- No database write - exports CSV for manual import
- Requires manual FK setup (`$f_machine_files_summary_count_fk`)
- Filters by `$allowedExtensions` array (40+ extensions hardcoded)
- Uses `Get-Acl` for file ownership vs Python's `win32security`

### Building Executables
```powershell
# Using PyInstaller (specs exist for reference)
pyinstaller file_info_version_22.spec
# Standalone .exe created in dist/ folder
```

### Database Maintenance

#### Running Stored Procedures
```sql
-- Aggregate file counts by extension
CALL sp_InsertOrUpdateFileCounts();

-- Get file size summary
CALL GetFileCount_FileSize_Summary();
```

#### Querying Scan Results
```sql
-- Check latest scans
SELECT * FROM audit_info ORDER BY start_time DESC LIMIT 10;

-- Find sensitive files
SELECT hostname, file_path, file_owner 
FROM d_file_details 
WHERE file_is_sensitive_data = '1';

-- Shared folder analysis (see SQLScripts/sql_Cased_Dimensions14Jan2024_v2.sql for examples)
SELECT shared_folder_name, shared_folder_path, shared_folder_description
FROM d_shared_folders
WHERE shared_folder_name IN ('backups', 'admin', 'scan');
```

## Code Conventions

### Logging
- **`loguru`** logger (not standard logging module): `logger.success()`, `logger.error()`, `logger.warning()`
- Logs written to `{hostname}_{ip}.log` and optionally to `app_log_file` MySQL table
- Pattern: `logger.remove()` at startup to clear default handlers

### UI/UX
- **`rich`** library for colored console output: `print("[bright_green]...[/bright_green]")`
- **`questionary`** for interactive selection menus: `select("prompt", choices=[...]).ask()`
- Exit mechanism: `keyboard.is_pressed('Esc')` loop at end

### Error Handling
- Broad try-except blocks with `logger.error(f"...: {str(e)}", exc_info=True)`
- Database errors trigger rollback in specific functions (e.g., `insert_log_file_to_mysql`)
- File access errors caught during `os.walk()` and logged without halting scan

## Machine Migration Utility

### Purpose
`machine_info_migration_centre.py` imports machine inventory from Azure Migration Center Excel exports into MySQL.

### Workflow
```powershell
# 1. Place Migration Center export in FileFinder_19/pc_data_info.xlsx
# 2. Run migration script
python machine_info_migration_centre.py
# 3. Data filtered by groupType='Assessment' and upserted to machine_info_migration_center table
```

### Excel Schema Expected
- `name`, `createDate` (DD-Mon-YYYY format), `collectedIpAddress`
- `model`, `osName`, `processorCount`, `memoryInMb`, `driveTotalFreeInGb`

**Note:** This utility operates independently - no FK relationship to scan tables.

## Excel Scanning Feature
When `ENABLE_EXCEL_FILE_DATA_SCAN=true`:
1. Reads `.xls`/`.xlsx` with `pandas.read_excel(sheet_name=None)` 
2. Stores sheet metadata (row/col counts) in `xls_file_sheet`
3. Captures first N rows (configurable via `ENABLE_EXCEL_FILE_DATA_SCAN_MIN_ROW`) in `xls_file_sheet_row`
4. Truncates data to 10 columns max, 255 chars per cell
5. Sets `is_truncate='yes'` flag if >10 columns exist

## Sensitive File Detection
Function `is_sensitive_file()`:
- Only checks files matching `IS_SENSITIVE_FILE_EXTENSIONS`
- Searches filename and optionally file content for patterns in `FILE_PATH_SCAN_SENSITIVE_PATTERNS`
- Result stored in `file_is_sensitive_data` boolean column

## Common Gotchas
- **Global variables**: Config values like `d_file_details_file_extensions` are set globally after env fetch
- **Time handling**: All timestamps use `time.time()` (Unix epoch) → converted via `FROM_UNIXTIME()` in SQL
- **Path truncation**: File paths truncated to 759 chars, shared folder paths to 2499 chars before insert
- **Platform checks**: `platform.system()` returns 'Windows' or 'Linux' - code branches extensively on this

## Testing/Debugging
- No automated tests present - manual testing required
- Check `{hostname}_{ip}.log` for execution trace
- Query `audit_info` table for scan performance metrics
- Use `ENABLE_FILE_EXT_COUNT_IN_SCAN=true` for detailed extension counts (slower)

## Troubleshooting & Maintenance

### Failed Scans
1. Check `audit_info` table for `activity_status` = 'error' (default is 'error', set to 'Completed' on success)
2. Review log files: `{hostname}_{ip}.log` or query `app_log_file` table
3. Verify database connectivity and credentials in `.env` or `env_info` table

### Common Issues
- **"Database connection failed"**: Check MySQL service running, verify host/port/credentials
- **"Permission denied" on file scan**: Run as Administrator (Windows) or with `sudo` (Linux)
- **Excel scan hangs**: Large `.xlsx` files can timeout - adjust `ENABLE_EXCEL_FILE_DATA_SCAN_MIN_ROW` to reduce row count
- **Path truncation warnings**: File paths >759 chars or shared folder paths >2499 chars are truncated (check logs)

### Performance Optimization
- Set `ENABLE_FILE_EXT_COUNT_IN_SCAN=false` to skip per-extension counting (faster summary scans)
- Use "File Count" mode instead of "File Data Scan" for quick machine inventory
- Filter by `N_DAYS` to scan only recently modified files
- Exclude system drives in "Specific Drive Scan" mode

### Database Cleanup
```sql
-- Delete old scan data for a hostname
DELETE FROM d_file_details WHERE f_machine_files_summary_count_fk = (
    SELECT f_machine_files_summary_count_pk FROM f_machine_files_summary_count WHERE hostname = 'OLD_HOST'
);

-- Recalculate aggregated counts
CALL sp_InsertOrUpdateFileCounts();
```
