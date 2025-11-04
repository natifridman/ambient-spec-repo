# Data Model: GitLab Support

**Feature**: GitLab Support for vTeam
**Branch**: `001-gitlab-support`
**Date**: 2025-11-04

This document defines the data structures and entities required for GitLab integration in vTeam.

---

## Core Entities

### 1. GitLabConnection

Represents a user's connection to GitLab (either GitLab.com or self-hosted instance).

**Storage**: Hybrid approach - metadata in ConfigMap, sensitive tokens in Secret

```go
// Stored in ConfigMap: gitlab-connections
type GitLabConnection struct {
    UserID       string    `json:"userId"`       // vTeam user identifier
    GitLabUserID string    `json:"gitlabUserId"` // GitLab user ID (from /user API)
    InstanceURL  string    `json:"instanceUrl"`  // e.g., "https://gitlab.com" or "https://gitlab.company.com"
    Username     string    `json:"username"`     // GitLab username
    UpdatedAt    time.Time `json:"updatedAt"`    // Last connection update
}

// Stored in Secret: gitlab-user-tokens
// Key: <userID>, Value: <PAT>
// Example: {"user-123": "glpat-xxxxxxxxxxxx"}
```

**Relationships**:
- One-to-one: User → GitLabConnection (each user has at most one GitLab connection)
- One-to-many: GitLabConnection → GitLab Repositories (user can access many repos)

**Validation Rules**:
- `UserID`: Required, non-empty, matches vTeam user ID
- `GitLabUserID`: Required, numeric string from GitLab API
- `InstanceURL`: Required, valid HTTPS URL, normalized (trailing slash removed)
- `Username`: Required, non-empty
- `UpdatedAt`: Auto-populated on creation/update

**State Transitions**:
1. **Not Connected** → **Connected**: User submits PAT via `/auth/gitlab/connect`
2. **Connected** → **Not Connected**: User disconnects via `/auth/gitlab/disconnect`
3. **Connected** → **Connected**: User updates PAT (same endpoint, updates existing)

---

### 2. GitLabRepository (extends GitRepository)

Represents a GitLab-hosted repository configured in a vTeam project.

```go
// Extends existing GitRepository type in types/common.go
type GitRepository struct {
    URL      string  `json:"url"`               // Repository URL
    Branch   *string `json:"branch,omitempty"`  // Optional branch override
    Provider string  `json:"provider,omitempty"` // NEW: "github" or "gitlab"
}

// Internal parsed representation (not persisted to CRD)
type ParsedGitLabRepo struct {
    Host     string  // "gitlab.com" or "gitlab.example.com"
    Owner    string  // Repository owner/namespace
    Repo     string  // Repository name
    APIURL   string  // Constructed API base URL (e.g., "https://gitlab.com/api/v4")
    ProjectID string // URL-encoded project path (owner%2Frepo) for API calls
}
```

**Relationships**:
- Many-to-one: GitLabRepository → GitLabConnection (repo access requires user connection)
- Many-to-one: GitLabRepository → ProjectSettings (multiple repos per project)

**Validation Rules**:
- `URL`: Required, must contain "gitlab" in domain, supports HTTPS/SSH formats
- `Provider`: Optional but recommended, auto-detected from URL if missing
- `Branch`: Optional, defaults to repository's default branch

**Derived Fields** (computed at runtime):
- `APIURL`: Constructed from `Host` (e.g., `https://gitlab.com/api/v4`)
- `ProjectID`: URL-encoded path for GitLab API (e.g., `namespace%2Fproject`)

---

### 3. GitLabPersonalAccessToken (PAT)

Represents authentication credentials for GitLab API and Git operations.

**Storage**: Kubernetes Secret only (never in ConfigMap or CRD)

```go
// Not a struct - stored as raw string in Secret
// Secret name: gitlab-user-tokens
// Secret key: <userID>
// Secret value: <PAT string>

// Metadata about token scopes (not stored, validated at runtime)
type GitLabTokenScopes struct {
    ReadAPI         bool  // Required for API browsing (minimum scope)
    ReadRepository  bool  // Required for clone operations
    WriteRepository bool  // Required for push operations
}
```

**Relationships**:
- One-to-one: PAT → GitLabConnection (token associated with user connection)

**Validation Rules**:
- Format: Must start with `glpat-` (GitLab PAT format) or `gitlab-ci-token` (CI token)
- Scopes: Validated via `/user` API call, must have minimum required scopes
- Expiration: GitLab API returns expiry date, stored in connection metadata

