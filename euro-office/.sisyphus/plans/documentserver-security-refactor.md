# DocumentServer Security Hardening & Refactoring Plan

## TL;DR

> **Quick Summary**: Comprehensive security hardening and modernization of Euro-Office DocumentServer to address critical buffer overflow vulnerabilities, update third-party dependencies, and refactor legacy C++ code patterns.
>
> **Deliverables**: 
> - Fixed buffer overflow vulnerabilities in DjVuFile library
> - Updated third-party dependencies (zlib, freetype, openjpeg)
> - Refactored ASCOfficeDrawingConverter (6026 lines → modular components)
> - Static analysis pipeline integrated into CI/CD
> - 80%+ test coverage for security-critical modules
>
> **Estimated Effort**: Large (8-12 weeks with 2 developers)
> **Parallel Execution**: YES - 4 waves
> **Critical Path**: Dependency updates → Buffer safety fixes → Input validation → Refactoring → Testing

---

## Context

### Original Request
"Look at document server in terms of security and refactoring"

### Interview Summary
**Key Discussions**:
- Comprehensive security audit revealed critical vulnerabilities
- Legacy C++ codebase with unsafe string functions
- Third-party dependencies have known CVEs
- God class (6026 lines) needs refactoring

**Research Findings**:
- 11,116 C/C++ source files analyzed
- Critical: strcpy/sprintf/vsprintf usage in DjVuFile, URL handling
- High: Manual memory management, outdated libraries (zlib 1.2.11, freetype 2.10.4)
- Medium: No input validation framework, missing security headers

### Metis Review
**Identified Gaps** (addressed):
- Need for rollback strategy when updating dependencies: Added feature flags and staging environment
- Testing coverage for buffer overflow fixes: Added fuzzing requirements
- Migration path for C++ modernization: Added gradual C++14/17 migration plan

---

## Work Objectives

### Core Objective
Secure the DocumentServer codebase by eliminating critical vulnerabilities, updating dependencies, and improving code maintainability through systematic refactoring.

### Concrete Deliverables
- Buffer overflow fixes in GString.cpp, GURL.cpp (replaced strcpy/sprintf)
- Updated zlib to 1.2.13+, freetype to 2.13+, openjpeg to 2.5+
- Modular drawing converter (split 6026-line file into 5-7 focused classes)
- Static analysis CI pipeline (clang-tidy, cppcheck)
- Security scanning pipeline (dependency-check, OWASP ZAP)
- Unit tests for security-critical functions (80%+ coverage)

### Definition of Done
- [ ] All critical/high security vulnerabilities remediated
- [ ] No compiler warnings (-Wall -Wextra -Werror passes)
- [ ] Static analysis shows 0 critical issues
- [ ] All third-party dependencies updated to secure versions
- [ ] Test coverage report shows 80%+ for security modules
- [ ] CI/CD pipeline includes security scanning

### Must Have
- Buffer safety fixes using strncpy/snprintf
- Dependency updates with CVE remediation
- Input validation for all user-facing APIs
- Static analysis integration

### Must NOT Have (Guardrails)
- NO breaking API changes without deprecation cycle
- NO removal of existing functionality
- NO hardcoded secrets in configuration
- NO disabling of compiler warnings
- NO changes to cryptographic implementations (use crypto++ as-is)

---

## Verification Strategy (MANDATORY)

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed. No exceptions.
> Acceptance criteria requiring "user manually tests/confirms" are FORBIDDEN.

### Test Decision
- **Infrastructure exists**: YES (Google Test found in codebase)
- **Automated tests**: YES (TDD for security fixes, tests-after for refactoring)
- **Framework**: Google Test (gtest) + fuzzing (libFuzzer)

### QA Policy
Every task MUST include agent-executed QA scenarios (see TODO template below).
Evidence saved to `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`.

- **Frontend/UI**: Use Playwright (playwright skill) — Navigate, interact, assert DOM, screenshot
- **TUI/CLI**: Use interactive_bash (tmux) — Run command, send keystrokes, validate output
- **API/Backend**: Use Bash (curl) — Send requests, assert status + response fields
- **Library/Module**: Use Bash (bun/node REPL) — Import, call functions, compare output

---

## Execution Strategy

### Parallel Execution Waves

> Maximize throughput by grouping independent tasks into parallel waves.
> Each wave completes before the next begins.
> Target: 5-8 tasks per wave. Fewer than 3 per wave (except final) = under-splitting.

```
Wave 1 (Start Immediately — Infrastructure & Assessment):
├── Task 1: Setup static analysis tooling [quick]
├── Task 2: Dependency audit & SBOM generation [quick]
├── Task 3: Security baseline testing [quick]
└── Task 4: CI/CD pipeline preparation [quick]

Wave 2 (Critical Security Fixes — MAX PARALLEL):
├── Task 5: Fix strcpy in GString.cpp [quick]
├── Task 6: Fix sprintf/vsprintf in GString.cpp [quick]
├── Task 7: Fix strcpy in GURL.cpp [quick]
├── Task 8: Add bounds checking to sscanf calls [quick]
└── Task 9: Memory management audit [deep]

Wave 3 (Dependency Updates — Careful Rolling):
├── Task 10: Update zlib to 1.2.13+ [medium]
├── Task 11: Update freetype to 2.13+ [medium]
├── Task 12: Update openjpeg to 2.5+ [medium]
├── Task 13: Update libdjvu [unspecified-high]
├── Task 14: Test regression suite after updates [deep]

Wave 4 (Refactoring — Modularization):
├── Task 15: Design converter interface [quick]
├── Task 16: Extract drawing conversion logic [unspecified-high]
├── Task 17: Split ASCOfficeDrawingConverter [unspecified-high]
├── Task 18: Implement plugin architecture [deep]
└── Task 19: Update build system for modules [medium]

Wave 5 (Input Validation & Security Hardening):
├── Task 20: Create input validation framework [deep]
├── Task 21: Add XSS protection [medium]
├── Task 22: Implement CSRF tokens [medium]
├── Task 23: Add CORS configuration [quick]
├── Task 24: Security headers for WebSocket [quick]

Wave 6 (Testing & Coverage):
├── Task 25: Unit tests for buffer fixes [unspecified-high]
├── Task 26: Fuzz testing for parsers [unspecified-high]
├── Task 27: Integration tests for conversion pipeline [deep]
├── Task 28: Coverage reporting setup [quick]
└── Task 29: Achieve 80%+ coverage target [deep]

Wave FINAL (After ALL tasks — 4 parallel reviews, then user okay):
├── Task F1: Plan compliance audit (oracle)
├── Task F2: Code quality review (unspecified-high)
├── Task F3: Real manual QA (unspecified-high)
└── Task F4: Scope fidelity check (deep)
-> Present results -> Get explicit user okay
```

### Dependency Matrix

- **1-4**: — — 5-9, 1
- **5-8**: — — 25
- **9**: — — 25, 2
- **10-13**: 1-2 — 14, 3
- **14**: 10-13 — 15-19, 4
- **15**: 14 — 16-18, 5
- **16-18**: 15 — 19, 27
- **19**: 16-18 — 20-24, 6
- **20-24**: 19 — 25-29, 7
- **25-29**: 5-8, 20-24 — F1-F4, 8
- **F1-F4**: All previous — user okay

---

## TODOs

