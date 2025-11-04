# Implementation Tasks: GitLab Support for vTeam

**Feature Branch**: `001-gitlab-support`
**Date**: 2025-11-04
**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

---

## Overview

This document provides an actionable, dependency-ordered task breakdown for implementing GitLab support in vTeam. Tasks are organized by user story to enable independent implementation and testing.

**Total Tasks**: 67
**MVP Scope**: User Story 1 (Project Configuration) - 22 tasks
**Parallel Opportunities**: 45+ tasks can run in parallel within phases

---

## Implementation Strategy

### MVP First (Recommended)
Start with **User Story 1 only** (tasks T001-T022) to deliver immediate value:
- Users can configure GitLab repositories
- Token validation works
- Foundation for all other stories

### Incremental Delivery
After MVP, implement in priority order:
1. **User Story 3** (AgenticSession) - Core value proposition
2. **User Story 2** (Repository browsing) - Nice-to-have for troubleshooting
3. **User Story 4** (Mixed providers) - Advanced use case
4. **User Story 5** (Repository seeding) - Convenience feature

---

## User Story Completion Order

```
Phase 1: Setup (blocking)
   ↓
Phase 2: Foundational (blocking)
   ↓
User Story 1 (P1) ────┐
                      ├──→ User Story 3 (P1)
                      │
                      └──→ User Story 2 (P2)
                            ↓
                      User Story 4 (P3)
                            ↓
                      User Story 5 (P3)
```

**Dependencies:**
- **User Story 1** blocks all others (must configure GitLab before using it)
- **User Story 3** can run in parallel with User Story 2 after US1
- **User Story 4** requires US1 + US3 (needs both providers working)
- **User Story 5** requires US1 + US3 (needs clone/push functionality)

---

## Phase 1: Setup & Infrastructure

**Goal**: Initialize GitLab integration package structure and dependencies

**Tasks** (6):

- [X] T001 Create gitlab package directory structure in components/backend/gitlab/
- [X] T002 Add GitLab package imports to components/backend/main.go
- [X] T003 [P] Create types for GitLabConnection in components/backend/types/gitlab.go
- [X] T004 [P] Create types for GitLabRepository in components/backend/types/gitlab.go
- [X] T005 [P] Create types for GitLabAPIError in components/backend/types/gitlab.go
- [X] T006 [P] Create ProviderType enum in components/backend/types/provider.go

---

## Phase 2: Foundational Layer

**Goal**: Implement shared utilities needed by all user stories

**Tasks** (10):

- [X] T007 Implement URL parser for GitLab repositories in components/backend/gitlab/parser.go
- [X] T008 Add URL normalization logic (HTTPS/SSH, .git suffix) in components/backend/gitlab/parser.go
- [X] T009 Add self-hosted instance detection in components/backend/gitlab/parser.go
- [X] T010 Add API URL construction logic in components/backend/gitlab/parser.go
- [X] T011 [P] Create GitLab HTTP client with 15-second timeout in components/backend/gitlab/client.go
- [X] T012 [P] Add error response parsing and GitLabAPIError mapping in components/backend/gitlab/client.go
- [X] T013 [P] Add provider detection function in components/backend/types/provider.go
- [X] T014 [P] Create Kubernetes Secret helper for PAT storage in components/backend/k8s/secrets.go
- [X] T015 [P] Add logging utilities with token redaction in components/backend/gitlab/logger.go
- [X] T016 [P] Update ProjectSettings CRD with optional provider field in components/manifests/crds/projectsettings.yaml

---

## Phase 3: User Story 1 - Configure vTeam Project with GitLab Repository (P1)

**Goal**: Enable users to configure GitLab repositories and validate PAT tokens

**Story Priority**: P1 (MVP - blocks all other stories)

**Independent Test**: Create vTeam project with GitLab URL `https://gitlab.com/owner/repo.git`, add PAT to runner secrets, verify successful validation and configuration display.