**Security Constraints**:
- Never logged in plaintext
- Redacted in error messages (`[REDACTED]` placeholder)
- Encrypted at rest (Kubernetes Secret encryption)
- Not exposed in API responses (only presence/absence indicated)

---

### 4. GitLabAPIError

Structured error type for GitLab API failures with user-friendly messages.

```go
type GitLabAPIError struct {
    StatusCode  int                 // HTTP status code
    Message     string              // User-friendly error message
    Remediation string              // Actionable guidance for user
    RawError    string              // Original error from GitLab API
    RequestID   string              // GitLab request ID for debugging
    Metadata    map[string]interface{} // Additional context
}

func (e *GitLabAPIError) Error() string {
    return fmt.Sprintf("%s. %s", e.Message, e.Remediation)
}
```

**Error Message Mapping** (SC-006: 90% user-friendly errors):
- `401 Unauthorized`: "GitLab token is invalid or expired. Please reconnect your GitLab account with a valid Personal Access Token."
- `403 Forbidden`: "GitLab token lacks required permissions. Ensure your token has {scope} scope and try again."
- `404 Not Found`: "GitLab repository '{owner}/{repo}' not found. Verify the repository URL and your access permissions."
- `429 Rate Limit`: "GitLab API rate limit exceeded. Please wait {reset_time} before retrying."
- `500 Internal Server Error`: "GitLab API is experiencing issues. Please try again in a few minutes or contact support if the issue persists."

**Relationships**:
- Many-to-one: GitLabAPIError → GitLab API Operation (each operation can produce specific errors)

---

### 5. GitLabBranch

Represents a Git branch in a GitLab repository.

```go
type GitLabBranch struct {
    Name      string    `json:"name"`
    Commit    Commit    `json:"commit"`
    Protected bool      `json:"protected"`
    Default   bool      `json:"default"`
}

type Commit struct {
    ID            string    `json:"id"`          // SHA
    ShortID       string    `json:"short_id"`
    Title         string    `json:"title"`
    Message       string    `json:"message"`
    AuthorName    string    `json:"author_name"`
    AuthorEmail   string    `json:"author_email"`
    CommittedDate time.Time `json:"committed_date"`
}
```

**Source**: GitLab API `/projects/:id/repository/branches`

**Relationships**:
- Many-to-one: GitLabBranch → GitLabRepository

---

### 6. GitLabTreeEntry

Represents a file or directory entry in a GitLab repository tree.

```go
type GitLabTreeEntry struct {
    ID   string `json:"id"`   // Object SHA
    Name string `json:"name"` // File/directory name
    Type string `json:"type"` // "blob" or "tree"
    Path string `json:"path"` // Full path from repository root
    Mode string `json:"mode"` // File mode (e.g., "100644")
}
```

**Source**: GitLab API `/projects/:id/repository/tree`

**Relationships**:
- Many-to-one: GitLabTreeEntry → GitLabRepository
- Hierarchical: TreeEntry → TreeEntry (directories contain files)

---

### 7. ProviderType (enum)

Distinguishes between Git hosting providers.

```go
type ProviderType string

const (
    ProviderGitHub ProviderType = "github"
    ProviderGitLab ProviderType = "gitlab"
)

// Provider detection logic
func DetectProvider(repoURL string) ProviderType {
    if strings.Contains(repoURL, "github.com") || strings.Contains(repoURL, "github.") {
        return ProviderGitHub
    }
    if strings.Contains(repoURL, "gitlab.com") || strings.Contains(repoURL, "gitlab.") {
        return ProviderGitLab
    }
    return "" // Unknown provider
}
```

**Usage**: Route API calls and authentication to appropriate provider handler.

---

## Data Flow Diagrams

### User Connects GitLab Account

```
[User]
  → POST /auth/gitlab/connect {personalAccessToken, instanceUrl}
  → [Backend: gitlab/token.go]
    → Validate PAT via GitLab /user API
    → Extract GitLabUserID, Username
    → Store connection metadata in ConfigMap
    → Store PAT in Secret (key: userID)
  → Response: {userId, gitlabUserId, instanceUrl, connected: true}
```

### AgenticSession Clones GitLab Repository

