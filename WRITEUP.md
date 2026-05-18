# LG Take-Home — CI Pipeline Write-Up

**Assignment:** Option 3 — CI Pipelines with GitHub Actions  
**LLM Used:** Claude Sonnet 4.6 (via Claude Code CLI)  
**Repos:** https://github.com/DaveHaus/lg-ci-shared and linked service repos below

---

## Why Option 3

CI/CD was the natural choice — my most recent role was a full Jenkins-to-GitLab migration, and GitHub Actions was the one gap in that experience. Go was a deliberate pick as well: I work primarily in Python and Bash for automation tasks, and this felt like the right context to start building in it.

---

## Prompt Progression — Broad to Narrow

My approach throughout was to use my existing GitLab and Jenkins experience as the frame of reference — asking Claude "I know how GitLab does X, what's the GitHub Actions equivalent?" rather than starting from scratch. Same with Go: I knew how Java and Python handle the runtime layer, so I asked how Go fits that mental model. The LLM became a translation layer between what I already knew and the new tools, not a replacement for knowing anything.

---

### Round 1 — Go's Build Model (Mapping Java/Python to Go)

**My frame:** Java needs a JVM, Python needs an interpreter. I knew Go compiled but wasn't sure where the runtime dependency lived. Claude confirmed Go compiles to a self-contained native binary — no runtime layer needed on the target machine. Also clarified `go get` (adds new deps) vs `go mod download` (pulls existing deps — what CI should use), and added `go vet` as a free built-in static analyzer worth including.


---

### Round 2 — Reuse Pattern (GitLab Components → GitHub Actions)

**My frame:** In my last role we built GitLab CI components — a shared Maven pipeline component that every Java service repo referenced via `include: component:`. One change to the component propagated to all repos. Does GitHub Actions have the same thing?

**What I asked Claude:** 3 teams, 5 repos — how do we avoid duplicating CI logic? GitLab has components and templates, does Actions have the same modularity?

**What Claude confirmed:** GitHub Actions has **Reusable Workflows** — the direct equivalent. One shared repo (`lg-ci-shared`), each service calls it by path via `uses:`.

| Concept | GitLab | GitHub Actions |
|---------|--------|---------------|
| Shared CI logic | `include: component: group/repo@version` | `uses: owner/repo/.github/workflows/file.yml@ref` |
| Version pinning | Component version tag | Version ref on the `uses:` path |
| Scope | Group namespace | Repo path |

---

### Round 3 — Manual Triggers (GitLab "Run Pipeline" → Actions)

**My frame:** In GitLab there's always a "Run Pipeline" button — you pick a branch, fill in any variables, and kick it off manually. Does Actions have the same?

**What I asked Claude:** How do I manually trigger a pipeline in GitHub Actions the same way as GitLab?

**Answer:** GitHub requires explicitly opting in via `workflow_dispatch` in the `on:` block. Once added, a "Run workflow" button appears in the Actions tab. GitLab exposes this by default; Actions requires you to declare it.

---

### Round 5 — Connecting the Services to the Business

The prompt gives three scenarios — price changes, sales tracking, store management. Even though I chose option 3, I used the other two scenarios as context. A fruit company with 200 stores doesn't have one service — it has sales, inventory, pricing, store management, and notifications to managers. I designed the 5 repos around that reality so the CI reuse pattern reflects an actual microservice architecture, not 5 arbitrary apps.

**Services chosen** to map to the apple business domain:

