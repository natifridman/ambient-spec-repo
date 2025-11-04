# Testing Strategy Summary: GitLab Integration

**Document Type**: Quick Reference
**Audience**: Engineering team, Product Manager
**Date**: 2025-11-04

## Overview

This document provides a concise summary of the testing strategy for GitLab integration, showing how testing decisions map to spec requirements and success criteria.

---

## Requirements Coverage Matrix

| Success Criteria | Testing Approach | Test Type | Location |
|-----------------|------------------|-----------|----------|
| **SC-003**: 95%+ success rate for Git operations | Unit tests for Git operations, integration tests for e2e flows | Unit + Integration | `gitlab/*_test.go`, `tests/integration/gitlab_session_test.go` |
| **SC-004**: 100% GitHub tests pass (zero regression) | Run GitHub integration tests on every PR | Regression | `.github/workflows/gitlab-tests.yml` (github-regression-tests job) |
| **SC-006**: 90% user-friendly error messages | Contract tests with 15+ GitLab API error scenarios | Contract | `tests/contract/gitlab_api_test.go` |
| **SC-008**: Handle 10k+ files without timeouts | Performance test with large repository simulation | Integration | `tests/integration/gitlab_performance_test.go` |

---

## Test Pyramid

```
                 /\
                /  \
               /    \
              / E2E  \    <-- 5 integration tests (~5 min)
             /--------\
            /          \
           /  Contract  \ <-- 10 contract tests (~1 min)
          /--------------\
         /                \
        /   Unit Tests     \ <-- 50+ unit tests (~30 sec)
       /____________________\
```

**Distribution**:
- **Unit tests**: 80% of tests, 20% of execution time
- **Contract tests**: 15% of tests, 30% of execution time
- **Integration tests**: 5% of tests, 50% of execution time

---

## Quick Start: Running Tests

### Developer Workflow

```bash
# 1. Run tests while coding (fast feedback)
cd components/backend
go test ./gitlab/... -v

# 2. Check coverage before committing
go test ./gitlab/... -cover

# 3. Run all tests before pushing
make test  # unit + contract tests

# 4. Run integration tests before creating PR (requires Kubernetes)
make test-integration
```

### CI/CD Workflow

```
Developer Push
    ↓
Lint + Unit Tests (30s) ← Fast feedback
    ↓
Pull Request Created
    ↓
Contract Tests (1m) ← API validation
    ↓
Integration Tests (5m) ← E2E validation
    ↓
GitHub Regression Tests (1m) ← Zero regression check
    ↓
Merge to Main (all checks pass)
```

---

## Test Examples by Category

### Unit Test Example
**Purpose**: Verify URL parsing handles all formats
```go
func TestParseGitLabURL_SelfHostedWithPort(t *testing.T) {
    host, owner, repo, err := ParseGitLabURL("https://gitlab.company.com:8443/team/project.git")

    assert.NoError(t, err)
    assert.Equal(t, "gitlab.company.com:8443", host)
    assert.Equal(t, "team", owner)
    assert.Equal(t, "project", repo)
}
```

### Contract Test Example
**Purpose**: Verify error messages are user-friendly (SC-006)
```go
func TestGitLabClient_InvalidToken_UserFriendlyError(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusUnauthorized)
        w.Write([]byte(`{"message":"401 Unauthorized"}`))
    }))
    defer server.Close()

    client := NewClient(server.URL, "invalid-token")
    _, err := client.GetBranches(context.Background(), "owner", "repo")

    assert.Error(t, err)
    assert.Contains(t, err.Error(), "Invalid GitLab token")
    assert.Contains(t, err.Error(), "verify your token has 'read_api' scope")
}
```