- [x] 1. Setup static analysis tooling

  **What to do**:
  - Install clang-tidy, cppcheck, and SonarQube scanner
  - Configure .clang-tidy with C++14/17 checks
  - Add cppcheck configuration for security rules
  - Create CI integration scripts

  **Must NOT do**:
  - Do not modify existing source code
  - Do not change build system yet

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `quick`
    - Reason: Infrastructure setup task, straightforward configuration
  - **Skills**: []
    - No additional skills needed for tool installation

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 2, 3, 4)
  - **Blocks**: Tasks 5-9 (security fixes need analysis)
  - **Blocked By**: None (can start immediately)

  **References** (CRITICAL - Be Exhaustive):

  **Pattern References** (existing code to follow):
  - `build/Makefile` - Current build configuration to extend

  **WHY Each Reference Matters**:
  - Makefile shows existing build process to integrate static analysis

  **Acceptance Criteria**:

  **If TDD (tests enabled):**
  - [ ] Test script created: scripts/verify-static-analysis.sh
  - [ ] Script runs clang-tidy and cppcheck successfully

  **QA Scenarios (MANDATORY — task is INCOMPLETE without these):**

  ```
  Scenario: Static analysis tools installed and configured
    Tool: Bash (tmux)
    Preconditions: Clean workspace in /home/weiss/workspace/euro-office/core
    Steps:
      1. Run `which clang-tidy` and verify exit code 0
      2. Run `which cppcheck` and verify exit code 0
      3. Run `clang-tidy --version` and capture output
      4. Run `cppcheck --version` and capture output
    Expected Result: Both tools installed and return version info
    Failure Indicators: Tools not found in PATH, version commands fail
    Evidence: .sisyphus/evidence/task-1-tools-installed.txt

  Scenario: Configuration files created
    Tool: Bash (tmux)
    Preconditions: Tools installed
    Steps:
      1. Check existence of .clang-tidy file in core/
      2. Check existence of cppcheck.suppressions in core/
      3. Run `clang-tidy --config-file=.clang-tidy` on a test file
    Expected Result: Config files exist and are valid
    Failure Indicators: Config files missing, validation errors
    Evidence: .sisyphus/evidence/task-1-config-files.txt
  ```

  **Evidence to Capture**:
  - [ ] Each evidence file named: task-{N}-{scenario-slug}.{ext}
  - [ ] Screenshots for UI, terminal output for CLI, response bodies for API

  **Commit**: YES (groups with 2-4)
  - Message: `ci: add static analysis tooling configuration`
  - Files: `.clang-tidy`, `cppcheck.suppressions`, `scripts/verify-static-analysis.sh`

- [x] 2. Dependency audit & SBOM generation

  **What to do**:
  - Run dependency-check on all third-party libraries
  - Generate Software Bill of Materials (SBOM)
  - Create vulnerability report with CVE mappings
  - Document current versions vs latest secure versions

  **Must NOT do**:
  - Do not update dependencies yet (only audit)
  - Do not modify package files

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `quick`
    - Reason: Audit task, straightforward scanning
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 3, 4)
  - **Blocks**: Tasks 10-14 (dependency updates)
  - **Blocked By**: None (can start immediately)

  **References** (CRITICAL - Be Exhaustive):

  **Pattern References** (existing code to follow):
  - `3DPARTY.md` - List of third-party dependencies
  - `Common/3dParty/` - Directory structure of third-party code

  **WHY Each Reference Matters**:
  - 3DPARTY.md provides authoritative list of dependencies to audit
  - 3dParty directory shows actual library versions in use

  **Acceptance Criteria**:

  **QA Scenarios (MANDATORY — task is INCOMPLETE without these):**

  ```
  Scenario: Dependency audit completed
    Tool: Bash (tmux)
    Preconditions: Dependency-check tool installed
    Steps:
      1. Run dependency-check scan on core/Common/3dParty/
      2. Parse output JSON for CVE findings
      3. Count critical/high severity vulnerabilities
      4. Generate summary report
    Expected Result: Report shows vulnerability counts by severity
    Failure Indicators: Scan fails, no output, parsing errors
    Evidence: .sisyphus/evidence/task-2-dependency-audit.json

  Scenario: SBOM generated
    Tool: Bash (tmux)
    Preconditions: Audit complete
    Steps:
      1. Generate SBOM in SPDX format
      2. Verify SBOM contains all libraries from 3DPARTY.md
      3. Check SBOM includes version info for each component
    Expected Result: Valid SPDX file with complete dependency list
    Failure Indicators: Missing libraries, invalid format
    Evidence: .sisyphus/evidence/task-2-sbom.spdx
  ```

  **Evidence to Capture**:
  - [ ] Each evidence file named: task-{N}-{scenario-slug}.{ext}
  - [ ] Screenshots for UI, terminal output for CLI, response bodies for API

  **Commit**: YES (groups with 1, 3, 4)
  - Message: `chore: generate dependency audit report and SBOM`
  - Files: `docs/dependency-audit.json`, `docs/SBOM.spdx`

- [x] 3. Security baseline testing

  **What to do**:
  - Create test suite for current security state
  - Document known vulnerabilities in test cases
  - Set up baseline metrics for comparison
  - Create fuzz test harness for parsers

  **Must NOT do**:
  - Do not fix vulnerabilities yet (only document)
  - Do not change production code

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `quick`
    - Reason: Test setup task
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 4)
  - **Blocks**: Tasks 25-29 (coverage improvements)
  - **Blocked By**: None (can start immediately)

  **References** (CRITICAL - Be Exhaustive):

  **Pattern References** (existing code to follow):
  - `Test/` directory - Existing test structure
  - `Common/3dParty/cryptopp/regtest*.cpp` - Test patterns from crypto++

  **WHY Each Reference Matters**:
  - Test directory shows existing test organization
  - Crypto++ tests provide examples of security test patterns

  **Acceptance Criteria**:

  **QA Scenarios (MANDATORY — task is INCOMPLETE without these):**

  ```
  Scenario: Baseline test suite created
    Tool: Bash (tmux)
    Preconditions: Test framework available
    Steps:
      1. Create security-baseline test file
      2. Add test cases for each known vulnerability
      3. Run test suite and verify all known issues fail as expected
      4. Capture test output showing baseline state
    Expected Result: Tests fail for known vulnerabilities (expected)
    Failure Indicators: Tests pass when they should fail, setup errors
    Evidence: .sisyphus/evidence/task-3-baseline-tests.txt

  Scenario: Fuzz test harness setup
    Tool: Bash (tmux)
    Preconditions: libFuzzer or AFL++ installed
    Steps:
      1. Create fuzz target for DjVuFile parser
      2. Create fuzz target for PDF parser
      3. Run initial fuzz test (1 minute)
      4. Verify fuzz harness captures crashes
    Expected Result: Fuzz harness runs without errors
    Failure Indicators: Compilation errors, harness crashes immediately
    Evidence: .sisyphus/evidence/task-3-fuzz-harness.cpp
  ```

  **Evidence to Capture**:
  - [ ] Each evidence file named: task-{N}-{scenario-slug}.{ext}
  - [ ] Screenshots for UI, terminal output for CLI, response bodies for API

  **Commit**: YES (groups with 1, 2, 4)
  - Message: `test: add security baseline test suite`
  - Files: `Test/security-baseline.cpp`, `Test/fuzz-harness.cpp`

