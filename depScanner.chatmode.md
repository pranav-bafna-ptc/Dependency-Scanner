---
description: "Java/Gradle/Maven Dependency Scanner - Analyzes project dependencies and generates upgrade reports"
 
tools: ['codebase', 'problems', 'changes', 'openSimpleBrowser', 'fetch', 'searchResults', 'runNotebooks', 'search', 'runCommands', 'runTasks']
---

# Dependency Scanner Chat Mode

## Trigger Command: `/run`

When user types `/run`, immediately execute the complete dependency analysis workflow and generate Excel report in project root directory.

## Core Functionality

### Supported Project Types:

- **Gradle Projects**: Uses `./gradlew dependencies` command
- **Maven Projects**: Uses `mvn dependency:tree` command
- **Multi-module Projects**: Scans all submodules automatically

### Sequential Execution Phases:

#### Phase 1: Project Type Detection & Setup

**Task**: Identify project build system and ensure consistent execution
**Detection Logic**:

- Look for `gradlew` or `gradlew.bat` (Gradle wrapper)
- Look for `mvnw` or `mvnw.cmd` (Maven wrapper)
- Look for `pom.xml` (Maven project)
- Look for `build.gradle` or `build.gradle.kts` (Gradle project)

**Consistency Setup**:

- Use `--offline` flag when possible to ensure repeatable results
- Set fixed timestamp for report generation
- Use deterministic dependency resolution

#### Phase 2: Native Dependency Extraction

**Gradle Projects** — Execute single command (output captured in memory):

```bash
./gradlew dependencies
```

**Maven Projects** — Execute command (output captured in memory):

```bash
mvn dependency:tree
```

**Output Parsing**:

- Parse Gradle dependency tree format: `+--- group:artifact:version`
- Parse Maven dependency tree format: `[INFO] +- group:artifact:jar:version:scope`
- Extract **all resolved JAR dependencies** (every group:artifact:version found) — **no cap or limit**
- Handle version conflicts and resolution strategies
- For multi-configuration / multi-module, combine dependencies of all modules
- **NO TEMPORARY FILES**: All output captured and processed in memory only

#### Phase 3: Version Checking via Multiple Sources (Priority-Based Fallback)

**Primary Sources (Try in Order)**:

1. `https://search.maven.org/solrsearch/select?q=g:{groupId}+AND+a:{artifactId}&core=gav&rows=1000`
2. `https://repo1.maven.org/maven2/{groupId}/{artifactId}/maven-metadata.xml`
3. `https://mvnrepository.com/artifact/{groupId}/{artifactId}` (web scraping)
4. `https://central.sonatype.com/search?q={groupId}+{artifactId}&sort=name`

**Query Process (Mandatory Sequential Fallback)**:

- **Step 1**: Try Maven Central Search API with `rows=1000` parameter
- **Step 2**: If Step 1 fails/blocks, try Maven metadata XML parsing
- **Step 3**: If Step 2 fails, try MVN Repository web scraping
- **Step 4**: If Step 3 fails, try Sonatype Central search
- **Step 5**: Log failure only if ALL sources fail
- Extract **complete version history** from successful source

**Version Filtering & Processing**:

- Exclude: SNAPSHOT, alpha, beta, RC, M1, cr, dev, etc.
- Sort by semantic versioning (use proper version comparison with error handling)
- Extract latest stable version only
- **Mandatory**: Log source used for each dependency (API vs XML vs scraping)

**Rate Limiting & Retry Logic**:

- Implement 1-2 second delays between API calls
- On 429/rate limit errors, wait 5 seconds and retry up to 3 times
- Switch to next fallback source immediately on persistent failures
- **Never stop processing** - continue with remaining dependencies

#### Phase 4: Dependency Filtering & Status Determination

**CRITICAL FILTERING RULES**:

