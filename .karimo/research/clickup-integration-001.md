# ClickUp Integration — Research Findings

**ID:** 001
**Created:** 2026-10-26
**Status:** approved
**Feeds:** [`specs/clickup-integration/SPEC.md`](../../specs/clickup-integration/SPEC.md)

---

## 1. Question

How can KARIMO use ClickUp as its project-communication channel: link a repo to
a ClickUp Space, mirror PRDs/tasks/status into ClickUp, ask humans for
clarification through ClickUp, and get (re)started from ClickUp when humans act?

---

## 2. KARIMO Internals Relevant to the Integration

| KARIMO element | Location | Relevance |
|----------------|----------|-----------|
| Pluggable provider manifests | `.karimo/providers/{name}/manifest.yaml`, `templates/PROVIDER_MANIFEST_SCHEMA.md` | Model for a `clickup` provider (manifest + `scripts/`) |
| Review provider selection in configure | `commands/configure.md` → "Basic Step 4", `--greptile` / `--code-review` / `--review` flags | Model for a ClickUp configure step and `--clickup` flag |
| Project config | `.karimo/config.yaml` (`templates/CONFIG_TEMPLATE.yaml`) | Home for `integrations.clickup` |
| Per-PRD overrides | `.karimo/prds/{slug}/.execution_config.json` | Per-PRD opt-out |
| Gitignored local state | `.karimo/local-state.json` | Store "last used interaction mode" without dirtying git |
| Execution state | `.karimo/prds/{slug}/status.json` (`templates/STATUS_SCHEMA.md`) | Source of status transitions to sync |
| Pause/resume points | PM `paused-at-gate`, `paused-wave-gate`, `needs-human-review`; resume via `/karimo:run` | Existing hook for "wait for human" |
| Human questions | `AskUserQuestion` in interviewer, configure, run, PM gates | Blocking-only; unavailable headless → needs a mode-aware ask protocol |
| Branch naming | `worktree/{prd-slug}-{task-id}`; PM derives state from it | Do **not** change branch names; put ClickUp IDs in PR titles instead |

**Observation:** `install.sh` does not reference `.karimo/providers/` in its
explicit copy list; confirm how providers reach target repos (MANIFEST loop)
before adding the `clickup` provider.

---

## 3. ClickUp Platform Facts

### 3.1 REST API v2

- Hierarchy: Workspace (team) → Space → Folder → List → Task → Subtask.
- Auth: personal token (`pk_…`) or OAuth. Personal token of a dedicated member
  is the simplest for headless use.
- **Rate limits (per token):** 100 req/min (Free Forever, Unlimited, Business),
  1,000 (Business Plus), 10,000 (Enterprise). 429 with `X-RateLimit-Limit`,
  `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers.
- **Space statuses cannot be created/edited via the public API** (community
  reports + open feature requests; `PUT /space/{id}` silently ignores status
  payloads). API-created Spaces get only default statuses.
  → Status workflow must come from a **Space template** or manual setup.
- Space tags *can* be created via API; task tags can be added/removed.
- Custom field *values* can be set via API; creating field definitions via the
  public API is unconfirmed → treat custom fields as optional.

### 3.2 Webhooks

- Events include `taskCreated`, `taskUpdated`, `taskStatusUpdated`,
  `taskAssigneeUpdated`, `taskCommentPosted`, `taskCommentUpdated`, `taskMoved`,
  list/folder/space events. `*` subscribes to all.
- Scoped to one Space, Folder, List, or Task (most specific wins).
- Signed: `X-Signature` header, HMAC-SHA256 hex of the body with the
  per-webhook secret.
- Webhook stops firing silently if its creator is deactivated.
- **`taskCommentPosted` fires for top-level comments only — threaded replies do
  not fire** (community report). Comment-driven designs require polling threads.

### 3.3 Automations

- "Call webhook" action supports **custom headers** (e.g. `Authorization`);
  header values are write-only after save.
- "Add a comment" action can @mention people.
- Usable as a trigger source for GitHub `repository_dispatch` (possibly via a
  small relay).

### 3.4 Agents and identity

- ClickUp Super Agents (Brain) can be @mentioned/assigned but run inside
  ClickUp; they cannot operate on a local repo or run Claude Code.
- **External agents cannot get a native mentionable identity** (only curated
  partners such as Cursor). Workaround: a regular member account (one seat)
  named "KARIMO" with its own API token.

### 3.5 Official MCP server

- Hosted at `https://mcp.clickup.com/mcp`, **OAuth only** (no API tokens).
- Fine for interactive sessions, awkward for a headless server.
- Decision: not used by KARIMO; REST API only.

### 3.6 GitHub integration

- ClickUp links commits/branches/PRs that mention the task ID (`CU-{id}` /
  `#{id}`). Using the ID in PR titles gives free traceability.

---

## 4. Trigger Options Evaluated

| Option | Mechanism | Latency | Hosting | Long runs | Verdict |
|--------|-----------|---------|---------|-----------|---------|
| D2 Polling worker | Poll "To Do ∧ assignee=KARIMO" per Space | 1–2 min | Linux server | Unlimited | **Phase 3 (primary)** |
| D3 GitHub Actions | Cron or ClickUp automation → `repository_dispatch` → Claude Code | 5–15 min / seconds | None | 6 h job cap, ephemeral state | Phase 4 (optional) |
| D1 Webhook listener | `taskStatusUpdated`/`taskAssigneeUpdated` → server | Seconds | Public HTTPS (or tunnel) | Unlimited | Phase 4 (optional accelerator; still needs reconciliation poll) |
| D4 ClickUp Super Agent | Native agent | — | None | — | Rejected: no repo access |

---

## 5. Key Insight: Status-Driven Reply Protocol

Using **task status** (Awaiting Input → To Do) as the "human answered" signal
removes the dependency on comment webhooks entirely. On the signal, KARIMO reads
all comments (top-level + thread replies via API) posted after the question.
This sidesteps the threaded-reply webhook gap and keeps the worker's polling to a
single filtered task query per Space.

---

## 6. Sources

- https://developer.clickup.com/docs/webhooks
- https://developer.clickup.com/docs/webhooksignature
- https://developer.clickup.com/docs/rate-limits
- https://feedback.clickup.com/feature-requests/p/custom-external-ai-agents-as-mentionable-participants-in-task-comments
- https://feedback.clickup.com/public-api/p/create-manage-task-statuses-on-lists-folders-space-via-api
- https://feedback.clickup.com/public-api/p/allow-statuses-to-be-updated-at-the-space-level-via-api
- https://feedback.clickup.com/public-api/p/provide-automation-action-webhook-header-customization
- https://help.clickup.com/hc/en-us/articles/31126817112343-Create-a-task-webhook-Automation
- https://help.clickup.com/hc/en-us/articles/6312097314199-Use-Automation-Actions
- https://clickup.com/blog/how-to-use-clickup-mcp-server/
