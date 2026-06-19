# rehosting/ci — shared GitHub Actions for the rehosting org

Central home for the CI logic that every `rehosting/*` repo otherwise
copy-pastes: Harbor/Arc-registry setup, image build-and-push with the registry
buildx cache, prebuilt-image pulls for test jobs, PR-cache cleanup, the
guest-utility toolchain→tarball→release pipeline, and Nix install+caching.

This repo holds two kinds of reusable building block:

- **Composite actions** (`actions/*`) — reusable *steps* you drop into a job.
- **Reusable workflows** (`.github/workflows/*` with `on: workflow_call`) —
  reusable *whole jobs/pipelines* you call from a consuming workflow.

The design rationale, the cross-repo duplication survey, and the per-repo
migration plan are maintained internally (they reference private-repo CI
details) — ask in the rehosting org for the design/migration docs.

## Composite actions

| Action | Purpose |
|--------|---------|
| `rehosting/ci/actions/arc-registry-setup@v1` | Trust Harbor's self-signed cert, (optionally) set up insecure buildx, log in. Toggle `install-cert` / `setup-buildx` / `login` for build vs pull vs cleanup jobs. |
| `rehosting/ci/actions/pull-image@v1` | `docker pull` a prebuilt image into the local daemon and retag it for test jobs to `docker run`. Replaces the slow `buildx build --load` heredoc. |
| `rehosting/ci/actions/cleanup-pr-cache@v1` | Delete a `cache-PR-<n>` artifact from Harbor via its REST API on PR close. |
| `rehosting/ci/actions/nix-setup@v1` | Install Nix and configure a binary cache (GitHub-Actions cache and/or Cachix). The drop-in building block for running Nix in any repo. |

### Prerequisite for `pull-image`: the dind daemon must trust Harbor

`pull-image` does a plain `docker pull`, which the **Docker daemon** performs —
not the runner. So this is a property of the **runner pool**, not the action:

- `--insecure-registry` on dind is **not** enough — it doesn't cover the OAuth
  `/service/token` TLS leg.
- On Docker 24+ with the containerd image store (the default in **Docker 29**),
  the daemon uses Go's **system cert pool** and ignores `/etc/docker/certs.d`.
- The pool must therefore ship Harbor's **issuing CA** (not just the leaf it
  serves in the handshake) into the dind **system trust store** — e.g. mount the
  CA at `/usr/local/share/ca-certificates` and run `update-ca-certificates`
  **before** dockerd starts.

Without this, `pull-image` fails with `x509: certificate signed by unknown
authority` on the token fetch, and the action's runner-side `install-cert` will
not fix it (hence it defaults off). The `rehosting-arc` pool is already set up
this way.

## Reusable workflows

| Workflow | Purpose |
|----------|---------|
| `rehosting/ci/.github/workflows/build-and-push.yml@v1` | Build a Docker image and push to Harbor with the `:cache` / `:cache-PR-<n>` registry cache. Folds in the REGISTRY/USER/CACHE/TARGET/EXTERNAL_REGISTRY_PASS env ladder. |
| `rehosting/ci/.github/workflows/toolchain-release.yml@v1` | Build a tarball inside a shared embedded-toolchain image and cut a versioned GitHub release. For the guest-utility repos. |
| `rehosting/ci/.github/workflows/nix-release.yml@v1` | Install Nix (cached), `nix build`, tar the result, and cut a versioned release. The Nix counterpart of `toolchain-release`. |
| `rehosting/ci/.github/workflows/update-flake-lock.yml@v1` | Scheduled job that opens a PR bumping `flake.lock`. |

## Versioning & pinning

Releases are cut as immutable tags `vMAJOR.MINOR.PATCH` (e.g. `v1.4.2`). A
**moving major tag** `v1` is force-updated to point at the latest `v1.x.y` on
each release. Consumers pin to the major tag:

```yaml
uses: rehosting/ci/actions/arc-registry-setup@v1
```

- `@v1` — recommended for all repos. Gets non-breaking fixes automatically;
  breaking changes only arrive on an explicit bump to `@v2`.
- `@v1.4.2` — pin exact when a repo needs to freeze behavior (e.g. mid-release).
- `@main` — **never** in consuming repos; only for testing a change from within
  this repo's own example workflows.

Breaking changes (renamed/removed inputs, changed defaults that alter behavior)
require a new major tag. Everything else is a minor/patch on the current major.

