# GitLab Support Implementation Quickstart

**Feature**: GitLab Support for vTeam
**Branch**: `001-gitlab-support`
**Date**: 2025-11-04

This guide provides a step-by-step walkthrough for implementing GitLab support in vTeam, from setting up your development environment to deploying the feature.

---

## Prerequisites

Before starting implementation, ensure you have:

- [ ] Go 1.24+ installed
- [ ] Kubernetes cluster access (kind for local testing)
- [ ] kubectl configured for your cluster
- [ ] Git repository cloned: `/workspace/sessions/agentic-session-1762276850/workspace/vTeam`
- [ ] GitLab.com account (for testing)
- [ ] GitLab Personal Access Token with `api`, `read_repository`, `write_repository` scopes

---

## Phase 0: Setup & Planning (Complete ✅)

### Artifacts Generated
- [x] `spec.md` - Feature specification with user stories
- [x] `plan.md` - Implementation plan with technical context
- [x] `research.md` - Testing strategy and best practices
- [x] `data-model.md` - Data structures and entities
- [x] `contracts/` - OpenAPI specifications for APIs
- [x] `quickstart.md` - This implementation guide

### Key Decisions Made
1. **HTTP Client**: Direct `net/http` calls (mirrors GitHub implementation)
2. **Testing**: Go native + testify, 70% coverage minimum
3. **Architecture**: New `gitlab/` package parallel to `github/`
4. **Storage**: ConfigMap for metadata, Secret for PAT tokens

---

## Phase 1: Foundation (Week 1)

### 1.1 Create GitLab Package Structure

```bash
cd /workspace/sessions/agentic-session-1762276850/workspace/vTeam/components/backend

# Create new gitlab package
mkdir gitlab
touch gitlab/parser.go        # URL parsing and provider detection
touch gitlab/token.go          # PAT management and validation
touch gitlab/client.go         # GitLab API client
touch gitlab/errors.go         # Error handling with user-friendly messages

# Create test files
touch gitlab/parser_test.go
touch gitlab/token_test.go
touch gitlab/client_test.go

# Create test fixtures
mkdir -p testdata/gitlab_api_responses
```

### 1.2 Implement URL Parser (`gitlab/parser.go`)

**Goal**: Parse GitLab URLs and detect self-hosted instances

```go
package gitlab

import (
	"fmt"
	"net/url"
	"strings"
)

// ParsedGitLabRepo represents a parsed GitLab repository URL
type ParsedGitLabRepo struct {
	Host      string // "gitlab.com" or "gitlab.example.com"
	Owner     string // Repository owner/namespace
	Repo      string // Repository name
	APIURL    string // Constructed API base URL
	ProjectID string // URL-encoded project path (owner%2Frepo)
}

// ParseGitLabURL parses a GitLab repository URL into components
// Supports formats:
//   - https://gitlab.com/owner/repo.git
//   - https://gitlab.com/owner/repo
//   - git@gitlab.com:owner/repo.git
//   - https://gitlab.company.com:8443/group/subgroup/repo.git
func ParseGitLabURL(gitURL string) (*ParsedGitLabRepo, error) {
	// Remove .git suffix if present
	gitURL = strings.TrimSuffix(gitURL, ".git")

	// Handle SSH format: git@gitlab.com:owner/repo
	if strings.HasPrefix(gitURL, "git@") {
		gitURL = strings.Replace(gitURL, ":", "/", 1)
		gitURL = strings.Replace(gitURL, "git@", "https://", 1)
	}

	// Parse URL
	u, err := url.Parse(gitURL)
	if err != nil {
		return nil, fmt.Errorf("invalid GitLab URL: %w", err)
	}

	// Validate scheme
	if u.Scheme != "https" && u.Scheme != "http" {
		return nil, fmt.Errorf("unsupported URL scheme: %s (must be https)", u.Scheme)
	}

	// Extract host (including port if present)
	host := u.Host

	// Extract path components: /owner/repo or /group/subgroup/repo
	pathParts := strings.Split(strings.Trim(u.Path, "/"), "/")
	if len(pathParts) < 2 {
		return nil, fmt.Errorf("invalid GitLab URL path: %s (expected /owner/repo)", u.Path)
	}

	// Handle nested groups: /group/subgroup/repo -> owner=group/subgroup, repo=repo
	repo := pathParts[len(pathParts)-1]
	owner := strings.Join(pathParts[:len(pathParts)-1], "/")

	// Construct API base URL
	apiURL := fmt.Sprintf("%s://%s/api/v4", u.Scheme, host)

	// Construct project ID (URL-encoded path)
	projectID := url.PathEscape(owner + "/" + repo)

	return &ParsedGitLabRepo{
		Host:      host,
		Owner:     owner,
		Repo:      repo,
		APIURL:    apiURL,
		ProjectID: projectID,
	}, nil
}

// IsGitLabURL checks if a URL points to GitLab (vs GitHub or other)
func IsGitLabURL(gitURL string) bool {
	return strings.Contains(gitURL, "gitlab.com") ||
		strings.Contains(gitURL, "gitlab.")
}
```

