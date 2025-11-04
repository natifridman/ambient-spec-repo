# GitLab Support for vTeam Project

**Feature Overview:**

Add support for GitLab repositories in the vTeam (Ambient Code Platform) backend, enabling users to work with GitLab.com and self-hosted GitLab instances alongside the existing GitHub integration. This feature allows developers who use GitLab as their primary Git hosting platform to leverage the full capabilities of the Ambient Code Platform for AI-powered code automation, eliminating the current GitHub-only limitation.

**Goals:**

* **Primary Users**: Developers and organizations using GitLab.com or self-hosted GitLab instances who want to use the Ambient Code Platform for AI-driven development workflows.

* **User Outcome**: Users can seamlessly integrate their GitLab repositories into vTeam projects, browse repository contents, and execute AI-powered code changes through AgenticSessions that can clone, commit, and push to GitLab repositories.

* **Current State vs Future State**:
  - **Today**: Users are limited to GitHub repositories only, excluding GitLab users from the platform entirely.
  - **With This Feature**: Users can choose between GitHub and GitLab repositories, or even use both within the same project, expanding the platform's addressable market to include the substantial GitLab user base.

* **Business Value**: Removes a critical barrier to adoption for GitLab users, enabling market expansion and meeting customer demands for multi-provider Git support.

**Out of Scope:**

* GitLab OAuth2 authentication flows (users will use Personal Access Tokens instead)
* Fork creation for GitLab repositories
* GitLab CI/CD pipeline integration
* GitLab Issues and Merge Request management
* GitLab-specific features (snippets, wikis, container registry, etc.)
* Support for GitLab Groups API in MVP
* Bitbucket, Azure DevOps, or other Git hosting providers

**Requirements:**

* **[MVP] GitLab Token Authentication**: Users can authenticate to GitLab using Personal Access Tokens (PAT), stored securely in project runner secrets (similar to existing GIT_TOKEN fallback mechanism).

* **[MVP] Repository URL Support**: System correctly parses and validates GitLab repository URLs in multiple formats:
  - HTTPS: `https://gitlab.com/owner/repo.git`
  - SSH: `git@gitlab.com:owner/repo.git`
  - Self-hosted: `https://gitlab.company.com/owner/repo.git`

* **[MVP] Repository Browsing - Tree**: Users can browse the file/directory structure of GitLab repositories through the API, equivalent to existing GitHub tree browsing functionality.

* **[MVP] Repository Browsing - Blob**: Users can read file contents from GitLab repositories through the API, equivalent to existing GitHub blob reading functionality.

* **[MVP] Repository Browsing - Branches**: Users can list all branches in a GitLab repository through the API, equivalent to existing GitHub branch listing functionality.

* **[MVP] Git Clone Operations**: AgenticSessions can clone GitLab repositories using token-injected HTTPS URLs (`https://oauth2:TOKEN@gitlab.com/owner/repo.git`).

* **[MVP] Git Commit Operations**: AgenticSessions can create commits in locally cloned GitLab repositories.

* **[MVP] Git Push Operations**: AgenticSessions can push commits to GitLab repositories with proper authentication and error handling.

* **[MVP] Self-Hosted GitLab Support**: System supports self-hosted GitLab instances by correctly constructing API base URLs (e.g., `https://gitlab.company.com/api/v4`).

* **[MVP] Provider Detection**: System automatically detects whether a repository URL is GitHub or GitLab based on the domain.

* **[MVP] GitLab API v4 Integration**: Backend implements GitLab REST API v4 endpoints for all required operations (tree, blob, branches).

* **[MVP] Token Validation**: System validates GitLab token access and permissions before attempting operations.

* **[MVP] Error Handling**: Clear, actionable error messages for GitLab-specific failures (invalid token, insufficient permissions, API rate limits, self-hosted connectivity issues).

* **[Non-MVP] Repository Provider Abstraction**: Refactor existing GitHub-specific code into a provider abstraction layer to cleanly support multiple Git hosting platforms.

**Done - Acceptance Criteria:**

