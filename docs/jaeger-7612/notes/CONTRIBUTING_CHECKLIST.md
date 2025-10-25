# Jaeger Contributing Checklist

**Project:** Jaeger - Distributed Tracing System
**Research Branch:** research/jaeger-7612
**Purpose:** Align with Jaeger contribution conventions for issue #7612 investigation
**Last Updated:** 2025-10-25

---

## Quick Reference

All contributions to Jaeger must follow these conventions. Each item includes source links for verification.

**Critical Requirements:**
- ✅ **ALL commits MUST be signed** with DCO (`git commit -s`)
- ✅ Use a **named branch** (not `main`) in your fork
- ✅ Open an **issue before significant changes**
- ✅ Run `make fmt`, `make lint`, `make test` before submitting PR

---

## Pre-Contribution Phase

### 1. Open an Issue First

- [ ] **Before making significant changes, open an issue describing:**
  - Requirement: What business use case are you solving?
  - Problem: What blocks you from solving it?
  - Proposal: What changes do you propose?
  - Open questions to address

**Source:** [CONTRIBUTING_GUIDELINES.md#making-a-change](../../CONTRIBUTING_GUIDELINES.md#making-a-change)

**Note:** For issue #7612, the investigation issue already exists. Document findings there or in linked issues.

### 2. Understand Assignment Policy

- [ ] **Do NOT request to be assigned to an issue**
  - Maintainers do not assign issues to contributors
  - Simply comment that you're working on it and submit a PR
  - If priorities change, it's okay to step away

**Source:** [CONTRIBUTING_GUIDELINES.md#assigning-issues](../../CONTRIBUTING_GUIDELINES.md#assigning-issues)

---

## Repository Setup

### 3. Fork and Clone Setup

- [ ] **Fork the repository** on GitHub to your personal org
- [ ] **Clone your fork** locally
- [ ] **Add upstream remote** (recommended):
  ```bash
  git remote add upstream git@github.com:jaegertracing/jaeger.git
  git fetch upstream main
  git branch --set-upstream-to=upstream/main main
  ```

**Source:** [CONTRIBUTING_GUIDELINES.md#creating-a-pull-request](../../CONTRIBUTING_GUIDELINES.md#creating-a-pull-request)

**Benefit:** This setup prevents needing to sync your fork's main branch.

### 4. Install Prerequisites

- [ ] **Install Go** (version per go.mod, currently 1.24.6+)
- [ ] **Setup GOPATH** and add `$GOPATH/bin` to PATH
- [ ] **Install GNU sed** (macOS only):
  ```bash
  brew install gnu-sed
  ```
- [ ] **Initialize submodules**:
  ```bash
  git submodule update --init --recursive
  ```
- [ ] **Install required tools**:
  ```bash
  make install-tools
  ```

**Source:** [CONTRIBUTING.md#getting-started](../../CONTRIBUTING.md#getting-started)

**Critical:** `make install-tools` includes `gofumpt`, `golangci-lint`, and other required tools.

---

## Development Workflow

### 5. Create Feature Branch

- [ ] **Create a named branch** (NEVER use `main`):
  ```bash
  git checkout -b feature/issue-7612-driver-research
  ```
- [ ] **Use descriptive branch names** (e.g., `fix/issue-1234`, `feature/add-opensearch`)

**Source:** [CONTRIBUTING_GUIDELINES.md#creating-a-pull-request](../../CONTRIBUTING_GUIDELINES.md#creating-a-pull-request)

**Warning:** PRs from `main` branch will fail CI checks and cannot be merged.

### 6. Code Quality Standards

- [ ] **Follow imports grouping** (3 groups):
  1. Standard library
  2. Other projects
  3. Jaeger project

  ```go
  import (
      "fmt"

      "go.uber.org/zap"

      "github.com/jaegertracing/jaeger/cmd/agent/app"
  )
  ```

**Source:** [CONTRIBUTING.md#imports-grouping](../../CONTRIBUTING.md#imports-grouping)

- [ ] **Configure IDE for gofumpt**:
  - VSCode example:
    ```json
    "go.formatTool": "gofumpt",
    "gopls": {
        "formatting.gofumpt": true
    }
    ```

**Source:** [CONTRIBUTING.md#auto-format](../../CONTRIBUTING.md#auto-format)

- [ ] **Add license headers** to new files:
  ```go
  // Copyright (c) 2025 The Jaeger Authors.
  // SPDX-License-Identifier: Apache-2.0
  ```

**Source:** [CONTRIBUTING_GUIDELINES.md#license](../../CONTRIBUTING_GUIDELINES.md#license)

**Note:** Use current year and "The Jaeger Authors" as copyright holder.

### 7. Testing Requirements

- [ ] **Maintain 95% code coverage** (repo-wide target)
- [ ] **Add `*_test.go` file** to all packages (even if empty):
  - If no tests possible, create `empty_test.go`
  - Add `.nocover` file only if external dependencies required (with comment explaining why)

**Source:** [CONTRIBUTING.md#testing-guidelines](../../CONTRIBUTING.md#testing-guidelines)

- [ ] **Run test suite before submitting**:
  ```bash
  make test
  ```

**Warning:** `make nocover` will fail if packages lack test files.

---

## Commit Conventions

### 8. Developer Certificate of Origin (DCO)

- [ ] **Sign EVERY commit** with DCO:
  ```bash
  git commit -s -m "Your commit message"
  ```

**Source:** [CONTRIBUTING_GUIDELINES.md#certificate-of-origin---sign-your-work](../../CONTRIBUTING_GUIDELINES.md#certificate-of-origin---sign-your-work)
**Reference:** [DCO](../../DCO)

**Critical:** The `-s` flag adds `Signed-off-by: Your Name <your.email@example.com>` automatically.

**What DCO Certifies:**
- (a) You created the contribution and have the right to submit it
- (b) Contribution is based on compatible open source work
- (c) Contribution was provided to you by someone who certified (a) or (b)
- (d) You understand the contribution is public and permanently recorded

**Enforcement:** DCO-bot checks all commits in PRs. Missing signatures will block merges.

### 9. Fixing Missing Sign-offs

**If latest commit is missing sign-off:**
```bash
git commit --amend -s
git push --force
```

**If commit in middle of history is missing sign-off:**
```bash
# Option 1: Squash all commits into one
git reset --soft <base-commit-hash>
git commit -s -m "Your PR title/message"
git push --force

# Option 2: Interactive rebase (advanced)
git rebase -i <base-commit-hash>
# Mark commits to edit, then for each:
git commit --amend -s
git rebase --continue
```

**Source:** [CONTRIBUTING_GUIDELINES.md#missing-sign-offs](../../CONTRIBUTING_GUIDELINES.md#missing-sign-offs)

**Note:** You do NOT need to squash commits yourself; maintainers use "Squash and merge" on GitHub.

### 10. Commit Message Format

- [ ] **Follow "good commit message" principles**:
  - **Limit title to 50 characters**
  - **Capitalize the first word**
  - **Do not end with a period**
  - **Use imperative mood** (e.g., "Add feature" not "Added feature")

**Source:** [CONTRIBUTING_GUIDELINES.md#creating-a-pull-request](../../CONTRIBUTING_GUIDELINES.md#creating-a-pull-request)
**Reference:** [How to Write a Git Commit Message](https://chris.beams.io/posts/git-commit/)

**Examples:**
- ✅ "Add support for OpenSearch 3.x driver"
- ✅ "Fix bulk request size limit validation"
- ✅ "Update ES driver compatibility matrix"
- ❌ "added opensearch support."
- ❌ "Fixed bug"

---

## Pull Request Submission

### 11. Pre-Submission Checklist

- [ ] **Run quality checks**:
  ```bash
  make fmt   # Auto-format code (commit changes)
  make lint  # Check linting errors
  make test  # Run all tests
  ```

**Source:** [CONTRIBUTING.md#contributing-code](../../CONTRIBUTING.md#contributing-code)

**Important:** Commit any changes from `make fmt` BEFORE pushing.

### 12. Push to Your Fork

- [ ] **Push your branch**:
  ```bash
  git push --set-upstream origin feature/issue-7612-driver-research
  ```
- [ ] **Do NOT repeatedly merge upstream main** unless resolving conflicts
  - Each merge triggers CI reset and requires maintainer re-approval

**Source:** [CONTRIBUTING_GUIDELINES.md#creating-a-pull-request](../../CONTRIBUTING_GUIDELINES.md#creating-a-pull-request)

### 13. Create Pull Request

- [ ] **Write descriptive PR title** (follows commit message format)
- [ ] **Add PR description** with:
  - Problem being solved (can reference issue: "Resolves #7612")
  - Summary of changes (explain _what_ and _why_, not _how_)

**Source:** [CONTRIBUTING_GUIDELINES.md#creating-a-pull-request](../../CONTRIBUTING_GUIDELINES.md#creating-a-pull-request)

### 14. Use Draft PRs (Recommended)

- [ ] **Open as Draft PR** if work is not ready for review
  - Allows CI to run and catch issues early
  - Signals to maintainers: "not ready for review yet"
  - Convert to "Ready for Review" when complete

**Best Practice:** Draft PRs are encouraged for large features or exploratory work like issue #7612 investigation.

**GitHub UI:** Use "Create Draft Pull Request" dropdown when opening PR.

---

## Communication Channels

### 15. Where to Ask Questions

**Before Starting Work:**
- [ ] **GitHub Issue #7612**: Best place for investigation discussions
  - URL: https://github.com/jaegertracing/jaeger/issues/7612

**During Development:**
- [ ] **CNCF Slack - #jaeger channel**:
  - Workspace: cloud-native.slack.com
  - Channel: #jaeger (https://cloud-native.slack.com/archives/CGG7NFUJ3)
  - Use for: Quick questions, clarifications, informal discussions

**Source:** [jaegertracing.io/get-involved](https://www.jaegertracing.io/get-involved/)

**Join Instructions:**
1. Get CNCF Slack invite: https://slack.cncf.io/
2. Join #jaeger channel once in workspace

**Formal Proposals:**
- [ ] **Mailing List**: jaeger-tracing@googlegroups.com
  - Use for: RFCs, major architecture discussions, governance topics

**Source:** [GOVERNANCE.md](../../GOVERNANCE.md)

### 16. Meeting Schedule

- [ ] **Bi-weekly video calls** (check #jaeger Slack for schedule)
  - Discuss issues and initiatives
  - Open to all contributors

**Source:** [jaegertracing.io/get-involved](https://www.jaegertracing.io/get-involved/)

**Tip:** Introduce yourself on Slack when starting significant work. Maintainers are friendly and responsive!

---

## PR Review Process

### 17. After Submitting PR

- [ ] **Be patient**: Maintainers review PRs as time permits
- [ ] **Respond to feedback**: Address comments and suggestions
- [ ] **Do NOT force-push** after reviews start (unless requested)
  - Adds new commits instead to preserve review context
  - Exception: Fixing DCO sign-off requires force-push

**Source:** [CONTRIBUTING_GUIDELINES.md#creating-a-pull-request](../../CONTRIBUTING_GUIDELINES.md#creating-a-pull-request)

### 18. Maintainer Merge Process

- [ ] **Maintainers will "Squash and merge"** your PR
  - All commits squashed into one
  - PR title becomes commit message
  - Individual commit messages preserved in body

**Source:** [CONTRIBUTING.md#merging-prs](../../CONTRIBUTING.md#merging-prs)

**Implication:** You don't need to manually squash; focus on clear individual commits during development.

---

## Advanced Topics

### 19. Deprecating CLI Flags (If Applicable)

If your work involves CLI changes:
- [ ] **Follow deprecation policy**:
  - Deprecated in release N → removed in N+2 or 3 months later
  - Add clear deprecation message with timeline
  - Log warnings when deprecated flags are used

**Source:** [CONTRIBUTING.md#deprecating-cli-flags](../../CONTRIBUTING.md#deprecating-cli-flags)

**Not directly applicable to issue #7612**, but good to know for driver configuration changes.

### 20. Using Feature Gates (If Applicable)

For breaking changes (like switching ES drivers):
- [ ] **Consider OTel Collector feature gates**:
  - Alpha: Disabled by default (experimental)
  - Beta: Enabled by default (user can disable)
  - Stable: Always enabled (2 releases later)
  - Removed: (2 releases after stable)

**Source:** [CONTRIBUTING.md#using-feature-gates-for-breaking-changes](../../CONTRIBUTING.md#using-feature-gates-for-breaking-changes)
**Reference:** [OTel Feature Gates](https://github.com/open-telemetry/opentelemetry-collector/blob/main/featuregate/README.md)

**Example:** Issue #7612 migration could use feature gate like `jaeger.new-es-driver` to allow gradual rollout.

---

## Governance & Maintainership

### 21. Path to Becoming a Maintainer

If you become a long-term contributor:
- [ ] **Maintainer requirements** (over 3+ months):
  - Review 10+ non-trivial PRs
  - Contribute 10+ non-trivial merged PRs
  - Demonstrate collaboration and understanding of codebase

**Source:** [GOVERNANCE.md#becoming-a-maintainer](../../GOVERNANCE.md#becoming-a-maintainer)

**Nomination Process:**
- Existing maintainer proposes you via mailing list or GitHub issue
- Two other maintainers must second
- 5 working days for objections
- Simple majority vote if needed

**Not immediately relevant**, but good to understand project structure.

---

## Issue #7612 Specific Notes

### 22. Research Phase Conventions

For this investigation issue:
- [ ] **Document research in `docs/jaeger-7612/`** (already in place)
- [ ] **Use research branch** `research/jaeger-7612` for all commits
- [ ] **All commits signed** with `-s` (DCO requirement)
- [ ] **Reference #7612** in commit messages

**Current branch status:**
```bash
git branch  # Should show: * research/jaeger-7612
git log -1  # Verify latest commits have Signed-off-by
```

### 23. Deliverable Expectations

When completing issue #7612 investigation:
- [ ] **Compile report** in markdown format
- [ ] **Include all matrices** (drivers, compatibility, API differences)
- [ ] **Provide recommendation** with rationale
- [ ] **Estimate migration effort** (person-weeks, risk assessment)
- [ ] **Submit as RFC** (via PR or GitHub discussion)

**Reference:** [ISSUE7612_DIGEST.md](ISSUE7612_DIGEST.md#investigation-objectives)

### 24. Community Feedback

Before finalizing recommendation:
- [ ] **Post summary in #7612 issue** for community input
- [ ] **Discuss on Slack #jaeger** for informal feedback
- [ ] **Optional: Present in bi-weekly meeting** for maintainer Q&A

**Goal:** Build consensus before committing to migration path.

---

## Common Pitfalls

### 25. Avoiding Common Mistakes

- [ ] ❌ **DON'T work on `main` branch** → CI fails, PR rejected
- [ ] ❌ **DON'T forget `-s` on commits** → DCO bot blocks merge
- [ ] ❌ **DON'T submit without running `make lint`** → CI fails, delays review
- [ ] ❌ **DON'T make huge PRs** → Hard to review, consider splitting
- [ ] ❌ **DON'T add AI attribution to commits** → Will be rejected (user hook policy)

**Best Practices:**
- ✅ **DO open issue first** for significant changes
- ✅ **DO use Draft PRs** for work-in-progress
- ✅ **DO ask questions on Slack** before getting stuck
- ✅ **DO write tests** alongside code
- ✅ **DO respond to reviews** promptly and respectfully

---

## Quick Commands Reference

```bash
# Setup
git remote add upstream git@github.com:jaegertracing/jaeger.git
git submodule update --init --recursive
make install-tools

# Development workflow
git checkout -b feature/my-feature
# ... make changes ...
make fmt && git add -A && git commit -s -m "Add feature"
make lint
make test
git push -u origin feature/my-feature

# Fixing missing DCO sign-off
git commit --amend -s && git push --force

# Before submitting PR
make fmt && make lint && make test
```

---

## Source Documents

All information in this checklist is derived from official Jaeger documentation:

1. **[CONTRIBUTING.md](../../CONTRIBUTING.md)** - Main contribution guide
2. **[CONTRIBUTING_GUIDELINES.md](../../CONTRIBUTING_GUIDELINES.md)** - Detailed workflow
3. **[GOVERNANCE.md](../../GOVERNANCE.md)** - Project governance
4. **[CODE_OF_CONDUCT.md](../../CODE_OF_CONDUCT.md)** - Community standards
5. **[DCO](../../DCO)** - Developer Certificate of Origin
6. **[jaegertracing.io/get-involved](https://www.jaegertracing.io/get-involved/)** - Contact info

**Verification Date:** 2025-10-25
**Last Repo Commit:** 82057d34 (as of checklist creation)

---

## Checklist Status for Issue #7612

Track your progress on the investigation:

### Setup Phase
- [x] Forked and cloned repository
- [x] Installed prerequisites
- [x] Created research branch `research/jaeger-7612`
- [x] Added upstream remote

### Research Phase
- [x] Conducted repository census
- [x] Analyzed issue #7612 deeply
- [ ] Compiled driver inventory (pending)
- [ ] Created compatibility matrix (pending)
- [ ] Performed API comparison (pending)

### Documentation Phase
- [x] All commits signed with DCO
- [x] Research documented in `docs/jaeger-7612/`
- [ ] Final report ready for RFC (pending)

### Community Engagement
- [ ] Posted progress update to #7612
- [ ] Discussed findings on Slack #jaeger
- [ ] Gathered maintainer feedback

**Next Steps:** Continue driver research per [ISSUE7612_DIGEST.md roadmap](ISSUE7612_DIGEST.md#suggested-research-approach).

---

**Contributing to Jaeger is rewarding!** Thank you for following these conventions. If you have questions, the community is here to help on Slack #jaeger. Good luck with your contribution! 🚀