**Test**: `gitlab/parser_test.go` (12 test cases as defined in research.md)

### 1.3 Implement Token Manager (`gitlab/token.go`)

**Goal**: Validate GitLab PATs and manage user connections

```go
package gitlab

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"time"
)

// TokenManager handles GitLab token validation and storage
type TokenManager struct {
	HTTPClient *http.Client
}

// GitLabUser represents the response from GitLab /user API
type GitLabUser struct {
	ID       int    `json:"id"`
	Username string `json:"username"`
	Name     string `json:"name"`
	Email    string `json:"email"`
}

// ValidateToken checks if a GitLab PAT is valid and returns user info
func (tm *TokenManager) ValidateToken(ctx context.Context, token, instanceURL string) (*GitLabUser, error) {
	// Construct API URL
	apiURL := fmt.Sprintf("%s/api/v4/user", strings.TrimSuffix(instanceURL, "/"))

	req, err := http.NewRequestWithContext(ctx, "GET", apiURL, nil)
	if err != nil {
		return nil, fmt.Errorf("failed to create request: %w", err)
	}

	// GitLab accepts Bearer token in Authorization header
	req.Header.Set("Authorization", "Bearer "+token)

	resp, err := tm.HTTPClient.Do(req)
	if err != nil {
		return nil, &GitLabAPIError{
			StatusCode:  0,
			Message:     "Failed to connect to GitLab API",
			Remediation: "Check your network connection and GitLab instance URL",
			RawError:    err.Error(),
		}
	}
	defer resp.Body.Close()

	// Handle HTTP errors
	if resp.StatusCode != http.StatusOK {
		body, _ := io.ReadAll(resp.Body)
		return nil, mapHTTPError(resp.StatusCode, string(body))
	}

	// Parse response
	var user GitLabUser
	if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
		return nil, fmt.Errorf("failed to parse GitLab user response: %w", err)
	}

	return &user, nil
}

// mapHTTPError converts GitLab API error codes to user-friendly messages
func mapHTTPError(statusCode int, body string) error {
	switch statusCode {
	case 401:
		return &GitLabAPIError{
			StatusCode:  401,
			Message:     "GitLab token is invalid or expired",
			Remediation: "Please reconnect your GitLab account with a valid Personal Access Token",
			RawError:    body,
		}
	case 403:
		return &GitLabAPIError{
			StatusCode:  403,
			Message:     "GitLab token lacks required permissions",
			Remediation: "Ensure your token has 'api' and 'read_repository' scopes and try again",
			RawError:    body,
		}
	case 429:
		return &GitLabAPIError{
			StatusCode:  429,
			Message:     "GitLab API rate limit exceeded",
			Remediation: "Please wait a few minutes before retrying",
			RawError:    body,
		}
	default:
		return &GitLabAPIError{
			StatusCode:  statusCode,
			Message:     fmt.Sprintf("GitLab API error (HTTP %d)", statusCode),
			Remediation: "Please try again or contact support if the issue persists",
			RawError:    body,
		}
	}
}
```

### 1.4 Implement Error Handling (`gitlab/errors.go`)

```go
package gitlab

import "fmt"

// GitLabAPIError represents a user-friendly error from GitLab API
type GitLabAPIError struct {
	StatusCode  int
	Message     string
	Remediation string
	RawError    string
	RequestID   string
	Metadata    map[string]interface{}
}

func (e *GitLabAPIError) Error() string {
	if e.Remediation != "" {
		return fmt.Sprintf("%s. %s", e.Message, e.Remediation)
	}
	return e.Message
}

// RedactToken replaces token values with [REDACTED] in strings
func RedactToken(s string) string {
	// Replace GitLab PAT pattern: glpat-xxxxxxxxxxxx
	re := regexp.MustCompile(`glpat-[a-zA-Z0-9_-]+`)
	return re.ReplaceAllString(s, "glpat-[REDACTED]")
}
```

