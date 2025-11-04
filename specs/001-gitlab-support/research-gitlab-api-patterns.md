# Research: GitLab API v4 Best Practices and Patterns

**Feature**: GitLab Support for vTeam
**Branch**: `001-gitlab-support`
**Date**: 2025-11-04
**Author**: Archie (Architect)

## Executive Summary

This document provides architectural decisions and implementation patterns for GitLab API v4 integration in vTeam. Based on analysis of the existing GitHub implementation and industry best practices, this research guides critical design decisions for Phase 1 implementation.

**Key architectural implications**:
- **Direct HTTP calls pattern** (mirrors existing GitHub implementation) over library abstraction
- **Token-injected URLs** for Git operations maintain consistency with current architecture
- **Self-hosted detection** via domain parsing enables multi-tenancy support
- **Stateless API client design** supports horizontal scaling patterns

**Long-term scalability considerations**:
- Rate limiting architecture supports future Redis-based distributed rate limiting
- API client abstraction enables future GraphQL migration path
- Provider detection pattern extensible to Bitbucket/Azure DevOps in 18-24 months

---

## Decision 1: Go HTTP Client Pattern for GitLab API

### Decision

**Use direct HTTP calls with `net/http` client**, mirroring the existing GitHub implementation pattern rather than adopting the go-gitlab library.

**HTTP Client Configuration**:
```go
// gitlab/client.go
type Client struct {
    baseURL    string // e.g., "https://gitlab.com/api/v4" or "https://gitlab.company.com/api/v4"
    token      string // Personal Access Token
    httpClient *http.Client
}

func NewClient(baseURL, token string) *Client {
    return &Client{
        baseURL: strings.TrimSuffix(baseURL, "/"),
        token:   token,
        httpClient: &http.Client{
            Timeout: 15 * time.Second, // Matches GitHub implementation
            Transport: &http.Transport{
                MaxIdleConns:        100,
                MaxIdleConnsPerHost: 10,
                IdleConnTimeout:     90 * time.Second,
            },
        },
    }
}
```

**Request Construction Pattern**:
```go
func (c *Client) doRequest(ctx context.Context, method, path string, body io.Reader) (*http.Response, error) {
    url := c.baseURL + path
    req, err := http.NewRequestWithContext(ctx, method, url, body)
    if err != nil {
        return nil, err
    }

    // GitLab API v4 headers
    req.Header.Set("PRIVATE-TOKEN", c.token) // GitLab standard header
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("User-Agent", "vTeam-Backend")

    return c.httpClient.Do(req)
}
```

### Rationale

**Why direct HTTP calls vs go-gitlab library?**

**Architectural Consistency**:
- Existing GitHub implementation uses direct HTTP calls (see `/workspace/sessions/agentic-session-1762276850/workspace/vTeam/components/backend/handlers/github_auth.go` lines 66-88)
- Pattern already proven: `doGitHubRequest()` function demonstrates team's familiarity with this approach
- Avoids cognitive overhead of learning library-specific abstractions
- System-wide consistency: same error handling patterns, timeout configurations, logging approaches

**System-Level Implications**:
- **Dependency management**: Reduces third-party dependencies from 1 major library to zero
- **Version compatibility**: go-gitlab library migrated from `github.com/xanzy/go-gitlab` to `gitlab.com/gitlab-org/api/client-go` in 2024 - demonstrates instability risk
- **Debugging surface**: Direct HTTP calls enable request/response logging without library internals
- **Security posture**: Smaller attack surface (no transitive dependencies from library)

**Long-Term Evolution**:
- **GraphQL migration path**: When GitLab GraphQL API becomes preferred (2-3 year horizon), direct HTTP approach easier to migrate than library abstraction
- **Multi-provider abstraction**: Future provider interface (`type RepositoryProvider interface`) simpler to implement with uniform HTTP patterns
- **Performance optimization**: Direct control enables request batching, connection pooling tuning specific to vTeam's usage patterns

**Technical Debt Considerations**:
- **Maintenance burden**: Library handles API changes automatically, direct calls require manual updates
- **Mitigation**: GitLab API v4 stable since 2016, breaking changes rare (documented in FR-022)
- **Pragmatic trade-off**: vTeam uses subset of API (4-5 endpoints) vs go-gitlab's 200+ - limited exposure to API changes

### Alternatives Considered

**Alternative 1: Adopt go-gitlab (xanzy/go-gitlab or gitlab.com/gitlab-org/api/client-go)**

**Pros**:
- Built-in pagination handling
- Automatic retries with exponential backoff
- Type-safe API surface (struct-based responses)
- Community maintenance (1,826 projects use it per web research)

**Cons (why rejected)**:
- **Architectural inconsistency**: GitHub uses direct HTTP, mixing patterns increases cognitive load
- **Library migration risk**: Official library moved packages in 2024, breaking existing users
- **Dependency weight**: 15+ transitive dependencies vs 0 for stdlib approach
- **Tight coupling**: Library changes force code updates even for unused endpoints
- **Learning curve**: Team must learn library patterns vs familiar HTTP client patterns

**Decision justification**: Consistency with existing architecture outweighs library conveniences for this project's scale (4-5 endpoints).

**Alternative 2: Create provider abstraction layer now**

**Proposal**: Abstract GitLab/GitHub behind `RepositoryProvider` interface immediately.

**Cons (why rejected)**:
- **Premature abstraction**: Only 2 providers (GitHub, GitLab) don't justify interface yet
- **YAGNI principle**: Third provider (Bitbucket, Azure DevOps) not in roadmap for 18 months
- **Implementation complexity**: Abstractions add 30-40% more code to test and maintain
- **Performance overhead**: Interface calls add vtable lookups (minor but measurable at scale)