- [x] 4. CI/CD pipeline preparation

  **What to do**:
  - Create CI configuration for static analysis
  - Add security scanning stage to pipeline
  - Set up automated testing on PR
  - Configure coverage reporting

  **Must NOT do**:
  - Do not modify existing CI configs yet
  - Do not change deployment process

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `quick`
    - Reason: Infrastructure task
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 3)
  - **Blocks**: All implementation tasks
  - **Blocked By**: Tasks 1-3 (need tooling and tests first)

  **References** (CRITICAL - Be Exhaustive):

  **Pattern References** (existing code to follow):
  - `.github/workflows/` - Existing CI structure (if any)
  - `build/Makefile` - Build process to integrate

  **WHY Each Reference Matters**:
  - Existing workflows show CI patterns to extend
  - Makefile shows build steps to add analysis to

  **Acceptance Criteria**:

  **QA Scenarios (MANDATORY — task is INCOMPLETE without these):**

  ```
  Scenario: CI pipeline configuration created
    Tool: Bash (tmux)
    Preconditions: Git repository
    Steps:
      1. Create .github/workflows/security-scan.yml
      2. Verify YAML syntax is valid
      3. Test pipeline locally with act or similar
      4. Verify all stages defined (build, test, scan)
    Expected Result: Valid CI config with all stages
    Failure Indicators: YAML syntax errors, missing stages
    Evidence: .sisyphus/evidence/task-4-ci-config.yml
  ```

  **Evidence to Capture**:
  - [ ] Each evidence file named: task-{N}-{scenario-slug}.{ext}
  - [ ] Screenshots for UI, terminal output for CLI, response bodies for API

  **Commit**: YES (groups with 1, 2, 3)
  - Message: `ci: add security scanning pipeline configuration`
  - Files: `.github/workflows/security-scan.yml`

- [x] 5. Fix strcpy in GString.cpp

  **What to do**:
  - Replace strcpy with strncpy or std::string
  - Replace strcat with strncat or std::string
  - Update all call sites in GString.cpp
  - Add bounds checking validation

  **Must NOT do**:
  - Do not change API signatures
  - Do not modify other files in this task

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `quick`
    - Reason: Straightforward string replacement
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 6, 7, 8)
  - **Blocks**: Task 25 (tests for buffer fixes)
  - **Blocked By**: Tasks 1-4 (need tooling and baseline)

  **References** (CRITICAL - Be Exhaustive):

  **Pattern References** (existing code to follow):
  - `DjVuFile/libdjvu/GString.cpp:1212` - strcpy usage to fix
  - `DjVuFile/libdjvu/GString.cpp:1217` - strcat usage to fix

  **WHY Each Reference Matters**:
  - Exact lines with vulnerable code that need fixing

  **Acceptance Criteria**:
  - [ ] grep -r "strcpy.*GString" shows no unsafe usage
  - [ ] clang-tidy shows 0 buffer overflow warnings in GString.cpp
  - [ ] Unit tests pass for GString functionality

  **QA Scenarios (MANDATORY — task is INCOMPLETE without these):**

  ```
  Scenario: strcpy replaced with safe alternative
    Tool: Bash (tmux)
    Preconditions: GString.cpp modified
    Steps:
      1. Run `grep -n "strcpy" DjVuFile/libdjvu/GString.cpp`
      2. Verify all strcpy are strncpy with bounds
      3. Compile modified code: cmake -B build && cmake --build build
      4. Run existing GString tests: ctest -R GString
    Expected Result: No unsafe strcpy, build succeeds, tests pass
    Failure Indicators: Unsafe strcpy found, build fails, tests fail
    Evidence: .sisyphus/evidence/task-5-strcpy-fix.txt

  Scenario: strcat replaced with safe alternative
    Tool: Bash (tmux)
    Preconditions: GString.cpp modified
    Steps:
      1. Run `grep -n "strcat" DjVuFile/libdjvu/GString.cpp`
      2. Verify all strcat are strncat with bounds
      3. Run static analysis: clang-tidy DjVuFile/libdjvu/GString.cpp
      4. Verify 0 buffer overflow warnings
    Expected Result: No unsafe strcat, no warnings
    Failure Indicators: Unsafe strcat found, warnings present
    Evidence: .sisyphus/evidence/task-5-strcat-fix.txt
  ```

  **Evidence to Capture**:
  - [ ] Each evidence file named: task-{N}-{scenario-slug}.{ext}
  - [ ] Screenshots for UI, terminal output for CLI, response bodies for API

  **Commit**: YES
  - Message: `fix(security): replace unsafe strcpy/strcat in GString.cpp`
  - Files: `DjVuFile/libdjvu/GString.cpp`

- [x] 6. Fix sprintf/vsprintf in GString.cpp

  **What to do**:
  - Replace sprintf with snprintf
  - Replace vsprintf with vsnprintf
  - Ensure all format strings are safe
  - Add buffer size validation

  **Must NOT do**:
  - Do not change API behavior
  - Do not modify other modules

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `quick`
    - Reason: Direct string function replacement
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 5, 7, 8)
  - **Blocks**: Task 25 (tests for buffer fixes)
  - **Blocked By**: Tasks 1-4 (need tooling and baseline)

  **References** (CRITICAL - Be Exhaustive):

  **Pattern References** (existing code to follow):
  - `DjVuFile/libdjvu/GString.cpp:1658` - vsprintf usage to fix
  - `DjVuFile/libdjvu/GString.cpp:1890` - sprintf usage to fix

  **WHY Each Reference Matters**:
  - Exact lines with vulnerable format string usage

  **Acceptance Criteria**:
  - [ ] grep -r "sprintf.*GString" shows only snprintf
  - [ ] grep -r "vsprintf.*GString" shows only vsnprintf
  - [ ] All format specifiers have corresponding arguments

  **QA Scenarios (MANDATORY — task is INCOMPLETE without these):**

  ```
  Scenario: sprintf replaced with snprintf
    Tool: Bash (tmux)
    Preconditions: GString.cpp modified
    Steps:
      1. Run `grep -n "sprintf" DjVuFile/libdjvu/GString.cpp`
      2. Verify all sprintf are snprintf with size param
      3. Compile and run tests
      4. Check static analysis for format string warnings
    Expected Result: Only snprintf used, tests pass
    Failure Indicators: Unsafe sprintf found, warnings present
    Evidence: .sisyphus/evidence/task-6-sprintf-fix.txt

  Scenario: vsprintf replaced with vsnprintf
    Tool: Bash (tmux)
    Preconditions: GString.cpp modified
    Steps:
      1. Run `grep -n "vsprintf" DjVuFile/libdjvu/GString.cpp`
      2. Verify all vsprintf are vsnprintf
      3. Test variadic function behavior unchanged
    Expected Result: Only vsnprintf used, behavior preserved
    Failure Indicators: Unsafe vsprintf found, behavior changed
    Evidence: .sisyphus/evidence/task-6-vsprintf-fix.txt
  ```

  **Evidence to Capture**:
  - [ ] Each evidence file named: task-{N}-{scenario-slug}.{ext}
  - [ ] Screenshots for UI, terminal output for CLI, response bodies for API

  **Commit**: YES (groups with 5)
  - Message: `fix(security): replace unsafe sprintf/vsprintf in GString.cpp`
  - Files: `DjVuFile/libdjvu/GString.cpp`

