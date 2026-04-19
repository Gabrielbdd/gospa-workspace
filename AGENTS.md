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

## Cross-impact assessment

When you work from this workspace and make a change inside one
submodule, your plan must declare the impact on the other — even when
that impact is "none". Stating it explicitly is the point.

- Change inside `repos/gofra` that touches a public contract, a
  `runtime/` package, the CLI, the starter, DX, or an operational
  concern → declare what it means for `repos/gospa`: what must change
  there, what will not change and why, or what was consciously deferred.
- Change inside `repos/gospa` that reveals a gap generic enough that any
  Gofra-based app would hit it → the fix belongs upstream in
  `repos/gofra` first, not worked around in the product. Flag this in
  the plan before implementing anything inside Gospa.
- "No impact" is a valid conclusion, but it must be stated — silence is
  not acceptable.

This assessment is a workspace-level rule. It lives here, not inside the
submodules, because the workspace is the only place that legitimately
knows about both repos at once. Each submodule keeps its own unilateral
boundary rule (framework must stay general; product concerns push back
upstream) and those stand alone.

## Long-running work — progress files

For work that spans more than one session:

- If the work lives entirely inside one submodule, use that submodule's
  convention: `repos/gofra/docs/project/agent-workflow.md` or
  `repos/gospa/docs/project/agent-workflow.md`.
- If the work genuinely crosses both submodules — coordinated changes
  that land in lockstep, migrations with a gofra-side and a gospa-side —
  keep a workspace-level progress file at
  `gospa-workspace/docs/progress/<kebab-slug>.md`. Mirror the required
  sections from the submodule convention (objective, context consulted,
  decisions taken, next steps, blockers, pending validations, current
  state with date).

Per-submodule progress files and the workspace progress file may
reference each other when work is coordinated, but each remains a
self-contained entry point.

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

## Push order — non-negotiable

A workspace commit that bumps submodule pointers references SHAs. Those
SHAs must exist on the submodule's remote before the workspace commit is
pushed, otherwise anyone who pulls the workspace and runs
`git submodule update` gets a broken pointer.

The order is therefore always the same and is not a choice:

1. Inside each touched submodule: commit, then `git push` to that
   submodule's remote.
2. Only after both submodule pushes land: commit the workspace bump (if
   not already committed) and `git push` the workspace.

If only one submodule changed, push that one and then the workspace. If
the workspace commit was already created before the submodule commits
were pushed — which is fine locally — the submodule pushes still go
first; the workspace push goes last.

Stated as a checklist for any change made through this workspace:

- [ ] Submodule-local commits are green (`go test ./...` in gofra; `mise
      run test` and `mise run build` in gospa).
- [ ] Submodule commits pushed to their respective remotes.
- [ ] Workspace commit that advances the pointers exists locally.
- [ ] `git diff --submodule=log` at the workspace confirms the pointer
      deltas match what was just pushed.
- [ ] Workspace pushed last.

Pushes go directly to `main` in both submodules and in the workspace.
That is the established convention here — no feature-branch-plus-PR
dance for routine work. **Never `--force` any of these three pushes.**
Not on the workspace, not on a submodule, under any circumstance, unless
the user has explicitly asked for a force-push in that exact sentence.

Never skip the submodule-first order to "save time" — the failure mode
is silent and lands on whoever clones next.

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