**Decision justification**: Defer abstraction until 3rd provider (GitLab is provider #2). Pattern will be clear then, avoid over-engineering now.

**Alternative 3: Use GitHub's go-github library pattern**

**Proposal**: Mirror GitHub's official SDK architecture (service-based structure).

**Cons (why rejected)**:
- **Not applicable**: vTeam doesn't use go-github either (uses direct HTTP)
- **Scope creep**: Would require refactoring GitHub code to match, expanding PR scope
- **Team preference**: Existing codebase demonstrates preference for HTTP client control

### Implementation Notes

**Error Handling Pattern** (mirrors GitHub implementation):
```go
func (c *Client) handleAPIError(resp *http.Response) error {
    body, _ := io.ReadAll(resp.Body)

    switch resp.StatusCode {
    case 401:
        return &GitLabError{
            StatusCode: 401,
            Message: "Invalid GitLab token. Please verify your Personal Access Token.",
            Remediation: "Create a new token at Settings > Access Tokens with required scopes",
        }
    case 403:
        return &GitLabError{
            StatusCode: 403,
            Message: "Insufficient permissions. Token lacks required scopes.",
            Remediation: "Add 'read_api', 'read_repository', 'write_repository' scopes to your token",
        }
    case 404:
        return &GitLabError{
            StatusCode: 404,
            Message: "Repository not found or you don't have access.",
            Remediation: "Verify repository URL and token permissions",
        }
    case 429:
        resetTime := resp.Header.Get("RateLimit-Reset")
        return &GitLabError{
            StatusCode: 429,
            Message: fmt.Sprintf("GitLab API rate limit exceeded. Resets at %s", resetTime),
            Remediation: "Wait for rate limit reset or reduce API call frequency",
        }
    default:
        return fmt.Errorf("GitLab API error %d: %s", resp.StatusCode, string(body))
    }
}
```

**GitLab-Specific Headers** (different from GitHub):
- **Authentication**: `PRIVATE-TOKEN: <token>` (not `Authorization: Bearer <token>`)
- **Rate Limiting**: `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` headers
- **Pagination**: `X-Total-Pages`, `X-Next-Page` headers (different from GitHub's Link header)

**Performance Characteristics**:
- Timeout: 15 seconds (matches GitHub pattern, reasonable for API calls)
- Connection pooling: 100 max idle connections, 10 per host (supports ~50 concurrent requests)
- Context cancellation: All requests accept `context.Context` for request lifecycle management

---

## Decision 2: Rate Limiting Strategy

### Decision

**Client-side rate limiting with token bucket algorithm** for GitLab.com, with bypass for self-hosted instances.

**Rate Limiter Configuration**:
```go
// gitlab/ratelimit.go
type RateLimiter struct {
    enabled      bool // false for self-hosted instances
    requestsPerMin int  // 300 for GitLab.com
    tokens       chan struct{}
    refillTicker *time.Ticker
}

func NewRateLimiter(host string) *RateLimiter {
    // Only enable for gitlab.com
    enabled := host == "gitlab.com"

    rl := &RateLimiter{
        enabled:        enabled,
        requestsPerMin: 300, // GitLab.com authenticated rate limit
        tokens:         make(chan struct{}, 300),
    }

    if enabled {
        // Fill bucket initially
        for i := 0; i < 300; i++ {
            rl.tokens <- struct{}{}
        }

        // Refill tokens at 5/second (300/minute)
        rl.refillTicker = time.NewTicker(200 * time.Millisecond)
        go rl.refillLoop()
    }

    return rl
}

func (rl *RateLimiter) Wait(ctx context.Context) error {
    if !rl.enabled {
        return nil // No rate limiting for self-hosted
    }

    select {
    case <-rl.tokens:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

**Integration into Client**:
```go
func (c *Client) doRequest(ctx context.Context, method, path string, body io.Reader) (*http.Response, error) {
    // Wait for rate limit token
    if err := c.rateLimiter.Wait(ctx); err != nil {
        return nil, fmt.Errorf("rate limit wait cancelled: %w", err)
    }

    // Proceed with request...
}
```

**Rate Limit Response Handling**:
```go
func (c *Client) handleRateLimitHeaders(resp *http.Response) {
    remaining := resp.Header.Get("RateLimit-Remaining")
    reset := resp.Header.Get("RateLimit-Reset")

    if remaining == "0" {
        resetTime, _ := time.Parse(time.RFC1123, reset)
        waitDuration := time.Until(resetTime)
        log.Printf("GitLab rate limit exhausted, reset in %v", waitDuration)

        // Optional: Adaptive rate limiting - slow down requests
        c.rateLimiter.Backoff(waitDuration)
    }
}
```

### Rationale

**Why token bucket over other algorithms?**

**Architectural Fit**:
- **Burst tolerance**: Token bucket allows burst of 300 requests immediately (useful for initial repository browsing)
- **Smooth rate**: Refills at 5 tokens/second, prevents sustained overload
- **Context-aware**: Respects request cancellation (critical for AgenticSession timeouts)
- **Stateless clients**: Each vTeam backend pod has independent rate limiter (no shared state)

**System-Level Implications**:
- **Horizontal scaling**: Rate limiting per pod, not per cluster - natural load distribution
- **Self-hosted optimization**: Bypasses rate limiting for self-hosted instances (typically higher limits or disabled)
- **Degraded mode**: On 429 response, adaptive backoff prevents retry storms

**Long-Term Scalability**:
- **Redis-backed rate limiting** (18-24 month horizon): Replace in-memory token bucket with Redis for cluster-wide limits
- **Per-user rate limiting**: Future enhancement to track limits per GitLab PAT (prevents single user exhausting quota)
- **Circuit breaker pattern**: When 50%+ requests return 429, open circuit and return errors immediately

**Compliance with Success Criteria**:
- **SC-002**: Repository browsing < 3 seconds - token bucket enables burst for initial page load
- **FR-021**: Detect and handle rate limits - proactive throttling prevents 429 errors in steady state

### Alternatives Considered

**Alternative 1: No rate limiting (rely on GitLab's 429 responses)**

**Pros**:
- Simplest implementation (no rate limiter code)
- Self-adjusting to actual GitLab limits (no hardcoded values)

**Cons (why rejected)**:
- **User experience**: 429 errors surface to users, requires retry logic everywhere
- **Cascading failures**: High request volume causes sustained 429s, blocking all operations
- **Resource waste**: Retry attempts consume network/CPU even when rate limited
- **Multi-tenant impact**: One user's burst affects others on same vTeam instance

**Decision justification**: Proactive rate limiting prevents user-facing errors and resource waste.

**Alternative 2: Exponential backoff on 429 responses only**

**Pros**:
- React to actual rate limits, not predictions
- Simple retry logic (already common pattern)

**Cons (why rejected)**:
- **Reactive vs proactive**: Still sends requests that will fail
- **Compounding effect**: Multiple concurrent operations all retry, amplifying load
- **Poor UX**: User sees "retry attempt 1... retry attempt 2..." messages
- **Performance**: Exponential backoff can lead to multi-minute delays (1s, 2s, 4s, 8s, 16s...)

**Decision justification**: Token bucket prevents 429s before they occur, better UX and performance.

**Alternative 3: Distributed rate limiting (Redis-backed)**

**Pros**:
- Cluster-wide rate limiting (accurate total limit)
- Per-user tracking across pods
- Persistent state (survives pod restarts)

**Cons (why rejected for MVP)**:
- **Infrastructure dependency**: Requires Redis deployment (not in vTeam stack today)
- **Complexity**: Network calls for every API request (latency overhead)
- **Over-engineering**: vTeam's scale (< 100 concurrent users) doesn't justify distributed approach
- **Cost**: Redis cluster adds operational burden

**Decision justification**: Defer to future phase when scale demands it (18-24 months). Per-pod limiting sufficient for MVP.

### Implementation Notes

**Rate Limit Testing Strategy**:
```go
// gitlab/ratelimit_test.go
func TestRateLimiter_BurstCapacity(t *testing.T) {
    rl := NewRateLimiter("gitlab.com")
    ctx := context.Background()

    // Should allow 300 immediate requests (full bucket)
    start := time.Now()
    for i := 0; i < 300; i++ {
        err := rl.Wait(ctx)
        assert.NoError(t, err)
    }
    duration := time.Since(start)

    // All 300 should complete in < 100ms (no blocking)
    assert.Less(t, duration, 100*time.Millisecond)

    // 301st request should block (bucket empty)
    done := make(chan bool)
    go func() {
        rl.Wait(ctx)
        done <- true
    }()

    select {
    case <-done:
        t.Fatal("Expected 301st request to block")
    case <-time.After(100 * time.Millisecond):
        // Expected: request still blocking
    }
}
```

**Self-Hosted Detection**:
```go
func (c *Client) isGitLabDotCom() bool {
    parsedURL, _ := url.Parse(c.baseURL)
    return parsedURL.Host == "gitlab.com"
}
```

**Monitoring and Observability**:
- Log rate limit hits: `log.Printf("Rate limiter: waited %v for token", duration)`
- Metrics: Expose Prometheus gauge for token bucket fill level
- Alerting: Fire alert when token bucket consistently < 10% (sustained high load)

**GitLab.com Rate Limit Details** (from web research):
- **Authenticated**: 300 requests/minute (5 req/second)
- **Unauthenticated**: 60 requests/minute (1 req/second)
- **Headers**: `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`
- **Self-hosted**: Configurable (often disabled or much higher)

---

## Decision 3: Pagination Pattern

### Decision

**Cursor-based pagination with automatic page fetching** for list operations (branches, tree entries).

**Pagination Implementation**:
```go
// gitlab/pagination.go
type PaginationOptions struct {
    Page    int // Current page (1-indexed)
    PerPage int // Items per page (max 100 for GitLab)
}

type PaginatedResponse struct {
    Items      []json.RawMessage // Raw JSON items
    TotalPages int
    NextPage   *int // nil if no next page
}

func (c *Client) getPaginated(ctx context.Context, path string, opts PaginationOptions) (*PaginatedResponse, error) {
    if opts.PerPage == 0 {
        opts.PerPage = 100 // GitLab maximum
    }
    if opts.Page == 0 {
        opts.Page = 1
    }

    // Construct URL with pagination params
    u := fmt.Sprintf("%s?page=%d&per_page=%d", path, opts.Page, opts.PerPage)
    resp, err := c.doRequest(ctx, "GET", u, nil)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    // Parse pagination headers
    totalPages := parseHeaderInt(resp.Header.Get("X-Total-Pages"))
    nextPage := parseHeaderInt(resp.Header.Get("X-Next-Page"))

    var items []json.RawMessage
    if err := json.NewDecoder(resp.Body).Decode(&items); err != nil {
        return nil, err
    }

    result := &PaginatedResponse{
        Items:      items,
        TotalPages: totalPages,
    }
    if nextPage > 0 {
        result.NextPage = &nextPage
    }

    return result, nil
}
```

**Auto-Pagination Helper** (for caller convenience):
```go
func (c *Client) getAllPages(ctx context.Context, path string) ([]json.RawMessage, error) {
    var allItems []json.RawMessage
    page := 1

    for {
        resp, err := c.getPaginated(ctx, path, PaginationOptions{Page: page, PerPage: 100})
        if err != nil {
            return nil, err
        }

        allItems = append(allItems, resp.Items...)

        if resp.NextPage == nil {
            break // Last page reached
        }
        page = *resp.NextPage

        // Safety: Cap at 100 pages (10,000 items with per_page=100)
        if page > 100 {
            return nil, fmt.Errorf("pagination safety limit exceeded (100 pages)")
        }
    }

    return allItems, nil
}
```

**Usage in Endpoint Functions**:
```go
func (c *Client) GetBranches(ctx context.Context, projectPath string) ([]Branch, error) {
    path := fmt.Sprintf("/projects/%s/repository/branches", url.PathEscape(projectPath))

    items, err := c.getAllPages(ctx, path)
    if err != nil {
        return nil, err
    }

    branches := make([]Branch, 0, len(items))
    for _, item := range items {
        var branch Branch
        if err := json.Unmarshal(item, &branch); err != nil {
            log.Printf("Failed to parse branch: %v", err)
            continue // Skip malformed items
        }
        branches = append(branches, branch)
    }

    return branches, nil
}
```

### Rationale

**Why auto-pagination vs manual page handling?**

**Architectural Pattern**:
- **Simplifies callers**: Handler code doesn't need pagination logic (matches GitHub implementation pattern)
- **Consistent with UI expectations**: Frontend expects complete lists, not paginated responses
- **Aligns with FR-010**: "System MUST handle GitLab API pagination correctly for all list operations"

**System-Level Implications**:
- **Memory management**: 100-page safety limit prevents OOM for pathological repositories
- **Latency trade-off**: Multiple sequential API calls (100ms each) vs single large response
  - Example: 500 branches = 5 pages = ~500ms total (acceptable per SC-002: < 3 seconds)
- **Rate limit impact**: Pagination consumes rate limit tokens (5 pages = 5 tokens)

**Long-Term Scalability**:
- **Streaming pagination** (future enhancement): Return channel of items, stream to caller
  - Enables UI progressive loading (show first 100 branches immediately)
  - Reduces memory footprint for large repositories
- **Parallel page fetching**: Fetch pages 2-N concurrently after page 1 determines total pages
  - Reduces latency for large repositories (500 branches: 500ms → 200ms)
  - Requires careful rate limit accounting

**Compliance with Success Criteria**:
- **SC-008**: "Handle repositories with up to 10,000 files and 500 branches without timeouts"
  - 500 branches = 5 pages × 100ms = 500ms (well under 3 second target)
  - 10,000 files = 100 pages × 100ms = 10 seconds (requires optimization or lazy loading)

### Alternatives Considered

**Alternative 1: Manual pagination (caller specifies page number)**

**Pros**:
- Caller controls page size and fetch strategy
- Supports UI pagination (show "Next Page" button)

**Cons (why rejected)**:
- **Complexity leakage**: Every caller must implement pagination loop
- **GitHub inconsistency**: Existing GitHub code uses auto-pagination (see `handlers/repo.go` lines 142-171)
- **Error-prone**: Easy to forget pagination, leading to truncated results (subtle bug)

**Decision justification**: Auto-pagination matches existing patterns and simplifies callers.

**Alternative 2: Fetch all pages concurrently**

**Pros**:
- Faster for large repositories (parallel network requests)
- Better utilization of rate limit burst capacity

**Cons (why rejected for MVP)**:
- **Complexity**: Requires goroutine coordination, error aggregation
- **Rate limit risk**: Burst of 100 requests exhausts token bucket
- **Ordering issues**: GitLab API may modify data between requests (rare but possible)

**Decision justification**: Sequential pagination simpler and sufficient for MVP. Revisit if SC-008 testing reveals performance issues.

**Alternative 3: Link header parsing (GitHub-style)**

**Proposal**: Use Link header pagination like GitHub (`rel="next"`, `rel="last"`).

**Cons (why rejected)**:
- **GitLab API difference**: GitLab uses `X-Next-Page` header, not Link header
- **Unnecessary abstraction**: Unified pagination for GitHub+GitLab not needed (separate clients)

### Implementation Notes

**GitLab Pagination Headers**:
```
X-Total: 250           # Total number of items
X-Total-Pages: 3       # Total number of pages
X-Per-Page: 100        # Items per page
X-Page: 1              # Current page
X-Next-Page: 2         # Next page number (absent on last page)
X-Prev-Page:           # Previous page number (absent on first page)
```

**Project Path Encoding** (critical for API URLs):
```go
// GitLab uses "namespace/project" format, must be URL-encoded
// Example: "gitlab-org/gitlab" → "gitlab-org%2Fgitlab"
projectPath := url.PathEscape("owner/repo")
apiPath := fmt.Sprintf("/projects/%s/repository/branches", projectPath)
// Result: /projects/owner%2Frepo/repository/branches
```

**Performance Testing** (SC-008 validation):
```go
func TestGetBranches_500Branches(t *testing.T) {
    // Mock server that returns 5 pages of 100 branches each
    server := setupMockPaginatedServer(t, 5, 100)
    defer server.Close()

    client := NewClient(server.URL, "test-token")
    start := time.Now()

    branches, err := client.GetBranches(context.Background(), "owner/huge-repo")
    duration := time.Since(start)

    assert.NoError(t, err)
    assert.Len(t, branches, 500)
    assert.Less(t, duration, 3*time.Second, "SC-002: < 3 seconds for typical repositories")
}
```

**Error Handling During Pagination**:
- **Network error on page 3 of 5**: Return partial results + error (caller decides retry)
- **Malformed JSON on single item**: Skip item, log warning, continue (resilient parsing)
- **Rate limit hit mid-pagination**: Return partial results + rate limit error

---

## Decision 4: Self-Hosted GitLab Support

### Decision

**URL-based detection with API base URL construction** from repository URLs.

**URL Parsing and Provider Detection**:
```go
// gitlab/parser.go
type ParsedGitLabURL struct {
    Host         string // "gitlab.com" or "gitlab.company.com"
    Namespace    string // "owner" or "group/subgroup"
    Project      string // "repo"
    IsGitLabDotCom bool
    APIBaseURL   string // "https://gitlab.com/api/v4" or "https://gitlab.company.com/api/v4"
}

func ParseGitLabURL(repoURL string) (*ParsedGitLabURL, error) {
    // Normalize URL (handle SSH format git@gitlab.com:owner/repo.git)
    repoURL = normalizeGitURL(repoURL)

    // Parse URL
    u, err := url.Parse(repoURL)
    if err != nil {
        return nil, fmt.Errorf("invalid repository URL: %w", err)
    }

    // Extract host (handle non-standard ports)
    host := u.Host // Includes port if present (gitlab.company.com:8443)

    // Extract namespace/project from path
    // Example: /owner/repo.git → namespace="owner", project="repo"
    // Example: /group/subgroup/repo.git → namespace="group/subgroup", project="repo"
    pathParts := strings.Split(strings.Trim(u.Path, "/"), "/")
    if len(pathParts) < 2 {
        return nil, fmt.Errorf("invalid GitLab URL path: expected /namespace/project")
    }

    project := strings.TrimSuffix(pathParts[len(pathParts)-1], ".git")
    namespace := strings.Join(pathParts[:len(pathParts)-1], "/")

    // Construct API base URL
    apiBaseURL := fmt.Sprintf("%s://%s/api/v4", u.Scheme, host)

    return &ParsedGitLabURL{
        Host:         host,
        Namespace:    namespace,
        Project:      project,
        IsGitLabDotCom: strings.HasPrefix(host, "gitlab.com"),
        APIBaseURL:   apiBaseURL,
    }, nil
}
```

**Provider Detection** (GitHub vs GitLab):
```go
// git/provider.go
func DetectProvider(repoURL string) (string, error) {
    normalized := normalizeGitURL(repoURL)

    if strings.Contains(normalized, "github.com") {
        return "github", nil
    }
    if strings.Contains(normalized, "gitlab.com") || strings.Contains(normalized, "gitlab") {
        return "gitlab", nil
    }

    return "", fmt.Errorf("unsupported Git provider: %s", repoURL)
}
```

**API Base URL Construction Examples**:
```
Repository URL                                 → API Base URL
--------------------------------------------------------------------------------
https://gitlab.com/owner/repo.git             → https://gitlab.com/api/v4
https://gitlab.company.com/team/repo.git      → https://gitlab.company.com/api/v4
https://gitlab.company.com:8443/team/repo.git → https://gitlab.company.com:8443/api/v4
git@gitlab.company.com:team/repo.git          → https://gitlab.company.com/api/v4
```

### Rationale

**Why URL-based detection over configuration?**

**Architectural Simplicity**:
- **Zero configuration**: Users provide repository URL, system determines provider automatically
- **Consistent with GitHub**: Existing GitHub implementation infers `api.github.com` from `github.com` URL (see `handlers/github_auth.go` lines 57-63)
- **Self-documenting**: URL contains all necessary information (host, path structure)

**System-Level Implications**:
- **Multi-tenancy**: Single vTeam instance supports multiple self-hosted GitLab instances simultaneously
- **No central registry**: No need to pre-configure list of self-hosted domains
- **User flexibility**: Users add new self-hosted instances without administrator intervention

**Long-Term Scalability**:
- **Provider-agnostic architecture**: Pattern extends to Bitbucket (bitbucket.org vs self-hosted), Azure DevOps
- **API versioning**: Future GitLab API v5 supported via URL parsing (detect `/api/v5` in URL)
- **Custom domains**: Supports GitLab instances at non-standard paths (e.g., `company.com/git` → `company.com/git/api/v4`)

**Edge Case Handling**:
- **Non-standard ports**: Preserves port in API URL (`:8443` → `gitlab.company.com:8443/api/v4`)
- **Subgroups**: Supports nested namespaces (`group/subgroup/project` → namespace encoding)
- **SSH URLs**: Normalizes to HTTPS format before parsing (avoids SSH parsing complexity)

### Alternatives Considered

**Alternative 1: Configuration-based provider mapping**

**Proposal**: Add `gitlab_instances` config map listing self-hosted domains.

**Cons (why rejected)**:
- **Operational overhead**: Admins must pre-register every self-hosted instance
- **Scalability bottleneck**: Central configuration limits multi-tenancy
- **User friction**: Users can't add new instances without admin help

**Decision justification**: URL-based detection is self-service and scales to unlimited instances.

**Alternative 2: API version negotiation (query /version endpoint)**

**Proposal**: Call GitLab `/version` endpoint to detect API version and capabilities.

**Cons (why rejected)**:
- **Additional latency**: Extra API call on every operation
- **Complexity**: Requires caching version info, handling version changes
- **Rate limit cost**: Consumes rate limit token for metadata
- **Overkill**: API v4 universal since GitLab 9.0 (2017), version detection not needed

**Decision justification**: Assume API v4, fail fast with clear error if incompatible.

**Alternative 3: DNS-based provider detection**

**Proposal**: Use DNS TXT records to advertise GitLab instances (`_gitlab._tcp.company.com`).

**Cons (why rejected)**:
- **Requires DNS infrastructure**: Not all organizations have DNS control
- **Network dependency**: DNS queries add latency and failure modes
- **Non-standard**: No industry standard for Git provider advertisement
- **Over-engineered**: URL-based detection sufficient for all known use cases

### Implementation Notes

**URL Normalization**:
```go
func normalizeGitURL(repoURL string) string {
    // Convert SSH to HTTPS
    // git@gitlab.com:owner/repo.git → https://gitlab.com/owner/repo.git
    if strings.HasPrefix(repoURL, "git@") {
        repoURL = strings.ReplaceAll(repoURL, ":", "/")
        repoURL = strings.Replace(repoURL, "git@", "https://", 1)
    }

    // Ensure .git suffix for consistency
    if !strings.HasSuffix(repoURL, ".git") {
        repoURL += ".git"
    }

    return repoURL
}
```

**Project Path Encoding** (critical for GitLab API):
```go
// GitLab uses URL-encoded "namespace/project" format in API paths
// Example: GET /api/v4/projects/gitlab-org%2Fgitlab/repository/branches
func projectPathForAPI(namespace, project string) string {
    fullPath := namespace + "/" + project
    return url.PathEscape(fullPath)
}
```

**Self-Hosted SSL Considerations**:
- **Standard certificates**: Kubernetes CA bundle automatically trusted (no special handling)
- **Self-signed certificates**: Users must add to Kubernetes CA bundle (documented in FR-009 edge case)
- **Certificate errors**: Return user-friendly error: "SSL certificate verification failed for gitlab.company.com. Contact administrator to add certificate to Kubernetes trust store."

**API Version Compatibility**:
- **Minimum**: GitLab 12.0 (released April 2019) - documented in assumptions
- **Current**: GitLab 17.x (as of 2025)
- **Testing**: Contract tests verify API v4 compatibility (no version-specific features)
- **Future-proofing**: If GitLab API v5 introduced, add version detection then (not preemptively)

**Provider Detection Testing**:
```go
func TestDetectProvider(t *testing.T) {
    tests := []struct {
        name     string
        repoURL  string
        wantProvider string
        wantErr  bool
    }{
        {"github https", "https://github.com/owner/repo.git", "github", false},
        {"gitlab.com https", "https://gitlab.com/owner/repo.git", "gitlab", false},
        {"gitlab self-hosted", "https://gitlab.company.com/owner/repo.git", "gitlab", false},
        {"gitlab ssh", "git@gitlab.company.com:owner/repo.git", "gitlab", false},
        {"bitbucket", "https://bitbucket.org/owner/repo.git", "", true}, // Not supported yet
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            provider, err := DetectProvider(tt.repoURL)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.wantProvider, provider)
            }
        })
    }
}
```

---

## Decision 5: Token Authentication and Scope Requirements

### Decision

**Personal Access Token (PAT) authentication** with three-tier scope validation.

**Token Scope Requirements**:
```go
// gitlab/token.go
type TokenScopes struct {
    ReadAPI        bool // Required for all operations
    ReadRepository bool // Required for browsing, cloning
    WriteRepository bool // Required for push operations
}