```
[AgenticSession Runner Pod]
  → Request: Clone https://gitlab.com/owner/repo.git
  → [Backend: git/operations.go]
    → Detect provider from URL (gitlab)
    → GetGitLabToken(projectName, userID)
      → Read Secret gitlab-user-tokens[userID]
    → InjectGitToken(url, token, "gitlab")
      → Returns: https://oauth2:TOKEN@gitlab.com/owner/repo.git
  → Execute: git clone <injected-url>
  → Store cloned repo in ephemeral volume
```

### User Browses GitLab Repository in UI

```
[Frontend]
  → GET /projects/:project/repo/branches
  → [Backend: handlers/repo.go]
    → Extract repository URL from ProjectSettings CRD
    → Detect provider (gitlab)
    → GetGitLabToken(projectName, userID)
    → [gitlab/client.go]
      → Parse repository URL (owner, repo)
      → Construct API URL: /api/v4/projects/owner%2Frepo/repository/branches
      → Call GitLab API with Authorization: Bearer <PAT>
      → Parse JSON response → []GitLabBranch
    → Return branches to frontend
  → [Frontend] Display branches in dropdown
```

---

## Database Schema (Kubernetes Resources)

### ConfigMap: gitlab-connections

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitlab-connections
  namespace: vteam-backend
data:
  user-123: |
    {
      "userId": "user-123",
      "gitlabUserId": "456",
      "instanceUrl": "https://gitlab.com",
      "username": "johndoe",
      "updatedAt": "2025-11-04T10:30:00Z"
    }
  user-456: |
    {
      "userId": "user-456",
      "gitlabUserId": "789",
      "instanceUrl": "https://gitlab.example.com",
      "username": "janesmith",
      "updatedAt": "2025-11-04T11:00:00Z"
    }
```

### Secret: gitlab-user-tokens

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-user-tokens
  namespace: vteam-backend
type: Opaque
data:
  user-123: Z2xwYXQteHh4eHh4eHh4eA==  # base64("glpat-xxxxxxxxxxxx")
  user-456: Z2xwYXQteXl5eXl5eXl5eQ==  # base64("glpat-yyyyyyyyyyyy")
```

### ProjectSettings CRD (Extended)

```yaml
apiVersion: vteam.ambient-code/v1alpha1
kind: ProjectSettings
metadata:
  name: my-project
  namespace: project-ns
spec:
  runnerSecretsName: ambient-runner-secrets
  repositories:
    - url: https://github.com/org/frontend.git
      branch: main
      provider: github  # Optional, auto-detected
    - url: https://gitlab.com/org/backend.git
      branch: develop
      provider: gitlab  # Optional, auto-detected
```

---

## API Request/Response Examples

### Connect GitLab Account

**Request:**
```json
POST /auth/gitlab/connect
Content-Type: application/json

{
  "personalAccessToken": "glpat-AbCdEfGhIjKlMnOp",
  "instanceUrl": "https://gitlab.com"
}
```

**Response (Success):**
```json
{
  "userId": "user-123",
  "gitlabUserId": "456",
  "username": "johndoe",
  "instanceUrl": "https://gitlab.com",
  "connected": true,
  "message": "GitLab account connected successfully"
}
```

**Response (Error - Invalid Token):**
```json
{
  "error": "GitLab token is invalid or expired. Please generate a new Personal Access Token with required scopes (api, read_repository, write_repository) and try again.",
  "statusCode": 401
}
```

### List GitLab Branches

**Request:**
```http
GET /projects/my-project/repo/branches?repoUrl=https://gitlab.com/org/backend.git
Authorization: Bearer <user-session-token>
```

**Response:**
```json
{
  "branches": [
    {
      "name": "main",
      "commit": {
        "id": "a1b2c3d4e5f6",
        "shortId": "a1b2c3d",
        "title": "feat: add GitLab support",
        "authorName": "John Doe",
        "committedDate": "2025-11-04T10:15:00Z"
      },
      "protected": true,
      "default": true
    },
    {
      "name": "develop",
      "commit": {
        "id": "f6e5d4c3b2a1",
        "shortId": "f6e5d4c",
        "title": "chore: update dependencies",
        "authorName": "Jane Smith",
        "committedDate": "2025-11-03T14:20:00Z"
      },
      "protected": false,
      "default": false
    }
  ]
}
```

---

## Validation & Constraints

### URL Parsing Constraints

**Valid GitLab URL Formats:**
- `https://gitlab.com/owner/repo.git` ✅
- `https://gitlab.com/owner/repo` ✅
- `git@gitlab.com:owner/repo.git` ✅
- `https://gitlab.example.com/group/subgroup/repo.git` ✅
- `https://gitlab.example.com:8443/owner/repo.git` ✅

