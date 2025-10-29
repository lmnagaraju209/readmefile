# SonarQube Pipeline Configuration Guide

## Overview

This Azure DevOps pipeline has been enhanced with intelligent SonarQube scanning capabilities that automatically adapt based on branch patterns. The pipeline ensures code quality through automated quality gates while providing different scanning strategies for different types of branches.

## 🎯 Key Features

- **Intelligent Branch Detection**: Automatically identifies feature/feat branches vs other branches
- **Conditional Scanning**: Different analysis modes based on branch type
- **PR Quality Enforcement**: Automatic PR failure for insufficient coverage on feature branches
- **Comprehensive Metrics**: Standardized dashboard display across all branches
- **Enhanced Debugging**: Detailed logging for troubleshooting

## 📋 Pipeline Structure

### Variables Configuration

The pipeline uses several key variables to determine scanning behavior:

```yaml
# Branch Detection Variables
- name: IsFeatureBranch
  value: ${{ or(startswith(variables['Build.SourceBranch'], 'refs/heads/feature/'), startswith(variables['Build.SourceBranch'], 'refs/heads/feat/')) }}

- name: SonarScanMode
  value: ${{ or(startswith(variables['Build.SourceBranch'], 'refs/heads/feature/'), startswith(variables['Build.SourceBranch'], 'refs/heads/feat/')) }}

- name: IsPullRequest
  value: ${{ or(eq(variables['Build.Reason'], 'PullRequest'), gt(variables['System.PullRequest.PullRequestId'], 0)) }}
```

## 🔄 Scanning Modes

### 1. Feature/Feat Branches (`feature/*`, `feat/*`)

**Scanning Strategy**: New Code Analysis Only
- **Target**: Only scans changed lines compared to `develop` branch
- **Reference Branch**: `develop`
- **Quality Gate**: Strict enforcement for PRs

**Quality Requirements**:
- ✅ **Coverage**: 80% minimum for new code
- ✅ **Duplication**: ≤3% for new code
- ✅ **Reliability Rating**: A minimum
- ✅ **Security Rating**: A minimum
- ✅ **Maintainability Rating**: A minimum

**PR Behavior**:
- **Pull Requests**: PR fails if quality gate doesn't pass
- **Direct Pushes**: Quality gate checked but build doesn't fail

### 2. Other Branches (develop, master, release, etc.)

**Scanning Strategy**: Full Repository Analysis
- **Target**: Scans entire codebase
- **Quality Gate**: Standard enforcement

**Quality Requirements**:
- ✅ **Coverage**: 80% minimum overall
- ✅ **Duplication**: ≤3% overall
- ✅ **Reliability Rating**: A minimum
- ✅ **Security Rating**: A minimum
- ✅ **Maintainability Rating**: A minimum

### 3. Pull Requests

**Scanning Strategy**: Differential Analysis
- **Target**: Compares source branch vs target branch
- **Quality Gate**: Enhanced with comprehensive metrics

## 📊 SonarQube Dashboard Metrics

All branches display consistent metrics on the SonarQube dashboard:

### New Code Section (Feature Branches)
- **Coverage**: 80% minimum threshold with circular progress indicator
- **Duplicated Lines (%)**: 3% maximum threshold with circular progress indicator
- **Reliability Rating**: A minimum with circular progress indicator
- **Security Rating**: A minimum with circular progress indicator
- **Maintainability Rating**: A minimum with circular progress indicator
- **New Issues**: Count of new issues found
- **Accepted Issues**: Count of accepted issues

### Overall Code Section (Other Branches)
- **Coverage**: 80% minimum threshold with circular progress indicator
- **Duplications**: 3% maximum threshold with circular progress indicator
- **Reliability Rating**: A minimum with circular progress indicator
- **Security Rating**: A minimum with circular progress indicator
- **Maintainability Rating**: A minimum with circular progress indicator
- **Security Hotspots**: Count with A rating indicator

## 🚀 Pipeline Steps

### 1. Environment Setup
```yaml
- Display Environment Variables
- Spring Security Vulnerability Check
- SonarQube Configuration Debug Information
```

### 2. SonarQube Analysis
```yaml
- SonarQubePrepare@7 (with conditional configuration)
- NodeTool@0
- Npm Install
- Build dist folder
- Run Jest Tests with Coverage
- Publish Code Coverage Results
- Publish Test Results
```

