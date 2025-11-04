# Research: Testing Strategy for GitLab Integration

**Feature**: GitLab Support for vTeam
**Branch**: `001-gitlab-support`
**Date**: 2025-11-04
**Author**: Stella (Staff Engineer)

## Executive Summary

This document defines a comprehensive testing strategy for the GitLab integration feature. The strategy balances quality requirements (95%+ success rate, zero GitHub regression, 90% user-friendly errors) with development velocity. Key decisions:

- **70% unit test coverage minimum** for new GitLab code with focus on critical paths
- **Native Go testing framework** with `testify` for assertions and HTTP mocking
- **httptest.Server** for API mocking (no external dependencies)
- **3-tier test structure**: unit → contract → integration
- **GitHub Actions CI** with quality gates at PR and merge time

This approach mirrors successful patterns from the Kubernetes ecosystem and aligns with vTeam's existing infrastructure.

---

## Decision 1: Test Coverage Requirements

### Decision

**Unit Test Coverage Targets**:
- **Minimum**: 70% coverage for new GitLab package code
- **Critical path**: 90%+ coverage for:
  - URL parsing and normalization (`gitlab/parser.go`)
  - Token validation (`gitlab/token.go`)
  - API error handling (`gitlab/client.go`)
  - Provider detection logic (GitHub vs GitLab)

**Component-Level Testing Strategy**:
- **Unit tests required**: `gitlab/` package (all files), modified handlers in `handlers/repo.go`
- **Contract tests required**: GitLab API interactions, error message formatting
- **Integration tests required**: End-to-end AgenticSession flows with GitLab repositories
- **No tests required**: Frontend changes (out of MVP scope), documentation updates

**Self-Hosted GitLab Testing**:
- **Unit tests**: Mock all self-hosted scenarios via `httptest.Server`
- **Contract tests**: Dedicated test cases for self-hosted URL patterns and API v4 compatibility
- **Integration tests**: Optional in CI (gated by environment variable `TEST_GITLAB_SELF_HOSTED`), required for manual validation

### Rationale