- [x] 7. Fix strcpy in GURL.cpp

  **What to do**:
  - Replace strcpy with strncpy or std::string
  - Add URL validation before copying
  - Ensure buffer sizes are calculated correctly
  - Handle edge cases (empty URLs, very long URLs)

  **Must NOT do**:
  - Do not change URL parsing logic
  - Do not modify other files

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `quick`
    - Reason: String safety fix
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 5, 6, 8)
  - **Blocks**: Task 25 (tests for buffer fixes)
  - **Blocked By**: Tasks 1-4 (need tooling and baseline)

  **References** (CRITICAL - Be Exhaustive):

  **Pattern References** (existing code to follow):
  - `DjVuFile/wasm/libdjvu/GURL.cpp:261` - strcpy usage to fix
  - `DjVuFile/wasm/libdjvu/GURL.cpp:305-308` - strcpy/strcat usage

  **WHY Each Reference Matters**:
  - Exact lines with URL buffer vulnerabilities

  **Acceptance Criteria**:
  - [ ] No unsafe strcpy in GURL.cpp
  - [ ] URL validation added before string operations
  - [ ] All URL tests pass

  **QA Scenarios (MANDATORY — task is INCOMPLETE without these):**

  ```
  Scenario: strcpy fixed in URL handling
    Tool: Bash (tmux)
    Preconditions: GURL.cpp modified
    Steps:
      1. Run `grep -n "strcpy" DjVuFile/wasm/libdjvu/GURL.cpp`
      2. Verify all strcpy are safe (strncpy or std::string)
      3. Test with various URL lengths (short, medium, very long)
      4. Verify no buffer overflows with valgrind
    Expected Result: Safe string handling, no overflows
    Failure Indicators: Unsafe strcpy, valgrind reports errors
    Evidence: .sisyphus/evidence/task-7-url-fix.txt
  ```

  **Evidence to Capture**:
  - [ ] Each evidence file named: task-{N}-{scenario-slug}.{ext}
  - [ ] Screenshots for UI, terminal output for CLI, response bodies for API

  **Commit**: YES (groups with 5, 6)
  - Message: `fix(security): replace unsafe strcpy in GURL.cpp`
  - Files: `DjVuFile/wasm/libdjvu/GURL.cpp`

- [x] 8. Add bounds checking to sscanf calls

  **What to do**:
  - Review all sscanf usage in PdfFile
  - Add width specifiers to prevent buffer overflows
  - Validate parsed values before use
  - Add error handling for malformed input

  **Must NOT do**:
  - Do not change parsing logic behavior
  - Do not modify other parsers

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `quick`
    - Reason: Format string safety
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 5, 6, 7)
  - **Blocks**: Task 25 (tests for buffer fixes)
  - **Blocked By**: Tasks 1-4 (need tooling and baseline)

  **References** (CRITICAL - Be Exhaustive):

  **Pattern References** (existing code to follow):
  - `PdfFile/SrcReader/Adaptors.cpp:80` - sscanf without bounds
  - `PdfFile/lib/xpdf/PdfAnnot.cpp:2702` - sscanf usage

  **WHY Each Reference Matters**:
  - Exact lines with unsafe sscanf that need width specifiers

  **Acceptance Criteria**:
  - [ ] All sscanf have width specifiers
  - [ ] No buffer overflow warnings in static analysis
  - [ ] PDF parsing tests pass

  **QA Scenarios (MANDATORY — task is INCOMPLETE without these):**

  ```
  Scenario: sscanf calls secured with width specifiers
    Tool: Bash (tmux)
    Preconditions: PdfFile code modified
    Steps:
      1. Run `grep -n "sscanf" PdfFile/SrcReader/Adaptors.cpp`
      2. Verify all sscanf have width limits (e.g., %255s not %s)
      3. Compile and run PDF parsing tests
      4. Test with malformed PDF input
    Expected Result: All sscanf safe, tests pass, no crashes
    Failure Indicators: Missing width specifiers, crashes on bad input
    Evidence: .sisyphus/evidence/task-8-sscanf-fix.txt
  ```

  **Evidence to Capture**:
  - [ ] Each evidence file named: task-{N}-{scenario-slug}.{ext}
  - [ ] Screenshots for UI, terminal output for CLI, response bodies for API

  **Commit**: YES (groups with 5, 6, 7)
  - Message: `fix(security): add bounds checking to sscanf in PdfFile`
  - Files: `PdfFile/SrcReader/Adaptors.cpp`, `PdfFile/lib/xpdf/PdfAnnot.cpp`

- [x] 9. Memory management audit

  **What to do**:
  - Identify all malloc/realloc/free usage
  - Convert to smart pointers where possible
  - Add RAII wrappers for manual memory
  - Create memory leak detection tests
  - Document memory ownership patterns

  **Must NOT do**:
  - Do not change external APIs
  - Do not remove necessary manual memory management

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `deep`
    - Reason: Complex refactoring requiring deep understanding of memory patterns
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: NO (sequential after Wave 2)
  - **Parallel Group**: Sequential (depends on Tasks 5-8)
  - **Blocks**: Task 25 (tests)
  - **Blocked By**: Tasks 5-8 (buffer fixes first)

  **References** (CRITICAL - Be Exhaustive):

  **Pattern References** (existing code to follow):
  - `MsBinaryFile/PptFile/Drawing/XmlStringWriter.h:77-107` - Manual memory example
  - `Common/3dParty/cryptopp/` - Good RAII patterns to follow

  **WHY Each Reference Matters**:
  - XmlStringWriter shows problematic manual memory
  - Crypto++ shows proper RAII patterns

  **Acceptance Criteria**:
  - [ ] valgrind reports 0 memory leaks in core modules
  - [ ] Smart pointers used where appropriate
  - [ ] Memory ownership documented

  **QA Scenarios (MANDATORY — task is INCOMPLETE without these):**

  ```
  Scenario: Memory leak detection
    Tool: Bash (tmux)
    Preconditions: Code modified
    Steps:
      1. Build with debug symbols
      2. Run valgrind on test suite: valgrind --leak-check=full ./tests
      3. Parse output for leak summary
      4. Document any remaining leaks
    Expected Result: 0 leaks in fixed code, documented leaks elsewhere
    Failure Indicators: New leaks introduced, tests crash
    Evidence: .sisyphus/evidence/task-9-valgrind-report.txt
  ```

  **Evidence to Capture**:
  - [ ] Each evidence file named: task-{N}-{scenario-slug}.{ext}
  - [ ] Screenshots for UI, terminal output for CLI, response bodies for API

  **Commit**: NO (part of larger refactoring)
  - Message: `refactor(memory): improve memory management in MsBinaryFile`
  - Files: `MsBinaryFile/PptFile/Drawing/XmlStringWriter.h`

- [ ] 10. Update zlib to 1.2.13+

  **What to do**:
  - Verify zlib-1.3.1 is present in OfficeUtils/src/
  - Update build system to use zlib-1.3.1 instead of 1.2.11
  - Remove zlib-1.2.11.backup to prevent confusion
  - Test compression/decompression functionality
  - Update 3DPARTY.md with new version

  **Must NOT do**:
  - Do not modify zlib source code
  - Do not change API interfaces

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `medium`
    - Reason: Dependency update requires careful build system changes
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (with Tasks 11, 12, 13)
  - **Blocks**: Task 14 (regression testing)
  - **Blocked By**: Task 2 (dependency audit)

  **References**:
  - `OfficeUtils/src/zlib-1.3.1/` - Target zlib version
  - `OfficeUtils/src/zlib-1.2.11.backup/` - Old version to remove
  - `3DPARTY.md` - Update version documentation

  **Acceptance Criteria**:
  - [ ] grep -r "zlib.*1.2.11" shows no references (except backup)
  - [ ] Build succeeds with zlib-1.3.1
  - [ ] Compression tests pass

  **QA Scenarios**:
  ```
  Scenario: zlib updated to secure version
    Tool: Bash (tmux)
    Preconditions: Codebase in working state
    Steps:
      1. Remove zlib-1.2.11.backup: rm -rf OfficeUtils/src/zlib-1.2.11.backup
      2. Verify zlib-1.3.1 exists and is referenced in build files
      3. Build project: cmake -B build && cmake --build build
      4. Run compression tests: ctest -R zlib
    Expected Result: Build succeeds, tests pass, old version removed
    Failure Indicators: Build fails, tests fail, old version still referenced
    Evidence: .sisyphus/evidence/task-10-zlib-update.txt
  ```

  **Commit**: YES
  - Message: `chore(deps): update zlib from 1.2.11 to 1.3.1`
  - Files: `OfficeUtils/src/`, `3DPARTY.md`

