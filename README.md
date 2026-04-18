# gospa-workspace

Local integration workspace for developing the **Gofra** framework and the
**Gospa** product side by side.

This repo is not an application. It is a thin orchestration layer that pins
specific commits of two independent repositories as submodules and ties them
together with a `go.work` file so you can edit both trees at the same time
without hacks in their `go.mod` files.

If you are looking for the projects themselves, they live here:

- Gofra framework: <https://github.com/Gabrielbdd/gofra>
- Gospa product:   <https://github.com/Gabrielbdd/gospa>

For the contract between this workspace and the two repos, read
[`AGENTS.md`](AGENTS.md).

## Layout

```
gospa-workspace/
├── AGENTS.md
├── README.md
├── go.work
├── .gitmodules
├── docs/
│   └── migration-plan.md
└── repos/
    ├── gofra/
    └── gospa/
```

## First-time setup

```bash
git clone --recurse-submodules git@github.com:Gabrielbdd/gospa-workspace.git
cd gospa-workspace
```

If you cloned without `--recurse-submodules`, fix it with:

```bash
git submodule update --init --recursive
```

## Everyday commands

Update both submodules to the SHAs this workspace pins:

```bash
git submodule update --init --recursive
```

Move a submodule to the latest `main` of its upstream:

```bash
cd repos/gofra
git fetch origin
git checkout origin/main
cd ../..
git add repos/gofra
git commit -m "chore: bump gofra submodule"
```

Inspect what `go.work` is using:

```bash
go work edit -print
```

## Running the two repos together

`go.work` at the workspace root lets Go resolve imports directly from the
local checkouts. No `replace` directives in `go.mod` are required for this to
work. From any shell inside the workspace:

```bash
cd repos/gospa
go test ./...          # resolves gofra from ../gofra via go.work
```

## Releases

Releases are cut inside the target repository, not here:

- Framework releases live on tags in the Gofra repo.
- Product releases (and container images) live in the Gospa repo.

This workspace only advances the submodule pointers once a release (or a
green commit on `main`) exists upstream.

## Current plan

The operational plan for the in-flight migration of both repos to their
canonical GitHub module paths lives at
[`docs/migration-plan.md`](docs/migration-plan.md). Agents should read that
before starting work.
