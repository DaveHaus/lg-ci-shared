# LG Go Services — CI Architecture

Shared reusable GitHub Actions workflow for 5 Go microservices. Each service repo references this workflow rather than maintaining its own CI logic.

## Architecture

```mermaid
flowchart TD
    dev([Developer])
    dev -->|git push / PR / manual trigger| sales
    dev -->|git push / PR / manual trigger| pricing
    dev -->|git push / PR / manual trigger| store
    dev -->|git push / PR / manual trigger| inventory
    dev -->|git push / PR / manual trigger| notification

    subgraph services[Service Repos]
        sales[sales\n.github/workflows/ci.yml]
        pricing[pricing\n.github/workflows/ci.yml]
        store[store\n.github/workflows/ci.yml]
        inventory[inventory\n.github/workflows/ci.yml]
        notification[notification\n.github/workflows/ci.yml]
    end

    sales -->|uses: DaveHaus/lg-ci-shared/.github/workflows/go-ci.yml@main| shared
    pricing -->|uses: DaveHaus/lg-ci-shared/.github/workflows/go-ci.yml@main| shared
    store -->|uses: DaveHaus/lg-ci-shared/.github/workflows/go-ci.yml@main| shared
    inventory -->|uses: DaveHaus/lg-ci-shared/.github/workflows/go-ci.yml@main| shared
    notification -->|uses: DaveHaus/lg-ci-shared/.github/workflows/go-ci.yml@main| shared

    subgraph shared[lg-ci-shared — Reusable Workflow]
        direction TB
        s1[1. Checkout code]
        s2[2. Setup Go toolchain]
        s3[3. go mod download]
        s4[4. go build ./...]
        s5[5. go vet ./...]
        s6[6. go test -v ./...]
        s1 --> s2 --> s3 --> s4 --> s5 --> s6
    end

    shared --> result{Pass / Fail}
    result -->|success| green([Pipeline Green])
    result -->|failure| red([Pipeline Red — blocks merge])
```

## Services

| Repo | Endpoint | Description |
|------|----------|-------------|
| [sales](https://github.com/DaveHaus/sales) | `POST /sales` | Record a sale |
| [pricing](https://github.com/DaveHaus/pricing) | `GET /price` | Current apple price |
| [store](https://github.com/DaveHaus/store) | `GET /stores` | Store and manager list |
| [inventory](https://github.com/DaveHaus/inventory) | `GET /inventory` | Stock levels per store |
| [notification](https://github.com/DaveHaus/notification) | `POST /notify` | Alert store managers |

## How It Works

Each service repo contains a minimal `ci.yml` that delegates entirely to this shared workflow:

```yaml
jobs:
  build-and-test:
    uses: DaveHaus/lg-ci-shared/.github/workflows/go-ci.yml@main
    with:
      service-name: sales
```

One change to `go-ci.yml` here propagates to all 5 services on their next run. No per-repo CI maintenance required.

## CI Stages

| Stage | Command | Purpose |
|-------|---------|---------|
| Dependencies | `go mod download` | Pull declared deps from go.mod |
| Build | `go build ./...` | Compile — proves code is valid Go |
| Vet | `go vet ./...` | Static analysis — catches common bugs |
| Test | `go test -v ./...` | Run unit tests with verbose output |

## Triggering a Pipeline

Pipelines run automatically on `push` to `main` and on pull requests. To trigger manually:

**GitHub UI:** Repo → Actions → CI → Run workflow → select branch → Run

**CLI:**
```bash
gh workflow run ci.yml --repo DaveHaus/sales
```