### Integration Test Example
**Purpose**: Verify e2e AgenticSession flow (SC-003)
```go
func TestAgenticSession_GitLab_CloneCommitPush(t *testing.T) {
    // Setup: Create test namespace with GitLab project
    testNS := createTestNamespace(t, k8sClient, "gitlab-e2e")
    defer cleanupTestNamespace(t, k8sClient, testNS)

    // Create ProjectSettings with GitLab repo
    createTestProjectSettings(t, k8sClient, testNS, &ProjectConfig{
        Repositories: []RepositoryConfig{
            {URL: "https://gitlab.com/test/repo.git", Provider: "gitlab"},
        },
    })

    // Create AgenticSession
    session := createTestAgenticSession(t, k8sClient, testNS, &SessionConfig{
        Task: "Add comment to README.md",
    })

    // Verify: Session completes successfully
    err := waitForSessionCompletion(context.Background(), k8sClient, session, 2*time.Minute)
    assert.NoError(t, err)
}
```

---

## Coverage Requirements

| Package | Minimum Coverage | Critical Functions Coverage |
|---------|-----------------|----------------------------|
| `gitlab/parser.go` | 70% | 90% (URL parsing functions) |
| `gitlab/client.go` | 70% | 90% (API error handling) |
| `gitlab/token.go` | 70% | 90% (Token validation) |
| `handlers/repo.go` | 60% | N/A (existing code) |
| `git/operations.go` | 60% | N/A (existing code) |

**How to verify**:
```bash
cd components/backend
go test ./gitlab/... -coverprofile=coverage.out
go tool cover -func=coverage.out | grep -E "gitlab/(parser|client|token)"
```

---

## Test Data Management

### JSON Fixtures Location
```
components/backend/testdata/gitlab_api_responses/
├── branches_response.json              # GET /projects/:id/repository/branches
├── tree_response.json                  # GET /projects/:id/repository/tree
├── file_response.json                  # GET /projects/:id/repository/files/:path/raw
├── error_401_invalid_token.json        # 401 Unauthorized
├── error_403_insufficient_scopes.json  # 403 Forbidden (scope issue)
├── error_404_not_found.json            # 404 Not Found
└── error_429_rate_limit.json           # 429 Too Many Requests
```

### How to Add New Fixtures
1. Make real API call to GitLab (using curl or Postman)
2. Save JSON response to `testdata/gitlab_api_responses/`
3. Reference in test: `os.ReadFile("testdata/gitlab_api_responses/new_fixture.json")`
4. Commit fixture to git (not sensitive data)

---

## Quality Gates (PR Merge Blockers)

| Check | Threshold | Enforcement |
|-------|-----------|-------------|
| Linting | Zero errors | golangci-lint on gitlab/ package |
| Unit test coverage | >= 70% | CI fails if below threshold |
| Unit test pass rate | 100% | All tests must pass |
| Contract test pass rate | 100% | All API contract tests pass |
| Integration test pass rate | 100% | All e2e tests pass |
| GitHub regression tests | 100% | Zero GitHub functionality breakage |
| Integration test timeout | < 10 minutes | Fails if tests hang |

**GitHub Actions Status Checks** (required):
- `lint-and-unit-tests`
- `contract-tests`
- `integration-tests`
- `github-regression-tests`

---

## Testing Self-Hosted GitLab

### Automated Testing (CI)
- **Unit tests**: Mock self-hosted URLs and API responses
- **Contract tests**: Test API v4 endpoint construction
- **Coverage**: 95% of self-hosted scenarios covered via mocks

### Manual Testing (Pre-Release)
Checklist for QA team:
- [ ] Self-hosted GitLab with standard HTTPS (port 443)
- [ ] Self-hosted GitLab with non-standard port (e.g., :8443)
- [ ] Self-hosted GitLab with custom path (e.g., /git)
- [ ] GitLab CE instance (Community Edition)
- [ ] GitLab EE instance (Enterprise Edition)
- [ ] Older GitLab versions (12.0, 13.0, 14.0)

**Why not automated?**
- Requires infrastructure (GitLab Docker container)
- Slow (2-3 minutes to start GitLab)
- Expensive (CI resource costs)
- Mocks provide equivalent functional coverage

---

## Debugging Failed Tests

### Unit Test Failure
```bash
# Run specific test with verbose output
go test ./gitlab/... -v -run TestParseGitLabURL

# Run with race detector
go test ./gitlab/... -race

# Show coverage for specific file
go test ./gitlab/... -coverprofile=coverage.out
go tool cover -html=coverage.out
```

