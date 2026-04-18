# gospa-workspace

This repository is a **workspace**, not an application.

Its job is to give humans and AI agents a single working tree that ties
together two otherwise independent repositories — the **Gofra** framework and
the **Gospa** product — so they can be developed in sync locally without
introducing cross-repo coupling that would have to be undone before release.

## Topology

```
gospa-workspace/
├── AGENTS.md              ← this file
├── README.md
├── go.work                ← versioned here, nowhere else
├── .gitmodules
├── docs/
│   └── migration-plan.md  ← current operational plan
└── repos/
    ├── gofra/             ← submodule → github.com/Gabrielbdd/gofra
    └── gospa/             ← submodule → github.com/Gabrielbdd/gospa
```

Responsibilities are split intentionally:

| Repo | What it owns |
| --- | --- |
| `repos/gofra` | Framework, `gofra` CLI, starter, reusable runtime packages. Source of truth for framework releases. |
| `repos/gospa` | Product (PSA). Depends on a published version of Gofra. Source of truth for product releases and deployments. |
| Workspace root | Shared context for agents, local `go.work`, cross-repo documentation, submodule SHA pinning. |

## Non-negotiable rules

### Hard boundary: this workspace does not bleed into the submodules

`repos/gofra` and `repos/gospa` are **autonomous repositories**. They must
remain correct and legible when cloned in isolation, with zero knowledge of
this workspace's existence.

Neither submodule's tracked files may mention:

- this workspace or any "sibling repo" relationship
- git submodules (as a mechanism they participate in)
- `go.work` files
- "parallel development" workflows
- external orchestration layers of any kind

All cross-repo coordination — decisions that involve both projects,
instructions for agents about "work here first then there", integration
sequencing, release ordering between gofra and gospa — belongs **only** in
this top-level repo. Agents working inside `repos/gofra` or `repos/gospa`
should receive no context about this workspace; the documentation inside
each submodule must stand on its own.

### Other rules

1. **`github.com/Gabrielbdd/gofra`** and **`github.com/Gabrielbdd/gospa`** are
   the canonical module paths.
2. The workspace does **not** substitute CI or the release process of either
   repo. All publishing happens inside the respective repos.
3. `go.work` is committed **only** at this workspace root. Never inside
   `repos/gofra` or `repos/gospa`.
4. No `go.mod` inside `repos/gofra` or `repos/gospa` may contain `replace ../`.
   Local multi-repo development uses `go.work` at this workspace level, not
   committed replaces inside the submodules.
5. `repos/gospa` depends on a **published** version of `github.com/Gabrielbdd/gofra`.
6. Every public change in Gofra updates the corresponding docs under
   `repos/gofra/docs/framework/` in the same change. That rule is enforced by
   a Stop hook inside the Gofra repo.
7. `main` of both repos should remain publishable at all times. Workspace
   commits that bump submodule SHAs only land after the target repo's own CI
   is green.

## Working on both repos at once

The typical flow:

```bash
# Bring both submodules to the SHAs pinned by this workspace
git submodule update --init --recursive

# Make a change inside one of the submodules
cd repos/gofra
# ... edit, test, commit, push inside the gofra repo ...

# Go back to the workspace and advance the submodule pointer
cd ../..
git add repos/gofra
git commit -m "chore: bump gofra submodule to <short-sha>"
```

For simultaneous development where Gospa needs to consume in-progress Gofra
changes, `go.work` at the workspace root makes Go resolve imports directly
from the local framework checkout — no `replace` in `go.mod` required, and no
pseudo-version rituals.

## What NOT to do here

- Do not add a `go.mod` to the workspace root. The workspace is not a module.
- Do not cut releases from this repo. Releases of Gofra happen in the Gofra
  repo; releases of Gospa happen in the Gospa repo.
- Do not resolve a product issue in Gospa by working around a missing
  framework capability. Push the fix back into Gofra first, then consume it.
- Do not vendor or submodule Gofra inside Gospa. The dependency is a normal
  Go module dependency via `go.mod`.

## Ongoing work

The current operational plan — migrating the existing state to the
topology described above — lives in
[`docs/migration-plan.md`](docs/migration-plan.md).