- [ ] 11. Update freetype to 2.13+

  **What to do**:
  - Verify freetype-2.13.2 is being used (already present)
  - Check if freetype-2.5.2 is still referenced anywhere
  - Update build system to use freetype-2.13.2 exclusively
  - Remove or archive freetype-2.5.2
  - Test font rendering functionality

  **Must NOT do**:
  - Do not modify freetype source
  - Do not change font API

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `medium`
    - Reason: Dependency update with potential rendering implications
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (with Tasks 10, 12, 13)
  - **Blocks**: Task 14
  - **Blocked By**: Task 2

  **References**:
  - `DesktopEditor/freetype-2.13.2/` - Target version (already present)
  - `DesktopEditor/freetype-2.5.2/` - Old version to remove

  **Acceptance Criteria**:
  - [ ] Build uses freetype-2.13.2
  - [ ] No references to freetype-2.5.2 in build files
  - [ ] Font rendering tests pass

  **QA Scenarios**:
  ```
  Scenario: freetype updated to secure version
    Tool: Bash (tmux)
    Preconditions: Codebase in working state
    Steps:
      1. Check build files for freetype version references
      2. Verify freetype-2.13.2 is the active version
      3. Build and run font rendering tests
      4. Compare rendering output with baseline
    Expected Result: Correct version used, rendering matches baseline
    Failure Indicators: Wrong version, rendering differences
    Evidence: .sisyphus/evidence/task-11-freetype-update.txt
  ```

  **Commit**: YES
  - Message: `chore(deps): update freetype to 2.13.2`
  - Files: `DesktopEditor/`, `3DPARTY.md`

- [ ] 12. Update openjpeg to 2.5+

  **What to do**:
  - Search for current openjpeg version in codebase
  - Download/openjpeg 2.5+ if not present
  - Integrate into build system
  - Test JPEG2000 decoding/encoding
  - Update documentation

  **Must NOT do**:
  - Do not break existing image handling
  - Do not change public API

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `medium`
    - Reason: Library integration with testing requirements
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (with Tasks 10, 11, 13)
  - **Blocks**: Task 14
  - **Blocked By**: Task 2

  **References**:
  - Search for openjpeg usage in codebase
  - OpenJPEG official docs: https://github.com/uclouvain/openjpeg

  **Acceptance Criteria**:
  - [ ] openjpeg 2.5+ integrated
  - [ ] JPEG2000 tests pass
  - [ ] No regression in image handling

  **QA Scenarios**:
  ```
  Scenario: openjpeg updated and integrated
    Tool: Bash (tmux)
    Preconditions: openjpeg 2.5+ downloaded
    Steps:
      1. Integrate openjpeg into build system
      2. Build project with openjpeg support
      3. Run JPEG2000 test suite
      4. Verify image quality matches baseline
    Expected Result: Build succeeds, tests pass, quality maintained
    Failure Indicators: Build errors, test failures, quality degradation
    Evidence: .sisyphus/evidence/task-12-openjpeg-update.txt
  ```

  **Commit**: YES
  - Message: `chore(deps): update openjpeg to 2.5+`
  - Files: `Common/3dParty/`, `3DPARTY.md`

- [ ] 13. Update libdjvu

  **What to do**:
  - Identify current libdjvu version in use
  - Download latest stable libdjvu (3.5.28+)
  - Replace existing libdjvu directory
  - Update build configuration
  - Test DjVu file parsing
  - Ensure GString.cpp and GURL.cpp fixes still work

  **Must NOT do**:
  - Do not break DjVu functionality
  - Do not lose existing security fixes

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `unspecified-high`
    - Reason: Complex library replacement with security implications
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (with Tasks 10, 11, 12)
  - **Blocks**: Task 14
  - **Blocked By**: Tasks 5-7 (buffer fixes must be preserved)

  **References**:
  - `DjVuFile/libdjvu/` - Current version
  - `DjVuFile/libdjvu-3.5.28.backup/` - May contain newer version
  - Tasks 5-7: Security fixes to preserve

  **Acceptance Criteria**:
  - [ ] libdjvu updated to latest stable
  - [ ] Security fixes from Tasks 5-7 preserved
  - [ ] DjVu parsing tests pass

  **QA Scenarios**:
  ```
  Scenario: libdjvu updated with security fixes preserved
    Tool: Bash (tmux)
    Preconditions: Tasks 5-7 completed
    Steps:
      1. Backup current libdjvu with fixes
      2. Replace with newer version
      3. Re-apply security fixes (GString.cpp, GURL.cpp)
      4. Build and run DjVu tests
      5. Run static analysis to verify no new issues
    Expected Result: New version with fixes applied, tests pass
    Failure Indicators: Build fails, tests fail, new security issues
    Evidence: .sisyphus/evidence/task-13-libdjvu-update.txt
  ```

  **Commit**: YES
  - Message: `chore(deps): update libdjvu to latest stable`
  - Files: `DjVuFile/libdjvu/`, `3DPARTY.md`

- [ ] 14. Test regression suite after updates

  **What to do**:
  - Run full test suite after all dependency updates
  - Compare results with baseline from Task 3
  - Document any regressions
  - Fix critical issues before proceeding
  - Generate regression report

  **Must NOT do**:
  - Do not skip any tests
  - Do not ignore failures

  **Recommended Agent Profile**:
  > Select category + skills based on task domain. Justify each choice.
  - **Category**: `deep`
    - Reason: Comprehensive testing requiring analysis of failures
  - **Skills**: []
    - No additional skills needed

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (after Tasks 10-13)
  - **Blocks**: Wave 4 (Refactoring)
  - **Blocked By**: Tasks 10-13

  **References**:
  - Task 3: Security baseline testing
  - Test suite: `Test/` directory

  **Acceptance Criteria**:
  - [ ] All tests pass or expected failures documented
  - [ ] No new regressions introduced
  - [ ] Regression report generated

  **QA Scenarios**:
  ```
  Scenario: Full regression test
    Tool: Bash (tmux)
    Preconditions: Tasks 10-13 complete
    Steps:
      1. Build project with updated dependencies
      2. Run full test suite: ctest --output-on-failure
      3. Compare with baseline from Task 3
      4. Document any new failures
      5. Fix critical regressions
    Expected Result: All tests pass or documented expected failures
    Failure Indicators: New test failures, crashes
    Evidence: .sisyphus/evidence/task-14-regression-report.txt
  ```

  **Commit**: NO (summary commit after Wave 3)
  - Message: `chore(deps): dependency updates completed - zlib, freetype, openjpeg, libdjvu`
  - Files: Multiple

---

## Wave 4: Refactoring (Tasks 15-19)