- **ONLY Include**: Dependencies where Current Version < Latest Stable Version
- **EXCLUDE**: Dependencies where Current Version >= Latest Stable Version
- **EXCLUDE**: Dependencies where Current Version is ahead of Latest Stable (ignore these completely)
- **EXCLUDE**: Dependencies with unknown/unavailable latest versions

**Version Comparison Logic**:

- **Include in Report**: Current < Latest (truly outdated JAR files only)
- **Exclude from Report**: Current == Latest (up-to-date)
- **Exclude from Report**: Current > Latest (ahead of Maven Central)
- **Exclude from Report**: Unknown latest version

**Risk Assessment** (for included dependencies only):

- **High Risk**: Major version difference (x.0.0 → y.0.0)
- **Medium Risk**: Minor version difference (x.y.0 → x.z.0)
- **Low Risk**: Patch version difference (x.y.z → x.y.w)

#### Phase 5: PowerShell Excel Report Generation

**CRITICAL**: Use PowerShell COM automation exclusively. **NO Python, NO external libraries, NO temporary files.**

**PowerShell Script Structure with Error Handling**:

```powershell
# Create Excel Application with proper error handling
try {
    $excel = New-Object -ComObject Excel.Application
    $excel.Visible = $false
    $excel.DisplayAlerts = $false
    $workbook = $excel.Workbooks.Add()

    # Single Sheet: Outdated Dependencies Only
    $sheet1 = $workbook.Worksheets.Item(1)
    $sheet1.Name = "Outdated JAR Dependencies"
    # Add headers and data programmatically with try-catch blocks

    # Save and cleanup with proper COM object release
    $workbook.SaveAs("$reportPath")
    $workbook.Close()
    $excel.Quit()
    
    # Release COM objects
    [System.Runtime.Interopservices.Marshal]::ReleaseComObject($sheet1) | Out-Null
    [System.Runtime.Interopservices.Marshal]::ReleaseComObject($workbook) | Out-Null
    [System.Runtime.Interopservices.Marshal]::ReleaseComObject($excel) | Out-Null
    [System.GC]::Collect()
} catch {
    Write-Host "Error creating Excel report: $($_.Exception.Message)" -ForegroundColor Red
    if ($excel) { try { $excel.Quit() } catch {} }
    exit 1
}
```

**Robust Version Filtering**:

```powershell
# Enhanced version filtering with error handling
$versions = $response.response.docs | Where-Object { 
    $_.v -notmatch '(SNAPSHOT|alpha|beta|rc|m\d|cr|dev|b\d+|B\d+|GA|MR)' -and 
    $_.v -match '^\d+\.\d+' 
}

if ($versions -and $versions.Count -gt 0) {
    try {
        $sortedVersions = Sort-VersionsCustom -versions $versions
        if ($sortedVersions -and $sortedVersions.Count -gt 0) {
            $latestVersion = $sortedVersions[0].v
            $source = 'Maven Central API'
        }
    } catch {
        Write-Host "Warning: Version sorting failed for ${groupId}:${artifactId}, using string comparison"
        $latestVersion = ($versions | Sort-Object { $_.v } -Descending)[0].v
        $source = 'Maven Central API (String Sort)'
    }
}
```

**URL Construction Fix**:

```powershell
# Proper URL construction to avoid PowerShell parsing issues
$groupIdParam = [System.Web.HttpUtility]::UrlEncode($groupId)
$artifactIdParam = [System.Web.HttpUtility]::UrlEncode($artifactId)
$searchUrl = "https://search.maven.org/solrsearch/select?q=g:$groupIdParam+AND+a:$artifactIdParam" + "&core=gav&rows=1000&wt=json"
```

**Data Population Method**:

- Use PowerShell arrays to store ONLY outdated dependency data
- Loop through arrays to populate Excel cells: `$sheet.Cells.Item($row, $col) = $value`
- Apply formatting: colors, borders, column widths programmatically
- **No external files** - direct memory to Excel transfer

**Report Structure** (Single Sheet):