---

## Phase 2: API Client (Week 2)

### 2.1 Implement GitLab API Client (`gitlab/client.go`)

**Goal**: Call GitLab API v4 endpoints for branches, tree, files

```go
package gitlab

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"time"
)

// Client is a GitLab API v4 client
type Client struct {
	BaseURL    string       // e.g., "https://gitlab.com/api/v4"
	Token      string       // Personal Access Token
	HTTPClient *http.Client
}

// NewClient creates a new GitLab API client
func NewClient(instanceURL, token string) *Client {
	return &Client{
		BaseURL:    fmt.Sprintf("%s/api/v4", strings.TrimSuffix(instanceURL, "/")),
		Token:      token,
		HTTPClient: &http.Client{Timeout: 15 * time.Second},
	}
}

// GitLabBranch represents a branch in GitLab
type GitLabBranch struct {
	Name      string    `json:"name"`
	Commit    Commit    `json:"commit"`
	Protected bool      `json:"protected"`
	Default   bool      `json:"default"`
}

type Commit struct {
	ID            string    `json:"id"`
	ShortID       string    `json:"short_id"`
	Title         string    `json:"title"`
	Message       string    `json:"message"`
	AuthorName    string    `json:"author_name"`
	AuthorEmail   string    `json:"author_email"`
	CommittedDate time.Time `json:"committed_date"`
}

// GetBranches retrieves all branches for a GitLab repository
func (c *Client) GetBranches(ctx context.Context, projectID string) ([]GitLabBranch, error) {
	endpoint := fmt.Sprintf("%s/projects/%s/repository/branches", c.BaseURL, projectID)

	req, err := http.NewRequestWithContext(ctx, "GET", endpoint, nil)
	if err != nil {
		return nil, fmt.Errorf("failed to create request: %w", err)
	}

	req.Header.Set("Authorization", "Bearer "+c.Token)

	resp, err := c.HTTPClient.Do(req)
	if err != nil {
		return nil, &GitLabAPIError{
			Message:     "Failed to connect to GitLab API",
			Remediation: "Check your network connection",
			RawError:    err.Error(),
		}
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		body, _ := io.ReadAll(resp.Body)
		return nil, mapHTTPError(resp.StatusCode, string(body))
	}

	var branches []GitLabBranch
	if err := json.NewDecoder(resp.Body).Decode(&branches); err != nil {
		return nil, fmt.Errorf("failed to parse branches response: %w", err)
	}

	return branches, nil
}

// Additional methods: GetTree, GetFile, etc. (implement similarly)
```

**Note**: Implement pagination logic as described in research-gitlab-api-patterns.md

---

## Phase 3: Handlers & Integration (Week 3)

### 3.1 Create GitLab Authentication Handlers

```bash
cd components/backend/handlers
touch gitlab_auth.go
```

**Implement**:
- `ConnectGitLabPAT(c *gin.Context)` - POST /auth/gitlab/connect
- `GetGitLabStatusGlobal(c *gin.Context)` - GET /auth/gitlab/status
- `DisconnectGitLabGlobal(c *gin.Context)` - POST /auth/gitlab/disconnect

### 3.2 Extend Repository Handlers

**Modify**: `handlers/repo.go`

Add provider detection logic:
```go
func GetRepositoryBranches(c *gin.Context) {
    repoURL := c.Query("repoUrl")

    // Detect provider
    if gitlab.IsGitLabURL(repoURL) {
        // Call GitLab API
        handleGitLabBranches(c, repoURL)
    } else {
        // Existing GitHub logic
        handleGitHubBranches(c, repoURL)
    }
}
```

### 3.3 Update main.go and routes.go

**Add GitLab routes**:
```go
// routes.go
api.POST("/auth/gitlab/connect", handlers.ConnectGitLabPAT)
api.GET("/auth/gitlab/status", handlers.GetGitLabStatusGlobal)
api.POST("/auth/gitlab/disconnect", handlers.DisconnectGitLabGlobal)
```

**Initialize GitLab TokenManager**:
```go
// main.go
gitlab.InitializeTokenManager()
handlers.GitlabTokenManager = gitlab.Manager
```

---

## Phase 4: Testing (Week 4)

### 4.1 Unit Tests

```bash
cd components/backend
go test ./gitlab/... -v -cover
```

**Coverage Check**:
```bash
go test ./gitlab/... -coverprofile=coverage.out
go tool cover -func=coverage.out | grep total
# Should be >= 70%
```