| Repo | Endpoint | Role |
|------|----------|------|
| [sales](https://github.com/DaveHaus/sales) | `POST /sales` | Record a sale (mirrors scenario #2 API exactly) |
| [pricing](https://github.com/DaveHaus/pricing) | `GET /price` | Current apple price |
| [store](https://github.com/DaveHaus/store) | `GET /stores` | Store and manager info |
| [inventory](https://github.com/DaveHaus/inventory) | `GET /inventory` | Stock levels per store |
| [notification](https://github.com/DaveHaus/notification) | `POST /notify` | Alert store managers |

---

## What Was Built

**`lg-ci-shared`** — single source of CI truth. Defines a reusable workflow with inputs (`go-version`, `service-name`) that runs on `ubuntu-latest` and executes: `go mod download` → `go build ./...` → `go vet ./...` → `go test -v ./...`

**Each service repo** — contains a minimal `ci.yml` (under 15 lines) that declares triggers (`push`, `pull_request`, `workflow_dispatch`) and delegates entirely to the shared workflow via `uses: DaveHaus/lg-ci-shared/.github/workflows/go-ci.yml@main`. No CI logic lives in the service repos themselves.

---

## Compare and Contrast — LLM Output vs Reality

### What the LLM Got Right Immediately
- Shared workflow pattern (reusable workflow + caller per service)
- `go mod download` vs `go get`
- `go vet` as a free add
- `./...` recursive package notation
- `workflow_dispatch` for manual trigger
- `httptest` for unit testing without spinning up a server

### What Needed Correction or Iteration

| Issue | What Happened | Resolution |
|-------|--------------|------------|
| Go version mismatch | Shared workflow defaulted to `1.22`, local brew installed `1.26.3` | Visible in Actions logs — flagged as prod gap, acceptable for this exercise |
| No manual trigger | Initial `ci.yml` only had `push` and `pull_request` | Added `workflow_dispatch` after confirming pipelines ran |
| Verbose test output | First run showed `[no test files]` then `ok ... 0.003s` with no detail | Added `-v` flag to `go test` in shared workflow — one change, all 5 services updated |
| No tests initially | `go test ./...` passed with `[no test files]` | Added `main_test.go` per service using `httptest` |

---

## Architecture Diagram

Full Mermaid diagram rendered at https://github.com/DaveHaus/lg-ci-shared — shows developer push → service `ci.yml` → shared workflow → 6 CI steps → pass/fail.

---

## Testing — What We Did and What's Next

### What We Did (Unit Tests with `httptest`)

Each service has a `main_test.go` that tests HTTP handler behavior without starting a server. Go's `net/http/httptest` package simulates the full HTTP request/response cycle in memory:

```go
req := httptest.NewRequest(http.MethodPost, "/sales", body)
w   := httptest.NewRecorder()
salesHandler(w, req)
// assert w.Code, w.Body
```

Tests run inside `go test ./...` — no port binding, no process management, quick. Proves handler logic is correct before the runner tears down.

**What each service tests:**
- Valid request returns correct status and payload
- Wrong HTTP method returns 405
- `/health` returns 200

---

### Nice to Have — Integration Tests (Spin Up and Hit It)

The next level would be starting the actual compiled binary on the runner, hitting it with real HTTP requests via `curl`, then tearing it down. This proves the whole binary boots, the router wires up correctly, and the HTTP layer behaves end-to-end — things `httptest` can't catch like port conflicts, middleware bugs, or startup failures.

Not included here because for simple services with no middleware or DB the unit tests cover the meaningful behavior. The full testing progression would be: unit tests → integration tests → contract tests → E2E against a deployed environment.

---

## GitLab vs GitHub Actions — Key Differences Observed

| Concept | GitLab CI | GitHub Actions |
|---------|-----------|---------------|
| Reusable CI | `include: component:` | `uses: owner/repo/.github/workflows/file.yml@ref` |
| Runner base | Runner tags (`docker`, `linux`) | `runs-on: ubuntu-latest` |
| Manual trigger | "Run Pipeline" button always available | `workflow_dispatch` must be explicitly added |
| Variables with dropdowns | `variables: options: [...]` | `workflow_dispatch: inputs: type: choice` |
| Verbose logs | Per-job log view | Per-step log view within a job |
| Artifact passing | `artifacts: paths:` | `actions/upload-artifact` + `actions/download-artifact` |
| Pipeline graph | Visual stage/job graph | Sequential step list within job |

---

## What I'd Add With More Time

1. **Branch protection rules** — require CI green before merge to `main`
2. **golangci-lint** — richer linting beyond `go vet` (errcheck, staticcheck, etc.)
3. **Docker build + push** — compile to image, push to registry (CD starting point)
4. **Version pinning** — lock the shared workflow reference to a specific version for reproducibility
5. **Integration test stage** — spin up binary, run curl assertions, tear down