**Acceptance Criteria**:
1. GitLab.com repositories can be configured
2. Self-hosted GitLab instances are detected and API URLs constructed correctly
3. Invalid tokens show clear error messages
4. Insufficient permissions are detected via test API call

**Tasks** (16):

### Token Validation & Authentication

- [ ] T017 [US1] Implement ValidateGitLabToken function in components/backend/gitlab/token.go
- [ ] T018 [US1] Add GitLab /user API call with Bearer token auth in components/backend/gitlab/token.go
- [ ] T019 [US1] Add token validation error handling (401, 403, 404) in components/backend/gitlab/token.go
- [ ] T020 [US1] Add test API call validation (repository access check) in components/backend/gitlab/token.go

### Connection Management

- [ ] T021 [P] [US1] Create ConfigMap helper for gitlab-connections in components/backend/k8s/configmap.go
- [ ] T022 [P] [US1] Implement StoreGitLabConnection function in components/backend/gitlab/connection.go
- [ ] T023 [P] [US1] Implement GetGitLabConnection function in components/backend/gitlab/connection.go
- [ ] T024 [US1] Add connection metadata storage (userID, gitlabUserID, instanceURL) in components/backend/gitlab/connection.go

### API Endpoints

- [ ] T025 [P] [US1] Create POST /auth/gitlab/connect handler in components/backend/handlers/gitlab_auth.go
- [ ] T026 [P] [US1] Create GET /auth/gitlab/status handler in components/backend/handlers/gitlab_auth.go
- [ ] T027 [P] [US1] Create POST /auth/gitlab/disconnect handler in components/backend/handlers/gitlab_auth.go
- [ ] T028 [US1] Register GitLab auth routes in components/backend/routes.go

### Project Configuration

- [ ] T029 [P] [US1] Update project creation handler to support GitLab URLs in components/backend/handlers/projects.go
- [ ] T030 [P] [US1] Add provider detection in project creation flow in components/backend/handlers/projects.go
- [ ] T031 [US1] Add GitLab repository validation on project save in components/backend/handlers/projects.go
- [ ] T032 [US1] Update project settings API to return provider information in components/backend/handlers/projects.go

**Parallel Execution Example**:
```bash
# After T016 completes, run these in parallel:
- T017-T020 (token validation)
- T021-T024 (connection management)
- T025-T028 (API endpoints)
- T029-T032 (project configuration)
```

---

## Phase 4: User Story 3 - Execute AgenticSession with GitLab Repository (P1)

**Goal**: Enable AgenticSessions to clone, commit, and push to GitLab repositories

**Story Priority**: P1 (Core value proposition)

**Depends On**: User Story 1 (must configure GitLab before using AgenticSessions)

**Independent Test**: Create AgenticSession with task "add comment to README", verify session clones GitLab repo, makes changes, commits, and pushes successfully. Check changes visible in GitLab UI.

**Acceptance Criteria**:
1. AgenticSessions can clone GitLab repositories with token auth
2. Commits are created successfully in local repository
3. Pushes to GitLab succeed and changes appear in GitLab UI
4. Token permission errors (403) show clear guidance
5. Self-hosted GitLab URLs are constructed correctly
6. Completion notifications include GitLab branch link

**Tasks** (14):

### Git Operations Integration

- [ ] T033 [P] [US3] Add GitLab token retrieval in components/backend/git/operations.go
- [ ] T034 [P] [US3] Implement token injection for GitLab URLs (oauth2:TOKEN@) in components/backend/git/operations.go
- [ ] T035 [US3] Add provider routing logic (GitHub vs GitLab) in components/backend/git/operations.go
- [ ] T036 [US3] Update clone operation to support GitLab in components/backend/git/operations.go

### Runner Pod Configuration