* **AC1**: A user can configure a vTeam project with a GitLab.com repository URL, and the system correctly identifies it as a GitLab repository.

* **AC2**: A user can configure a vTeam project with a self-hosted GitLab repository URL (e.g., `https://gitlab.internal.company.com/team/repo.git`), and the system correctly identifies and connects to the self-hosted instance.

* **AC3**: A user can store a GitLab Personal Access Token in the project's runner secrets, and the system uses this token for authentication.

* **AC4**: Through the vTeam API, a user can list all branches in a GitLab repository and receive accurate results.

* **AC5**: Through the vTeam API, a user can browse the directory structure (tree) of a GitLab repository at a specific branch and receive accurate results.

* **AC6**: Through the vTeam API, a user can read file contents (blob) from a GitLab repository at a specific branch/path and receive accurate results.

* **AC7**: An AgenticSession can successfully clone a GitLab repository using token authentication.

* **AC8**: An AgenticSession can create commits and push changes to a GitLab repository, and those changes appear correctly in the GitLab UI.

* **AC9**: A user can use both GitHub and GitLab repositories within the same vTeam project (e.g., main repo on GitHub, supporting repo on GitLab).

* **AC10**: All existing GitHub functionality continues to work without degradation or breaking changes after GitLab support is added.

* **AC11**: When GitLab API calls fail (invalid token, rate limits, network issues), users receive clear error messages indicating the problem and suggested remediation steps.

* **AC12**: The system handles GitLab's API pagination correctly when browsing large repositories or branch lists.

**Use Cases - i.e. User Experience & Workflow:**

**Use Case 1: Configure Project with GitLab Repository**

**Main Success Scenario:**
1. User creates a new vTeam project
2. User enters GitLab repository URL: `https://gitlab.com/mycompany/myapp.git`
3. User adds GitLab Personal Access Token to project runner secrets with key `GIT_TOKEN`
4. System validates token has required scopes (read_repository, write_repository)
5. Project is successfully configured and ready for AgenticSessions

**Alternative Flow - Self-Hosted GitLab:**
1. User creates a new vTeam project
2. User enters self-hosted GitLab URL: `https://gitlab.internal.company.com/platform/service.git`
3. User adds GitLab PAT to runner secrets
4. System detects self-hosted instance and constructs API URL: `https://gitlab.internal.company.com/api/v4`
5. System validates connectivity and token
6. Project configured successfully

**Use Case 2: Browse GitLab Repository Contents**

**Main Success Scenario:**
1. User opens project settings or repository browser
2. User selects "Browse Repository" for GitLab-backed project
3. System calls GitLab API `/projects/:id/repository/tree?ref=main`
4. UI displays directory/file structure
5. User clicks on a file
6. System calls GitLab API `/projects/:id/repository/files/:file_path/raw?ref=main`
7. UI displays file contents

**Use Case 3: AgenticSession with GitLab Repository**

**Main Success Scenario:**
1. User creates AgenticSession with task: "Add error handling to authentication module"
2. System retrieves GitLab token from project runner secrets
3. Claude Code runner pod clones GitLab repository using: `git clone https://oauth2:TOKEN@gitlab.com/owner/repo.git`
4. AI agent analyzes code and makes changes
5. Agent commits changes: `git commit -m "Add error handling"`
6. Agent pushes to branch: `git push origin feature-branch`
7. Changes appear in GitLab repository
8. User receives session completion notification with link to GitLab branch

**Alternative Flow - Permission Error:**
1. Steps 1-3 as above
2. Agent attempts to push changes
3. GitLab returns 403 Forbidden (token lacks write permission)
4. System returns error: "GitLab push failed: Token does not have write_repository permission. Please update your GitLab PAT to include 'write_repository' scope."
5. User updates token in runner secrets
6. User retries AgenticSession

**Use Case 4: Mixed GitHub/GitLab Project**

