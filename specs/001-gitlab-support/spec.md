# Feature Specification: GitLab Support for vTeam

**Feature Branch**: `001-gitlab-support`
**Created**: 2025-11-04
**Status**: Draft
**Input**: User description: "Add GitLab support to the vTeam project. Currently supports only GitHub"

## Clarifications

### Session 2025-11-04

- Q: What level of observability (logging, metrics, tracing) should be implemented for GitLab integration operations? → A: Match current pattern - Use standard library logging (log.Printf / fmt.Printf) for GitLab operations, logging only warnings and critical errors
- Q: What retry strategy should be used for transient GitLab API failures (network errors, timeouts, 5xx responses)? → A: Match GitHub implementation - No automatic retries, use 15-second HTTP client timeout, fail immediately and return errors to user
- Q: How should GitLab PATs be protected in transit and at runtime within AgenticSession pods? → A: Match GitHub implementation - Store in Kubernetes Secret referenced by ProjectSettings, inject via EnvFrom as environment variables and mount as read-only volume at /var/run/runner-secrets/, use security context with all capabilities dropped and privilege escalation disabled
- Q: How should the system handle GitLab API rate limit errors (HTTP 429)? → A: Match GitHub implementation - Treat 429 responses like any other error, fail immediately without parsing rate limit headers, return error response body to user
- Q: Should GitLab tokens with scopes exceeding minimum requirements (e.g., full api scope instead of read_api + read_repository) be accepted or rejected? → A: Match GitHub implementation - No scope inspection or validation, validate tokens by attempting test API call (access user info or repository), accept any token that successfully authenticates regardless of granted scopes

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Configure vTeam Project with GitLab Repository (Priority: P1)

A developer wants to use the Ambient Code Platform with their GitLab-hosted repository to leverage AI-powered code automation. They need to configure a new vTeam project pointing to their GitLab.com or self-hosted GitLab repository, authenticate using a Personal Access Token, and have the system validate the connection.

**Why this priority**: This is the foundational capability - without the ability to configure and connect to GitLab repositories, no other functionality is possible. This represents the minimum viable feature that unlocks GitLab users.

**Independent Test**: Can be fully tested by creating a new vTeam project with a GitLab repository URL, adding a GitLab PAT to runner secrets, and verifying the system successfully validates the token and connection. Delivers immediate value by confirming GitLab connectivity.

**Acceptance Scenarios**:

1. **Given** a user has a GitLab.com repository, **When** they create a vTeam project with the URL `https://gitlab.com/owner/repo.git` and store a valid PAT in runner secrets, **Then** the system successfully validates the repository and displays it as configured
2. **Given** a user has a self-hosted GitLab instance, **When** they configure a project with `https://gitlab.company.com/team/repo.git`, **Then** the system detects the self-hosted instance and constructs the correct API URL (`https://gitlab.company.com/api/v4`)
3. **Given** a user provides an invalid GitLab PAT, **When** the system attempts validation via test API call, **Then** the system displays a clear error message indicating the token is invalid and provides guidance on creating a valid token
4. **Given** a user provides a GitLab PAT with insufficient permissions to access the repository, **When** the system validates the token via test API call, **Then** the system displays an error indicating the token cannot access the repository

---

### User Story 2 - Browse GitLab Repository Contents (Priority: P2)

A user with a configured GitLab project wants to browse their repository's file structure, view branches, and read file contents through the vTeam interface, similar to how they can with GitHub repositories.

**Why this priority**: Repository browsing is essential for users to understand repository structure before creating AgenticSessions and for troubleshooting. It provides immediate feedback that the integration is working correctly.

**Independent Test**: Can be fully tested by configuring a GitLab project (from Story 1), then using the vTeam API to list branches, browse directory trees, and read file contents. Delivers value by enabling repository exploration without implementation work.

**Acceptance Scenarios**:

1. **Given** a configured GitLab project, **When** a user requests the list of branches, **Then** the system returns all branches from the GitLab repository with accurate names and metadata
2. **Given** a configured GitLab project and a specific branch, **When** a user browses the root directory, **Then** the system displays the directory tree structure with files and folders
3. **Given** a user is browsing a GitLab repository, **When** they select a specific file, **Then** the system retrieves and displays the file contents accurately
4. **Given** a large GitLab repository with many branches or files, **When** the system retrieves the data, **Then** the system correctly handles API pagination and returns complete results
5. **Given** a GitLab API rate limit is reached, **When** the user attempts to browse, **Then** the system returns an error with GitLab's response body indicating the rate limit was exceeded

---

### User Story 3 - Execute AgenticSession with GitLab Repository (Priority: P1)

A developer wants to create an AgenticSession that can clone their GitLab repository, make AI-powered code changes, commit those changes, and push them back to GitLab, just as they currently do with GitHub repositories.