- [ ] T037 [P] [US3] Update AgenticSession operator to inject GitLab PAT from Secrets in components/operator/internal/handlers/sessions.go
- [ ] T038 [P] [US3] Add EnvFrom configuration for runner-secrets in components/operator/internal/handlers/sessions.go
- [ ] T039 [P] [US3] Add volume mount for /var/run/runner-secrets/ in components/operator/internal/handlers/sessions.go
- [ ] T040 [US3] Update security context (drop all capabilities) in components/operator/internal/handlers/sessions.go

### Push Operations & Error Handling

- [ ] T041 [P] [US3] Implement push error detection (403 Forbidden) in components/backend/git/operations.go
- [ ] T042 [P] [US3] Add user-friendly error messages for permission failures in components/backend/git/operations.go
- [ ] T043 [US3] Add self-hosted GitLab URL construction for push operations in components/backend/git/operations.go

### Completion Notifications

- [ ] T044 [P] [US3] Add GitLab branch URL construction in components/backend/handlers/sessions.go
- [ ] T045 [P] [US3] Update session completion notification with GitLab link in components/backend/handlers/sessions.go
- [ ] T046 [US3] Add provider-specific notification templates in components/backend/handlers/sessions.go

**Parallel Execution Example**:
```bash
# After US1 completes, run these in parallel:
- T033-T036 (git operations)
- T037-T040 (runner pod config)
- T041-T043 (push operations)
- T044-T046 (notifications)
```

---

## Phase 5: User Story 2 - Browse GitLab Repository Contents (P2)

**Goal**: Enable repository browsing (branches, files, directories) through vTeam API

**Story Priority**: P2 (Valuable for troubleshooting but not blocking)

**Depends On**: User Story 1 (must configure GitLab before browsing)

**Independent Test**: Configure GitLab project, use vTeam API to list branches, browse directory tree, read file contents. Verify accurate results and pagination for large repos.

**Acceptance Criteria**:
1. List branches returns all branches with accurate metadata
2. Directory tree browsing works for root and subdirectories
3. File contents are retrieved accurately
4. Pagination handles repositories with 10,000+ files
5. Rate limit errors (429) return GitLab's response body

**Tasks** (15):

### GitLab API Client Methods

- [ ] T047 [P] [US2] Implement GetBranches with pagination in components/backend/gitlab/client.go
- [ ] T048 [P] [US2] Implement GetTree with pagination in components/backend/gitlab/client.go
- [ ] T049 [P] [US2] Implement GetFileContents in components/backend/gitlab/client.go
- [ ] T050 [P] [US2] Add pagination helper (handle X-Next-Page headers) in components/backend/gitlab/client.go
- [ ] T051 [P] [US2] Add rate limit error handling (429) in components/backend/gitlab/client.go

### API Endpoints

- [ ] T052 [P] [US2] Create GET /projects/:project/repo/branches handler in components/backend/handlers/repo.go
- [ ] T053 [P] [US2] Create GET /projects/:project/repo/tree handler in components/backend/handlers/repo.go
- [ ] T054 [P] [US2] Create GET /projects/:project/repo/blob handler in components/backend/handlers/repo.go
- [ ] T055 [US2] Add provider detection and routing in repo handlers in components/backend/handlers/repo.go

### Response Mapping

- [ ] T056 [P] [US2] Map GitLabBranch to common Branch type in components/backend/types/repository.go
- [ ] T057 [P] [US2] Map GitLabTreeEntry to common TreeEntry type in components/backend/types/repository.go
- [ ] T058 [P] [US2] Add project ID URL encoding for GitLab API in components/backend/gitlab/parser.go

### Error Handling

- [ ] T059 [P] [US2] Add specific error messages for browsing failures in components/backend/gitlab/client.go
- [ ] T060 [P] [US2] Add remediation guidance for common browsing errors in components/backend/gitlab/client.go
- [ ] T061 [US2] Register repository browsing routes in components/backend/routes.go

**Parallel Execution Example**:
```bash
# After US1 completes, run these in parallel with US3:
- T047-T051 (GitLab API methods)
- T052-T055 (API endpoints)
- T056-T058 (response mapping)
- T059-T061 (error handling)
```

---

