# Migration Plan — gofra + gospa workspace

Operational plan for migrating the current state of this workspace and its
two submodules to the canonical topology described in
[`../AGENTS.md`](../AGENTS.md).

## End state

- `github.com/Gabrielbdd/gofra` is the framework module path, with a first
  public release (`v0.1.0`) consumed by the starter.
- `github.com/Gabrielbdd/gospa` is the product module path, scaffolded from
  `gofra new` against the published release.
- No `replace ../` survives in either repo's `go.mod`.
- The workspace `go.work` references both submodules.

## Decisions already locked in

- Owner on GitHub: `Gabrielbdd` (capital G).
- No pseudo-version. The starter keeps a local `replace` until Gofra's first
  public tag exists; only then does the starter switch to a published
  `require`.
- `go.work` starts with only `./repos/gofra`; `./repos/gospa` is added in
  Phase 5, after the scaffold.
- Starter remains Postgres-only. No Restate, Zitadel, Jaeger, or new
  `buf lint` in this round.
- GHCR / image publish is off the critical path. Phase 6, only if Gospa ends
  up needing it.
- Workspace documentation stays lean: `AGENTS.md`, `README.md`, and this
  file. Nothing else here until there is concrete content to justify it.

## Phases

### Phase 0 — Bootstrap the workspace

- Create `AGENTS.md`, `README.md`, `docs/migration-plan.md` and `go.work`.
- `go.work` references only `./repos/gofra` at this point.
- Commit: `chore: bootstrap workspace context`.

### Phase 1 — Migrate Gofra to GitHub

Inside `repos/gofra`:

- Rename the module from `databit.com.br/gofra` to
  `github.com/Gabrielbdd/gofra`.
- Replace the old path across every file that references it (imports, tests,
  docs, `annotations.proto`, generated `annotations.pb.go`).
- Rewrite `AGENTS.md` and `CLAUDE.md` to describe the new gofra + gospa
  siblings topology. Remove every mention of a `psa/` submodule inside this
  repo.
- Add a minimal `.github/workflows/ci.yml` running `go test ./...`.
- `go test ./...` and `mise run smoke:new` must be green.
- Commit: `refactor: migrate module path to github.com/Gabrielbdd/gofra`.
- Push `main` to GitHub.

### Phase 2 — Align the starter contract with reality

Inside `repos/gofra`:

- Bump `internal/scaffold/starter/full/mise.toml.tmpl` to Go 1.25.
- Add `tasks.test` and `tasks.build` to the starter's `mise.toml.tmpl`.
- Fix framework README drift: `mise run gen:runtimeconfig` →
  `mise run gen:config`. Audit the rest of the command list.
- Update the starter README and `docs/framework/reference/starter/`.
- Keep the `replace` directive in `go.mod.tmpl`. It is removed in Phase 4.
- Commit: `docs: align starter contract with current repo reality`.

### Phase 3 — Make the starter day-1 deployable

Inside `repos/gofra`:

- Add `Dockerfile` (multi-stage, distroless runtime), `.dockerignore`,
  `.github/workflows/ci.yml` to `internal/scaffold/starter/full/`.
- Expand `generator_test.go` to assert the new files exist and that
  `go build ./...` works inside the generated app.
- Add `docs/framework/reference/starter/deployment.md`.
- No GHCR yet.
- Commit: `feat(starter): add Dockerfile and CI baseline`.

### Phase 4 — Publish Gofra and drop the `replace`

Inside `repos/gofra`:

- Extend CI to include `mise run smoke:new`.
- Cut tag `v0.1.0`, push.
- Rewrite `go.mod.tmpl` to `require github.com/Gabrielbdd/gofra v0.1.0`
  without any `replace`.
- Hardcode the framework module and version in `internal/scaffold/generator.go`.
  Remove `FrameworkDir`, `DetectFramework`, `LoadFramework`, and the
  `--framework-dir` flag in `cmd/gofra`.
- Update docs to match.
- Commit: `feat(scaffold): consume published gofra release`.
- Cut tag `v0.1.1` for the CLI/starter, push.

### Phase 5 — Scaffold Gospa

Inside the workspace:

- Run `go run github.com/Gabrielbdd/gofra/cmd/gofra@v0.1.1 new --module github.com/Gabrielbdd/gospa ./repos/gospa`.
- Commit the bootstrapped app inside `repos/gospa`.
- Advance the submodule SHA.
- Add `use ./repos/gospa` to the workspace `go.work`.

### Phase 6 — Image publish (off the critical path)

Add a GHCR workflow to the starter (and mirror to Gospa) only when Gospa
actually needs to distribute container images.

## Verification after Phase 5

- `rg -n "databit\.com\.br/gofra" repos/gofra` returns zero.
- `repos/gofra/internal/scaffold/starter/full/go.mod.tmpl` has no `replace`.
- `go test ./...` and `mise run smoke:new` pass inside `repos/gofra`.
- Outside the workspace: `go install github.com/Gabrielbdd/gofra/cmd/gofra@latest`
  followed by `gofra new /tmp/demo && cd /tmp/demo && mise trust --yes && mise run test && mise run build && docker build -t demo:ci .`
  all succeed.
- `go work edit -print` at the workspace root shows both submodules.