### Integration Test Failure
```bash
# Check Kubernetes resources
kubectl get pods -n vteam-test-gitlab
kubectl logs -n vteam-test-gitlab <pod-name>

# Debug in CI (GitHub Actions)
# Logs automatically collected on failure (see workflow)

# Run locally with kind
make k8s-setup
make test-integration
make k8s-teardown
```

### Contract Test Failure
```bash
# Run with HTTP request/response logging
TEST_HTTP_DEBUG=1 go test ./tests/contract/... -v

# Check JSON fixtures
cat testdata/gitlab_api_responses/branches_response.json | jq .
```

---

## Performance Testing

### Large Repository Test
**Goal**: Verify SC-008 (10,000+ files without timeout)

```bash
# Run performance tests (skipped in short mode)
go test ./tests/integration/... -v -run TestLargeRepository

# Expected results:
# - 10,000 files retrieved in < 5 seconds
# - Memory usage < 100MB
# - No goroutine leaks
```

### Concurrent Sessions Test
**Goal**: Verify 10+ simultaneous AgenticSessions work

```bash
go test ./tests/integration/... -v -run TestConcurrentSessions

# Expected results:
# - 10 sessions complete successfully
# - No resource conflicts
# - Total time < 5 minutes (parallelism works)
```

---

## Maintenance

### Adding New GitLab API Endpoint
1. Add JSON fixture to `testdata/gitlab_api_responses/`
2. Write unit test for client function
3. Write contract test for API interaction
4. Update integration test if e2e flow affected
5. Verify coverage stays >= 70%

### Updating GitLab API Version
1. Update fixtures with new API response format
2. Run all tests to identify breaking changes
3. Update client code to handle both old and new formats
4. Add contract tests for version compatibility

### Adding New Error Scenario
1. Get real GitLab error response (via curl)
2. Add fixture: `error_<code>_<scenario>.json`
3. Write contract test verifying user-friendly message
4. Update error message mapping in client code
5. Verify SC-006 metric (90% user-friendly errors)

---

## Metrics and Monitoring

### Test Health Metrics (Track in CI)
- **Unit test execution time**: Should stay < 30 seconds
- **Integration test execution time**: Should stay < 5 minutes
- **Test flakiness rate**: Should be < 1% (tests pass consistently)
- **Coverage trend**: Track over time, should increase or stay stable

### Success Criteria Validation
- **SC-003** (95% success rate): Track in production via observability
- **SC-004** (zero regression): GitHub tests must pass on every PR
- **SC-006** (90% friendly errors): Manual review of error scenarios
- **SC-008** (10k files): Performance test validates before release

---

## FAQ

**Q: Do I need to write tests for my GitLab code?**
A: Yes, 70% coverage is required and enforced by CI.

**Q: Can I skip integration tests during development?**
A: Yes, use `-short` flag: `go test ./... -short`. Integration tests run in CI.

**Q: How do I test self-hosted GitLab scenarios?**
A: Use httptest mocks in unit tests. Manual QA for real self-hosted instances.

**Q: What if GitHub regression tests fail?**
A: PR is blocked. Fix the regression or update test expectations with justification.

**Q: How long do tests take?**
A: Unit (30s), Contract (1m), Integration (5m), Total (7m) - faster with caching.

**Q: Can I use a real GitLab instance in tests?**
A: Not in CI. Use httptest mocks. Real GitLab testing is for staging/QA.

---

## Resources

- **Full testing strategy**: [research.md](./research.md)
- **Feature specification**: [spec.md](./spec.md)
- **Implementation plan**: [plan.md](./plan.md)
- **Backend Makefile**: `/workspace/sessions/agentic-session-1762276850/workspace/vTeam/components/backend/Makefile`
- **CI workflow**: `.github/workflows/gitlab-tests.yml` (to be created)

---

**Author**: Stella (Staff Engineer)
**Last Updated**: 2025-11-04
**Status**: Ready for Phase 1 (Implementation)