## Phase 6: User Story 4 - Use Mixed GitHub and GitLab Repositories (P3)

**Goal**: Support projects with both GitHub and GitLab repositories simultaneously

**Story Priority**: P3 (Advanced use case for multi-provider organizations)

**Depends On**: User Story 1 + User Story 3 (both providers must work independently)

**Independent Test**: Configure project with both GitHub and GitLab repositories, create AgenticSession that clones both, verify correct authentication for each provider.

**Acceptance Criteria**:
1. Provider is correctly identified for each repository URL
2. Appropriate authentication method used (GitHub App vs GitLab PAT)
3. Both repositories update successfully without interference
4. Provider-specific errors are clearly indicated

**Tasks** (9):

### Multi-Provider Support

- [ ] T062 [P] [US4] Add support for mixed repository arrays in ProjectSettings in components/backend/types/project.go
- [ ] T063 [P] [US4] Update provider detection to handle multiple repos in components/backend/handlers/projects.go
- [ ] T064 [US4] Add per-repository token routing logic in components/backend/git/operations.go

### AgenticSession Updates

- [ ] T065 [P] [US4] Update session clone logic to handle multiple providers in components/backend/git/operations.go
- [ ] T066 [P] [US4] Add provider-specific error aggregation in components/backend/handlers/sessions.go
- [ ] T067 [US4] Update session status reporting with per-provider results in components/backend/handlers/sessions.go

### Error Handling

- [ ] T068 [P] [US4] Add mixed-provider error messages in components/backend/types/errors.go
- [ ] T069 [P] [US4] Add provider failure indication in session results in components/backend/handlers/sessions.go
- [ ] T070 [US4] Add provider-specific remediation guidance in components/backend/types/errors.go

**Parallel Execution Example**:
```bash
# After US1 + US3 complete, run these in parallel:
- T062-T064 (multi-provider support)
- T065-T067 (session updates)
- T068-T070 (error handling)
```

---

## Phase 7: User Story 5 - Seed GitLab Repository with Required Structure (P3)

**Goal**: Automatically seed GitLab repositories with .claude/ directory structure

**Story Priority**: P3 (Convenience feature for onboarding)

**Depends On**: User Story 1 + User Story 3 (needs clone and push functionality)

**Independent Test**: Configure project with GitLab repository missing `.claude/` directories, trigger seeding, verify system clones, adds files, commits, and pushes structure to GitLab.

**Acceptance Criteria**:
1. Missing structure is detected
2. User is offered to seed repository
3. Seeding clones, copies templates, commits, and pushes to GitLab
4. All required directories and files are present after seeding
5. Permission errors provide clear guidance

**Tasks** (8):

### Repository Seeding Logic

- [ ] T071 [P] [US5] Implement DetectMissingStructure function in components/backend/handlers/repo_seed.go
- [ ] T072 [P] [US5] Implement SeedRepository function in components/backend/handlers/repo_seed.go
- [ ] T073 [US5] Add template copy logic for .claude/ structure in components/backend/handlers/repo_seed.go

### API Endpoints

- [ ] T074 [P] [US5] Create POST /projects/:project/repo/seed handler in components/backend/handlers/repo_seed.go
- [ ] T075 [P] [US5] Create GET /projects/:project/repo/seed-status handler in components/backend/handlers/repo_seed.go
- [ ] T076 [US5] Register repository seeding routes in components/backend/routes.go

### Error Handling

- [ ] T077 [P] [US5] Add seeding error messages with permission guidance in components/backend/handlers/repo_seed.go
- [ ] T078 [US5] Add seeding progress tracking and status updates in components/backend/handlers/repo_seed.go

**Parallel Execution Example**:
```bash
# After US1 + US3 complete, run these in parallel with US4:
- T071-T073 (seeding logic)
- T074-T076 (API endpoints)
- T077-T078 (error handling)
```

---

## Phase 8: Polish & Cross-Cutting Concerns

**Goal**: Add logging, monitoring, documentation, and final integration