**Invalid Formats:**
- `http://gitlab.com/owner/repo` ❌ (HTTP not secure)
- `ftp://gitlab.com/owner/repo` ❌ (Wrong protocol)
- `gitlab.com/owner/repo` ❌ (No protocol)

### Token Validation

**Required Scopes:**
- Minimum: `api` or `read_api` (for API access)
- For clone: `read_repository`
- For push: `write_repository`

**Validation Logic:**
```go
func ValidateGitLabToken(token string, instanceURL string) (*TokenValidation, error) {
    // Call GitLab /user API
    resp, err := http.Get(instanceURL + "/api/v4/user",
        headers: {"Authorization": "Bearer " + token})

    if resp.StatusCode == 401 {
        return nil, &GitLabAPIError{
            StatusCode: 401,
            Message: "GitLab token is invalid or expired",
            Remediation: "Please reconnect with a valid token",
        }
    }

    // Parse scopes from response (if GitLab provides them)
    // Validate minimum scopes present

    return &TokenValidation{
        Valid: true,
        UserID: resp.Body.ID,
        Username: resp.Body.Username,
        Scopes: extractedScopes,
    }, nil
}
```

---

## Migration & Backward Compatibility

### Phase 1: Add GitLab Support (No Breaking Changes)

**Existing GitRepository type:**
```go
type GitRepository struct {
    URL    string  `json:"url"`
    Branch *string `json:"branch,omitempty"`
}
```

**Extended (backward compatible):**
```go
type GitRepository struct {
    URL      string  `json:"url"`
    Branch   *string `json:"branch,omitempty"`
    Provider string  `json:"provider,omitempty"` // NEW: Optional field
}
```

**Compatibility:**
- Existing projects without `provider` field: Auto-detect from URL
- New projects with `provider` field: Use explicit provider
- No CRD version bump required (optional field addition)

### Phase 2: Multi-Provider Projects

**Support projects with both GitHub and GitLab repositories:**
- ProjectSettings can have mixed `repositories` array
- Each repository's provider detected independently
- Token retrieval routes to appropriate provider (GitHub App or GitLab PAT)

---

## Performance Considerations

### Pagination (SC-008: 10,000+ files)

**GitLab API Pagination:**
- Default page size: 20 items
- Max page size: 100 items
- Headers: `X-Total-Pages`, `X-Next-Page`, `X-Per-Page`

**Implementation:**
```go
func (c *Client) GetAllBranches(ctx context.Context, owner, repo string) ([]GitLabBranch, error) {
    var allBranches []GitLabBranch
    page := 1
    perPage := 100  // Max page size

    for {
        branches, nextPage, err := c.GetBranchesPage(ctx, owner, repo, page, perPage)
        if err != nil {
            return nil, err
        }

        allBranches = append(allBranches, branches...)

        if nextPage == 0 {
            break // No more pages
        }

        page = nextPage

        // Safety limit: prevent infinite loop
        if page > 100 {
            return nil, fmt.Errorf("exceeded pagination limit (100 pages)")
        }
    }

    return allBranches, nil
}
```

### Caching Strategy

**Token Caching:**
- PATs are long-lived (no expiration unless set by user)
- Cache in-memory after first retrieval from Secret
- Invalidate cache on user disconnect or token update

**API Response Caching:**
- Not implemented in MVP (GitLab API rate limits are generous: 300 req/min)
- Future enhancement: Cache branches/tree for 1-5 minutes

---

## Security & Privacy

### Token Storage

**Never Store in:**
- Logs (redact tokens: `[REDACTED]`)
- API responses (only return connection status)
- ConfigMaps (only non-sensitive metadata)
- Git commit messages or URLs in logs

**Always Store in:**
- Kubernetes Secrets (encrypted at rest)
- Memory (short-lived, cleared after use)

### Error Handling

**Don't Expose:**
- Token values in error messages
- Internal system paths
- Stack traces to end users

**Do Expose:**
- User-friendly error messages
- Actionable remediation steps
- Link to documentation for complex issues

---

## Next Steps

This data model enables:
1. **Phase 1 Implementation**: Create `gitlab/` package with structs and types
2. **API Contract Definition**: Define OpenAPI schemas in `/contracts/`
3. **Handler Implementation**: Implement GitLab-aware handlers in `handlers/`
4. **Testing**: Unit tests for validation logic, contract tests for API interactions