- [ ] 15. Design converter interface

  **What to do**:
  - Analyze ASCOfficeDrawingConverter to identify responsibilities
  - Define abstract base class for drawing conversion
  - Create interface specifications for each conversion type
  - Document module boundaries and dependencies
  - Design backward-compatible API for existing callers

  **Must NOT do**:
  - Do not modify implementation yet (design only)
  - Do not break existing API contracts

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Design and documentation task
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (first in Wave 4)
  - **Blocks**: Tasks 16-18
  - **Blocked By**: Task 14

  **References**:
  - `ASCOfficeDrawingConverter.cpp` - 6026-line god class to refactor
  - Design patterns: Strategy pattern for conversion algorithms

  **Acceptance Criteria**:
  - [ ] Interface design document created
  - [ ] Module boundaries clearly defined
  - [ ] API compatibility matrix documented

  **QA Scenarios**:
  ```
  Scenario: Interface design complete
    Tool: Read file
    Steps:
      1. Review design document
      2. Verify all responsibilities from god class are mapped
      3. Confirm API compatibility plan
    Expected Result: Complete design with clear boundaries
    Evidence: .sisyphus/evidence/task-15-design.md
  ```

  **Commit**: YES
  - Message: `docs: design converter interface for refactoring`
  - Files: `docs/converter-interface-design.md`

- [ ] 16. Extract drawing conversion logic

  **What to do**:
  - Identify distinct conversion operations in ASCOfficeDrawingConverter
  - Create separate classes: ShapeConverter, TextConverter, ImageConverter, etc.
  - Move implementation logic to new classes
  - Keep delegation in original class during transition

  **Must NOT do**:
  - Do not change external behavior
  - Do not remove functionality

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Complex extraction requiring deep understanding
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 4 (with Task 17, 18)
  - **Blocks**: Task 19
  - **Blocked By**: Task 15

  **References**:
  - Task 15: Interface design
  - `ASCOfficeDrawingConverter.cpp` lines by functionality

  **Acceptance Criteria**:
  - [ ] New converter classes created
  - [ ] Logic moved without behavior change
  - [ ] Tests still pass

  **QA Scenarios**:
  ```
  Scenario: Conversion logic extracted
    Tool: Bash (tmux)
    Steps:
      1. Build with new classes
      2. Run conversion tests
      3. Verify output matches baseline
    Expected Result: All tests pass, output identical
    Evidence: .sisyphus/evidence/task-16-extraction.txt
  ```

  **Commit**: YES
  - Message: `refactor: extract drawing conversion logic to separate classes`
  - Files: `Converter/*.cpp`, `Converter/*.h`

- [ ] 17. Split ASCOfficeDrawingConverter

  **What to do**:
  - Break 6026-line file into 5-7 focused classes
  - Create: DrawingConverterBase, ShapeConverter, TextConverter, ImageConverter, GroupConverter, EffectConverter
  - Move remaining code to appropriate classes
  - Update includes and dependencies

  **Must NOT do**:
  - Do not create new god classes
  - Do not increase coupling

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Large refactoring with many files
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 4 (with Tasks 16, 18)
  - **Blocks**: Task 19
  - **Blocked By**: Tasks 15-16

  **References**:
  - Task 15: Interface design
  - Task 16: Extracted logic

  **Acceptance Criteria**:
  - [ ] Original file removed or reduced to <500 lines
  - [ ] 5-7 new focused classes created
  - [ ] No circular dependencies

  **QA Scenarios**:
  ```
  Scenario: God class split complete
    Tool: Bash (tmux)
    Steps:
      1. Check file sizes: wc -l ASCOfficeDrawingConverter.cpp
      2. Verify new class structure
      3. Build and test
    Expected Result: Small coordinator file, focused classes
    Evidence: .sisyphus/evidence/task-17-split.txt
  ```

  **Commit**: YES
  - Message: `refactor: split ASCOfficeDrawingConverter into modular components`
  - Files: `Converter/*.cpp`, `Converter/*.h`

- [ ] 18. Implement plugin architecture

  **What to do**:
  - Create plugin loading mechanism
  - Define plugin interface (IDrawingConverterPlugin)
  - Implement registry for format handlers
  - Support dynamic loading of new converters

  **Must NOT do**:
  - Do not make it complex
  - Do not require recompilation for new formats

  **Recommended Agent Profile**:
  - **Category**: `deep`
    - Reason: Architecture design with runtime implications
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 4 (with Tasks 16-17)
  - **Blocks**: Task 19
  - **Blocked By**: Task 15

  **References**:
  - Task 15: Interface design
  - Plugin pattern examples

  **Acceptance Criteria**:
  - [ ] Plugin interface defined
  - [ ] Registry implementation complete
  - [ ] Can load converter plugin dynamically

  **QA Scenarios**:
  ```
  Scenario: Plugin architecture working
    Tool: Bash (tmux)
    Steps:
      1. Build plugin system
      2. Create test plugin
      3. Load plugin at runtime
      4. Verify converter works through plugin
    Expected Result: Plugin loads and converts correctly
    Evidence: .sisyphus/evidence/task-18-plugin.txt
  ```

  **Commit**: YES
  - Message: `feat: add plugin architecture for drawing converters`
  - Files: `Plugin/*.cpp`, `Plugin/*.h`

- [ ] 19. Update build system for modules

  **What to do**:
  - Update Makefile/CMakeLists.txt for new structure
  - Create proper dependency ordering
  - Add build targets for each module
  - Verify incremental builds work

  **Must NOT do**:
  - Do not break existing build process
  - Do not increase build time significantly

  **Recommended Agent Profile**:
  - **Category**: `medium`
    - Reason: Build system configuration
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (last in Wave 4)
  - **Blocks**: Wave 5
  - **Blocked By**: Tasks 16-18

  **References**:
  - `build/Makefile` or `CMakeLists.txt`
  - Tasks 16-18: New modules

  **Acceptance Criteria**:
  - [ ] Build succeeds with new structure
  - [ ] Incremental builds work
  - [ ] All modules link correctly

  **QA Scenarios**:
  ```
  Scenario: Build system updated
    Tool: Bash (tmux)
    Steps:
      1. Clean build: make clean && make
      2. Incremental build: touch file.cpp && make
      3. Verify all targets build
    Expected Result: Clean and incremental builds succeed
    Evidence: .sisyphus/evidence/task-19-build.txt
  ```

  **Commit**: YES
  - Message: `build: update build system for modular converter architecture`
  - Files: `build/Makefile`, `CMakeLists.txt`

---

## Wave 5: Input Validation & Security Hardening (Tasks 20-24)

- [ ] 20. Create input validation framework

  **What to do**:
  - Design input validation interface
  - Implement validators for common input types (strings, numbers, file paths)
  - Add sanitization functions for user input
  - Create validation middleware for APIs
  - Integrate with existing codebase

  **Must NOT do**:
  - Do not use regex-only validation
  - Do not skip error handling

  **Recommended Agent Profile**:
  - **Category**: `deep`
    - Reason: Security-critical framework design
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (first in Wave 5)
  - **Blocks**: Tasks 21-24
  - **Blocked By**: Task 19

  **References**:
  - OWASP Input Validation Cheat Sheet
  - Existing API endpoints

  **Acceptance Criteria**:
  - [ ] Validation framework implemented
  - [ ] All input points use validation
  - [ ] Invalid input rejected gracefully

  **QA Scenarios**:
  ```
  Scenario: Input validation working
    Tool: Bash (curl)
    Steps:
      1. Send valid input to API
      2. Send invalid input (SQL injection, XSS)
      3. Verify invalid input rejected
      4. Check error messages don't leak info
    Expected Result: Valid accepted, invalid rejected safely
    Evidence: .sisyphus/evidence/task-20-validation.txt
  ```

  **Commit**: YES
  - Message: `feat: add input validation framework`
  - Files: `Validation/*.cpp`, `Validation/*.h`

