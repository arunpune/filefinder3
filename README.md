# FileFinder - Enterprise File Inventory & Analysis Tool

**Version 6** | Last Updated: February 26, 2024

FileFinder is a cross-platform (Windows/Linux) file scanning and analysis tool that inventories files across drives, detects sensitive data patterns, analyzes Excel file content, and stores comprehensive metadata in MySQL for reporting and analysis.

## 📋 Table of Contents

- [Overview](#overview)
- [Contact Information](#contact-information)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [MySQL Database Setup](#mysql-database-setup)
  - [Python Environment Setup](#python-environment-setup)
- [Configuration](#configuration)
- [Usage](#usage)
- [Building Executable](#building-executable)
- [PowerBI Integration](#powerbi-integration)
- [Troubleshooting](#troubleshooting)
- [Architecture](#architecture)

---

## Overview

FileFinder scans file systems across Windows and Linux machines to:
- 📂 Inventory files across all drives
- 🔍 Detect sensitive data patterns (passwords, credit cards, etc.)
- 📊 Analyze Excel file contents (sheets and rows)
- 💾 Store metadata in MySQL database
- 📈 Generate reports via PowerBI
- 🔐 Track file ownership and permissions
- 🗂️ Enumerate network shares (Windows)

---

## Contact Information

For any inquiries or assistance, please contact:

📧 **Email**: arunkumar.nair@canspirit.ai

---

## Prerequisites

### System Requirements

- **Operating System**: Windows 10/11 or Linux (Ubuntu 20.04+)
- **Python**: 3.11 or higher
- **MySQL**: 8.0 or higher
- **RAM**: 4GB minimum (8GB recommended)
- **Disk Space**: 2GB for application + space for database

### Required Software

1. **Python 3.11+** - [Download Python](https://www.python.org/downloads/)
2. **MySQL Server 8.0+** - [Download MySQL](https://dev.mysql.com/downloads/mysql/)
3. **Visual Studio Code** (recommended) - [Download VS Code](https://code.visualstudio.com/)
4. **PowerBI Desktop** (optional, for reporting) - Available from Microsoft Store

---

## Installation

### MySQL Database Setup

#### Step 1: Configure MySQL for Remote Access

1. Navigate to the MySQL configuration directory:
   ```
   C:\ProgramData\MySQL\MySQL Server 8.0
   ```

2. Locate and edit the `my.ini` file (make it writable if needed)

3. Add the following line to allow connections from any IP address:
   ```ini
   bind-address=0.0.0.0
   ```

4. Restart MySQL service:
   ```powershell
   # Windows
   Restart-Service MySQL80
   
   # Or via Services GUI
   services.msc → MySQL80 → Restart
   ```

#### Step 2: Create Database User and Grant Permissions

1. Open **MySQL Workbench** or connect via command line:
   ```powershell
   mysql -u root -p
   ```

2. Execute the following SQL script to create the application user:
   ```sql
   CREATE USER 'arungt'@'localhost' IDENTIFIED BY 'fi!ef!ndgt!23';
   GRANT ALL PRIVILEGES ON *.* TO 'arungt'@'localhost' WITH GRANT OPTION;
   FLUSH PRIVILEGES;
   SHOW GRANTS FOR 'arungt'@'localhost';
   ```

#### Step 3: Import Database Schema

1. Locate the SQL schema file: `1_sql_file_info_create.sql` (in SQLScripts folder)

2. Execute the script in MySQL Workbench:
   ```sql
   SOURCE /path/to/1_sql_file_info_create.sql;
   ```
   
   Or via command line:
   ```powershell
   mysql -u arungt -p < 1_sql_file_info_create.sql
   ```

3. Verify tables were created:
   ```sql
   USE lisney_files_info8;
   SHOW TABLES;
   ```

Expected tables:
- `f_machine_files_summary_count`
- `d_file_details`
- `d_shared_folders`
- `xls_file_sheet`
- `xls_file_sheet_row`
- `audit_info`
- `app_log_file`
- `env_info`
- `machine_info_migration_center`
- `f_machine_files_count_sp`

---

### Python Environment Setup

#### Step 1: Clone or Download the Repository

```powershell
cd C:\Projects  # Or your preferred directory
git clone <repository-url>
cd filefinder3\FileFinder_19
```

#### Step 2: Create Virtual Environment

Navigate to the FileFinder directory and create a virtual environment:

```powershell
# Navigate to the project directory
cd <filefinder-directory>

# Create virtual environment
python -m venv venv
```

#### Step 3: Activate Virtual Environment

**Windows (PowerShell)**:
```powershell
.\venv\Scripts\activate
```

**Windows (Command Prompt)**:
```cmd
venv\Scripts\activate.bat
```

**Linux/Mac**:
```bash
source venv/bin/activate
```

#### Step 4: Install Dependencies

```powershell
pip install -r requirements.txt
```

For Linux, use:
```bash
pip install -r "requirements - Linux.txt"
```

#### Step 5: Verify Installation

```powershell
python --version  # Should show Python 3.11+
pip list          # Shows installed packages
```

---

## Configuration

### Environment Variables (.env file)

The `.env` file contains all configuration settings. It must be in the same directory as the Python script.

**Location**: `FileFinder_19/.env`

```properties
# MySQL Database Connection
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_DATABASE=lisney_files_info8
MYSQL_USERNAME=arungt
MYSQL_PASSWORD=fi!ef!ndgt!23

# Configuration Source
ENABLE_ENV_FROM_DB=false
# Set to 'true' to load config from env_info table instead of .env file

# Application Logging
ENABLE_APP_LOG_TO_DB=true
# Store application logs in app_log_file table

# File Scanning Settings
D_FILE_DETAILS_FILE_EXTENSIONS=.xlsx,.xls,.pdf,.doc,.docx,.txt
# Comma-separated list of extensions to scan
# Use "all" to scan all file types (includes system files)

N_DAYS=0
# Number of days to filter files by modification date
# 0 = scan all files regardless of date, it does not scan all the files
# >0 = Use scan only files modified in last N days, to scan all the files

# Extension Counting
ENABLE_FILE_EXT_COUNT_IN_SCAN=false
# true = count files by extension (slower, more detailed)
# false = only count total files (faster)

# Sensitive Data Detection
IS_SENSITIVE_FILE_EXTENSIONS=.xls,.xlsx,.doc,.docx,.pdf
FILE_PATH_SCAN_SENSITIVE_PATTERNS=password,creditcard,ssn,confidential
# Keywords to identify sensitive files

# Excel Content Scanning
ENABLE_EXCEL_FILE_DATA_SCAN=false
# true = read Excel contents and store in database
# false = only scan Excel metadata

ENABLE_EXCEL_FILE_DATA_SCAN_MIN_ROW=3
# Number of rows to extract from each Excel sheet
```

### Configuration Checklist

Before running FileFinder, verify:

- ✅ MySQL server is running
- ✅ Database `lisney_files_info8` exists
- ✅ User `arungt` has proper permissions
- ✅ `.env` file has correct credentials
- ✅ Virtual environment is activated
- ✅ All dependencies are installed

---

## Usage

### Running FileFinder

1. **Activate Virtual Environment** (if not already activated):
   ```powershell
   .\venv\Scripts\activate
   ```

2. **Run the Application**:
   ```powershell
   python file_info_version_22.py
   ```

3. **Follow Interactive Prompts**:

   **Step 1**: Enter your employee username
   ```
   Enter your Employee username: john.doe
   ```

   **Step 2**: Select scan type
   - **File Count** - Quick scan, counts files by type
   - **File Data Scan** - Detailed scan, extracts full metadata

   **Step 3**: Select Operating System
   - **Windows** - Scans all drives and network shares
   - **Linux** - Scans from root (/) or specific path

   **Step 4**: Choose scan scope
   - **All Drive Scan** - Scans all detected drives
   - **Specific Drive Scan** - Scan a single drive/path

4. **Monitor Progress**:
   - Watch console output for progress messages
   - Log file is created: `{hostname}_{ip}.log`
   - Press `Esc` when prompted to exit

### Example Workflow

```powershell
PS C:\Projects\filefinder3\FileFinder_19> python file_info_version_22.py

Your IP Address: 192.168.1.100
Your Host Name: DESKTOP-PC
Operating System: Windows
OS Version: 10

Enter your Employee username: john.doe
? Select the type of scan: File Data Scan
? Select the Operating System: Windows

Drives Detected on this PC:
1. C:\
2. D:\
3. E:\ (removable)

? Select the type of scan: All Drive Scan

Performing a full scan for data files...
The Tool is now scanning for Data Files. Please Wait...
```

---

## Building Executable

To create a standalone `.exe` file that doesn't require Python installation:

### Step 1: Install PyInstaller

```powershell
pip install pyinstaller
```

### Step 2: Build Executable

```powershell
pyinstaller --onefile file_info_version_22.py
```

Or use the provided spec file:
```powershell
pyinstaller file_info_version_22.spec
```

### Step 3: Locate Output Files

After successful build, the executable will be in:
```
dist/
  └── file_info_version_22.exe
```

### Step 4: Deploy Executable

For deployment to other machines, copy these files together:
1. `file_info_version_22.exe` (from `dist/` folder)
2. `.env` (configuration file)
3. `1_sql_file_info_create.sql` (database schema)

**Important**: The `.env` file MUST be in the same directory as the `.exe` file.

### Step 5: Run Executable

```powershell
.\file_info_version_22.exe
```

The executable provides the same interactive interface as the Python script.

---

## PowerBI Integration

### Install PowerBI

1. **Download PowerBI Desktop**:
   - Open **Microsoft Store**
   - Search for "Power BI Desktop"
   - Click **Install**

2. **Install MySQL Connector**:
   - Download [MySQL Connector/NET](https://dev.mysql.com/downloads/connector/net/)
   - Install the connector
   - Restart PowerBI Desktop

### Connect to FileFinder Database

1. **Open PowerBI Desktop**

2. **Get Data from MySQL**:
   - Click **Get Data** → **Database** → **MySQL database**
   - Enter connection details:
     - **Server**: `localhost:3306`
     - **Database**: `lisney_files_info8`

3. **Enter Credentials**:
   - **Username**: `arungt`
   - **Password**: `fi!ef!ndgt!23`

4. **Select Tables**:
   - Choose tables to import:
     - `f_machine_files_summary_count`
     - `d_file_details`
     - `d_shared_folders`
     - `audit_info`

5. **Configure Security Settings**:
   - Go to **File** → **Options and Settings** → **Options**
   - Navigate to **Security**
   - Under **Privacy levels**, select **"Ignore Privacy Levels"** (Not Recommended)
   - Click **OK**

   ⚠️ **Note**: This setting is needed for cross-database queries but may reduce security.

### Sample PowerBI Queries

```sql
-- File count by extension per machine
SELECT 
    hostname,
    SUM(total_n_xls) as Excel_Files,
    SUM(total_n_pdf) as PDF_Files,
    SUM(total_n_doc + total_n_docx) as Word_Files
FROM f_machine_files_summary_count
GROUP BY hostname;

-- Sensitive files by owner
SELECT 
    file_owner,
    COUNT(*) as sensitive_file_count,
    SUM(file_size_bytes) as total_size_bytes
FROM d_file_details
WHERE file_is_sensitive_data = '1'
GROUP BY file_owner;

-- Recent scans
SELECT 
    hostname,
    employee_username,
    start_time,
    end_time,
    duration,
    activity_status
FROM audit_info
ORDER BY start_time DESC;
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. Virtual Environment Not Activating

**Error**: `Execution of scripts is disabled on this system`

**Solution**:
```powershell
Set-ExecutionPolicy Unrestricted -Force
```

Then retry activating the virtual environment.

#### 2. Long Path Issues (Windows)

**Error**: `Path is too long` or file access errors

**Solution**:
Enable long paths in Windows Registry:

1. Open Registry Editor (`Win + R` → `regedit`)
2. Navigate to:
   ```
   Computer\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\FileSystem
   ```
3. Set `LongPathsEnabled` to `1`
4. Restart computer

Or via PowerShell (Administrator):
```powershell
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
-Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

#### 3. MySQL Connection Failed

**Error**: `Can't connect to MySQL server`

**Checklist**:
- ✅ MySQL service is running: `services.msc` → MySQL80
- ✅ Check `.env` credentials match MySQL user
- ✅ Database exists: `SHOW DATABASES LIKE 'lisney_files_info8';`
- ✅ User has permissions: `SHOW GRANTS FOR 'arungt'@'localhost';`
- ✅ Firewall allows port 3306

#### 4. Permission Denied During Scan

**Error**: `Permission denied` when scanning system folders

**Solution**:
- **Windows**: Run Command Prompt/PowerShell as Administrator
- **Linux**: Run with `sudo`: `sudo python3 file_info_version_22.py`

#### 5. Excel Scan Hangs

**Issue**: Application freezes when scanning large Excel files

**Solutions**:
- Set `ENABLE_EXCEL_FILE_DATA_SCAN=false` in `.env`
- Reduce `ENABLE_EXCEL_FILE_DATA_SCAN_MIN_ROW` to `3` or lower
- Exclude Excel files from scan: Remove `.xls,.xlsx` from `D_FILE_DETAILS_FILE_EXTENSIONS`

#### 6. PyInstaller Build Fails

**Error**: Module import errors during build

**Solution**:
```powershell
# Clear PyInstaller cache
pyinstaller --clean file_info_version_22.spec

# Or rebuild from scratch
rmdir -r build dist
pyinstaller file_info_version_22.spec
```

---

## Architecture

### System Components

```
┌─────────────────┐
│  FileFinder.exe │ ← Python Application
│  or .py script  │
└────────┬────────┘
         │
         ├──→ File System Scanner
         │    ├── Drive Detection (psutil)
         │    ├── File Walk (os.walk)
         │    └── Metadata Extraction
         │
         ├──→ Sensitive Data Detector
         │    └── Pattern Matching
         │
         ├──→ Excel Analyzer
         │    └── pandas + openpyxl
         │
         └──→ MySQL Database
              ├── f_machine_files_summary_count
              ├── d_file_details (3M+ rows)
              ├── xls_file_sheet
              ├── xls_file_sheet_row
              ├── d_shared_folders
              └── audit_info
```

### Data Flow

1. **Configuration Loading** → Load from `.env` or `env_info` table
2. **Drive Detection** → Enumerate drives using `psutil`
3. **File Scanning** → Walk directory tree with `os.walk()`
4. **Metadata Extraction** → Get file size, owner, timestamps
5. **Sensitive Detection** → Check filenames/content for patterns
6. **Excel Analysis** → Read sheets and rows (if enabled)
7. **Database Upsert** → Insert/update MySQL tables
8. **Audit Logging** → Record scan execution details

### Database Schema Summary

| Table | Purpose | Rows (Typical) |
|-------|---------|----------------|
| `f_machine_files_summary_count` | Machine-level summary | 1 per machine |
| `d_file_details` | Individual file metadata | 3,000,000+ |
| `xls_file_sheet` | Excel sheet metadata | 100,000+ |
| `xls_file_sheet_row` | Excel row data | 500,000+ |
| `d_shared_folders` | Network shares | 50-100 |
| `audit_info` | Scan execution log | 1 per scan |

---

## Additional Resources

### File Structure

```
filefinder3/
├── .github/
│   └── copilot-instructions.md
├── FileFinder_19/
│   ├── file_info_version_22.py       # Main scanner (1100+ lines)
│   ├── file_info_mapfolders.py       # Simplified scanner
│   ├── machine_info_migration_centre.py  # Azure migration import
│   ├── .env                           # Configuration
│   ├── pyproject.toml                 # Poetry dependencies
│   ├── requirements.txt               # Windows dependencies
│   ├── requirements - Linux.txt       # Linux dependencies
│   └── *.spec                         # PyInstaller specs
├── PowerShell/
│   └── PS__ArunV2_Final_8Feb2024.ps1  # PowerShell alternative
├── SQLScripts/
│   ├── 1_sql_file_info_create.sql     # Database schema
│   └── sql_Cased_Dimensions14Jan2024_v2.sql
└── README.md                          # This file
```

### Useful Commands

```powershell
# Check Python version
python --version

# Check MySQL status
sc query MySQL80

# View installed packages
pip list

# Test database connection
mysql -u arungt -p -e "SHOW DATABASES;"

# Check log files
Get-Content *.log -Tail 50

# Monitor scan progress
Get-Content {hostname}_{ip}.log -Wait
```

---

## License

Copyright © 2024. All rights reserved.

---

## Version History

- **v6** (Feb 2024): Database-driven config, PowerBI integration
- **v22** (Feb 2024): Excel content scanning
- **v19** (Nov 2023): Initial Windows/Linux support

---

**Last Updated**: February 26, 2024  
**Maintained By**: IT Asset Management Team