### 4.2 Integration Tests

**Setup Kind Cluster**:
```bash
cd components/backend
make k8s-setup  # Or manually create kind cluster
```

**Run Integration Tests**:
```bash
go test ./tests/integration/... -v -timeout=10m
```

### 4.3 Manual Testing Checklist

- [ ] Connect GitLab.com account with valid PAT
- [ ] Verify connection status in UI
- [ ] Browse GitLab repository branches
- [ ] Browse GitLab repository file tree
- [ ] Read file contents from GitLab repo
- [ ] Create AgenticSession with GitLab repo
- [ ] Verify AgenticSession can clone GitLab repo
- [ ] Verify AgenticSession can push to GitLab repo
- [ ] Test mixed GitHub + GitLab project
- [ ] Test self-hosted GitLab instance (if available)
- [ ] Disconnect GitLab account

---

## Common Development Commands

### Run Backend Locally
```bash
cd components/backend
export GITLAB_INSTANCE_URL=https://gitlab.com
go run main.go routes.go
```

### Lint Code
```bash
golangci-lint run ./gitlab/...
```

### Run Specific Test
```bash
go test ./gitlab -run TestParseGitLabURL -v
```

### Debug with Delve
```bash
dlv test ./gitlab -- -test.run TestParseGitLabURL
```

### Check Test Coverage
```bash
go test ./gitlab/... -coverprofile=coverage.out
go tool cover -html=coverage.out -o coverage.html
open coverage.html
```

---

## Debugging Tips

### GitLab API Errors

**Problem**: Getting 401 Unauthorized
**Solution**:
1. Verify token format starts with `glpat-`
2. Check token hasn't expired in GitLab settings
3. Verify token has minimum scopes: `api`, `read_repository`

**Problem**: Getting 404 Not Found
**Solution**:
1. Check repository URL is correct
2. Verify user has access to repository
3. For self-hosted, verify instance URL is correct

### Local Development

**Problem**: Can't connect to local Kubernetes cluster
**Solution**:
```bash
kubectl config current-context
kubectl get pods -n vteam-backend
```

**Problem**: Changes not reflected
**Solution**:
```bash
# Rebuild and restart backend
make build
kubectl delete pod -n vteam-backend -l app=backend
```

---

## Deployment Checklist

Before deploying to production:

- [ ] All unit tests passing (>= 70% coverage)
- [ ] All integration tests passing
- [ ] GitHub regression tests passing (zero failures)
- [ ] golangci-lint passing with zero errors
- [ ] Manual QA completed for all user stories
- [ ] Self-hosted GitLab testing completed (if supported)
- [ ] Documentation updated (API docs, user guides)
- [ ] CHANGELOG.md updated with new features
- [ ] Security review completed (token storage, error messages)

---

## Troubleshooting

### Issue: Tests failing with "connection refused"

**Cause**: Kubernetes cluster not running

**Fix**:
```bash
kind create cluster --name vteam-test
kubectl apply -f components/manifests/crds/
```

### Issue: "invalid token" error despite valid PAT

**Cause**: Token might be for wrong GitLab instance

**Fix**: Verify `instanceUrl` matches the GitLab instance where PAT was created

### Issue: Rate limit errors during development

**Cause**: Too many API calls to GitLab

**Fix**: Implement token bucket rate limiter (see research-gitlab-api-patterns.md)

---

## Next Steps After Implementation

1. **User Documentation**: Write end-user guide for GitLab setup
2. **Admin Documentation**: Document token management and troubleshooting
3. **Monitoring**: Add metrics for GitLab API calls and error rates
4. **Performance Optimization**: Implement caching for frequently accessed data
5. **OAuth Flow**: Add GitLab OAuth as alternative to PAT (future enhancement)

---

## Resources

**Internal Documentation**:
- [Feature Specification](./spec.md)
- [Implementation Plan](./plan.md)
- [Research Document](./research.md)
- [Data Model](./data-model.md)
- [API Contracts](./contracts/)

**External References**:
- [GitLab API v4 Documentation](https://docs.gitlab.com/ee/api/api_resources.html)
- [GitLab Personal Access Tokens Guide](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)
- [Go Testing Documentation](https://pkg.go.dev/testing)
- [vTeam Repository](https://github.com/org/vTeam)

**Support**:
- GitLab Feature Slack Channel: `#gitlab-support`
- vTeam Engineering Team: `#vteam-dev`
- On-call Engineer: See PagerDuty schedule