- [ ] 21. Add XSS protection

  **What to do**:
  - Implement output encoding for HTML contexts
  - Add Content-Security-Policy headers
  - Sanitize user-generated content
  - Protect against DOM-based XSS

  **Must NOT do**:
  - Do not rely on single protection layer
  - Do not encode already-safe content

  **Recommended Agent Profile**:
  - **Category**: `medium`
    - Reason: Security implementation
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 5 (with Tasks 22-24)
  - **Blocks**: Task 25
  - **Blocked By**: Task 20

  **References**:
  - OWASP XSS Prevention Cheat Sheet
  - Task 20: Validation framework

  **Acceptance Criteria**:
  - [ ] XSS payloads blocked
  - [ ] CSP headers present
  - [ ] Output encoding implemented

  **QA Scenarios**:
  ```
  Scenario: XSS protection active
    Tool: Bash (curl)
    Steps:
      1. Send XSS payload in input
      2. Verify payload encoded in output
      3. Check CSP headers present
      4. Attempt DOM-based XSS
    Expected Result: All XSS attempts blocked
    Evidence: .sisyphus/evidence/task-21-xss.txt
  ```

  **Commit**: YES
  - Message: `security: add XSS protection measures`
  - Files: `Security/xss-protection.cpp`

- [ ] 22. Implement CSRF tokens

  **What to do**:
  - Generate unique CSRF tokens per session
  - Add token validation middleware
  - Include tokens in forms and AJAX requests
  - Implement token rotation

  **Must NOT do**:
  - Do not use predictable tokens
  - Do not store tokens in URL

  **Recommended Agent Profile**:
  - **Category**: `medium`
    - Reason: Security implementation
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 5 (with Tasks 21, 23, 24)
  - **Blocks**: Task 25
  - **Blocked By**: Task 20

  **References**:
  - OWASP CSRF Prevention Cheat Sheet

  **Acceptance Criteria**:
  - [ ] CSRF tokens generated
  - [ ] Token validation enforced
  - [ ] Invalid requests rejected

  **QA Scenarios**:
  ```
  Scenario: CSRF protection working
    Tool: Bash (curl)
    Steps:
      1. Make request without token
      2. Make request with invalid token
      3. Make request with valid token
      4. Attempt CSRF attack simulation
    Expected Result: Only valid token requests succeed
    Evidence: .sisyphus/evidence/task-22-csrf.txt
  ```

  **Commit**: YES
  - Message: `security: implement CSRF token protection`
  - Files: `Security/csrf-protection.cpp`

- [ ] 23. Add CORS configuration

  **What to do**:
  - Define allowed origins
  - Configure allowed methods and headers
  - Implement preflight request handling
  - Set appropriate credentials policy

  **Must NOT do**:
  - Do not allow * origin in production
  - Do not over-permit methods

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Configuration task
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 5 (with Tasks 21-22, 24)
  - **Blocks**: Task 25
  - **Blocked By**: Task 20

  **References**:
  - OWASP CORS Policy

  **Acceptance Criteria**:
  - [ ] CORS headers present
  - [ ] Only allowed origins accepted
  - [ ] Preflight handled correctly

  **QA Scenarios**:
  ```
  Scenario: CORS configured correctly
    Tool: Bash (curl)
    Steps:
      1. Send request from allowed origin
      2. Send request from blocked origin
      3. Send preflight request
      4. Verify CORS headers
    Expected Result: Only allowed origins succeed
    Evidence: .sisyphus/evidence/task-23-cors.txt
  ```

  **Commit**: YES
  - Message: `config: add CORS configuration`
  - Files: `config/cors.json`, `Security/cors.cpp`

- [ ] 24. Security headers for WebSocket

  **What to do**:
  - Add WSS (WebSocket Secure) support
  - Implement origin validation
  - Add rate limiting for connections
  - Configure message size limits

  **Must NOT do**:
  - Do not use unencrypted WebSockets
  - Do not allow unlimited message sizes

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Security configuration
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 5 (with Tasks 21-23)
  - **Blocks**: Task 25
  - **Blocked By**: Task 20

  **References**:
  - WebSocket security best practices

  **Acceptance Criteria**:
  - [ ] WSS only (no WS)
  - [ ] Origin validation active
  - [ ] Rate limiting enforced

  **QA Scenarios**:
  ```
  Scenario: WebSocket security active
    Tool: Bash (tmux)
    Steps:
      1. Attempt WS connection (should fail)
      2. Attempt WSS connection
      3. Attempt from wrong origin
      4. Test rate limiting
    Expected Result: Only secure, authorized connections succeed
    Evidence: .sisyphus/evidence/task-24-websocket.txt
  ```

  **Commit**: YES
  - Message: `security: add WebSocket security headers and protections`
  - Files: `WebSocket/security.cpp`

---

## Wave 6: Testing & Coverage (Tasks 25-29)

- [ ] 25. Unit tests for buffer fixes

  **What to do**:
  - Create unit tests for GString.cpp fixes
  - Create unit tests for GURL.cpp fixes
  - Create unit tests for PdfFile sscanf fixes
  - Create unit tests for RAII wrapper
  - Achieve 100% coverage on security-critical code

  **Must NOT do**:
  - Do not write tests that pass without implementation
  - Do not skip edge cases

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Comprehensive test writing
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 6 (with Tasks 26-29)
  - **Blocks**: Task 29
  - **Blocked By**: Tasks 5-9 (security fixes)

  **References**:
  - Tasks 5-9: Security fixes to test
  - `Test/` directory for patterns

  **Acceptance Criteria**:
  - [ ] All buffer fixes have tests
  - [ ] Tests cover happy and error paths
  - [ ] 100% coverage on security modules

  **QA Scenarios**:
  ```
  Scenario: Buffer fix tests passing
    Tool: Bash (tmux)
    Steps:
      1. Run unit tests: ctest -R buffer
      2. Check coverage: gcov -l
      3. Verify all security functions tested
    Expected Result: All tests pass, 100% coverage
    Evidence: .sisyphus/evidence/task-25-unit-tests.txt
  ```

  **Commit**: YES
  - Message: `test: add unit tests for buffer overflow fixes`
  - Files: `Test/buffer-fixes.test.cpp`

- [ ] 26. Fuzz testing for parsers

  **What to do**:
  - Set up libFuzzer or AFL++
  - Create fuzz targets for DjVu parser
  - Create fuzz targets for PDF parser
  - Run fuzzing for 24+ hours
  - Analyze and fix discovered crashes

  **Must NOT do**:
  - Do not skip crash analysis
  - Do not ignore edge cases found

  **Recommended Agent Profile**:
  - **Category**: `unspecified-high`
    - Reason: Fuzzing setup and analysis
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 6 (with Tasks 25, 27-29)
  - **Blocks**: Task 29
  - **Blocked By**: Tasks 5-8 (parser fixes)

  **References**:
  - libFuzzer documentation
  - Task 3: Fuzz harness setup

  **Acceptance Criteria**:
  - [ ] Fuzz targets created
  - [ ] 24+ hours of fuzzing completed
  - [ ] No crashes found or all fixed

  **QA Scenarios**:
  ```
  Scenario: Fuzz testing complete
    Tool: Bash (tmux)
    Steps:
      1. Build fuzz targets
      2. Run fuzzer for extended period
      3. Check for crashes
      4. Analyze any findings
    Expected Result: No crashes, or all fixed
    Evidence: .sisyphus/evidence/task-26-fuzz-report.txt
  ```

  **Commit**: YES
  - Message: `test: add fuzz testing for document parsers`
  - Files: `Test/fuzz-*.cpp`

