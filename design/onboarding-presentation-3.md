---
marp: true
theme: default
style: |
  section {
    font-size: 24px;
    padding-top: 20px;
  }
  h1 {
    font-size: 36px;
  }
  h2 {
    font-size: 32px;
    margin-top: 0;
  }
  pre, code {
    font-size: 18px;
  }
  table {
    font-size: 20px;
  }
---

# Go MCP SDK: Maintainer Onboarding

## Presentation 3: Open Source Processes

---

# Agenda (60 minutes)

1. **Governance & Working Group** (15 min)
   - Project structure
   - Working Group meetings
   - Decision making process

2. **Issue & PR Management** (15 min)
   - Triage workflow
   - Labels and lifecycle
   - Proposal process

3. **Tracking the MCP Spec** (15 min)
   - Protocol changes
   - SDK tiering requirements
   - Conformance testing

4. **Releases & Maintenance** (15 min)
   - Release process
   - Backward compatibility
   - Security handling

---

# Part 1: Governance & Working Group

---

## Project Structure

**Administered by:** Go team (Google) and Anthropic

**Working Group:** Set of maintainers who can approve and merge PRs

**Key repositories:**
- [modelcontextprotocol/go-sdk](https://github.com/modelcontextprotocol/go-sdk) - This SDK
- [modelcontextprotocol/modelcontextprotocol](https://github.com/modelcontextprotocol/modelcontextprotocol) - MCP spec
- [modelcontextprotocol/conformance](https://github.com/modelcontextprotocol/conformance) - Conformance tests

**Communication channels:**
- GitHub Issues & Discussions (primary)
- Discord (real-time contributor discussion)
- Proposal meetings (complex design decisions)

---

## Proposal Meetings

Regular virtual meetings to review proposals that need discussion beyond GitHub.

**When proposals go to a meeting:**
- Complex or controversial changes
- Multiple valid approaches needing debate
- Proposals that stalled on the issue tracker

**Format:**
- Announced via pinned GitHub issue with agenda
- Open to all participants
- Recorded with notes published afterward

Similar to the [Go Tools call](https://go.dev/wiki/golang-tools).

---

## SDK Working Group

Monthly meeting for SDK maintainers across all languages.

**Purpose:**
- Coordinate cross-SDK features and consistency
- Discuss spec changes affecting SDKs
- Share implementation approaches

**Schedule:**
- Monthly at 6pm London time (see [meet.modelcontextprotocol.io](https://meet.modelcontextprotocol.io/))
- Discord channel: `#sdk-wg`

**How to join:**
- Attend the next meeting from the calendar
- Follow `#general-sdk-dev` on Discord for async discussion

---

## Decision Making Process

```
                    ┌─────────────────┐
                    │  Issue Filed    │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
     ┌────────────────┐           ┌─────────────────┐
     │  Bug / Simple  │           │    Proposal     │
     │    Fix         │           │  (API change)   │
     └───────┬────────┘           └────────┬────────┘
             │                             │
             ▼                             ▼
     ┌────────────────┐           ┌─────────────────┐
     │  PR Review     │           │  Week-long      │
     │  & Merge       │           │  Discussion     │
     └────────────────┘           └────────┬────────┘
                                           │
                          ┌────────────────┴────────────────┐
                          │                                 │
                          ▼                                 ▼
                 ┌─────────────────┐              ┌─────────────────┐
                 │  Straightforward│              │   Complicated   │
                 │  → Approve      │              │ → Proposal Mtg  │
                 └─────────────────┘              └─────────────────┘
```

---

## Antitrust Considerations

The SDK follows the [LF Projects Antitrust Policy](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/ANTITRUST.md).

**In practice, avoid discussing:**
- Pricing or commercial terms
- Customer relationships
- Product roadmaps beyond the SDK
- Competitive strategies

**Keep discussions:**
- Technical and focused on the SDK
- Open and transparent
- Documented in GitHub (not private channels)

---

## Discord

The [MCP Discord](https://discord.gg/6CSzBmMkjX) is open for real-time contributor discussion.

**Public channels include:**
- SDK development (`#typescript-sdk-dev`, etc.)
- Working Group discussions
- Community feedback and brainstorming
- Office hours and maintainer availability

**Not for:**
- General MCP user support (use GitHub Discussions)
- Product/service marketing

**Important:** Technical decisions must still be documented in GitHub Issues/Discussions. Discord is for real-time collaboration, not permanent records.

---

# Part 2: Issue & PR Management

---

## Issue Triage Workflow

**When a new issue arrives:**

1. **Read and understand** - What is the user experiencing?
2. **Categorize** - Bug, enhancement, proposal, question?
3. **Label** - Apply appropriate labels
4. **Respond** - Acknowledge, ask for clarification, or address
5. **Link** - Connect to related issues or PRs

**Tier 1 requirement:** Issues triaged within 2 days

---

## Issue Labels

| Label | Meaning |
|-------|---------|
| `bug` | Something isn't working |
| `enhancement` | New feature or improvement |
| `proposal` | API change requiring discussion |
| `proposal-accepted` | Approved for implementation |
| `help wanted` | Good for external contributors |
| `documentation` | Docs need updating |
| `duplicate` | Already reported (link to original) |
| `question` | General question (may redirect) |

---

## Bug Reports

Users should answer five questions:

1. What did you do?
2. What did you see?
3. What did you expect to see?
4. What version of the Go MCP SDK are you using?
5. What version of Go are you using?

**If info is missing:** Ask for it before investigating.

**If it's a real bug:** Acknowledge, prioritize, fix or assign.

---

## The Proposal Process

**What requires a proposal:**
- New API
- Change to existing API signature or behavior
- New module dependency

**Process:**
1. File issue with `proposal` label
2. Allow at least one week for discussion
3. Maintainer approves → `proposal-accepted` label
4. Implementation can proceed

Similar to [Go proposal process](https://github.com/golang/proposal), but lighter weight.

---

## PR Review

**For external contributors:**
- Check if issue exists and contributor claimed it
- Issues labeled `help wanted` are pre-approved for contribution
- For unlabeled issues, contributor should ask before starting

**Review checklist:**
- Does it solve the stated problem?
- Is it well tested?
- Does it follow [Google Go style guide](https://google.github.io/styleguide/go/)?
- Does it have proper copyright header?
- Does commit message follow [Go format](https://go.dev/wiki/CommitMessage)?

---

## Copyright Header

All Go files must have:

```go
// Copyright 2025 The Go MCP SDK Authors. All rights reserved.
// Use of this source code is governed by an MIT-style
// license that can be found in the LICENSE file.
```

---

## Commit Message Format

Follow the [Go commit message format](https://go.dev/wiki/CommitMessage):

```
mcp: add support for resource subscriptions

Implements client.Subscribe/Unsubscribe and server-side
subscription tracking via ServerOptions.SubscribeHandler.

Fixes #123
```

**Format:**
- First line: `package: brief description` (lowercase, no period)
- Blank line
- Detailed explanation if needed
- Issue reference at the end

---

## Timeouts

**Two-week timeout policy:**

If a contributor hasn't responded to:
- Questions on their issue
- Comments on their PR

The issue or PR may be closed.

Can be reopened when contributor resumes.

---

## Dependencies Policy

**Goal:** Minimize dependencies (bugs, churn, conflicts for users)

**New dependencies require a proposal** including:
- Stability evaluation
- Security assessment
- Ecosystem establishment

**Dependency updates:**
- Can be done without proposal (if same major version)
- Prefer immediately after SDK release
- Run `govulncheck` after any change:

```bash
go run golang.org/x/vuln/cmd/govulncheck@latest
```

---

# Part 3: Tracking the MCP Spec

---

## MCP Spec Repository

Watch: [modelcontextprotocol/modelcontextprotocol](https://github.com/modelcontextprotocol/modelcontextprotocol)

**Key things to monitor:**
- SEPs (Spec Enhancement Proposals)
- Protocol version changes
- Breaking changes

**Current supported versions:**
- 2025-11-25 (latest)
- 2025-06-18
- 2025-03-26
- 2024-11-05

---

## When Spec Changes Land

1. **Update protocol types**
   - `mcp/protocol.go` (often auto-generated from spec)
   - Add new types, fields, methods

2. **Implement features**
   - Add new handlers/methods as needed
   - Adjust existing behavior

3. **Update conformance tests**
   - Internal: `mcp/testdata/conformance/`
   - Official: verify with `./scripts/conformance.sh`

4. **Document changes**
   - Update `docs/` as needed
   - Update version compatibility table in README

---

## SDK Tiering System

MCP is introducing tiers for SDKs ([SEP-1730](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1730)):

| Tier | Conformance | Features | Issues | Security |
|------|-------------|----------|--------|----------|
| **Tier 1** | 100% | Before release | 2 days | 7 days |
| **Tier 2** | 80% | Within 6 months | Active | Active |
| **Tier 3** | — | — | — | — |

**Tier 1 requirements in detail:**
- All conformance tests pass
- New features implemented before protocol release
- Issues triaged within 2 days
- Security bugs fixed within 7 days

**Goal:** Maintain Tier 1 status

---

## Conformance Testing

**Internal tests:**
```bash
go test -run TestServerConformance ./mcp
```

**Official conformance suite:**
```bash
./scripts/conformance.sh
```

**Save results:**
```bash
./scripts/conformance.sh --result_dir ./conformance-results
```

**Test against local conformance repo:**
```bash
./scripts/conformance.sh --conformance_repo ~/src/conformance
```

---

## Staying Ahead of Spec Changes

**Tier 1 requires features before protocol release**

This means:
- Watch the spec repo for upcoming changes
- Participate in SEP discussions
- Implement features during spec development
- Be ready when new protocol version ships

---

# Part 4: Releases & Maintenance

---

## Semantic Versioning

Post-v1.0.0, the SDK follows semver:

| Version | Meaning |
|---------|---------|
| `v1.x.y` | Backward compatible |
| `v1.x+1.0` | New features, backward compatible |
| `v1.x.y+1` | Bug fixes only |
| `v2.0.0` | Breaking changes |

**Backward compatibility is required** — changes that break existing code need a major version bump.

---

## V2 Tracking

Known issues to address in v2 are tracked in `docs/rough_edges.md`:

- `EventStore.Open` unnecessary (can be no-op)
- Invalid tool names only log instead of panic (SEP-986)
- Inconsistent naming (`ResourceUpdatedNotificationsParams`)
- Default capabilities should be empty

Review this file before releases and when planning v2.

---

## Release Checklist

**Before releasing:**

1. Run full test suite:
   ```bash
   go test -race ./...
   ```

2. Run conformance tests:
   ```bash
   ./scripts/conformance.sh
   ```

3. Run vulnerability check:
   ```bash
   go run golang.org/x/vuln/cmd/govulncheck@latest
   ```

4. Check `docs/rough_edges.md` for blockers

5. Update version compatibility table in README if needed

---

## Creating a Release

Releases are created via **GitHub Releases** (not `git tag` directly).

**Process:**
1. Go to [Releases page](https://github.com/modelcontextprotocol/go-sdk/releases)
2. Click "Draft a new release"
3. Create new tag (e.g., `v1.3.0`)
4. Write release notes (see previous releases for format)
5. Publish

The tag is created automatically when you publish.

---

## Release Notes Format

Look at previous releases for the format. Generally include:

- **Summary** of what changed
- **New features** with examples
- **Bug fixes** with issue references
- **Breaking changes** (if any, for major versions)
- **Contributors** acknowledgment

---

## Security Handling

**Tier 1 requirement:** Security bugs fixed within 7 days

**When a security issue is reported:**
1. Acknowledge promptly
2. Assess severity
3. Develop fix privately if needed
4. Release patch version
5. Disclose responsibly

**Proactive measures:**
- Run `govulncheck` after dependency changes
- Monitor for CVEs in dependencies
- Keep dependencies updated

---

## README Generation

**README.md is auto-generated — don't edit directly!**

**To update:**
1. Edit `internal/readme/README.src.md`
2. Run `go generate ./internal/readme`
3. Commit both files together

CI checks that README is in sync.

---

## Documentation Updates

**When to update docs:**
- New features added
- Behavior changes
- API additions

**Doc files:**
- `docs/` - Feature documentation
- `internal/readme/README.src.md` - README source
- `internal/docs/*.src.md` - Other doc sources

**Regenerate after changes:**
```bash
go generate ./internal/docs
```

---

## Code of Conduct

The project follows the [MCP Code of Conduct](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/CODE_OF_CONDUCT.md).

For conduct-related issues: mcp-coc@anthropic.com

---

## Summary

**Governance:**
- Working Group makes decisions
- Proposals require week-long discussion
- GitHub is the primary venue

**Issue/PR Management:**
- Triage within 2 days (Tier 1)
- Label appropriately
- Proposals for API changes

**Spec Tracking:**
- Watch the spec repo
- Implement before release (Tier 1)
- Run conformance tests

**Releases:**
- Via GitHub Releases
- Full test suite first
- Maintain backward compatibility

---

## Resources

- **CONTRIBUTING.md:** Full contribution guidelines
- **Design Document:** `design/design.md`
- **Rough Edges:** `docs/rough_edges.md`
- **MCP Spec Repo:** [modelcontextprotocol/modelcontextprotocol](https://github.com/modelcontextprotocol/modelcontextprotocol)
- **Conformance Repo:** [modelcontextprotocol/conformance](https://github.com/modelcontextprotocol/conformance)
- **Go Code of Conduct:** https://go.dev/conduct

---

## Q&A

Questions about open source processes or maintainer responsibilities?