Inside this repo, the composite actions and reusable workflows reference each
other by the **same major tag** (e.g. `pull-image` calls
`arc-registry-setup@v1`), so a consumer pinned at `@v1` always gets an
internally-consistent set. When developing, this self-reference is what you bump
last before tagging a release.

## Consuming-repo examples

### Build + push on a PR (replaces the inline build job)

```yaml
jobs:
  build_container:
    uses: rehosting/ci/.github/workflows/build-and-push.yml@v1
    with:
      image: rehosting/penguin
      build-args: APT_MIRROR=MIT
      cache-from-extra: |
        type=registry,ref=harbor.harbor.svc.cluster.local/external/rehosting/penguin:cache_last_published,mode=max
    secrets:
      registry: ${{ secrets.REHOSTING_ARC_REGISTRY }}
      registry-user: ${{ secrets.REHOSTING_ARC_REGISTRY_USER }}
      registry-password: ${{ secrets.REHOSTING_ARC_REGISTRY_PASSWORD }}
```

### Pull a prebuilt image in a test job

```yaml
jobs:
  run_tests:
    runs-on: rehosting-arc
    steps:
      - uses: actions/checkout@v5
      - uses: rehosting/ci/actions/pull-image@v1
        with:
          registry: ${{ secrets.REHOSTING_ARC_REGISTRY || 'harbor.harbor.svc.cluster.local' }}
          target: ${{ secrets.REHOSTING_ARC_REGISTRY || 'harbor.harbor.svc.cluster.local/external' }}
          image: rehosting/penguin
          tag: ${{ github.sha }}
          local-tag: rehosting/penguin:latest
          username: ${{ secrets.REHOSTING_ARC_REGISTRY_USER || 'external' }}
          password: ${{ secrets.REHOSTING_ARC_REGISTRY_PASSWORD || vars.EXTERNAL_REGISTRY_PASS }}
          install-python: "true"   # penguin's test runners are Python scripts
```

### Clean up the PR cache on close

```yaml
on:
  pull_request:
    types: [closed]
jobs:
  cleanup:
    if: ${{ github.event.pull_request.head.repo.full_name == github.repository }}
    runs-on: rehosting-arc
    steps:
      - uses: rehosting/ci/actions/cleanup-pr-cache@v1
        with:
          registry: ${{ secrets.REHOSTING_ARC_REGISTRY || 'harbor.harbor.svc.cluster.local' }}
          repository: penguin
          reference: cache-PR-${{ github.event.number }}
          username: ${{ secrets.REHOSTING_ARC_REGISTRY_USER || 'external' }}
          password: ${{ secrets.REHOSTING_ARC_REGISTRY_PASSWORD || vars.EXTERNAL_REGISTRY_PASS }}
```

### Guest-utility tarball release

```yaml
on:
  push:
    branches: [master]
jobs:
  release:
    uses: rehosting/ci/.github/workflows/toolchain-release.yml@v1
    with:
      artifact: busybox-latest.tar.gz
      build-command: /app/build.sh
      package-command: tar -czf busybox-latest.tar.gz build
      submodules: recursive
```

### Nix: just the setup (drop into any job)

The minimal way to run Nix in a repo — install + GitHub-Actions store cache:

```yaml
jobs:
  check:
    runs-on: rehosting-arc
    steps:
      - uses: actions/checkout@v4
      - uses: rehosting/ci/actions/nix-setup@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          extra-nix-config: |
            max-jobs = 8
            cores = 8
      - run: nix flake check
      - run: nix build
```

Cachix instead of (or as well as) the GitHub cache:

```yaml
      - uses: rehosting/ci/actions/nix-setup@v1
        with:
          cache-backend: cachix          # or "both"
          cachix-name: unblob
          cachix-auth-token: ${{ secrets.CACHIX_AUTH_TOKEN }}
          cachix-extra-pull-names: pyperscan
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Nix build → tarball → release (hyperfs)

```yaml
on:
  push:
    branches: [main]
jobs:
  release:
    uses: rehosting/ci/.github/workflows/nix-release.yml@v1
    with:
      artifact: hyperfs.tar.gz
      apt-packages: xz-utils curl p7zip-full jq sqlite3
      flake-check: true
```

### Scheduled flake.lock bump

```yaml
on:
  schedule: [{ cron: "0 0 * * 0" }]
  workflow_dispatch:
jobs:
  update:
    uses: rehosting/ci/.github/workflows/update-flake-lock.yml@v1
    with:
      pr-title: "Update flake.lock"
    secrets:
      token: ${{ secrets.CREATE_PR_TOKEN }}
```