**Why this priority**: This is the core value proposition of the Ambient Code Platform - enabling AI-driven code automation. Without this, GitLab support is incomplete and doesn't deliver the primary user benefit.

**Independent Test**: Can be fully tested by creating an AgenticSession with a task (e.g., "add a comment to README"), verifying the session clones the GitLab repo, makes changes, commits, and pushes successfully. Delivers the complete end-to-end value of the platform for GitLab users.

**Acceptance Scenarios**:

1. **Given** a configured GitLab project with valid write permissions, **When** a user creates an AgenticSession with a code modification task, **Then** the system successfully clones the GitLab repository using token authentication
2. **Given** an AgenticSession has cloned a GitLab repository, **When** the AI agent makes code changes and commits, **Then** the commit is created successfully in the local repository
3. **Given** an AgenticSession has committed changes, **When** the system pushes to GitLab, **Then** the changes appear in the GitLab repository and are visible in the GitLab UI
4. **Given** an AgenticSession attempts to push with a token lacking write permissions, **When** the push fails due to GitLab 403 Forbidden response, **Then** the system returns a clear error message indicating insufficient permissions and guidance on ensuring the token has write_repository scope
5. **Given** an AgenticSession is working with a self-hosted GitLab instance, **When** Git operations are performed, **Then** the system correctly constructs authentication URLs with the self-hosted domain
6. **Given** a user completes an AgenticSession with GitLab, **When** they receive the completion notification, **Then** the notification includes a direct link to the branch in GitLab

---

### User Story 4 - Use Mixed GitHub and GitLab Repositories (Priority: P3)

An organization uses both GitHub and GitLab repositories and wants to configure a single vTeam project that can work with repositories from both providers, enabling AgenticSessions to interact with code from multiple sources.

**Why this priority**: This addresses a real but less common scenario - organizations in transition between providers or with split ecosystems. It's valuable but not essential for initial GitLab adoption.

**Independent Test**: Can be fully tested by configuring a project with both a GitHub repository and a GitLab repository, then creating an AgenticSession that clones both and verifies authentication works correctly for each provider. Delivers value for multi-provider organizations.

**Acceptance Scenarios**:

1. **Given** a vTeam project configured with both GitHub and GitLab repositories, **When** the system performs operations, **Then** it correctly identifies which provider each repository uses based on the URL
2. **Given** an AgenticSession working with mixed repositories, **When** cloning repositories, **Then** the system uses the appropriate authentication method for each provider (GitHub App token for GitHub, PAT for GitLab)
3. **Given** an AgenticSession makes changes to both GitHub and GitLab repositories, **When** pushing changes, **Then** both repositories are updated successfully without interference between providers
4. **Given** one provider fails (e.g., GitLab token expires) while the other succeeds, **When** reporting results, **Then** the system clearly indicates which provider failed and provides specific error details

---

### User Story 5 - Seed GitLab Repository with Required Structure (Priority: P3)

A user creates a new vTeam project with a GitLab repository that lacks the required `.claude/` directory structure. They want the system to automatically seed the repository with necessary templates and agent configurations so they can immediately start using AgenticSessions.

**Why this priority**: Repository seeding improves the onboarding experience for new GitLab users, but it's not blocking since users can manually add required files. It's a convenience feature that reduces friction.

**Independent Test**: Can be fully tested by configuring a vTeam project with a GitLab repository missing `.claude/` directories, triggering repository seeding, and verifying the system clones, adds files, commits, and pushes the structure to GitLab. Delivers value by automating setup.

**Acceptance Scenarios**:

1. **Given** a GitLab repository without `.claude/` directories, **When** a user triggers repository seeding, **Then** the system detects the missing structure and offers to seed the repository
2. **Given** a user confirms repository seeding, **When** the system performs the operation, **Then** it clones the GitLab repository, copies spec-kit templates and agent configurations, commits the changes, and pushes them to GitLab
3. **Given** repository seeding completes, **When** the user views their GitLab repository, **Then** all required directories and files are present and the repository is ready for AgenticSessions
4. **Given** a seeding operation fails due to permissions, **When** the error occurs, **Then** the system provides clear guidance on required token permissions and allows retry

---

### Edge Cases

- **What happens when a user provides a GitLab URL in SSH format** (`git@gitlab.com:owner/repo.git`)? System must normalize it to HTTPS format for API operations while preserving user intent for Git operations.

- **How does the system handle GitLab repository URLs without the `.git` suffix?** System must accept both formats (`https://gitlab.com/owner/repo` and `https://gitlab.com/owner/repo.git`) and normalize them correctly.

