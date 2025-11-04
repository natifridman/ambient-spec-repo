# Implementation Plan: GitLab Support for vTeam

**Branch**: `001-gitlab-support` | **Date**: 2025-11-04 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-gitlab-support/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

This feature adds GitLab repository support to the vTeam Ambient Code Platform, enabling users to configure GitLab-hosted repositories (GitLab.com and self-hosted instances), authenticate using Personal Access Tokens, browse repository contents, and execute AgenticSessions that can clone, modify, commit, and push code changes to GitLab repositories. The implementation will mirror the existing GitHub integration pattern while introducing GitLab-specific URL parsing, API client implementation, and token-based authentication instead of GitHub App-based auth.

## Technical Context

**Language/Version**: Go 1.24.0
**Primary Dependencies**: Gin (web framework), Kubernetes client-go v0.34.0, gorilla/websocket, golang-jwt
**Storage**: Kubernetes Secrets (for PAT tokens), Kubernetes CRDs (for vTeam project configuration)
**Testing**: Go testing framework (70% minimum coverage for new code, 90% for critical paths)
**Target Platform**: Kubernetes cluster (Linux containers)
**Project Type**: Web application (Go backend + frontend)
**Performance Goals**: API response time <3s for repository browsing, <200ms for token validation, support 10,000+ file repositories
**Constraints**: Must not degrade existing GitHub performance, maintain backward compatibility
**Scale/Scope**: Support 100+ concurrent AgenticSessions, handle repositories with 10,000+ files and 500+ branches
**Existing Architecture**: Backend has separate packages for git/, github/, k8s/, handlers/, types/. GitHub integration uses app.go and token.go pattern. Extension points identified: new gitlab/ package, handlers/gitlab_auth.go, handlers/repo.go modifications, git/operations.go provider abstraction

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Initial Status (Pre-Phase 0)**: PASSED
**Post-Phase 1 Status**: PASSED ✅

Since the project constitution is a template and not yet defined, this feature follows general engineering principles:

- **Backward Compatibility**: ✅ All existing GitHub functionality continues to work. No breaking changes to existing APIs, CRDs, or user workflows. Design validates this through parallel package structure.

- **Code Organization**: ✅ Follows existing package structure (github/ → gitlab/ parallel pattern). New gitlab/ package mirrors github/ organization with client.go, token.go, parser.go. Shared abstractions in git/ package.

- **Testing Requirements**: ✅ Comprehensive testing strategy defined in research.md:
  - 70% minimum coverage for new GitLab code
  - 90% coverage for critical paths (URL parsing, token validation, error handling)
  - Unit, contract, and integration tests specified
  - GitHub regression tests required on every PR

- **Security**: ✅ Security-first approach validated:
  - Tokens stored exclusively in Kubernetes Secrets (never ConfigMaps)
  - Token redaction in all logs and error messages ([REDACTED] placeholder)
  - No plaintext credentials in API responses
  - Error messages don't expose internal system details

- **Observability**: ✅ Error handling design meets SC-006 (90% user-friendly errors):
  - Structured GitLabAPIError type with Message and Remediation fields
  - HTTP status code mapping defined for all common errors (401, 403, 404, 429, 500)
  - User-actionable guidance in all error messages
  - Request IDs for debugging without exposing sensitive data

**Post-Design Assessment**:
- No architectural complexity introduced beyond necessary provider abstraction
- Design patterns align with existing codebase conventions
- All success criteria measurable and testable
- Performance constraints addressed (pagination for 10k+ files, rate limiting)

**No violations requiring justification.**

## Project Structure

### Documentation (this feature)

```
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root: /workspace/sessions/agentic-session-1762276850/workspace/vTeam)

```
components/
├── backend/                 # Go backend API server
│   ├── gitlab/             # NEW: GitLab integration package (parallel to github/)
│   │   ├── client.go       # GitLab API client
│   │   ├── token.go        # Token validation and auth
│   │   └── parser.go       # URL parsing and normalization
│   ├── github/             # EXISTING: GitHub integration (reference pattern)
│   │   ├── app.go          # GitHub App integration
│   │   └── token.go        # GitHub token management
│   ├── git/                # EXISTING: Generic Git operations (shared by both)
│   ├── handlers/           # MODIFY: Add GitLab-aware handlers
│   ├── types/              # MODIFY: Add GitLab-specific types/structs
│   ├── k8s/                # MODIFY: Handle GitLab PAT secrets
│   └── main.go             # MODIFY: Register GitLab routes
├── frontend/               # EXISTING: React frontend (no changes in MVP)
├── runners/                # EXISTING: AgenticSession execution environment
└── manifests/              # MODIFY: CRD updates for GitLab fields

tests/                       # NEW: Test structure to be defined in Phase 0
├── integration/
│   └── gitlab/             # End-to-end GitLab integration tests
└── unit/
    └── gitlab/             # Unit tests for GitLab package
```

**Structure Decision**: Web application (backend + frontend). This feature primarily adds a new `gitlab/` package in the backend component, following the existing `github/` package pattern. The git/ package provides shared Git operation functionality for both providers.

## Complexity Tracking

*Fill ONLY if Constitution Check has violations that must be justified*

**No complexity violations identified.**

The GitLab integration follows established patterns from the existing GitHub implementation and does not introduce architectural complexity beyond the necessary provider abstraction.