**Main Success Scenario:**
1. User configures project with primary repo on GitHub
2. User adds supporting repository on GitLab (different team/organization)
3. System detects provider for each repository based on URL
4. AgenticSession clones both repositories using appropriate tokens
5. Agent makes changes across both repos
6. System pushes to GitHub using GitHub App token
7. System pushes to GitLab using PAT from runner secrets
8. Both repositories updated successfully

**Use Case 5: Repository Seeding with GitLab**

**Main Success Scenario:**
1. User creates new project with GitLab repository that lacks required structure
2. System detects missing `.claude/agents/` directory
3. User triggers repository seeding
4. System clones GitLab repo
5. System copies spec-kit templates and agent configurations
6. System commits and pushes seeding changes to GitLab
7. Repository now ready for AgenticSessions

**Documentation Considerations:**

* **New Documentation Required:**
  - GitLab Integration Setup Guide (how to create Personal Access Token, required scopes)
  - Self-Hosted GitLab Configuration Guide (network requirements, API URL construction, SSL certificate considerations)
  - Token Storage Best Practices (using runner secrets, token rotation recommendations)
  - GitLab API Rate Limits and Troubleshooting

* **Existing Documentation to Update:**
  - Project Configuration Guide: Add GitLab as alternative to GitHub
  - Repository Requirements: Note provider-specific differences
  - Authentication Guide: Add GitLab token section alongside GitHub App
  - Troubleshooting Guide: Add GitLab-specific error scenarios

* **API Documentation:**
  - Document GitLab URL format requirements
  - Document expected GitLab token scopes: `read_api`, `read_repository`, `write_repository`
  - Document self-hosted GitLab endpoint detection logic

* **Related Existing Documentation:**
  - GitHub Integration: https://github.com/natifridman/vTeam (existing implementation to reference)

**Questions to answer:**

* **Q1 - Architecture**: Should we implement a repository provider abstraction layer to cleanly separate GitHub and GitLab logic, or keep them as separate parallel implementations?
  - **Option A**: Create `RepositoryProvider` interface with `GitHubProvider` and `GitLabProvider` implementations
  - **Option B**: Duplicate/fork existing GitHub code into separate GitLab package
  - **Recommendation**: Provider abstraction for maintainability and future extensibility

* **Q2 - Token Storage**: Should GitLab tokens be stored in Kubernetes Secrets instead of ConfigMaps for better security?
  - **Current State**: GitHub App installations stored in ConfigMap `github-app-installations`
  - **Consideration**: GitLab PATs are more sensitive than GitHub installation IDs
  - **Recommendation**: Use Secrets for GitLab tokens, keep ConfigMap for GitHub App metadata

* **Q3 - Provider Detection**: How should the system detect Git provider from URLs?
  - **Option A**: Simple domain matching (`github.com` vs `gitlab.com` vs self-hosted)
  - **Option B**: API probe (try GitHub API, fallback to GitLab)
  - **Option C**: Explicit user selection in UI
  - **Recommendation**: Domain matching with optional explicit override

* **Q4 - API Client**: Should we use GitLab's official Go SDK or implement direct REST API calls?
  - **Current State**: GitHub integration uses direct `net/http` calls, no SDK
  - **Trade-offs**: SDK adds dependency but provides type safety and API evolution handling
  - **Recommendation**: Direct REST calls for consistency with existing GitHub implementation

* **Q5 - Token Scopes**: What minimum GitLab token scopes are required for MVP?
  - **Read Operations**: `read_api` + `read_repository`
  - **Write Operations**: `write_repository`
  - **Question**: Should we support `api` (full access) or enforce minimal scopes?
  - **Recommendation**: Document minimal scopes, but accept `api` for simplicity

* **Q6 - Self-Hosted Versioning**: What minimum GitLab version should be supported for self-hosted instances?
  - **Consideration**: GitLab API v4 introduced in GitLab 9.0 (2017)
  - **Recommendation**: Support GitLab 12.0+ (released 2019, still widely deployed)

* **Q7 - URL Normalization**: How should the system handle various GitLab URL formats?
  - **Examples**: `.git` suffix optional, HTTPS vs SSH, trailing slashes
  - **Recommendation**: Implement robust URL parser that normalizes all variants