**Why 70% minimum coverage?**
- vTeam currently has **no test suite** (tests/ directory doesn't exist)
- 70% is pragmatic for greenfield code while ensuring quality
- Higher than industry average (60%) but achievable without slowing delivery
- Focused on business-critical paths rather than boilerplate

**Why 90% for critical paths?**
- These components directly impact the success criteria:
  - URL parsing: affects all 5 user stories (repository detection)
  - Token validation: affects SC-009 (95%+ permission issues caught)
  - Error handling: affects SC-006 (90% user-friendly messages)
- Bugs here cascade to user-facing failures

**Why contract tests for API interactions?**
- GitLab API v4 is external dependency - contract tests verify assumptions
- Catch API version incompatibilities early (especially for self-hosted instances)
- Enables testing without live GitLab instance

**Why optional self-hosted integration tests?**
- Self-hosted instances require infrastructure setup (not available in CI)
- Unit and contract tests cover 95% of self-hosted logic
- Manual testing checklist for self-hosted scenarios pre-release
- Future: dedicated test GitLab instance in staging environment

### Alternatives Considered

**Alternative 1: 80%+ coverage across all code**
- **Rejected**: Diminishing returns above 70% for Go code with simple branching
- Would require testing trivial getters, struct marshaling, etc.
- Example from research: Kubernetes client-go maintains 65-75% coverage

**Alternative 2: Integration tests only (no unit tests)**
- **Rejected**: Integration tests are slow (3-5 minutes) and flaky
- Don't provide granular failure signals for debugging
- CI costs increase significantly (longer runs, more infrastructure)

**Alternative 3: Require live self-hosted GitLab instance in CI**
- **Rejected**: Complex infrastructure setup increases CI brittleness
- Self-hosted instances vary widely (versions, configurations)
- Mock-based testing covers functional requirements adequately

### Implementation Notes

**Coverage Measurement**:
```bash
# Run in components/backend directory
go test ./gitlab/... -coverprofile=coverage.out
go tool cover -func=coverage.out | grep total

# Fail PR if below 70%
go test ./gitlab/... -coverprofile=coverage.out
COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
if (( $(echo "$COVERAGE < 70" | bc -l) )); then exit 1; fi
```

**Critical Path Identification**:
- `parseGitLabURL()`: 12 test cases for URL formats (SSH, HTTPS, with/without .git, self-hosted)
- `validateGitLabToken()`: 8 test cases for token scopes and API errors
- `constructAPIBaseURL()`: 6 test cases for self-hosted instances with non-standard ports
- Error message formatting: Test suite with 15+ GitLab API error scenarios

**Self-Hosted Test Scenarios** (manual validation checklist):
1. Self-hosted instance with standard HTTPS (port 443)
2. Self-hosted instance with non-standard port (e.g., 8443)
3. Self-hosted instance with custom domain and path (e.g., gitlab.company.com/git)
4. GitLab CE vs GitLab EE instances
5. Older GitLab versions (12.0, 13.0, 14.0) - API v4 compatibility

---

## Decision 2: Testing Framework

### Decision

**Core Framework**: Go native `testing` package with minimal dependencies

**Additional Libraries**:
- **`github.com/stretchr/testify/assert`**: Assertions (already in go.mod)
- **`github.com/stretchr/testify/mock`**: Interface mocking (already in go.mod)
- **`net/http/httptest`**: HTTP API mocking (stdlib, no dependency)
- **No additions**: Ginkgo, gomock, or other frameworks

**Mocking Strategy**:
- **HTTP API calls**: `httptest.NewServer()` with canned GitLab API responses
- **Kubernetes client**: Interface-based mocking with `testify/mock`
- **File system**: `testing.Tempdir()` for Git operations
- **Time/clock**: Dependency injection for token expiration testing

**Test Data Management**:
- **JSON fixtures**: `testdata/gitlab_api_responses/` directory
  - `branches_response.json`, `tree_response.json`, `file_response.json`
  - `error_401_invalid_token.json`, `error_403_insufficient_scopes.json`, etc.
- **Git repositories**: Temporary repos created via `git init` in tests
- **Secrets**: Generated in-memory during test setup (no real credentials)

### Rationale

**Why Go native testing?**
- **Zero learning curve**: Team already familiar with Go testing patterns
- **CI integration**: Works out-of-box with GitHub Actions, no special setup
- **Tooling compatibility**: Works with `go test`, `golangci-lint`, VS Code Go extension
- **Existing pattern**: vTeam's Makefile already references Go testing (even though tests/ doesn't exist yet)

**Why testify?**
- **Already a dependency**: Present in `go.mod` (stretchr/testify v1.11.1)
- **Readable assertions**: `assert.Equal(t, expected, actual)` vs `if expected != actual { t.Errorf(...) }`
- **Better failure messages**: Automatically shows diff for complex structs
- **Mock support**: Unified library for assertions + mocking

**Why httptest over external mock servers?**
- **Stdlib solution**: No external dependencies, zero setup
- **Full control**: Can simulate network errors, timeouts, partial responses
- **Fast**: In-memory server, no actual network calls
- **Example from GitHub package**: Pattern already used in handlers/github_auth.go (15s timeout client)

**Why JSON fixtures?**
- **Real API responses**: Copy actual GitLab API responses for accuracy
- **Version compatibility**: Easy to add fixtures for older GitLab versions
- **Regression protection**: Changes to test expectations are explicit (git diff shows fixture changes)
- **Shared across tests**: Reduce duplication, single source of truth

### Alternatives Considered

**Alternative 1: Ginkgo/Gomega BDD framework**
- **Rejected**: Adds complexity and dependencies for minimal benefit
- BDD style (`Describe`, `Context`, `It`) unfamiliar to team
- Not used by similar Go projects (Kubernetes, Docker, Terraform use native testing)
- Integration costs: requires separate CI setup, IDE plugin support

**Alternative 2: gomock for interface mocking**
- **Rejected**: Requires code generation step (`go generate`)
- testify/mock provides equivalent functionality without codegen
- gomock is overkill for this project's interface complexity