func ValidateToken(ctx context.Context, apiBaseURL, token string) (*TokenScopes, error) {
    // Call GitLab API to validate token and check scopes
    // Endpoint: GET /api/v4/personal_access_tokens/self
    client := &http.Client{Timeout: 10 * time.Second}
    req, _ := http.NewRequestWithContext(ctx, "GET", apiBaseURL+"/personal_access_tokens/self", nil)
    req.Header.Set("PRIVATE-TOKEN", token)

    resp, err := client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("token validation failed: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode == 401 {
        return nil, fmt.Errorf("invalid token: authentication failed")
    }

    var tokenInfo struct {
        Scopes []string `json:"scopes"`
        Active bool     `json:"active"`
        Revoked bool    `json:"revoked"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&tokenInfo); err != nil {
        return nil, fmt.Errorf("failed to parse token info: %w", err)
    }

    if tokenInfo.Revoked {
        return nil, fmt.Errorf("token has been revoked")
    }

    if !tokenInfo.Active {
        return nil, fmt.Errorf("token is not active (may be expired)")
    }

    // Check for required scopes
    scopes := &TokenScopes{}
    for _, scope := range tokenInfo.Scopes {
        switch scope {
        case "api": // 'api' scope includes read_api, read_repository, write_repository
            scopes.ReadAPI = true
            scopes.ReadRepository = true
            scopes.WriteRepository = true
        case "read_api":
            scopes.ReadAPI = true
        case "read_repository":
            scopes.ReadRepository = true
        case "write_repository":
            scopes.WriteRepository = true
            scopes.ReadRepository = true // write_repository implies read_repository
        }
    }

    return scopes, nil
}
```

**Scope Validation Tiers** (enforced at operation time):
```go
func (c *Client) requireScopes(operation string, scopes *TokenScopes) error {
    switch operation {
    case "list_branches", "get_tree", "get_file":
        if !scopes.ReadAPI || !scopes.ReadRepository {
            return fmt.Errorf("operation %s requires 'read_api' and 'read_repository' scopes", operation)
        }
    case "clone":
        if !scopes.ReadRepository {
            return fmt.Errorf("clone operation requires 'read_repository' scope")
        }
    case "push":
        if !scopes.WriteRepository {
            return fmt.Errorf("push operation requires 'write_repository' scope")
        }
    }
    return nil
}
```

**Token-Injected Git URL Format**:
```go
// gitlab/auth.go
func InjectTokenIntoGitURL(repoURL, token string) (string, error) {
    u, err := url.Parse(repoURL)
    if err != nil {
        return "", err
    }

    // GitLab supports oauth2:<token>@ format (same as GitHub)
    u.User = url.UserPassword("oauth2", token)

    return u.String(), nil
}

// Example results:
// https://gitlab.com/owner/repo.git
// → https://oauth2:glpat-xyz@gitlab.com/owner/repo.git
//
// https://gitlab.company.com:8443/owner/repo.git
// → https://oauth2:glpat-xyz@gitlab.company.com:8443/owner/repo.git
```

### Rationale

**Why PAT over OAuth2 application?**

**Architectural Consistency**:
- **Mirrors GitHub pattern**: vTeam already supports GitHub App + PAT fallback (see `git/operations.go` lines 78-149)
- **Simplifies MVP**: OAuth2 requires callback endpoints, state management, refresh tokens (significant complexity)
- **User familiarity**: Developers comfortable with PAT workflow (GitHub, GitLab, Bitbucket all support PATs)

**System-Level Implications**:
- **Token storage**: Stored in Kubernetes Secrets (same as GitHub PAT fallback)
- **Token lifecycle**: Users responsible for token rotation (no automatic refresh like OAuth2)
- **Security posture**: Tokens have full user permissions (can't be scoped per-repository like GitHub App installations)

**Long-Term Evolution**:
- **OAuth2 support** (12-18 month horizon): Add alongside PAT, don't replace
  - Use case: SaaS vTeam instances with many users
  - Benefit: Automatic token refresh, per-installation permissions
- **Token rotation**: Future enhancement to detect expiring tokens and notify users
- **Token proxy**: Centralized token management service (enterprise feature)

**Compliance with Functional Requirements**:
- **FR-005**: Validate `read_api`, `read_repository` during project configuration
- **FR-006**: Validate `write_repository` before AgenticSession push operations
- **FR-015**: Provide actionable error messages for invalid/insufficient tokens

### Alternatives Considered

**Alternative 1: OAuth2 application integration**

**Pros**:
- Automatic token refresh (no expiration issues)
- Per-installation permissions (more secure)
- Industry best practice for SaaS applications

**Cons (why rejected for MVP)**:
- **Complexity**: Requires OAuth2 callback endpoints, state management, refresh token storage
- **Infrastructure**: Needs persistent storage for OAuth2 state (Redis or database)
- **User experience**: Multi-step OAuth flow vs simple PAT copy-paste
- **Self-hosted compatibility**: Self-hosted GitLab instances may disable OAuth2 applications
- **Scope creep**: Adds 3-4 weeks to MVP timeline

**Decision justification**: PAT sufficient for MVP, OAuth2 can be added later if demand justifies complexity.

**Alternative 2: Git credential helper**

**Proposal**: Use Git's credential helper system for token management.

**Cons (why rejected)**:
- **Limited to Git operations**: Doesn't help with GitLab API calls (branch list, etc.)
- **Configuration complexity**: Requires configuring credential helper on runner pods
- **Token injection pattern**: vTeam already uses token-injected URLs (consistent pattern)

**Alternative 3: API scope (full permissions)**

**Proposal**: Require `api` scope only (includes all read/write permissions).

**Cons (why rejected)**:
- **Over-permissive**: Grants API access to issues, merge requests, settings, etc. (not needed)
- **Security principle**: Least privilege - only request scopes actually needed
- **User concern**: Users hesitant to grant full API access for simple Git operations

**Decision justification**: Granular scopes (read_api, read_repository, write_repository) align with security best practices.

### Implementation Notes

**Token Storage in Kubernetes**:
```yaml
# project-namespace/runner-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ambient-runner-secrets
  namespace: project-namespace
type: Opaque
data:
  GITLAB_TOKEN: Z2xwYXQteHl6MTIzYWJjZGVm  # base64(glpat-xyz123abcdef)
  GIT_TOKEN: Z2xwYXQteHl6MTIzYWJjZGVm     # Backward compat with GitHub PAT pattern
```

**Token Validation Caching**:
- **Cache duration**: 5 minutes (balance between API calls and freshness)
- **Cache key**: `gitlab:token:sha256(token)` (avoid storing raw tokens in cache)
- **Cache invalidation**: On 401 response (token revoked/expired)

**Error Messages for Token Issues**:
```
Invalid token:
"GitLab authentication failed. Please verify your Personal Access Token.

To create a new token:
1. Visit https://gitlab.com/-/user_settings/personal_access_tokens
2. Create token with scopes: read_api, read_repository, write_repository
3. Store token in project runner secret as GITLAB_TOKEN"

Insufficient scopes:
"GitLab token lacks required permissions. Current scopes: [read_api, read_repository]

To enable push operations, add 'write_repository' scope:
1. Visit https://gitlab.com/-/user_settings/personal_access_tokens
2. Edit your token and add 'write_repository' scope
3. Update token in project runner secret"
```

**Token Expiration Handling**:
- **Detection**: 401 response during operation indicates expired token
- **User notification**: Display expiration date if available in token metadata
- **Remediation**: Provide link to create new token with same scopes

**GitLab Token Format** (for validation):
- **Format**: `glpat-` prefix + 20 character alphanumeric string
- **Example**: `glpat-xyz123abcdefghij4567`
- **Validation regex**: `^glpat-[a-zA-Z0-9_\-]{20}$`

---

## Decision 6: Error Handling and Retry Strategies

### Decision

**Layered error handling** with user-friendly messages and selective retry logic.

**Error Type Hierarchy**:
```go
// gitlab/errors.go
type GitLabError struct {
    StatusCode  int
    Message     string // User-friendly message
    Remediation string // Actionable guidance
    Cause       error  // Original error (for logging)
}

func (e *GitLabError) Error() string {
    return e.Message
}

func (e *GitLabError) Unwrap() error {
    return e.Cause
}

// Error categories
var (
    ErrInvalidToken = &GitLabError{
        StatusCode:  401,
        Message:     "Invalid GitLab token",
        Remediation: "Verify token at Settings > Access Tokens",
    }

    ErrInsufficientPermissions = &GitLabError{
        StatusCode:  403,
        Message:     "Insufficient permissions",
        Remediation: "Add required scopes: read_api, read_repository, write_repository",
    }

    ErrRepositoryNotFound = &GitLabError{
        StatusCode:  404,
        Message:     "Repository not found",
        Remediation: "Verify repository URL and token permissions",
    }

    ErrRateLimitExceeded = &GitLabError{
        StatusCode:  429,
        Message:     "GitLab API rate limit exceeded",
        Remediation: "Wait for rate limit reset or reduce request frequency",
    }
)
```

**Retry Strategy**:
```go
// gitlab/retry.go
type RetryConfig struct {
    MaxRetries      int
    InitialDelay    time.Duration
    MaxDelay        time.Duration
    RetryableErrors []int // Status codes to retry
}

var DefaultRetryConfig = RetryConfig{
    MaxRetries:      3,
    InitialDelay:    1 * time.Second,
    MaxDelay:        10 * time.Second,
    RetryableErrors: []int{408, 429, 500, 502, 503, 504}, // Timeout, rate limit, server errors
}

func (c *Client) doRequestWithRetry(ctx context.Context, method, path string, body io.Reader) (*http.Response, error) {
    var lastErr error

    for attempt := 0; attempt <= c.retryConfig.MaxRetries; attempt++ {
        if attempt > 0 {
            // Exponential backoff: 1s, 2s, 4s, 8s (capped at maxDelay)
            delay := time.Duration(1<<uint(attempt-1)) * c.retryConfig.InitialDelay
            if delay > c.retryConfig.MaxDelay {
                delay = c.retryConfig.MaxDelay
            }

            log.Printf("Retry attempt %d after %v", attempt, delay)

            select {
            case <-time.After(delay):
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }

        resp, err := c.doRequest(ctx, method, path, body)
        if err != nil {
            // Network error - retry
            lastErr = err
            continue
        }

        // Check if status code is retryable
        retryable := false
        for _, code := range c.retryConfig.RetryableErrors {
            if resp.StatusCode == code {
                retryable = true
                break
            }
        }

        if !retryable {
            return resp, nil // Success or non-retryable error
        }

        // Retryable error - consume response and retry
        io.Copy(io.Discard, resp.Body)
        resp.Body.Close()
        lastErr = fmt.Errorf("GitLab API returned %d", resp.StatusCode)
    }

    return nil, fmt.Errorf("request failed after %d retries: %w", c.retryConfig.MaxRetries, lastErr)
}
```

**Error Response Parsing**:
```go
func (c *Client) parseGitLabError(resp *http.Response) error {
    body, _ := io.ReadAll(resp.Body)

    // Try to parse GitLab error response
    var gitlabErr struct {
        Message string `json:"message"`
        Error   string `json:"error"`
    }
    json.Unmarshal(body, &gitlabErr)

    baseErr := &GitLabError{
        StatusCode: resp.StatusCode,
        Cause:      fmt.Errorf("GitLab API error: %s", string(body)),
    }

    // Map status codes to user-friendly messages
    switch resp.StatusCode {
    case 401:
        baseErr.Message = "Invalid GitLab token. Authentication failed."
        baseErr.Remediation = "Create a new Personal Access Token at Settings > Access Tokens with required scopes"
    case 403:
        baseErr.Message = "Insufficient permissions. Token lacks required scopes."
        baseErr.Remediation = "Add scopes: read_api, read_repository, write_repository"
    case 404:
        baseErr.Message = fmt.Sprintf("Repository not found: %s", gitlabErr.Message)
        baseErr.Remediation = "Verify repository URL and token permissions"
    case 429:
        resetTime := resp.Header.Get("RateLimit-Reset")
        baseErr.Message = fmt.Sprintf("GitLab API rate limit exceeded. Resets at %s", resetTime)
        baseErr.Remediation = "Wait for rate limit reset or reduce API call frequency"
    case 500, 502, 503, 504:
        baseErr.Message = "GitLab server error. Please try again."
        baseErr.Remediation = "If error persists, check GitLab status page or contact administrator"
    default:
        baseErr.Message = fmt.Sprintf("GitLab API error: %s", gitlabErr.Message)
        baseErr.Remediation = "Check GitLab documentation or contact support"
    }

    return baseErr
}
```

### Rationale

**Why structured error types over plain errors?**

**Architectural Pattern**:
- **Aligns with SC-006**: "90% of GitLab API errors result in user-friendly error messages"
- **Consistent with FR-015**: "Provide clear, actionable error messages"
- **Enables error categorization**: Handlers can differentiate auth failures from network errors

**System-Level Implications**:
- **Observability**: Structured errors enable error rate monitoring per category (auth vs rate limit vs server)
- **User experience**: Remediation guidance reduces support burden
- **Debugging**: Wrapped errors preserve context while surfacing root cause

**Long-Term Scalability**:
- **Error budgets**: Track error rates by category (SLO: < 5% auth errors, < 1% rate limit errors)
- **Circuit breakers**: Open circuit when 50%+ errors in category (e.g., GitLab server errors)
- **Automated remediation**: Detect expired tokens, trigger notification workflow

**Retry Strategy Justification**:
- **Transient failures**: 500/502/503/504 errors typically resolve within seconds (network glitches, GitLab deployments)
- **Rate limiting**: 429 errors benefit from backoff (token bucket refills over time)
- **Non-retryable**: 401/403/404 errors permanent (retrying wastes resources and delays user feedback)

### Alternatives Considered

**Alternative 1: Retry all errors uniformly**

**Cons (why rejected)**:
- **Wastes resources**: Retrying 401 (invalid token) will never succeed
- **Poor UX**: User waits 10 seconds for retries before seeing "invalid token" message
- **Rate limit amplification**: Retrying 429 errors exhausts rate limit faster

**Alternative 2: No automatic retries (caller decides)**

**Pros**:
- Caller has full control over retry logic
- Simpler client implementation

**Cons (why rejected)**:
- **Complexity leakage**: Every caller must implement retry logic
- **Inconsistent behavior**: Some callers retry, others don't
- **Doesn't address SC-003**: "95%+ success rate" requires handling transient failures

**Alternative 3: Circuit breaker pattern**

**Proposal**: Open circuit after N consecutive failures, fast-fail subsequent requests.

**Deferred (not rejected)**:
- **Future enhancement**: Add circuit breaker in Phase 2 if observability data shows need
- **Current decision**: Retry logic sufficient for MVP
- **Triggers for implementation**: > 5% error rate sustained for 5 minutes

### Implementation Notes

**Error Handling in Handlers**:
```go
// handlers/repo.go
func ListGitLabBranches(c *gin.Context) {
    branches, err := gitlabClient.GetBranches(ctx, projectPath)
    if err != nil {
        var gitlabErr *gitlab.GitLabError
        if errors.As(err, &gitlabErr) {
            // Structured GitLab error
            c.JSON(gitlabErr.StatusCode, gin.H{
                "error":       gitlabErr.Message,
                "remediation": gitlabErr.Remediation,
            })
            return
        }

        // Generic error
        c.JSON(http.StatusInternalServerError, gin.H{
            "error": "Failed to list branches: " + err.Error(),
        })
        return
    }

    c.JSON(http.StatusOK, gin.H{"branches": branches})
}
```

**Logging Strategy**:
```go
// Log at different levels based on error type
func logGitLabError(err error) {
    var gitlabErr *gitlab.GitLabError
    if errors.As(err, &gitlabErr) {
        switch gitlabErr.StatusCode {
        case 401, 403, 404:
            // User error - log at INFO level
            log.Printf("GitLab user error: %s (remediation: %s)", gitlabErr.Message, gitlabErr.Remediation)
        case 429:
            // Rate limit - log at WARN level
            log.Printf("GitLab rate limit exceeded: %s", gitlabErr.Message)
        case 500, 502, 503, 504:
            // Server error - log at ERROR level with full context
            log.Printf("GitLab server error: %s (cause: %v)", gitlabErr.Message, gitlabErr.Cause)
        }
    }
}
```

**Prometheus Metrics**:
```go
// gitlab/metrics.go
var (
    gitlabAPIErrors = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "vteam_gitlab_api_errors_total",
            Help: "Total GitLab API errors by status code",
        },
        []string{"status_code", "operation"},
    )

    gitlabAPIRetries = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "vteam_gitlab_api_retries_total",
            Help: "Total GitLab API retry attempts",
        },
        []string{"operation"},
    )
)
```

**Testing Error Scenarios**:
```go
func TestClient_HandleErrors(t *testing.T) {
    tests := []struct {
        name            string
        mockStatusCode  int
        mockBody        string
        wantErrType     error
        wantRemediation string
    }{
        {
            name:            "invalid token",
            mockStatusCode:  401,
            mockBody:        `{"message": "401 Unauthorized"}`,
            wantErrType:     gitlab.ErrInvalidToken,
            wantRemediation: "Create a new Personal Access Token",
        },
        {
            name:            "rate limit exceeded",
            mockStatusCode:  429,
            mockBody:        `{"message": "Rate limit exceeded"}`,
            wantErrType:     gitlab.ErrRateLimitExceeded,
            wantRemediation: "Wait for rate limit reset",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                w.WriteHeader(tt.mockStatusCode)
                w.Write([]byte(tt.mockBody))
            }))
            defer server.Close()

            client := gitlab.NewClient(server.URL, "test-token")
            _, err := client.GetBranches(context.Background(), "owner/repo")

            assert.Error(t, err)
            assert.ErrorIs(t, err, tt.wantErrType)
            assert.Contains(t, err.Error(), tt.wantRemediation)
        })
    }
}
```

---

## Summary of Architectural Decisions

| Decision Area | Choice | Architectural Rationale |
|--------------|--------|-------------------------|
| **HTTP Client Pattern** | Direct `net/http` calls | Mirrors GitHub implementation, zero dependencies, full control |
| **Rate Limiting** | Token bucket (client-side) | Proactive throttling, context-aware, horizontally scalable |
| **Pagination** | Auto-pagination with safety limits | Simplifies callers, consistent with GitHub pattern, protects memory |
| **Self-Hosted Support** | URL-based detection | Zero configuration, multi-tenant ready, provider-agnostic |
| **Token Authentication** | PAT with granular scopes | MVP simplicity, aligns with GitHub PAT fallback, security best practice |
| **Error Handling** | Structured errors + selective retry | User-friendly messages (SC-006), transient failure resilience (SC-003) |

---

## Integration Points with Existing Architecture

### Shared Git Operations Package

**Current Pattern** (`git/operations.go`):
```go
func GetGitHubToken(ctx context.Context, k8sClient *kubernetes.Clientset, dynClient dynamic.Interface, project, userID string) (string, error) {
    // Try GitHub App first
    // Fall back to project runner secret GIT_TOKEN
}
```

**Proposed Extension**:
```go
func GetGitProviderToken(ctx context.Context, k8sClient *kubernetes.Clientset, dynClient dynamic.Interface, project, userID, provider string) (string, error) {
    switch provider {
    case "github":
        return GetGitHubToken(ctx, k8sClient, dynClient, project, userID)
    case "gitlab":
        return GetGitLabToken(ctx, k8sClient, dynClient, project, userID)
    default:
        return "", fmt.Errorf("unsupported provider: %s", provider)
    }
}

func GetGitLabToken(ctx context.Context, k8sClient *kubernetes.Clientset, dynClient dynamic.Interface, project, userID string) (string, error) {
    // GitLab doesn't have App concept - only PAT
    // Read from project runner secret GITLAB_TOKEN or GIT_TOKEN
    settings, _ := getProjectSettings(ctx, dynClient, project)
    secretName := settings.RunnerSecret
    if secretName == "" {
        secretName = "ambient-runner-secrets"
    }

    secret, err := k8sClient.CoreV1().Secrets(project).Get(ctx, secretName, v1.GetOptions{})
    if err != nil {
        return "", fmt.Errorf("no GitLab credentials available")
    }

    // Try GITLAB_TOKEN first, fall back to GIT_TOKEN (backward compat)
    if token, ok := secret.Data["GITLAB_TOKEN"]; ok && len(token) > 0 {
        return string(token), nil
    }
    if token, ok := secret.Data["GIT_TOKEN"]; ok && len(token) > 0 {
        return string(token), nil
    }

    return "", fmt.Errorf("no GitLab token in runner secret")
}
```

### Handler Pattern Consistency

**Existing GitHub Pattern** (`handlers/repo.go`):
```go
func ListUserForks(c *gin.Context) {
    // 1. Extract parameters
    project := c.Param("projectName")
    upstreamRepo := c.Query("upstreamRepo")

    // 2. Get K8s clients
    reqK8s, reqDyn := GetK8sClientsForRequestRepo(c)

    // 3. Get token
    token, err := GetGitHubTokenRepo(c.Request.Context(), reqK8s, reqDyn, project, userID.(string))

    // 4. Parse repo URL
    owner, repoName, err := parseOwnerRepo(upstreamRepo)

    // 5. Make API call
    api := githubAPIBaseURL("github.com")
    resp, err := doGitHubRequest(ctx, http.MethodGet, url, "Bearer "+token, "", nil)

    // 6. Return response
    c.JSON(http.StatusOK, gin.H{"forks": all})
}
```

**Proposed GitLab Handler** (parallel structure):
```go
func ListGitLabBranches(c *gin.Context) {
    // 1. Extract parameters (same pattern)
    project := c.Param("projectName")
    repoURL := c.Query("repo")

    // 2. Get K8s clients (same pattern)
    reqK8s, reqDyn := GetK8sClientsForRequestRepo(c)

    // 3. Detect provider and get token
    provider, _ := git.DetectProvider(repoURL)
    if provider != "gitlab" {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Not a GitLab repository"})
        return
    }
    token, err := git.GetGitLabToken(c.Request.Context(), reqK8s, reqDyn, project, userID.(string))

    // 4. Parse repo URL (GitLab-specific)
    parsed, err := gitlab.ParseGitLabURL(repoURL)

    // 5. Make API call (GitLab client)
    client := gitlab.NewClient(parsed.APIBaseURL, token)
    branches, err := client.GetBranches(c.Request.Context(), parsed.Namespace+"/"+parsed.Project)

    // 6. Return response (same pattern)
    c.JSON(http.StatusOK, gin.H{"branches": branches})
}
```

---

## Phase 1 Implementation Roadmap

### Package Structure
```
components/backend/
├── gitlab/                    # NEW: GitLab-specific logic
│   ├── client.go              # HTTP client, API calls
│   ├── client_test.go
│   ├── parser.go              # URL parsing, provider detection
│   ├── parser_test.go
│   ├── token.go               # Token validation, scope checking
│   ├── token_test.go
│   ├── ratelimit.go           # Rate limiting logic
│   ├── ratelimit_test.go
│   ├── pagination.go          # Pagination helpers
│   ├── pagination_test.go
│   ├── errors.go              # Error types and messages
│   └── errors_test.go
├── git/                       # MODIFIED: Provider abstraction
│   ├── operations.go          # Add GetGitLabToken, DetectProvider
│   └── operations_test.go     # Add tests for new functions
├── handlers/                  # MODIFIED: New GitLab endpoints
│   ├── repo.go                # Add GitLab handlers alongside GitHub
│   └── repo_test.go
└── testdata/                  # NEW: Test fixtures
    └── gitlab_api_responses/
        ├── branches_response.json
        ├── tree_response.json
        └── error_*.json
```

### Implementation Sequence

**Week 1: Foundation**
1. Create `gitlab/` package structure
2. Implement URL parsing (`parser.go`)
3. Implement provider detection (`git/operations.go`)
4. Unit tests for parsing (12 test cases)

**Week 2: API Client**
5. Implement HTTP client (`client.go`)
6. Implement rate limiter (`ratelimit.go`)
7. Implement pagination (`pagination.go`)
8. Unit tests for client (20+ test cases)

**Week 3: Token Management**
9. Implement token validation (`token.go`)
10. Implement token-injected URLs (`git/operations.go`)
11. Implement error handling (`errors.go`)
12. Unit tests for token logic (15+ test cases)

**Week 4: Integration**
13. Add GitLab handlers to `handlers/repo.go`
14. Update existing handlers to support both providers
15. Contract tests for GitLab API
16. Integration tests for e2e flows

---

## References

**GitLab API Documentation**:
- [GitLab API v4 Overview](https://docs.gitlab.com/ee/api/api_resources.html)
- [REST API Authentication](https://docs.gitlab.com/api/rest/authentication/)
- [Projects API](https://docs.gitlab.com/ee/api/projects.html)
- [Repository API](https://docs.gitlab.com/ee/api/repositories.html)
- [Rate Limits](https://docs.gitlab.com/ee/security/rate_limits.html)

**Go GitLab Libraries**:
- [go-gitlab (xanzy) - Migrated](https://github.com/xanzy/go-gitlab)
- [gitlab.com/gitlab-org/api/client-go](https://gitlab.com/gitlab-org/api/client-go)
- [Internal Working of GitLab Go Client](https://mincong.io/en/go-gitlab/)

**Existing vTeam Architecture**:
- GitHub implementation: `/workspace/sessions/agentic-session-1762276850/workspace/vTeam/components/backend/handlers/github_auth.go`
- Git operations: `/workspace/sessions/agentic-session-1762276850/workspace/vTeam/components/backend/git/operations.go`
- Repository handlers: `/workspace/sessions/agentic-session-1762276850/workspace/vTeam/components/backend/handlers/repo.go`

**Industry Best Practices**:
- [Kubernetes client-go patterns](https://github.com/kubernetes/client-go)
- [HashiCorp retryablehttp](https://github.com/hashicorp/go-retryablehttp)
- [Go HTTP Client Best Practices](https://go.dev/doc/effective_go)

---

**Next Steps**: This research document provides architectural foundation for Phase 1 implementation. Proceed to create data model definitions and API contracts in `data-model.md`.