- **What happens when a self-hosted GitLab instance uses a non-standard API port?** System must correctly parse URLs like `https://gitlab.company.com:8443/owner/repo.git` and construct API URLs as `https://gitlab.company.com:8443/api/v4`.

- **How does the system behave when a GitLab API call times out due to network issues?** System must use a 15-second HTTP client timeout (matching GitHub implementation), fail immediately without retries, and return clear error messages indicating network connectivity problems or timeout occurred.

- **What happens when a user's GitLab PAT expires mid-session?** System must detect authentication failures during operations and provide clear messages indicating token expiration and how to update it.

- **How does the system handle very large GitLab repositories (thousands of files/branches)?** System must implement pagination for all API calls and handle large responses without memory issues or timeouts.

- **What happens when a user configures a GitLab repository URL that doesn't exist?** System must validate repository existence during configuration and provide clear error messages indicating the repository was not found.

- **How does the system handle GitLab API version incompatibilities with older self-hosted instances?** System must detect API version during initial connection and provide clear messaging if the GitLab instance is too old to support required endpoints.

- **What happens when a self-hosted GitLab instance requires custom SSL certificates?** System must support standard Kubernetes certificate authority configuration and provide clear error messages for SSL verification failures.

- **How does the system handle concurrent AgenticSessions accessing the same GitLab repository?** System must ensure proper Git operation isolation to prevent conflicts, similar to existing GitHub behavior.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST accept GitLab repository URLs in multiple formats (HTTPS with/without .git suffix, SSH format) and correctly parse owner/repository information

- **FR-002**: System MUST detect whether a repository URL points to GitHub, GitLab.com, or a self-hosted GitLab instance based on the domain

- **FR-003**: System MUST construct correct API base URLs for self-hosted GitLab instances (e.g., `https://gitlab.company.com/api/v4`) when provided with repository URLs

- **FR-004**: System MUST store GitLab Personal Access Tokens securely in Kubernetes Secrets referenced by ProjectSettings CR, inject tokens into runner pods via EnvFrom as environment variables and mount as read-only volume at /var/run/runner-secrets/, and configure pod security context to drop all capabilities and disable privilege escalation

- **FR-005**: System MUST validate GitLab tokens during project configuration by attempting a test API call (e.g., GET /user or repository access), accepting any token that successfully authenticates regardless of scopes granted (document minimum recommended scopes as read_api, read_repository for reference)

- **FR-006**: System SHOULD document that write_repository scope is required for AgenticSessions performing push operations, but validation is access-based (push failures will surface permission errors from GitLab API at runtime)

- **FR-007**: System MUST retrieve branch lists from GitLab repositories via GitLab API v4 (`/projects/:id/repository/branches`)

- **FR-008**: System MUST retrieve directory tree structure from GitLab repositories via GitLab API v4 (`/projects/:id/repository/tree`)

- **FR-009**: System MUST retrieve file contents from GitLab repositories via GitLab API v4 (`/projects/:id/repository/files/:file_path/raw`)

- **FR-010**: System MUST handle GitLab API pagination correctly for all list operations (branches, tree entries)

- **FR-011**: System MUST construct token-injected Git URLs for GitLab repositories in the format `https://oauth2:TOKEN@gitlab.com/owner/repo.git` for clone operations

- **FR-012**: AgenticSessions MUST be able to clone GitLab repositories using token-injected URLs

- **FR-013**: AgenticSessions MUST be able to commit changes to locally cloned GitLab repositories

- **FR-014**: AgenticSessions MUST be able to push commits to GitLab repositories with proper authentication

- **FR-015**: System MUST provide clear, actionable error messages for GitLab-specific failures including invalid tokens, insufficient permissions, API rate limits, and network connectivity issues

- **FR-016**: System MUST map common GitLab API error codes (401, 403, 404, 429, 500) to user-friendly error messages with remediation guidance

- **FR-017**: System MUST support simultaneous operation with both GitHub and GitLab repositories within the same project

- **FR-018**: System MUST use appropriate authentication for each repository based on provider (GitHub App token for GitHub, PAT for GitLab)

- **FR-019**: System MUST maintain backward compatibility - all existing GitHub functionality must continue to work without degradation after GitLab support is added

- **FR-020**: System MUST normalize GitLab repository URLs to a consistent internal format regardless of input format (HTTPS/SSH, with/without .git)

- **FR-021**: System MUST treat GitLab API rate limit errors (HTTP 429) like other API errors, fail immediately, and return GitLab's error response body to users (GitLab.com has 300 requests/minute limit for authenticated users)

- **FR-022**: System MUST support GitLab API v4 endpoints for all required operations

- **FR-023**: System MUST encode GitLab project paths correctly when constructing API URLs (handling special characters, slashes, etc.)