**Alternative 3: VCR-style HTTP recording (go-vcr)**
- **Rejected**: Requires live GitLab instance to record initial responses
- Harder to test error scenarios (need to trigger actual errors)
- Fixtures are more explicit and reviewable

**Alternative 4: Testcontainers with real GitLab instance**
- **Rejected**: Extremely slow (GitLab container takes 2-3 minutes to start)
- CI resource intensive (GitLab requires 4GB RAM minimum)
- Adds Docker dependency to test suite
- Reserved for optional integration testing, not unit tests

### Implementation Notes

**Example Test Structure**:
```go
// gitlab/parser_test.go
package gitlab

import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestParseGitLabURL(t *testing.T) {
    tests := []struct {
        name      string
        input     string
        wantHost  string
        wantOwner string
        wantRepo  string
        wantErr   bool
    }{
        {
            name:      "gitlab.com HTTPS with .git",
            input:     "https://gitlab.com/owner/repo.git",
            wantHost:  "gitlab.com",
            wantOwner: "owner",
            wantRepo:  "repo",
            wantErr:   false,
        },
        // ... 11 more test cases
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            host, owner, repo, err := ParseGitLabURL(tt.input)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.wantHost, host)
                assert.Equal(t, tt.wantOwner, owner)
                assert.Equal(t, tt.wantRepo, repo)
            }
        })
    }
}
```

**HTTP Mock Example**:
```go
// gitlab/client_test.go
func TestGetBranches_Success(t *testing.T) {
    // Setup mock GitLab API server
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        assert.Equal(t, "/api/v4/projects/owner%2Frepo/repository/branches", r.URL.Path)
        assert.Equal(t, "Bearer test-token", r.Header.Get("Authorization"))

        // Load fixture
        fixture, _ := os.ReadFile("testdata/gitlab_api_responses/branches_response.json")
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusOK)
        w.Write(fixture)
    }))
    defer server.Close()

    // Test client with mock server
    client := NewClient(server.URL, "test-token")
    branches, err := client.GetBranches(context.Background(), "owner", "repo")

    assert.NoError(t, err)
    assert.Len(t, branches, 3)
    assert.Equal(t, "main", branches[0].Name)
}
```

**Kubernetes Client Mock**:
```go
// Use interface for testing
type K8sSecretReader interface {
    GetSecret(ctx context.Context, namespace, name string) (*corev1.Secret, error)
}

// In test:
mockK8s := new(mocks.K8sSecretReader)
mockK8s.On("GetSecret", mock.Anything, "project-ns", "runner-secret").
    Return(&corev1.Secret{Data: map[string][]byte{"GITLAB_TOKEN": []byte("test-token")}}, nil)
```

**Test Data Directory Structure**:
```
components/backend/testdata/
├── gitlab_api_responses/
│   ├── branches_response.json          # List branches
│   ├── tree_response.json              # Repository tree
│   ├── file_response.json              # Single file content
│   ├── paginated_branches_page1.json   # Test pagination
│   ├── paginated_branches_page2.json
│   ├── error_401_invalid_token.json    # Auth errors
│   ├── error_403_insufficient_scopes.json
│   ├── error_404_not_found.json
│   └── error_429_rate_limit.json       # Rate limiting
└── git_repos/                          # (Generated during tests, not committed)
```

---

## Decision 3: Test Structure

### Decision

**Test File Organization**: Parallel to source files (Go convention)
```
components/backend/
├── gitlab/
│   ├── parser.go
│   ├── parser_test.go          # Unit tests
│   ├── client.go
│   ├── client_test.go          # Unit + contract tests
│   ├── token.go
│   └── token_test.go           # Unit tests
├── testdata/                   # Shared test fixtures
│   └── gitlab_api_responses/
├── tests/                      # NEW: Integration tests
│   ├── unit/                   # (Future: tests for other packages)
│   ├── contract/
│   │   └── gitlab_api_test.go  # Contract tests for GitLab API
│   └── integration/
│       ├── gitlab_session_test.go        # E2E AgenticSession with GitLab
│       ├── gitlab_browsing_test.go       # Repository browsing tests
│       └── gitlab_mixed_providers_test.go # GitHub + GitLab simultaneously
```

