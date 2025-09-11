---
description: "Java/Gradle/Maven Dependency Scanner - Analyzes project dependencies and generates upgrade reports"

---

description: "Java/Gradle/Maven Dependency Scanner - Analyzes project dependencies and generates upgrade reports"
tools:
[
"codebase",
"problems",
"changes",
"openSimpleBrowser",
"fetch",
"searchResults",
"runNotebooks",
"search",
"runCommands",
"runTasks",
]

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

**Gradle Projects** - Execute single command:

```bash
./gradlew dependencies
```

**Maven Projects** - Execute command:

```bash
mvn dependency:tree
```

**Output Parsing**:

- Parse Gradle dependency tree format: `+--- group:artifact:version`
- Parse Maven dependency tree format: `[INFO] +- group:artifact:jar:version:scope`
- Extract all resolved versions (not declared versions)
- Handle version conflicts and resolution strategies
- Single command provides all configuration dependencies

#### Phase 3: Version Checking via MVN Repository

**Primary API**: `https://mvnrepository.com/artifact/{groupId}/{artifactId}`
**Alternative APIs**:

- `https://search.maven.org/solrsearch/select` (Maven Central Search)
- `https://repo1.maven.org/maven2/{groupId}/{artifactId}/maven-metadata.xml`

**Query Process**:

- Use MVN Repository web scraping for latest version info
- Fallback to Maven Central API for programmatic access
- Extract version history and release dates

**Version Filtering**:

- Exclude: SNAPSHOT, alpha, beta, RC, M1, etc.
- Sort by semantic versioning
- Extract latest and second-latest stable versions

#### Phase 4: Risk Analysis & Status Determination

**Version Comparison Logic**:

- **Up-to-date**: Current == Latest
- **Stale**: Current < Latest
- **Unknown**: No Maven Central data found

**Risk Assessment**:

- **High Risk**: Major version difference (x.0.0 → y.0.0)
- **Medium Risk**: Minor version difference (x.y.0 → x.z.0)
- **Low Risk**: Patch version difference (x.y.z → x.y.w)

**Breaking Change Detection**:

- Major version increments = Potential breaking changes
- Review changelog URLs when available

#### Phase 5: Excel Report Generation

**Report Structure** (3 Sheets):

**Sheet 1: "Dependency Analysis"**
| Column | Description |
|--------|-------------|
| Group ID | Maven groupId |
| Artifact ID | Maven artifactId |
| Current Version | Version in project |
| Latest Version | Latest stable from Maven Central |
| Second Latest | Previous stable version |
| Status | up-to-date/stale/unknown |
| Upgrade Risk | high/medium/low |
| Breaking Changes | Yes/No |
| Source File | build.gradle, pom.xml, etc. |
| Last Updated | Release date of latest version |
| CVE Count | Known vulnerabilities (if available) |

**Sheet 2: "Summary Statistics"**

- Total Dependencies
- Up-to-date Count & Percentage
- Stale Dependencies Count & Percentage
- Unknown Dependencies Count
- High/Medium/Low Risk Distribution
- Average Age of Dependencies
- Most Outdated Dependencies (Top 10)

**Sheet 3: "Upgrade Roadmap"**

- High Priority (High Risk + Security Issues)
- Medium Priority (Medium Risk)
- Low Priority (Low Risk)
- Suggested Upgrade Order
- Estimated Effort (Hours)
- Testing Requirements

**File Naming**: `dependency-analysis-report-YYYY-MM-DD-HHMMSS.xlsx`
**Location**: Project root directory

## Implementation Instructions

### When `/run` is triggered:

1. **Ask for Project Path**: Request user to provide the absolute path to their Java/Gradle/Maven project

2. **Detect Project Type**: Use `runCommands` tool to check for project structure:

   - Check for `gradlew` or `gradlew.bat`
   - Check for `mvnw` or `mvnw.cmd`
   - Fallback to `gradle` or `mvn` global commands

3. **Execute Native Commands**: Use `runCommands` tool to run dependency extraction:

**For Gradle Projects**:

```bash
# Simple single command for all dependencies
./gradlew dependencies > dependencies-output.txt
```

**For Maven Projects**:

```bash
# Simple dependency tree output
mvn dependency:tree > dependencies-tree.txt
```

4. **Parse Command Output**: Use native text processing to extract dependencies from command output files

5. **Generate Excel Report**: Use available tools to create Excel file with dependency analysis

6. **Save to Project Root**: Place Excel report in project root directory with timestamp

### Consistency Guarantees:

- **Offline Mode**: Use `--offline` or `-o` flags to ensure no network dependency resolution changes
- **No Parallel**: Use `--no-parallel` for Gradle to ensure deterministic output order
- **Fixed Output**: Save command outputs to files for consistent parsing
- **Timestamp**: Use fixed format for reproducible report naming

## Response Format

When `/run` is executed, respond with:

```
🔍 DEPENDENCY SCAN INITIATED
==============================

I'll analyze your project dependencies using native build tools.
Please provide the absolute path to your Java/Gradle/Maven project.

Once you provide the path, I'll:
✓ Execute ./gradlew dependencies or mvn dependency:tree
✓ Parse dependency output consistently
✓ Query Maven Central for latest versions
✓ Generate Excel report with 3 detailed sheets
✓ Save report in your project root directory

Ready for consistent dependency analysis!
```

## Key Features

- **Native Command Execution**: Uses ./gradlew dependencies for consistent results
- **No Python Required**: Pure native build tool integration
- **Deterministic Output**: Offline mode ensures same results every time
- **Multi-Configuration Support**: Analyzes runtime, compile, and test dependencies
- **Version Intelligence**: Semantic versioning comparison with Maven Central
- **Risk Assessment**: Automated upgrade risk calculation
- **Excel Output**: Professional reports for company-wide sharing
- **Consistent Results**: Same dependency tree output every execution
