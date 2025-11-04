# SonarQube Scanning Process - Step by Step Guide

This document explains the complete SonarQube code analysis and coverage scanning process implemented in the Azure DevOps pipeline (`bff.yaml`).

## Table of Contents

1. [Overview](#overview)
2. [Branch Types and Analysis Modes](#branch-types-and-analysis-modes)
3. [Complete Step-by-Step Process](#complete-step-by-step-process)
4. [Coverage Generation and Processing](#coverage-generation-and-processing)
5. [SonarQube Analysis Configuration](#sonarqube-analysis-configuration)
6. [Quality Gate Evaluation](#quality-gate-evaluation)
7. [Troubleshooting](#troubleshooting)

---

## Overview

The pipeline performs comprehensive code quality analysis using SonarQube, including:
- **Code Analysis**: Detects bugs, vulnerabilities, code smells, and security hotspots
- **Coverage Analysis**: Measures test coverage using Jest-generated LCOV reports
- **Code Duplication**: Identifies duplicated code blocks
- **Quality Ratings**: Calculates reliability, security, and maintainability ratings
- **Quality Gate**: Evaluates code against predefined quality standards

---

## Branch Types and Analysis Modes

The pipeline adapts its behavior based on the branch type:

### 1. **Pull Request (PR)**
- **Analysis Mode**: New Code Analysis (compared to target branch)
- **Quality Gate**: **ENFORCED** - Build fails if quality gate doesn't pass
- **Coverage Threshold**: 80% minimum for new code
- **Purpose**: Prevent merging code that doesn't meet quality standards

### 2. **Feature Branch** (e.g., `feature/*`, `feat/*`)
- **Analysis Mode**: New Code Analysis (compared to `develop` branch)
- **Quality Gate**: **ENFORCED** for PRs, **Informational** for direct pushes
- **Reference Branch**: `develop`
- **Coverage Threshold**: 80% minimum for new code
- **Purpose**: Track quality of new changes compared to develop

### 3. **Develop Branch** (and other non-feature branches)
- **Analysis Mode**: Full Repository Analysis (all code)
- **Quality Gate**: **Informational** - Build continues even if gate fails
- **Coverage**: Overall code coverage percentage
- **Purpose**: Establish baseline coverage for feature branches to compare against

---

## Complete Step-by-Step Process

### Phase 1: Pre-Analysis Configuration

#### Step 1.1: Display Configuration Debug Information
**Task**: `SonarQube: Debug Configuration`

- Displays branch type detection
- Shows analysis mode (New Code vs Full Repository)
- Lists quality gate thresholds
- Explains PR enforcement status

**Output Example**:
```
SCAN MODE: New Code Analysis Only (Feature Branch)
DASHBOARD METRICS: New Code section will show:
  - Coverage: 80% minimum threshold
  - Duplicated Lines: 3% maximum threshold
  - Reliability Rating: A minimum
  - Security Rating: A minimum
  - Maintainability Rating: A minimum
```

---

#### Step 1.2: SonarQube Prepare Analysis
**Task**: `SonarQubePrepare@7`

This step configures SonarQube analysis based on branch type:

##### **For Pull Requests:**
```yaml
sonar.projectKey: Employee_Digital_14031_digital_soh_bff
sonar.sources: src
sonar.tests: src
sonar.typescript.lcov.reportPaths: coverage/lcov.info
sonar.javascript.lcov.reportPaths: coverage/lcov.info
sonar.scm.provider: git
sonar.pullrequest.key: <PR_ID>
sonar.pullrequest.base: <target_branch>
sonar.pullrequest.branch: <source_branch>
sonar.qualitygate.wait: true  # Enforces quality gate
```

##### **For Feature Branches:**
```yaml
sonar.branch.name: <feature_branch_name>
sonar.branch.target: develop
sonar.branch.analysisType: BRANCH
sonar.newCode.referenceBranch: develop
sonar.qualitygate.wait: true  # Enforces quality gate
```

##### **For Develop/Other Branches:**
```yaml
sonar.branch.name: <branch_name>
sonar.qualitygate.wait: false  # Doesn't block build
```

**Key Configuration Points**:
- **Source Directory**: `src` (all TypeScript/JavaScript files)
- **Test Directory**: `src` (test files)
- **Coverage Format**: LCOV (`coverage/lcov.info`)
- **Exclusions**: Config files, test utilities, migrations, etc.
- **Test Inclusions**: `**/*.test.*`, `test/**`, `**.test.*`

---

#### Step 1.3: Verify Branch Coverage Prerequisites
**Task**: `Verify Branch Coverage Prerequisites`

**For Feature Branches**:
- Checks if `develop` branch has coverage data in SonarQube
- Validates that coverage file exists locally
- Provides instructions if `develop` branch lacks coverage

**Critical Check**:
```
⚠⚠⚠ REQUIRED: The 'develop' branch MUST have coverage data in SonarQube ⚠⚠⚠
```

**Why This Matters**:
- Feature branch "New Code" coverage compares against `develop` branch
- If `develop` has no coverage, SonarQube cannot calculate "New Code" coverage
- Result: Coverage shows "Not computed" in dashboard

---

### Phase 2: Build and Test Execution

#### Step 2.1: Install Node.js Dependencies
**Task**: `NodeTool@0` + Custom NPM Install

- Installs Node.js version 22.2.0
- Runs `npm install` to install project dependencies
- Validates `package.json` exists
- Uses `package-lock.json` if available for reproducible builds

---

#### Step 2.2: Build Application
**Task**: `Npm@1` - Build dist folder

- Runs `npm run build`
- Compiles TypeScript/JavaScript source code
- Generates production-ready artifacts

---

#### Step 2.3: Run Jest Tests with Coverage
**Task**: `Run Jest Tests with Coverage`

**Command**: `npm run test:cov`

**What Happens**:
1. Jest executes all test files matching test patterns
2. Generates coverage report in LCOV format
3. Creates `coverage/lcov.info` file

**Coverage File Structure** (LCOV format):
```
SF:src/utils/helper.ts
DA:10,1
DA:11,0
DA:12,1
...
end_of_record
```

**Validation**:
- Checks if `coverage/lcov.info` exists
- Counts source files (SF:) entries
- Counts coverage data (DA:) entries
- Reports file size and location

**Critical**: If coverage file is missing, SonarQube will show "Not computed" for coverage!

---

### Phase 3: Coverage File Processing

#### Step 3.1: Transform Coverage File Paths
**Task**: `Transform Coverage Paths for SonarQube`

**Problem**: Jest generates paths without `src/` prefix (due to `rootDir='src'` in Jest config)

**Example**:
- **Jest Output**: `SF:utils/helper.ts`
- **SonarQube Expects**: `SF:src/utils/helper.ts` (to match `sonar.sources=src`)

**Solution**: Script prepends `src/` to all file paths in `coverage/lcov.info`

**Process**:
1. Creates backup of original coverage file
2. Uses `awk` to transform paths: `SF:utils/helper.ts` → `SF:src/utils/helper.ts`
3. Validates transformation (all paths should have `src/` prefix)
4. Removes backup file

**Why This Matters**:
- SonarQube matches coverage data to source files by path
- If paths don't match, coverage won't be associated with files
- Result: Coverage shows 0% or "Not computed"

---

#### Step 3.2: Validate Coverage Report
**Task**: `Debug: Validate Coverage Report Exists`

**Checks**:
- Verifies `coverage/` directory exists
- Validates `coverage/lcov.info` file exists
- Displays file statistics (line count, file count)
- Shows sample coverage entries
- Validates file paths match source structure

**Output**:
```
✓ lcov.info file exists
File size: 1523 lines
Found 45 source files in coverage report
✓ Coverage data appears to be present
```

---

#### Step 3.3: Pre-SonarQube Coverage Verification
**Task**: `Pre-SonarQube: Final Coverage Verification`

Multiple verification steps ensure coverage file is ready:

**Step 3.3a**: `Pre-SonarQube: Final Coverage Verification`
- Validates coverage file exists at expected location
- Checks file paths match source structure
- Verifies all paths start with `src/`

**Step 3.3b**: `Pre-Analysis: Ensure Coverage File Accessible`
- Ensures coverage file is in `$(Build.SourcesDirectory)/coverage/lcov.info`
- Copies coverage file if needed to correct location
- Validates file is accessible to SonarQube scanner

**Step 3.3c**: `🔍 CRITICAL: Final Coverage File Verification`
- Final check before SonarQube analysis
- Validates file content and structure
- Provides diagnostic information if issues found

---

### Phase 4: SonarQube Analysis

#### Step 4.1: SonarQube Code Analysis
**Task**: `Sonar Run Code Analysis` (`SonarQubeAnalyze@7`)

**What Happens**:
1. SonarQube scanner reads source code from `sonar.sources=src`
2. Analyzes code for:
   - **Bugs**: Code errors that could cause runtime issues
   - **Vulnerabilities**: Security flaws (SQL injection, XSS, etc.)
   - **Code Smells**: Maintainability issues
   - **Security Hotspots**: Security-sensitive code requiring review
3. Reads coverage file from `coverage/lcov.info`
4. Matches coverage data to source files
5. Calculates metrics:
   - Coverage percentage
   - Duplicated lines
   - Code complexity
   - Technical debt
6. For feature branches: Compares against reference branch (`develop`)
7. For PRs: Compares against target branch
8. Uploads results to SonarQube server

**Analysis Types**:

**New Code Analysis** (Feature Branches/PRs):
- Identifies code changed compared to reference branch
- Calculates coverage only for new/changed lines
- Shows metrics in "New Code" tab in SonarQube dashboard

**Full Repository Analysis** (Develop/Other Branches):
- Analyzes entire codebase
- Calculates overall coverage percentage
- Shows metrics in "Overall Code" tab in SonarQube dashboard

**Quality Gate Evaluation**:
- Evaluates code against quality gate conditions
- For PRs/Feature Branches: **Fails build** if gate doesn't pass
- For Develop/Other Branches: **Continues build** even if gate fails

---

#### Step 4.2: Post-Analysis Coverage Status Check
**Task**: `📊 Post-Analysis: Coverage Import Status`

**For Feature Branches**:
- Verifies coverage was imported successfully
- Provides troubleshooting steps if coverage shows "Not computed"
- Checks SonarQube project settings
- Validates develop branch has coverage

**Common Issues and Solutions**:
1. **Coverage shows "Not computed"**:
   - Check: SonarQube project "New Code Definition" must be "Reference branch: develop"
   - Check: Develop branch must have coverage in SonarQube
   - Check: Coverage file paths must match source structure

2. **Coverage file not found**:
   - Verify: `coverage/lcov.info` exists after test execution
   - Verify: File paths are transformed correctly (with `src/` prefix)

---

#### Step 4.3: Quality Gate Status Information
**Task**: `📊 Quality Gate Status Information (Default Gate)`

**For Develop/Other Branches**:
- Explains why quality gate might fail
- Default quality gate checks "New Code" metrics
- For full repository scans, this can cause false failures
- Provides solutions:
  1. Create custom quality gate for develop branch
  2. Update project-level "New Code" definition
  3. Accept quality gate failure (current approach)

**Current Behavior**:
- Build continues even if quality gate fails
- Coverage data is still collected
- Metrics are still visible in dashboard
- Develop branch serves as reference for feature branches

---

#### Step 4.4: Publish Quality Gate Results
**Task**: `SonarQube: Publish Quality Gate Result` (`SonarQubePublish@7`)

**What Happens**:
1. Waits for SonarQube server to process analysis (up to 300 seconds)
2. Retrieves quality gate status
3. Publishes results to Azure DevOps pipeline
4. Displays quality gate status (Passed/Failed) in pipeline summary

**Quality Gate Status**:
- ✅ **Passed**: All conditions met
- ❌ **Failed**: One or more conditions not met
- ⚠️ **Informational**: Status shown but doesn't block build (for develop branch)

---

### Phase 5: Post-Analysis Diagnostics

#### Step 5.1: Post-SonarQube Coverage Analysis Diagnostic
**Task**: `Diagnostic: Post-SonarQube Coverage Analysis`

**For Feature Branches**:
- Provides detailed diagnostic information
- Explains how to verify develop branch coverage
- Lists common issues and solutions
- Validates coverage file content

---

## Coverage Generation and Processing

### Coverage File Format (LCOV)

The pipeline uses **LCOV format** for coverage reporting:

```
SF:src/utils/helper.ts          # Source file path
DA:10,1                         # Line 10, executed 1 time
DA:11,0                         # Line 11, executed 0 times (not covered)
DA:12,5                         # Line 12, executed 5 times
LF:15                           # Lines with function definitions
LH:12                           # Lines hit (covered)
end_of_record                   # End of file record
```

### Coverage File Transformation

**Why Transformation is Needed**:
- Jest `rootDir='src'` generates paths without `src/` prefix
- SonarQube expects paths to match `sonar.sources=src`
- Path mismatch causes coverage to not be associated with files

**Transformation Process**:
```bash
# Before transformation:
SF:utils/helper.ts

# After transformation:
SF:src/utils/helper.ts
```

### Coverage Exclusions

Files excluded from coverage calculation:
- Test files: `**/*.test.*`, `**/*.spec.*`, `**/test/**`
- Storybook files: `**/*.stories.*`
- Configuration files: `**.yml`, `**.d.ts`
- Build artifacts: `webpack.*.js`, `Dockerfile`

---

## SonarQube Analysis Configuration

### Source and Test Configuration

```yaml
sonar.sources: src                    # Source code directory
sonar.tests: src                      # Test files directory
sonar.test.inclusions:               # Test file patterns
  - **/*.test.*
  - test/**
  - **.test.*
```

### File Exclusions

```yaml
sonar.exclusions:                     # Files excluded from analysis
  - **.yml
  - **.d.ts
  - eslint/**
  - migrations/**
  - test-report.xml
  - **/*.stories.tsx
  - snapshotTest/**
  - static/**
  - src/monitor.ts
  - src/index.tsx
  - webpack.*.js
  - **/node_modules/**
  - **/Dockerfile
```

### Coverage Configuration

```yaml
sonar.typescript.lcov.reportPaths: coverage/lcov.info
sonar.javascript.lcov.reportPaths: coverage/lcov.info
sonar.coverage.exclusions:            # Excluded from coverage calculation
  - **/*.test.*
  - **/*.spec.*
  - **/test/**
  - **/tests/**
  - **/*.stories.*
```

### Branch-Specific Configuration

#### Feature Branches:
```yaml
sonar.branch.name: <feature_branch>
sonar.branch.target: develop
sonar.newCode.referenceBranch: develop
sonar.coverage.newCode.minimum: 80
sonar.duplication.newCode.minimum: 3
sonar.reliability.newCode.minimum: A
sonar.security.newCode.minimum: A
sonar.maintainability.newCode.minimum: A
```

#### Pull Requests:
```yaml
sonar.pullrequest.key: <PR_ID>
sonar.pullrequest.base: <target_branch>
sonar.pullrequest.branch: <source_branch>
```

---

## Quality Gate Evaluation

### Quality Gate Conditions

The default SonarQube quality gate checks:

1. **Coverage on New Code** >= 80%
2. **Duplicated Lines on New Code** <= 3%
3. **Maintainability Rating on New Code** = A
4. **Reliability Rating on New Code** = A
5. **Security Rating on New Code** = A
6. **No new Blocker or Critical issues**

### Quality Gate Behavior by Branch Type

| Branch Type | Quality Gate Wait | Build Fails on Gate Failure |
|------------|-------------------|------------------------------|
| Pull Request | `true` | ✅ Yes (enforced) |
| Feature Branch (PR) | `true` | ✅ Yes (enforced) |
| Feature Branch (Direct Push) | `true` | ⚠️ No (`continueOnError: true`) |
| Develop/Other Branches | `false` | ❌ No (`continueOnError: true`) |

### Why Develop Branch Quality Gate Fails

The default quality gate checks **"New Code"** metrics, but develop branch performs **full repository analysis**. This mismatch causes:

1. **"New Code" = 0%**: If no changes in last 30 days (date-based definition)
2. **"New Code" = All Code**: If comparing against itself
3. **False Failures**: Quality gate fails even when overall code quality is good

**Solutions**:
- Create custom quality gate for develop (checks "Overall Code" metrics)
- Update project "New Code Definition" to reference another branch
- Accept quality gate failure (current approach - build continues)

---

## Troubleshooting

### Issue 1: Coverage Shows "Not computed" in Feature Branch

**Symptoms**:
- Feature branch dashboard shows "Coverage: Not computed"
- "New Code" tab shows no coverage percentage

**Root Causes**:
1. **Develop branch has no coverage**: Feature branches compare against develop
2. **SonarQube project settings**: "New Code Definition" not set to "Reference branch: develop"
3. **Coverage file not found**: Coverage file missing or incorrect path

**Solutions**:
1. **Verify develop branch has coverage**:
   - Go to SonarQube dashboard
   - Switch to `develop` branch
   - Check "Overall Code" tab - should show coverage percentage
   - If "Not computed", run pipeline on develop branch first

2. **Check SonarQube project settings**:
   - Go to: Project Settings → General Settings → New Code
   - Set "New Code Definition" to "Reference branch: develop"
   - Save and re-run pipeline

3. **Verify coverage file**:
   - Check pipeline logs for "Coverage file exists" messages
   - Verify `coverage/lcov.info` is generated
   - Check file paths are transformed correctly (with `src/` prefix)

---

### Issue 2: Quality Gate Always Fails on Develop Branch

**Symptoms**:
- Develop branch quality gate shows "Failed" status
- Build continues (doesn't fail)

**Root Cause**:
- Default quality gate checks "New Code" metrics
- Develop branch performs full repository analysis
- Mismatch causes false failures

**Solutions**:
1. **Create custom quality gate** (requires SonarQube admin):
   - Quality Gates → Create Quality Gate
   - Add conditions for "Overall Code" metrics (not "New Code")
   - Assign to develop branch

2. **Update "New Code Definition"**:
   - Project Settings → General Settings → New Code
   - Set to "Reference branch: master" (or another branch)
   - This gives develop a proper reference for "New Code" calculation

3. **Accept current behavior** (recommended):
   - Quality gate failure is informational only
   - Build continues successfully
   - Coverage data is collected
   - Develop branch serves as reference for feature branches

---

### Issue 3: Coverage File Not Found

**Symptoms**:
- Pipeline error: "coverage/lcov.info was NOT generated!"
- SonarQube shows "Not computed" for coverage

**Solutions**:
1. **Check test execution**:
   - Verify `npm run test:cov` completes successfully
   - Check Jest configuration in `package.json`
   - Ensure test files are being executed

2. **Verify Jest configuration**:
   - Check `jest.config.js` or `package.json` jest config
   - Verify `coverageReporters: ['lcov']` is configured
   - Ensure `collectCoverageFrom` includes source files

3. **Check file paths**:
   - Verify coverage file is in `coverage/lcov.info` (relative to project root)
   - Check working directory during test execution
   - Verify file exists before SonarQube analysis

---

### Issue 4: Coverage Paths Don't Match

**Symptoms**:
- Coverage file exists but shows 0% coverage
- SonarQube analysis logs show "No coverage information"

**Root Cause**:
- Coverage file paths don't match SonarQube source paths
- Jest generates paths without `src/` prefix
- SonarQube expects paths with `src/` prefix

**Solution**:
- Pipeline automatically transforms paths (Step 3.1)
- Verify transformation step runs successfully
- Check pipeline logs for "All paths now have 'src/' prefix" message
- If transformation fails, check `coverage/lcov.info` file format

---

## Summary

The SonarQube scanning process involves:

1. **Configuration**: Set up analysis parameters based on branch type
2. **Test Execution**: Run Jest tests with coverage generation
3. **Coverage Processing**: Transform coverage file paths for SonarQube compatibility
4. **Code Analysis**: SonarQube analyzes code and imports coverage data
5. **Quality Gate**: Evaluate code against quality standards
6. **Results Publishing**: Display results in SonarQube dashboard and pipeline

**Key Points**:
- Feature branches analyze "New Code" compared to `develop`
- Develop branch must have coverage for feature branches to show coverage
- Quality gate is enforced for PRs/feature branch PRs
- Quality gate is informational for develop branch (build continues)
- Coverage file paths must match source structure for proper coverage calculation

---

## Additional Resources

- **SonarQube Dashboard**: https://devtools.metlife.com/sonar/dashboard
- **Project**: Employee_Digital_14031_digital_soh_bff
- **Pipeline File**: `bff.yaml`

---

## Pipeline Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Configure SonarQube Analysis (Based on Branch Type)      │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Install Dependencies & Build Application                 │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. Run Jest Tests → Generate coverage/lcov.info             │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. Transform Coverage Paths (Add src/ prefix)               │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. Validate Coverage File (Multiple verification steps)     │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. SonarQube Code Analysis + Coverage Import                │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. Quality Gate Evaluation                                   │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 8. Publish Results to SonarQube Dashboard                  │
└─────────────────────────────────────────────────────────────┘
```

---

**Last Updated**: 2025-01-23  
**Pipeline Version**: bff.yaml  
**SonarQube Server**: https://devtools.metlife.com/sonar