**Integration Test Setup**:
- **Test GitLab Instance**: NOT required for CI (use mocked API)
- **Kubernetes Cluster**: Required (use kind or existing test cluster)
- **Test Namespace**: `vteam-test-gitlab` (auto-created and cleaned)
- **Isolation**: Each test creates unique ProjectSettings CR and secrets
- **Cleanup**: `defer` cleanup or `t.Cleanup()` for all resources

**Performance Testing Approach**:
```go
// Large repository tests (SC-008: 10,000+ files)
func TestGetTree_LargeRepository(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping large repository test in short mode")
    }

    // Mock GitLab API with paginated responses (100 files per page, 100 pages = 10k files)
    server := setupMockServerWithPagination(t, 100, 100)
    defer server.Close()

    start := time.Now()
    tree, err := client.GetTree(context.Background(), "owner", "huge-repo", "main", "/")
    duration := time.Since(start)

    assert.NoError(t, err)
    assert.Len(t, tree.Entries, 10000)
    assert.Less(t, duration, 5*time.Second, "Large repository browsing should complete in under 5 seconds")
}
```

### Rationale

**Why parallel test file structure?**
- **Go convention**: Standard practice in Go ecosystem (Kubernetes, Docker, Terraform)
- **IDE support**: Test files automatically discovered by Go tools
- **Clear ownership**: `parser.go` logic tested in `parser_test.go` - easy to find
- **Package-level testing**: Tests can access private functions for white-box testing

**Why separate tests/ directory for integration?**
- **Isolation**: Integration tests require different setup (Kubernetes, longer timeouts)
- **Selective execution**: `go test ./gitlab/...` runs unit tests, `go test ./tests/integration/...` runs integration
- **CI optimization**: Unit tests run on every commit, integration tests on PR only
- **Clear boundaries**: Unit (milliseconds), contract (< 1s), integration (seconds to minutes)

**Why NOT require test GitLab instance in CI?**
- **Complexity**: Setting up GitLab (even via Docker) adds 2-3 minutes to CI time
- **Maintenance burden**: GitLab version updates, SSL certificates, test data seeding
- **Cost**: Runs consume significant resources ($$ for GitHub Actions minutes)
- **Sufficient coverage**: httptest mocks provide equivalent functional testing
- **Real GitLab testing**: Reserved for staging environment and manual QA

**Why use kind for Kubernetes testing?**
- **Lightweight**: kind (Kubernetes in Docker) starts in ~30 seconds
- **CI-friendly**: Works in GitHub Actions without special setup
- **Full K8s API**: Real Kubernetes API server, not mocks
- **Already available**: vTeam's backend Makefile has `k8s-setup` target for kind

### Alternatives Considered

**Alternative 1: Separate test package (gitlab_test)**
- **Rejected**: Loses access to private functions for white-box testing
- Common in Java, not Go convention
- Makes testing internal helpers harder (need to export everything)