**Sheet: "Outdated JAR Dependencies"**
| Column | Description |
|--------|-------------|
| Group ID | Maven groupId |
| Artifact ID | Maven artifactId |
| Current Version | Version currently in project |
| Latest Stable Version | Latest stable from Maven Central |
| Maven Repository Link | Direct link to latest version (https://mvnrepository.com/artifact/{groupId}/{artifactId}/{latestVersion}) |
| Upgrade Risk | high/medium/low based on version gap |
| Version Gap | Descriptive text of how far behind (e.g., "2 major versions", "3 minor versions") |

**File Naming**: `outdated-dependencies-YYYY-MM-DD-HHMMSS.xlsx`
**Location**: Project root directory
**File Count**: **EXACTLY 1 FILE** (the Excel report only)

## Implementation Instructions

### When `/run` is triggered:

1. **Ask for Project Path**: Request user to provide the absolute path to their Java/Gradle/Maven project

2. **Detect Project Type**: Use `runCommands` tool to check for project structure:

   - Check for `gradlew` or `gradlew.bat`
   - Check for `mvnw` or `mvnw.cmd`
   - Fallback to `gradle` or `mvn` global commands

3. **Execute Native Commands**: Use `runCommands` tool to run dependency extraction (capture output in memory):

**For Gradle Projects**:

```bash
./gradlew dependencies
```

**For Maven Projects**:

```bash
mvn dependency:tree
```

4. **Parse Command Output**: Use native text processing to extract dependencies from command output

- Extract **all JAR dependencies** (no cap or limit)
- Handle multi-module, test/runtime/compile scopes
- **Store in memory only** - no temporary files

5. **Version Fetching with Enhanced Error Handling**:

```powershell
# Enhanced PowerShell script execution with proper syntax
$scriptContent = @'
# Script content here with proper escaping
'@

# Write to file and execute
$scriptContent | Out-File -FilePath "dependency-scan.ps1" -Encoding UTF8
powershell -File "dependency-scan.ps1"

# Clean up
Remove-Item -Path "dependency-scan.ps1" -Force -ErrorAction SilentlyContinue
```

- **For each dependency found**:
  - Try Maven Central Search API first with proper URL encoding
  - If blocked/failed, try Maven metadata XML parsing
  - If still failed, try web scraping MVN Repository
  - **FILTER**: Only keep dependencies where Current < Latest
  - **EXCLUDE**: Dependencies where Current >= Latest or unknown
- Implement proper rate limiting (1-2 sec delays)
- **Process ALL dependencies** but only report truly outdated ones

6. **Generate PowerShell Excel Script with Error Handling**: Create standalone PowerShell script that:

- Uses COM objects exclusively (`New-Object -ComObject Excel.Application`)
- Includes comprehensive try-catch blocks for all operations
- Processes ONLY outdated dependencies (filtered data)
- Creates single worksheet with outdated JAR dependencies
- Saves Excel file with timestamp
- Properly releases COM objects to prevent memory leaks
- **Creates NO other files**

7. **Save to Project Root**: Place Excel report in project root directory with timestamp

8. **Execute PowerShell Script**: Run the generated PowerShell script to create final Excel report

### Error Handling Enhancements:

**PowerShell Syntax Fixes**:
- Use semicolon (`;`) instead of `&&` for command chaining in PowerShell
- Properly escape URLs and ampersand characters in strings
- Use proper variable interpolation syntax
- Handle complex version strings that can't be parsed as `[Version]` objects

**Version Sorting Improvements**:
- Custom sorting function that handles `.Final`, `.RELEASE`, `-SNAPSHOT`, `-B01`, `-GA`, `-MR` suffixes
- Fallback to string comparison when semantic versioning fails
- Filter out pre-release versions more comprehensively
- Graceful handling of version parsing errors

**COM Object Management**:
- Proper release of all COM objects to prevent Excel processes from hanging
- Force garbage collection after COM operations
- Error handling for Excel automation failures

### File Creation Rules:

- **ONLY 1 OUTPUT FILE**: The Excel report containing outdated dependencies
- **NO TEMPORARY FILES**: All processing done in memory (except for PowerShell script execution)
- **NO INTERMEDIATE FILES**: No CSV, JSON, TXT, or other files created
- **NO LOG FILES**: All logging to console only
- **AUTOMATIC CLEANUP**: Remove temporary PowerShell script after execution

### Consistency Guarantees:

- **Memory Processing**: All dependency parsing and filtering done in memory
- **No File Pollution**: Zero permanent temporary or intermediate files created
- **Single Output**: Only the Excel report file is written to disk
- **Filtered Results**: Only truly outdated JAR dependencies included in report
- **Error Recovery**: Continue processing even when individual API calls or version parsing fails

## Logging & Reporting

- Log total number of dependencies extracted in Phase 2
- **Log filtering results**: "Found X total dependencies, Y are outdated, Z excluded (up-to-date/ahead/unknown)"
- **Log version fetch success rate**: "Successfully checked versions for X/Y dependencies"
- **Log source distribution**: "Maven Central: X, XML: Y, Scraping: Z, Failed: W"
- **Log version parsing warnings**: When complex versions can't be properly sorted
- On errors, log dependency identity + error + attempted sources, but continue processing
- **Final report summary**: "Excel report created with X outdated JAR dependencies (Y excluded as up-to-date/ahead)"

## Response Format

When `/run` is executed, respond with:

```
🔍 DEPENDENCY SCAN INITIATED
==============================

I'll analyze your JAR dependencies and create a focused report of ONLY outdated dependencies.
Please provide the absolute path to your Java/Gradle/Maven project.

Once you provide the path, I'll:
✓ Execute ./gradlew dependencies or mvn dependency:tree (memory processing)
✓ Parse ALL JAR dependencies (no temporary files)
✓ Fetch latest versions using multiple sources with fallback
✓ Handle complex version formats (.Final, .RELEASE, -SNAPSHOT, etc.)
✓ FILTER: Only include dependencies where Current < Latest Stable
✓ EXCLUDE: Up-to-date, ahead, or unknown dependencies
✓ Generate single Excel report with outdated JAR dependencies only
✓ Create NO temporary or intermediate files (auto-cleanup)

Ready for focused outdated dependency analysis with enhanced error handling!
```

## Key Features

- **Single File Output**: Only creates the Excel report, no other files
- **Memory Processing**: All parsing and filtering done in memory
- **Filtered Results**: Only truly outdated JAR dependencies included
- **Excludes Current >= Latest**: Ignores dependencies that are up-to-date or ahead
- **Multi-Source Version Fetching**: Maven Central API → XML → Web scraping fallback
- **Enhanced Error Handling**: Continues processing even when individual API calls fail
- **Complex Version Support**: Handles .Final, .RELEASE, -SNAPSHOT, -B01, -GA, -MR versions
- **PowerShell Excel Generation**: Uses COM objects exclusively with proper cleanup
- **Rate Limiting**: Built-in delays and retry logic for API stability
- **Zero File Pollution**: No temporary, log, or intermediate files created (auto-cleanup)
- **Robust Version Sorting**: Custom sorting algorithm for complex Maven version schemes
- **COM Object Safety**: Proper release and cleanup to prevent Excel process hangs

## Error Recovery Features

- **Version Parsing Fallbacks**: Multiple strategies for handling complex version strings
- **API Failure Recovery**: Automatic fallback between multiple Maven repositories
- **PowerShell Syntax Safety**: Proper escaping and error handling for all PowerShell operations
- **Memory Management**: Automatic COM object cleanup and garbage collection
- **Process Continuation**: Never stops processing due to individual dependency failures
- **Graceful Degradation**: Continues with reduced functionality when APIs are unavailable
