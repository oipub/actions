# OiPub Actions

Reusable GitHub Actions workflows shared by the OiPub service repositories.
They are consumed with `uses: oipub/actions/.github/workflows/<name>@main`.

## Container registry

The registry host is **not hardcoded anywhere in this repository**. It comes
from the `CR_REGISTRY` configuration variable, which each caller passes:

```yaml
with:
  registry: ${{ vars.CR_REGISTRY }}        # image jobs
  imageRegistry: ${{ vars.CR_REGISTRY }}   # deploy job
```

Set `CR_REGISTRY` once as an **organization variable** so a registry move is a
single change; a repository variable of the same name overrides it if one
service ever needs to publish somewhere else.

The input is required and has no default. Because an unset `vars.*` resolves to
an empty string — which GitHub still accepts as a "provided" required input —
every job that uses the registry guards on it and fails fast with a clear
message rather than pushing to a malformed reference.

Registry credentials come from the `CR_USERNAME` / `CR_PASSWORD` secrets, which
each calling repository forwards. The host is a variable rather than a secret
deliberately: masking it would redact it from build logs and from the deploy
job's diff output, which is where you look when a deploy goes wrong.

## Workflows

### `git-version.job.yml`

Derives the next semantic version from git history via `codacy/git-version`.
Read-only.

| | |
|---|---|
| Inputs | `baseBranch` (default `main`), `runsOn` (default `ubuntu-24.04`) |
| Outputs | `version` |
| Permissions | `contents: read` |

### `cr-docker-image-dotnet.job.yml`

Stamps `AppVersion.cs` with the release version, commits it back to the branch,
then builds and pushes the service image.

| | |
|---|---|
| Inputs | `registry`*, `imageName`*, `imageTag`*, `githubUsername`, `context`, `dockerfile`, `runsOn` |
| Secrets | `CR_USERNAME`*, `CR_PASSWORD`*, `GITHUB_PAT` (private NuGet restore) |
| Outputs | `image`, `digest` |
| Permissions | `contents: write` (for the version stamp commit) |

### `cr-docker-image-react.job.yml`

Builds and pushes the React app image. No git writes.

| | |
|---|---|
| Inputs | `registry`*, `imageName`*, `imageTag`*, `context`, `dockerfile`, `runsOn` |
| Secrets | `CR_USERNAME`*, `CR_PASSWORD`* |
| Outputs | `image`, `digest` |
| Permissions | `contents: read` |

### `git-tag.job.yml`

Tags the built commit, skipping the tag if it already exists.

| | |
|---|---|
| Inputs | `version`*, `runsOn` |
| Permissions | `contents: write` |

### `npm-version.job.yml`

Writes the released version into `package.json` and pushes it.

| | |
|---|---|
| Inputs | `version`*, `nodeVersion` (default `22`), `runsOn` |
| Permissions | `contents: write` |

### `deploy.job.yml`

Pins the new tag in the deployment repository's compose file and pushes. That
push is what triggers the actual deploy.

| | |
|---|---|
| Inputs | `deploymentRepo`*, `serviceName`*, `imageRegistry`*, `imageName`*, `imageTag`*, `deploymentOwner` (default `oipub`), `composeFile`, `runsOn` |
| Secrets | `GITHUB_PAT`* (push access to the deployment repo) |
| Permissions | none — it works entirely over the PAT |

**`serviceName` is a protocol, not a label.** It becomes the commit subject
prefix (`<serviceName>: Updated the Docker image version to <tag>`), which
`test-env-deploy` parses to choose which `deploy-*.sh` to run on the VM.
Changing it silently downgrades a targeted deploy to `deploy-all.sh`.

The compose rewrite matches on the *image name*, not the registry host, so a
registry migration rewrites the line rather than silently matching nothing. If
no line changes, the job fails loudly and prints the compose file's image lines.

## Conventions

- Runners are pinned to `ubuntu-24.04` rather than `ubuntu-latest`, so a
  GitHub-side `-latest` migration cannot change build behaviour unannounced.
  Every job takes a `runsOn` input if a caller needs to override it.
- Images are single-platform `linux/amd64`. No QEMU is set up, because nothing
  cross-builds and the runners are already amd64.
- Docker layer caching uses the GitHub Actions cache, scoped per image name so
  services do not evict each other.
- Secrets are passed to shell steps through `env:`, never interpolated into the
  script body.
- The jobs that push to a branch (`cr-docker-image-dotnet`, `npm-version`,
  `deploy`) rebase and retry on a rejected push, since several pipelines can
  land on the same branch at once.

## Troubleshooting

**"No container registry configured".** The `CR_REGISTRY` variable is unset, or
is set somewhere the calling repository cannot see it (an organization variable
restricted to selected repositories, for instance).

**Image push fails with a manifest or media-type error.** Buildx attaches
provenance attestations by default. If the registry rejects them, add
`provenance: false` to the `docker/build-push-action` step in the two
`cr-docker-image-*` jobs.

**Deploy job fails with "No image line for ... was updated".** The `imageName`
input does not match any `image:` line in the deployment repo's compose file.
The job log prints the lines it found.