**Alternative 2: Single tests/ directory for all tests**
- **Rejected**: Breaks Go tooling expectations (`go test ./gitlab/...` wouldn't find tests)
- Harder to navigate - separation between unit and integration is spatial, not organizational

**Alternative 3: Require real GitLab instance for all tests**
- **Rejected**: See rationale above (slow, expensive, brittle)
- Only viable for staging/production validation, not CI

**Alternative 4: No integration tests, rely on contract tests**
- **Rejected**: Doesn't validate Kubernetes interactions (secrets, CRDs)
- AgenticSession e2e flow requires real K8s for Runner pod orchestration
- Contract tests don't catch integration issues (e.g., secret encoding, permission errors)

### Implementation Notes

**Test Execution Patterns**:
```bash
# Unit tests only (fast, run on every save)
cd components/backend
go test ./gitlab/... -v

# Contract tests (API interactions, < 1 second each)
go test ./tests/contract/... -v

# Integration tests (requires Kubernetes)
go test ./tests/integration/... -v -timeout=5m

# Performance tests (skipped by default)
go test ./gitlab/... -v -run TestLargeRepository

# Short mode (skip slow tests)
go test ./... -short
```

**Integration Test Template**:
```go
// tests/integration/gitlab_session_test.go
package integration

import (
    "context"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "k8s.io/client-go/kubernetes"
    // ... other imports
)

func TestAgenticSession_GitLab_EndToEnd(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test in short mode")
    }

    // Setup: Create test namespace
    ctx := context.Background()
    k8sClient := getTestK8sClient(t)
    testNS := createTestNamespace(t, k8sClient, "gitlab-e2e-test")
    defer cleanupTestNamespace(t, k8sClient, testNS)

    // Setup: Create ProjectSettings CR with GitLab repo
    projectSettings := createTestProjectSettings(t, k8sClient, testNS, &ProjectConfig{
        Repositories: []RepositoryConfig{
            {
                URL:      "https://gitlab.com/test-org/test-repo.git",
                Provider: "gitlab",
            },
        },
    })

    // Setup: Create secret with GitLab PAT
    createTestSecret(t, k8sClient, testNS, "runner-secret", map[string]string{
        "GITLAB_TOKEN": "glpat-test-token-12345",
    })

    // Test: Create AgenticSession
    session := createTestAgenticSession(t, k8sClient, testNS, &SessionConfig{
        Task:   "Add comment to README.md",
        Branch: "test-branch",
    })

    // Wait for session completion (timeout 2 minutes)
    err := waitForSessionCompletion(ctx, k8sClient, session, 2*time.Minute)
    assert.NoError(t, err)

    // Verify: Session status indicates success
    finalSession := getSession(t, k8sClient, testNS, session.Name)
    assert.Equal(t, "Completed", finalSession.Status.Phase)
    assert.Contains(t, finalSession.Status.Message, "Changes pushed successfully")
}
```

**Performance Test Configuration**:
- **Large repo test**: 10,000 files, verify no memory leaks or timeouts
- **Many branches test**: 500 branches, verify pagination works correctly
- **Concurrent sessions**: 10 simultaneous AgenticSessions with GitLab repos
- **Rate limiting**: Simulate GitLab rate limit (429 errors), verify retry logic

**Test Data Lifecycle**:
- **Setup**: Create Kubernetes resources (namespace, secrets, CRDs)
- **Execution**: Perform operations (API calls, Git commands, AgenticSession)
- **Verification**: Assert expected state (status, logs, Git history)
- **Cleanup**: Delete resources via `defer` or `t.Cleanup()`
- **Idempotency**: Tests can run multiple times without conflicts (unique namespaces)

---

## Decision 4: CI/CD Integration

### Decision

**GitHub Actions Workflow**: New workflow `.github/workflows/gitlab-tests.yml`

**Test Execution Tiers**:

1. **On every commit** (developer push):
   - Linting: `golangci-lint` on GitLab package
   - Unit tests: `go test ./gitlab/... -short`
   - Execution time: ~30 seconds

2. **On pull request** (PR opened/updated):
   - All unit tests: `go test ./gitlab/... -cover`
   - Contract tests: `go test ./tests/contract/...`
   - Coverage check: Fail if < 70%
   - Execution time: ~2 minutes

3. **Before merge** (required check):
   - Integration tests: `go test ./tests/integration/...`
   - Requires kind cluster setup
   - Execution time: ~5 minutes

4. **On merge to main** (post-merge validation):
   - Full test suite including performance tests
   - E2E regression suite for GitHub (verify zero regression)
   - Execution time: ~10 minutes

**Regression Testing Strategy**:
- **GitHub functionality**: Run existing GitHub integration tests (when they exist) on every PR
- **Backward compatibility**: Verify GitHub-only projects still work after GitLab code changes
- **Mixed provider tests**: Ensure GitHub + GitLab projects work simultaneously
- **Test organization**: GitHub tests in `tests/integration/github_regression_test.go`

**Pre-Merge Quality Gates**:
```yaml
# .github/workflows/gitlab-tests.yml
required_checks:
  - golangci-lint (gitlab package)
  - unit-tests (gitlab package, >= 70% coverage)
  - contract-tests (GitLab API interactions)
  - integration-tests (AgenticSession e2e)
  - github-regression-tests (zero failures)
```

**Merge Blocking Conditions**:
- Any test failure
- Coverage below 70% for new GitLab code
- Linting errors in GitLab package
- Integration test timeout (> 10 minutes indicates performance regression)
- GitHub regression test failure (breaks SC-004: zero regression requirement)

### Rationale

**Why tiered test execution?**
- **Developer experience**: Fast feedback on every commit (<30s unit tests)
- **PR efficiency**: Catch most issues before expensive integration tests
- **Resource optimization**: Integration tests (requiring K8s) only when PR is serious
- **Cost control**: Kind cluster setup costs ~$0.02/run, only run when necessary

**Why new workflow vs extending existing?**
- **Isolation**: GitLab testing is feature-specific, shouldn't impact unrelated code
- **Conditional execution**: Only run when GitLab code changes (path filtering)
- **Parallel execution**: Can run alongside existing workflows (go-lint.yml, frontend-lint.yml)
- **Maintainability**: Clear ownership, easier to modify without breaking other workflows

**Why GitHub regression tests on every PR?**
- **SC-004 requirement**: "100% of GitHub integration tests must still pass"
- **Early detection**: Catch regressions before merge, not after
- **Confidence**: Developers can refactor shared code (git/ package) safely
- **Cost**: GitHub tests are fast (< 1 minute) since they already exist

**Why integration tests before merge, not after?**
- **Quality gate**: Don't allow broken code into main branch
- **Rollback prevention**: Avoid "fix forward" commits that clutter git history
- **Team velocity**: Unbroken main branch means other developers aren't blocked
- **Compliance**: Required for production deployments in regulated environments

### Alternatives Considered

**Alternative 1: Run all tests on every commit**
- **Rejected**: Wastes CI resources and developer time
- Integration tests take 5+ minutes, blocks rapid iteration
- Developer workflow: push 5-10 commits while coding, don't need full suite each time

**Alternative 2: Only run integration tests on merge to main**
- **Rejected**: Too late to catch issues, broken main branch
- Violates team practice (see existing go-lint.yml runs on PR)
- Regression fixing requires emergency patches or reverts

**Alternative 3: Manual QA instead of automated tests**
- **Rejected**: Doesn't scale, human error prone
- Spec requires measurable success criteria (SC-003: 95% success rate)
- Can't verify SC-008 (10k files) or SC-006 (90% error messages) manually

**Alternative 4: Nightly test runs instead of PR-time tests**
- **Rejected**: Slow feedback loop (issues discovered 12-24 hours later)
- Developer has moved on to different task, context switching cost
- Doesn't meet "shift left" testing philosophy

### Implementation Notes

**GitHub Actions Workflow Example**:
```yaml
# .github/workflows/gitlab-tests.yml
name: GitLab Integration Tests

on:
  push:
    branches: [main, 'feature/gitlab-*']
    paths:
      - 'components/backend/gitlab/**'
      - 'components/backend/git/**'
      - 'components/backend/handlers/repo.go'
      - 'components/backend/tests/**'
  pull_request:
    branches: [main]

jobs:
  lint-and-unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version-file: 'components/backend/go.mod'
          cache-dependency-path: 'components/backend/go.sum'

      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v8
        with:
          working-directory: components/backend
          args: --timeout=5m ./gitlab/...

      - name: Run unit tests with coverage
        run: |
          cd components/backend
          go test ./gitlab/... -coverprofile=coverage.out -covermode=atomic

      - name: Check coverage threshold
        run: |
          cd components/backend
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
          echo "Coverage: $COVERAGE%"
          if (( $(echo "$COVERAGE < 70" | bc -l) )); then
            echo "Error: Coverage $COVERAGE% is below 70% threshold"
            exit 1
          fi

      - name: Upload coverage to Codecov (optional)
        uses: codecov/codecov-action@v5
        with:
          files: components/backend/coverage.out
          flags: gitlab-package

  contract-tests:
    runs-on: ubuntu-latest
    needs: lint-and-unit-tests
    steps:
      - uses: actions/checkout@v5

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version-file: 'components/backend/go.mod'

      - name: Run contract tests
        run: |
          cd components/backend
          go test ./tests/contract/... -v -timeout=2m

  integration-tests:
    runs-on: ubuntu-latest
    needs: contract-tests
    steps:
      - uses: actions/checkout@v5

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version-file: 'components/backend/go.mod'

      - name: Set up Kind cluster
        uses: helm/kind-action@v1
        with:
          cluster_name: vteam-test

      - name: Install test CRDs
        run: |
          kubectl apply -f components/manifests/crds/

      - name: Run integration tests
        run: |
          cd components/backend
          TEST_NAMESPACE=vteam-test-gitlab \
          CLEANUP_RESOURCES=true \
          go test ./tests/integration/... -v -timeout=10m

      - name: Collect logs on failure
        if: failure()
        run: |
          kubectl get pods -n vteam-test-gitlab
          kubectl logs -n vteam-test-gitlab --all-containers --tail=100

  github-regression-tests:
    runs-on: ubuntu-latest
    needs: lint-and-unit-tests
    steps:
      - uses: actions/checkout@v5

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version-file: 'components/backend/go.mod'

      - name: Set up Kind cluster
        uses: helm/kind-action@v1
        with:
          cluster_name: vteam-test

      - name: Run GitHub integration tests
        run: |
          cd components/backend
          # Once GitHub tests exist, run them here
          # go test ./tests/integration/github_*.go -v -timeout=5m
          echo "GitHub regression tests will run once baseline tests exist"
```

**Branch Protection Rules** (configure in GitHub):
```json
{
  "required_status_checks": {
    "strict": true,
    "checks": [
      "lint-and-unit-tests",
      "contract-tests",
      "integration-tests",
      "github-regression-tests"
    ]
  },
  "required_pull_request_reviews": {
    "required_approving_review_count": 1
  }
}
```

**Coverage Reporting** (optional, future enhancement):
- Integrate Codecov or Coveralls for coverage tracking over time
- Set up coverage diff comments on PRs (show coverage change)
- Track coverage trends per package (ensure GitLab package doesn't degrade)

**Test Failure Notifications**:
- GitHub PR status checks (built-in)
- Slack notifications on main branch test failures (optional)
- CODEOWNERS file ensures GitLab team is auto-assigned to PRs touching gitlab/ package

**Performance Regression Detection**:
```bash
# Optional: Add benchmark tests for performance-critical paths
go test ./gitlab/... -bench=. -benchmem -run=^$ > bench_new.txt

# Compare with baseline (store in git or CI cache)
benchstat bench_baseline.txt bench_new.txt

# Fail if performance regresses > 20%
```

---

## Summary of Key Decisions

| Decision Area | Choice | Key Justification |
|--------------|--------|-------------------|
| **Coverage Target** | 70% minimum, 90% critical paths | Balances quality with velocity, focus on business value |
| **Testing Framework** | Go native + testify + httptest | Zero learning curve, already in dependencies, stdlib support |
| **Test Structure** | Parallel files + separate integration/ | Go convention, clear boundaries, selective execution |
| **CI Strategy** | Tiered execution (commit → PR → merge) | Fast feedback, resource optimization, quality gates |
| **API Mocking** | httptest.Server with JSON fixtures | No external dependencies, full control, realistic responses |
| **Self-Hosted Testing** | Unit/contract tests + manual QA | Adequate coverage without complex CI infrastructure |
| **Regression Testing** | Run GitHub tests on every PR | Enforce SC-004 (zero regression) requirement |

---

## Implementation Checklist

### Phase 0: Test Infrastructure Setup
- [ ] Create `components/backend/testdata/gitlab_api_responses/` directory
- [ ] Add sample JSON fixtures for common GitLab API responses
- [ ] Create `.github/workflows/gitlab-tests.yml` workflow
- [ ] Configure branch protection rules for GitLab feature branch
- [ ] Update backend Makefile with GitLab-specific test targets

### Phase 1: Unit Test Development (Parallel with Implementation)
- [ ] Write tests for URL parsing (12 test cases)
- [ ] Write tests for token validation (8 test cases)
- [ ] Write tests for API client (15+ test cases)
- [ ] Write tests for error message formatting (15+ scenarios)
- [ ] Verify 70% coverage threshold met

### Phase 2: Contract Test Development
- [ ] Define GitLab API v4 contracts for endpoints used
- [ ] Implement contract tests for branches API
- [ ] Implement contract tests for tree API
- [ ] Implement contract tests for file content API
- [ ] Test self-hosted URL patterns and API v4 compatibility

### Phase 3: Integration Test Development
- [ ] Set up kind cluster in CI
- [ ] Write e2e test for AgenticSession with GitLab repo
- [ ] Write test for repository browsing flow
- [ ] Write test for mixed GitHub + GitLab project
- [ ] Write performance test for large repository (10k files)

### Phase 4: Regression Testing
- [ ] Create baseline GitHub integration tests (if not exist)
- [ ] Verify GitHub tests pass after GitLab changes
- [ ] Add test for backward compatibility of existing projects
- [ ] Document manual QA checklist for self-hosted GitLab

---

## Open Questions for Phase 1

**Question 1**: Should we add mutation testing to verify test quality?
- **Context**: Mutation testing changes code and checks if tests catch the change
- **Recommendation**: No for MVP, revisit after 6 months if coverage metrics plateau
- **Rationale**: High setup cost, diminishing returns for greenfield code

**Question 2**: Should we measure and enforce cyclomatic complexity limits?
- **Context**: Prevent overly complex functions that are hard to test
- **Recommendation**: Yes, add to golangci-lint config (limit: 15)
- **Rationale**: Already using golangci-lint, simple addition to `.golangci.yml`

**Question 3**: Should we test against multiple GitLab versions (12.x, 13.x, 14.x)?
- **Context**: Self-hosted instances may run older GitLab versions
- **Recommendation**: Not in CI, document minimum supported version (GitLab 12.0+)
- **Rationale**: API v4 stable since GitLab 12.0, breaking changes unlikely

**Question 4**: Should we implement load testing for GitLab API rate limits?
- **Context**: GitLab.com has 300 req/min limit for authenticated users
- **Recommendation**: Add to manual QA checklist, not automated CI
- **Rationale**: Rate limit testing requires sustained load (5+ minutes), expensive in CI

---

## References

**Go Testing Best Practices**:
- [Effective Go - Testing](https://go.dev/doc/effective_go#testing)
- [Go Testing Package Documentation](https://pkg.go.dev/testing)
- [Testify Documentation](https://github.com/stretchr/testify)

**Similar Project Testing Strategies**:
- [Kubernetes client-go testing patterns](https://github.com/kubernetes/client-go/tree/master/testing)
- [Docker CLI testing approach](https://github.com/docker/cli/tree/master/cli/command)
- [Terraform AWS provider testing](https://github.com/hashicorp/terraform-provider-aws/tree/main/internal/service)

**vTeam Existing Infrastructure**:
- `/workspace/sessions/agentic-session-1762276850/workspace/vTeam/components/backend/Makefile` (test targets)
- `/workspace/sessions/agentic-session-1762276850/workspace/vTeam/.github/workflows/go-lint.yml` (CI pattern)
- `/workspace/sessions/agentic-session-1762276850/workspace/vTeam/components/backend/.golangci.yml` (linting config)

**GitLab API Documentation**:
- [GitLab API v4 Overview](https://docs.gitlab.com/ee/api/api_resources.html)
- [GitLab API Rate Limits](https://docs.gitlab.com/ee/api/index.html#rate-limits)
- [GitLab Projects API](https://docs.gitlab.com/ee/api/projects.html)

---

**Next Steps**: Proceed to Phase 1 (data-model.md) to define GitLab-specific data structures and API contracts.