**Tasks** (11):

### Logging & Observability

- [ ] T079 [P] Add standardized logging for all GitLab API calls in components/backend/gitlab/client.go
- [ ] T080 [P] Add token redaction in all log statements in components/backend/gitlab/logger.go
- [ ] T081 [P] Add request ID tracking for debugging in components/backend/gitlab/client.go

### Documentation

- [ ] T082 [P] Create GitLab integration user guide in docs/gitlab-integration.md
- [ ] T083 [P] Create GitLab PAT setup instructions in docs/gitlab-token-setup.md
- [ ] T084 [P] Create self-hosted GitLab configuration guide in docs/gitlab-self-hosted.md
- [ ] T085 [P] Update main README with GitLab support announcement in README.md

### Testing

- [ ] T086 [P] Run GitHub regression tests to verify no degradation in tests/integration/github/
- [ ] T087 [P] Verify backward compatibility with existing projects in tests/integration/backward_compat/
- [ ] T088 Create end-to-end integration test for GitLab flow in tests/integration/gitlab/

### Final Integration

- [ ] T089 Update API documentation with GitLab endpoints in docs/api/openapi.yaml

**Parallel Execution Example**:
```bash
# After all user stories complete, run these in parallel:
- T079-T081 (logging)
- T082-T085 (documentation)
- T086-T088 (testing)
- T089 (API docs)
```

---

## Task Summary

### By Phase

| Phase | Tasks | Can Run in Parallel |
|-------|-------|---------------------|
| Phase 1: Setup | 6 | 4 |
| Phase 2: Foundational | 10 | 7 |
| Phase 3: User Story 1 (P1) | 16 | 13 |
| Phase 4: User Story 3 (P1) | 14 | 11 |
| Phase 5: User Story 2 (P2) | 15 | 13 |
| Phase 6: User Story 4 (P3) | 9 | 7 |
| Phase 7: User Story 5 (P3) | 8 | 6 |
| Phase 8: Polish | 11 | 9 |
| **Total** | **89** | **70** |

### By User Story

| User Story | Priority | Tasks | Dependencies |
|------------|----------|-------|--------------|
| US1: Configure GitLab | P1 | 16 | Phase 1 + 2 |
| US2: Browse Repository | P2 | 15 | US1 |
| US3: AgenticSession | P1 | 14 | US1 |
| US4: Mixed Providers | P3 | 9 | US1 + US3 |
| US5: Repository Seeding | P3 | 8 | US1 + US3 |

### MVP Scope (Recommended First Iteration)

**Tasks T001-T032** (38 tasks total)
- Phase 1: Setup (6 tasks)
- Phase 2: Foundational (10 tasks)
- Phase 3: User Story 1 (16 tasks)
- Selected Phase 8: Logging + docs (6 tasks)

**Estimated effort**: 1-2 weeks for experienced Go developer

**Deliverable**: Users can configure and validate GitLab repositories in vTeam projects

---

## Validation Checklist

Before marking this feature complete, verify:

- [ ] All 5 user stories have passing independent tests
- [ ] GitHub regression tests pass (zero degradation)
- [ ] Backward compatibility verified with existing projects
- [ ] Documentation complete and accurate
- [ ] Error messages meet SC-006 (90% user-friendly)
- [ ] Performance meets SC-002 (<3s browsing, <200ms validation)
- [ ] Security review completed (token storage, redaction, error messages)
- [ ] All task IDs follow format `- [ ] T### [labels] Description with file path`

---

## Notes

**Task ID Format**: All tasks use strict checklist format with sequential IDs (T001-T089)

**Parallel Markers**: `[P]` indicates task can run in parallel with others in same phase

**Story Labels**: `[US1]`, `[US2]`, etc. map tasks to user stories from spec.md

**File Paths**: All tasks include specific file paths for implementation

**Independent Testing**: Each user story phase includes independent test criteria

**Incremental Value**: Each completed user story delivers measurable value independently