- **FR-024**: System MUST configure GitLab HTTP client with 15-second timeout matching GitHub implementation, fail immediately on timeout or network errors without automatic retries, and return clear error messages to users

- **FR-025**: System MUST log GitLab operations using Go standard library logging (log.Printf / fmt.Printf) consistent with existing GitHub integration, logging warnings for configuration issues and critical errors for operation failures

### Key Entities

- **GitLab Repository**: Represents a Git repository hosted on GitLab.com or self-hosted GitLab instance. Key attributes: URL (HTTPS or SSH format), owner/namespace, repository name, provider type (gitlab.com vs self-hosted), API base URL for self-hosted instances.

- **GitLab Personal Access Token**: Represents authentication credentials for GitLab API and Git operations. Key attributes: token value (sensitive), scopes/permissions (read_api, read_repository, write_repository), expiration date, associated user identity.

- **Repository Provider**: Abstract concept identifying the Git hosting service (GitHub vs GitLab). Key attributes: provider type (github/gitlab), domain name, API base URL, authentication method (App installation vs PAT).

- **Project Configuration**: Represents a configured vTeam project. Key attributes: one or more repository references (each with provider type, URL, and authentication), runner secrets containing tokens, project metadata.

- **AgenticSession Git Operations**: Represents Git operations performed within a session. Key attributes: repository URL with embedded authentication, local clone path, Git operations performed (clone/commit/push), operation status and error messages.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users with GitLab repositories can complete project configuration in under 5 minutes, including token creation and validation

- **SC-002**: Repository browsing operations (list branches, browse tree, read files) for GitLab repositories complete in under 3 seconds for typical repositories (under 1000 files)

- **SC-003**: AgenticSessions can successfully clone, modify, commit, and push to GitLab repositories with 95%+ success rate (excluding user configuration errors like invalid tokens)

- **SC-004**: All existing GitHub functionality continues to work with zero regression - 100% of GitHub integration tests pass after GitLab support is added

- **SC-005**: Users with mixed GitHub/GitLab projects can successfully run AgenticSessions that interact with both providers without manual intervention

- **SC-006**: 90% of GitLab API errors result in user-friendly error messages that clearly explain the problem and provide actionable remediation steps

- **SC-007**: Self-hosted GitLab instances running GitLab 12.0 or newer are successfully supported with full functionality

- **SC-008**: System correctly handles GitLab API pagination for repositories with up to 10,000 files and 500 branches without timeouts or memory issues

- **SC-009**: Token validation during project configuration catches 95%+ of authentication and repository access issues before users attempt to use AgenticSessions (via test API calls)

- **SC-010**: Repository seeding operations for GitLab repositories complete in under 2 minutes for typical template sizes (under 100 files)

## Out of Scope

The following capabilities are explicitly excluded from this feature to maintain focused scope:

- **GitLab OAuth2 authentication flows**: Users will authenticate using Personal Access Tokens only; full OAuth2 application integration is not included
- **Fork creation for GitLab repositories**: Forking workflows are not supported in this version
- **GitLab CI/CD pipeline integration**: No integration with GitLab CI/CD, runners, or pipeline configuration
- **GitLab Issues and Merge Request management**: No automated creation or management of GitLab Issues or Merge Requests from vTeam
- **GitLab-specific features**: No integration with GitLab Snippets, Wikis, Container Registry, Package Registry, or other GitLab-specific features
- **GitLab Groups API**: No support for GitLab Group-level operations in MVP
- **Other Git providers**: Bitbucket, Azure DevOps, Gitea, and other Git hosting platforms are not included
- **Migration tooling**: No automated tools for migrating projects from GitHub to GitLab or vice versa
- **GitLab Enterprise-specific features**: No support for GitLab EE-only features like advanced security scanning, compliance frameworks, or audit events

## Assumptions

- Users creating GitLab integrations understand how to create Personal Access Tokens in GitLab and are comfortable with token-based authentication
- Self-hosted GitLab instances are accessible from the Kubernetes cluster where vTeam runs (no special firewall rules or VPN requirements beyond standard network connectivity)
- GitLab instances use standard HTTPS ports (443) or standard API paths unless explicitly configured otherwise
- Users with self-hosted GitLab instances are using GitLab 12.0 or newer (API v4 fully supported)
- GitLab API rate limits (300 requests/minute for authenticated users on GitLab.com) are sufficient for typical vTeam usage patterns
- Users understand the difference between read and write token scopes and can configure appropriate permissions
- Repository URLs provided by users are valid and accessible with the provided credentials
- Standard Git operations (clone, commit, push) behavior is consistent between GitHub and GitLab repositories
- Users with mixed GitHub/GitLab projects understand which repositories use which authentication method
- Existing GitHub implementation patterns (direct HTTP API calls, token-based auth) are appropriate templates for GitLab implementation