* **Q8 - Error Handling Strategy**: How should GitLab API errors be surfaced to users?
  - **Current State**: GitHub errors sometimes cryptic
  - **Recommendation**: Map common GitLab API errors (401, 403, 404, 429) to user-friendly messages

* **Q9 - Concurrent Provider Support**: How should the system handle operations when project has both GitHub and GitLab repos?
  - **Consideration**: Token resolution, error handling, UI feedback
  - **Recommendation**: Independent token resolution per repo, aggregate error reporting

* **Q10 - Migration Path**: Should there be a tool to help users migrate existing GitHub projects to GitLab?
  - **Scope**: Out of MVP, but consider in architecture decisions
  - **Recommendation**: Design APIs to support future migration tooling

**Background & Strategic Fit:**

The Ambient Code Platform (vTeam) currently supports only GitHub repositories, creating a significant barrier for organizations and developers who standardize on GitLab for source control management. GitLab has substantial market presence, particularly in:

* **Enterprise self-hosted deployments**: Many large organizations run self-hosted GitLab for security, compliance, and control reasons
* **European markets**: GitLab is particularly popular in European enterprises
* **DevOps-first organizations**: GitLab's integrated CI/CD appeals to platform engineering teams

**Strategic Rationale:**

1. **Market Expansion**: Unlocks the GitLab user base (estimated 30M+ users) as potential Ambient Code Platform users
2. **Customer Requests**: Addresses specific customer demands for GitLab support
3. **Competitive Parity**: Competing AI coding platforms support multiple Git providers
4. **Platform Maturity**: Demonstrates platform flexibility and multi-provider architecture
5. **Future Extensibility**: Establishes pattern for adding additional providers (Bitbucket, Azure DevOps)

**Technical Strategic Fit:**

* Aligns with Kubernetes-native architecture (provider-agnostic storage, configuration-driven)
* Leverages existing Git operations foundation (`git/operations.go`)
* Minimal changes to core AgenticSession/CRD architecture
* Natural extension point: Backend handler layer already abstracts repository operations

**Timing Considerations:**

* Low risk: Additive feature, no breaking changes to existing GitHub functionality
* High impact: Removes adoption blocker for significant user segment
* Moderate effort: Well-scoped MVP, clear requirements, existing patterns to follow

**Customer Considerations:**

* **Enterprise Self-Hosted GitLab Users**:
  - Require network connectivity from Kubernetes cluster to self-hosted GitLab instance
  - May need custom CA certificates for SSL/TLS validation
  - May have API rate limits configured differently than GitLab.com
  - Often have strict token management policies (rotation, least privilege)
  - Documentation must address firewall rules, network policies, security scanning concerns

* **GitLab.com SaaS Users**:
  - Subject to GitLab.com rate limits (300 requests/minute for authenticated users)
  - May use GitLab Groups for organization-wide projects
  - Expect OAuth2 flows similar to other SaaS integrations (defer to post-MVP)

* **Mixed Environment Users** (both GitHub and GitLab):
  - Need clear UI affordances to distinguish repository providers
  - May have different authentication mechanisms per provider
  - Expect consistent UX across providers despite API differences

* **Security-Conscious Organizations**:
  - Require secure token storage (Kubernetes Secrets, not ConfigMaps)
  - Need token scope documentation and validation
  - May require audit logging for Git operations
  - Expect least-privilege access patterns

* **Migration Scenarios**:
  - Organizations migrating from GitHub to GitLab need both providers functional during transition
  - Existing projects must continue working without modification
  - Backward compatibility critical for enterprise adoption

* **Compliance Requirements**:
  - Self-hosted GitLab often used for compliance reasons (GDPR, SOC2, FedRAMP)
  - Platform must not introduce new compliance risks
  - Token handling must meet enterprise security standards

* **Support Expectations**:
  - Clear troubleshooting documentation for self-hosted connectivity issues
  - GitLab API version compatibility matrix
  - Example configurations for common enterprise setups
  - Migration guides from GitHub-only to multi-provider configurations