- [ ] 27. Integration tests for conversion pipeline

  **What to do**:
  - Create end-to-end conversion tests
  - Test full document processing pipeline
  - Verify output quality matches expectations
  - Test with real-world documents

  **Must NOT do**:
  - Do not test in isolation
  - Do not use only synthetic data

  **Recommended Agent Profile**:
  - **Category**: `deep`
    - Reason: Complex integration testing
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 6 (with Tasks 25-26, 28-29)
  - **Blocks**: Task 29
  - **Blocked By**: Wave 4 (refactoring complete)

  **References**:
  - Test documents corpus
  - Wave 4: Refactored converters

  **Acceptance Criteria**:
  - [ ] Integration tests pass
  - [ ] Output quality verified
  - [ ] Real documents tested

  **QA Scenarios**:
  ```
  Scenario: Conversion pipeline integration
    Tool: Bash (tmux)
    Steps:
      1. Run integration test suite
      2. Compare output with baseline
      3. Verify no regressions
    Expected Result: All conversions produce correct output
    Evidence: .sisyphus/evidence/task-27-integration.txt
  ```

  **Commit**: YES
  - Message: `test: add integration tests for conversion pipeline`
  - Files: `Test/integration-tests.cpp`

- [ ] 28. Coverage reporting setup

  **What to do**:
  - Configure gcov/lcov for coverage reporting
  - Set up coverage dashboard
  - Create coverage thresholds
  - Integrate with CI/CD pipeline

  **Must NOT do**:
  - Do not set thresholds too low
  - Do not skip coverage in CI

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Tooling configuration
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 6 (with Tasks 25-27, 29)
  - **Blocks**: Task 29
  - **Blocked By**: Task 4 (CI setup)

  **References**:
  - Task 4: CI/CD pipeline
  - gcov/lcov documentation

  **Acceptance Criteria**:
  - [ ] Coverage reports generated
  - [ ] Dashboard accessible
  - [ ] CI integration working

  **QA Scenarios**:
  ```
  Scenario: Coverage reporting working
    Tool: Bash (tmux)
    Steps:
      1. Build with coverage flags
      2. Run tests
      3. Generate coverage report
      4. Verify report accuracy
    Expected Result: Coverage report generated correctly
    Evidence: .sisyphus/evidence/task-28-coverage.txt
  ```

  **Commit**: YES
  - Message: `ci: add coverage reporting to pipeline`
  - Files: `.github/workflows/coverage.yml`, `scripts/coverage.sh`

- [ ] 29. Achieve 80%+ coverage target

  **What to do**:
  - Review coverage report
  - Identify uncovered code
  - Add tests for uncovered paths
  - Verify 80%+ coverage achieved
  - Document coverage by module

  **Must NOT do**:
  - Do not artificially inflate coverage
  - Do not ignore critical uncovered code

  **Recommended Agent Profile**:
  - **Category**: `deep`
    - Reason: Comprehensive test improvement
  - **Skills**: []

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (last in Wave 6)
  - **Blocks**: Final Verification
  - **Blocked By**: Tasks 25-28

  **References**:
  - Task 28: Coverage report
  - All previous tasks

  **Acceptance Criteria**:
  - [ ] Overall coverage >= 80%
  - [ ] Security modules >= 90%
  - [ ] Coverage report published

  **QA Scenarios**:
  ```
  Scenario: Coverage target achieved
    Tool: Bash (tmux)
    Steps:
      1. Generate coverage report
      2. Check overall coverage
      3. Verify security module coverage
      4. Document results
    Expected Result: 80%+ overall, 90%+ security modules
    Evidence: .sisyphus/evidence/task-29-coverage-target.txt
  ```

  **Commit**: YES
  - Message: `test: achieve 80%+ code coverage target`
  - Files: Coverage reports, additional tests

---

## Final Verification Wave (MANDATORY — after ALL implementation tasks)

> 4 review agents run in PARALLEL. ALL must APPROVE. Present consolidated results to user and get explicit "okay" before completing.
>
> **Do NOT auto-proceed after verification. Wait for user's explicit approval before marking work complete.**
> **Never mark F1-F4 as checked before getting user's okay.** Rejection or user feedback -> fix -> re-run -> present again -> wait for okay.

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Read the plan end-to-end. For each "Must Have": verify implementation exists (read file, curl endpoint, run command). For each "Must NOT Have": search codebase for forbidden patterns — reject with file:line if found. Check evidence files exist in .sisyphus/evidence/. Compare deliverables against plan.
  Output: `Must Have [N/N] | Must NOT Have [N/N] | Tasks [N/N] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  Run `tsc --noEmit` + linter + `bun test`. Review all changed files for: `as any`/`@ts-ignore`, empty catches, console.log in prod, commented-out code, unused imports. Check AI slop: excessive comments, over-abstraction, generic names (data/result/item/temp).
  Output: `Build [PASS/FAIL] | Lint [PASS/FAIL] | Tests [N pass/N fail] | Files [N clean/N issues] | VERDICT`

- [ ] F3. **Real Manual QA** — `unspecified-high` (+ `playwright` skill if UI)
  Start from clean state. Execute EVERY QA scenario from EVERY task — follow exact steps, capture evidence. Test cross-task integration (features working together, not isolation). Test edge cases: empty state, invalid input, rapid actions. Save to `.sisyphus/evidence/final-qa/`.
  Output: `Scenarios [N/N pass] | Integration [N/N] | Edge Cases [N tested] | VERDICT`

- [ ] F4. **Scope Fidelity Check** — `deep`
  For each task: read "What to do", read actual diff (git log/diff). Verify 1:1 — everything in spec was built (no missing), nothing beyond spec was built (no creep). Check "Must NOT do" compliance. Detect cross-task contamination: Task N touching Task M's files. Flag unaccounted changes.
  Output: `Tasks [N/N compliant] | Contamination [CLEAN/N issues] | Unaccounted [CLEAN/N files] | VERDICT`

---

## Commit Strategy

- **Security fixes**: `fix(security): replace unsafe string function in [module]`
- **Dependency updates**: `chore(deps): update [library] from v1 to v2`
- **Refactoring**: `refactor([module]): split [class] into modular components`
- **Tests**: `test([module]): add coverage for [feature]`

---

## Success Criteria

### Verification Commands
```bash
# Static analysis
clang-tidy --warnings-as-errors=* src/**/*.cpp

# Security scanning
dependency-check -project DocumentServer -scan ./third-party/

# Test coverage
ctest --output-on-failure && gcov -l | grep "coverage:"

# Build with strict flags
cmake -DCMAKE_CXX_FLAGS="-Wall -Wextra -Werror" -B build && cmake --build build
```

### Final Checklist
- [ ] All critical/high security vulnerabilities remediated
- [ ] No compiler warnings (-Wall -Wextra -Werror passes)
- [ ] Static analysis shows 0 critical issues
- [ ] All third-party dependencies updated to secure versions
- [ ] Test coverage report shows 80%+ for security modules
- [ ] CI/CD pipeline includes security scanning