### 3. SonarQube Processing
```yaml
- SonarQubeAnalyze@7 (with conditional failure logic)
- SonarQubePublish@7
- PR Linting and Quality Gate Check
- PR Failure Enforcement (for feature branches)
```

### 4. Additional Security Scans
```yaml
- Veracode SAST Scan Preparation
- Veracode SCA Scan (if enabled)
- Docker Image Build and Push
```

## 🔧 Configuration Details

### SonarQube Properties by Branch Type

#### Pull Requests
```yaml
sonar.coverage.newCode.minimum=80
sonar.duplication.newCode.minimum=3
sonar.reliability.newCode.minimum=A
sonar.security.newCode.minimum=A
sonar.maintainability.newCode.minimum=A
```

#### Feature Branches
```yaml
sonar.newCode.referenceBranch=develop
sonar.coverage.newCode.minimum=80
sonar.duplication.newCode.minimum=3
sonar.reliability.newCode.minimum=A
sonar.security.newCode.minimum=A
sonar.maintainability.newCode.minimum=A
```

#### Other Branches
```yaml
sonar.coverage.minimum=80
sonar.duplication.minimum=3
sonar.reliability.minimum=A
sonar.security.minimum=A
sonar.maintainability.minimum=A
```

## 🚨 PR Failure Conditions

Feature branch PRs will fail if any of these conditions are not met:

1. **Coverage < 80%** for new code
2. **Duplication > 3%** for new code
3. **Reliability Rating** worse than A
4. **Security Rating** worse than A
5. **Maintainability Rating** worse than A

## 📝 Debug Information

The pipeline includes comprehensive debug logging that shows:

- Current branch information
- Feature branch detection status
- Scanning mode (New Code vs Full Repository)
- Quality gate requirements
- PR enforcement status
- Dashboard metrics that will be displayed

## 🔍 Troubleshooting

### Common Issues

1. **PR Failing Due to Coverage**
   - Check if new code has sufficient test coverage
   - Ensure test files are properly configured
   - Verify coverage reports are generated correctly

2. **Quality Gate Failures**
   - Review SonarQube dashboard for specific issues
   - Check for code duplications
   - Address security and maintainability issues

3. **Branch Detection Issues**
   - Ensure branch names follow `feature/*` or `feat/*` pattern
   - Check pipeline logs for branch detection status

### Debug Steps

1. Check the "SonarQube: Debug Configuration" step output
2. Review SonarQube dashboard for detailed metrics
3. Examine quality gate conditions in SonarQube
4. Verify coverage report generation

## 📈 Benefits

### For Development Teams
- **Automated Quality Assurance**: No manual quality checks required
- **Consistent Standards**: Same quality requirements across all branches
- **Clear Feedback**: Detailed error messages and logging
- **Flexible Workflow**: Different rules for different branch types

### For Code Quality
- **Prevent Technical Debt**: Enforced quality standards
- **Maintain Coverage**: 80% coverage requirement
- **Reduce Duplication**: ≤3% duplication limit
- **Security Standards**: A rating for security issues

### For Operations
- **Reduced Manual Intervention**: Automated quality gates
- **Consistent Dashboard**: Same metrics across all branches
- **Clear Reporting**: Comprehensive logging and status updates
- **Maintainable Configuration**: Well-documented and structured

## 🎯 Best Practices

1. **Branch Naming**: Use `feature/*` or `feat/*` prefixes for feature branches
2. **Test Coverage**: Write tests for new code to meet 80% coverage requirement
3. **Code Quality**: Address SonarQube issues promptly
4. **Regular Monitoring**: Check SonarQube dashboard regularly
5. **Documentation**: Keep this README updated with any changes

## 📞 Support

For issues or questions regarding this pipeline configuration:

1. Check the pipeline logs for detailed error information
2. Review the SonarQube dashboard for quality gate details
3. Consult the debug output for configuration status
4. Refer to Azure DevOps documentation for pipeline-specific issues

---

**Last Updated**: $(date)
**Pipeline Version**: Enhanced SonarQube Configuration v1.0
**Compatible With**: Azure DevOps, SonarQube 7+, Node.js 22.2.0
# readmefile
