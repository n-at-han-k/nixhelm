# AGENTS.md

## Project Overview

nixhelm is a Nix flake that provides Helm charts as Nix derivations. Charts are defined as simple Nix attribute sets in `charts/<repo_name>/<chart_name>/default.nix` and are auto-discovered by the [haumea](https://github.com/nix-community/haumea) module loader. The `helmupdater` Python CLI tool manages chart creation, updates, and hash computation.

## Repository Structure

```
charts/
  <repo_name>/
    <chart_name>/
      default.nix       # The only file per chart -- a 4-field Nix attribute set
flake.nix               # Flake definition; uses haumea to load all charts
src/helmupdater/        # Python CLI for managing charts
```

## Chart Definition Format

Every chart is a `default.nix` with exactly 4 fields:

```nix
{
  repo = "<repository URL>";
  chart = "<chart name>";
  version = "<version string>";
  chartHash = "<SRI hash, e.g. sha256-...>";
}
```

HTTP repository example (`charts/argoproj/argo-cd/default.nix`):

```nix
{
  repo = "https://argoproj.github.io/argo-helm/";
  chart = "argo-cd";
  version = "9.4.3";
  chartHash = "sha256-4f1moa/OOwcMb+bjeTPqIHNcUfgs4eF3yzPDWRTQ5so=";
}
```

OCI registry example (`charts/forgejo-helm/forgejo/default.nix`):

```nix
{
  repo = "oci://code.forgejo.org/forgejo-helm";
  chart = "forgejo";
  version = "16.2.0";
  chartHash = "sha256-TAeZbC8+n22kjkdR3gaQn1nUNar/m984OuoKPqf3cwc=";
}
```

The format is identical for HTTP and OCI -- only the `repo` URL scheme differs (`https://` vs `oci://`).

## Adding a New Helm Chart

Run the `helmupdater init` command from the repo root:

```sh
nix run .#helmupdater -- init <REPO_URL> <REPO_NAME>/<CHART_NAME> --commit
```

### HTTP Repository

```sh
nix run .#helmupdater -- init "https://prometheus-community.github.io/helm-charts" prometheus-community/prometheus --commit
```

### OCI Registry

```sh
nix run .#helmupdater -- init "oci://ghcr.io/myorg/charts" myorg/nginx --commit
```

### What Happens Under the Hood

1. `<REPO_NAME>/<CHART_NAME>` is split into a repo name and chart name.
2. The directory `charts/<repo_name>/<chart_name>/` is created.
3. A placeholder `default.nix` is written with version `0.0.0` and a dummy hash.
4. The file is staged with `git add` (Nix flakes only see git-tracked files).
5. The registry is queried for available versions (HTTP fetches `index.yaml`; OCI lists tags).
6. The latest stable version is selected (pre-release versions are filtered out).
7. The correct `chartHash` is computed by triggering a `nix build`, capturing the hash mismatch error, and extracting the real hash from the error output.
8. The final `default.nix` is written with the correct version and hash.
9. If `--commit` was passed, the change is committed with message `<repo>/<chart>: init at <version>`.

## Other Helmupdater Commands

| Command | Description |
|---|---|
| `init <repo_url> <repo/chart> [--commit]` | Create a new chart definition (described above). |
| `update <repo/chart> [--commit] [--build]` | Update a single chart to the latest stable version. |
| `update-all [--commit] [--build]` | Update all charts to their latest stable versions. |
| `rehash <repo/chart> [--commit] [--build]` | Recompute the hash without changing the version. |
| `build <repo/chart>` | Build the Nix derivation for a chart. |

All commands are invoked as `nix run .#helmupdater -- <command> [args]`.

## Important Notes

- Files must be git-tracked for Nix to see them. The `init` command handles this automatically.
- Only stable versions are selected. Pre-release versions (alpha, beta, rc, dev) are skipped.
- The `chartHash` is an SRI hash computed automatically -- never write it by hand.
- Charts hosted in git repos can be fetched as flake inputs directly instead of being added here (see [this issue](https://github.com/farcaller/nixhelm/issues/10)).
- Charts are updated nightly via GitHub Actions.
