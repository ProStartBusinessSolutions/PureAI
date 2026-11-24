# Deploying PureAI

How PureAI gets from this repository to the DGX Spark.

**Status: plan.** The repository changes in step 1 are still to do.

## How it fits together

```
commit + tag vX.Y.Z.B ──► build ARM64 image ──► ghcr.io/prostartbusinesssolutions/pureai ──► Portainer on the Spark
                          (GitHub Actions,         (private package)                        "Pull and redeploy"
                           or a developer laptop)
```

- **Hardware:** the DGX Spark is ARM64 with an NVIDIA GB10 GPU, so the image is built for `linux/arm64` only.
- **What's public:** the image is private, and the code is public in this repository.
- **Deploying:** Portainer on the Spark pulls the image with its own registry login and runs it as a stack.
- **Data:** everything PureAI keeps (accounts, settings, memory, models, keys) lives in `/PureAI/Data` on the Spark, never in the image.
- **llama.cpp:** the CUDA build isn't compiled here. It's copied from A·R·I's public image `ghcr.io/ari-labz/llama-cpp-cuda:<ref>`, so a build is mostly the .NET server and the web UI. To compile llama.cpp ourselves instead, add `--build-arg LLAMA_IMAGE=llama-bin` to the build (about 20 minutes).

## Rules

- **Only deploy what's pushed and tagged.** Every build comes from a commit that's pushed to this repository and tagged `vX.Y.Z.B`, for example `v0.11.0.1`, and the image carries the same version. The source of whatever is running is then always public, which is what the AGPL asks of a hosted, modified version.
- **Four-part tags are ours.** A·R·I tags its releases with three parts (`v0.11.0`), so the two never clash.
- **Nothing ProStart-specific goes in the repository or the image.** This repository is public. Accounts, tokens, API keys and the PureBill connection are set up in the control panel, and they live in `/PureAI/Data`.
- **Keep the credit.** The About screen and the startup log keep the A·R·I credit. See [NOTICE](NOTICE).

## 1. Repository changes (one-time, to do)

| File | Change | Why |
|---|---|---|
| `.github/workflows/docker.yml` | Set `IMAGE` to `ghcr.io/prostartbusinesssolutions/pureai`. Run on `v*.*.*.*` tags and by hand only, not on every push to `main`. | Publishes our image, and only for tagged builds. |
| `Dockerfile` | Point the `org.opencontainers.image.source` label at this repository. | Links the package to PureAI rather than A·R·I. |
| `Dockerfile` | Run the .NET and web UI build stage on the build machine's own platform (`FROM --platform=$BUILDPLATFORM`), cross-compiling for `linux-arm64`. | Lets an x64 laptop build the ARM64 image quickly. .NET under emulation is slow and unreliable. GitHub Actions builds on a native ARM64 runner, so it doesn't need this. |
| `docker/compose.pureai.yml` | New file: the Portainer stack (below). | |
| `ARI.UI/public/controlpanel.html` | About screen: add PureAI as entry 2 of the credit chain, with a link to this repository. | The credit chain, and the AGPL's source offer to people using PureAI over the network. |

## 2. One-time setup

### GitHub

1. Turn on Actions for this repository, then turn off `release.yml` and `llama-cuda.yml`:

       gh workflow disable release.yml
       gh workflow disable llama-cuda.yml

   `release.yml` builds A·R·I's desktop apps (on macOS runners too), and `llama-cuda.yml` publishes to A·R·I's registry. Only `docker.yml` should run.
2. Do the first release (step 3). The first push creates the `pureai` package under the organisation's Packages.
3. Check the package shows as **Private**. If it doesn't, change it in the package settings.
4. If the package was first created from a laptop rather than by the workflow: open the package settings, go to **Manage Actions access**, and give this repository **Write**.

ARM64 runners are free for public repositories, and GitHub doesn't currently charge for Container registry storage.

### Tokens

| Used by | Token | Scope |
|---|---|---|
| GitHub Actions | `GITHUB_TOKEN` (automatic) | `packages: write`, already set in the workflow |
| A developer laptop | Classic personal access token | `write:packages` |
| Portainer on the Spark | Classic personal access token from a named account | `read:packages` only |

The Container registry only accepts classic personal access tokens.

### The Spark

1. Check Docker can see the GPU:

       docker run --rm --gpus all nvidia/cuda:13.0.1-base-ubuntu24.04 nvidia-smi

2. Create the data folders on the internal drive:

       sudo mkdir -p /PureAI/Data /PureAI/Projects

3. **Portainer → Registries → Add registry.** Choose **GitHub**, or **Custom registry** with the URL `ghcr.io`. Turn authentication on. The username is the named account and the password is its `read:packages` token.
   A `docker login` in the Spark's terminal doesn't help: Portainer only uses its own registry logins.
4. **Portainer → Stacks → Add stack.** Name it `pureai`, paste in `docker/compose.pureai.yml`, then deploy it.
5. The container's log shows the first-run setup token. Use Portainer's **Logs** for the container, or run `docker logs pureai`.
   Then open `http://<spark>:5074`, create the admin account, download a model and set up the agents.
6. Allow port 5074 from the office network.

### The stack: `docker/compose.pureai.yml`

```yaml
name: pureai

services:
  pureai:
    image: ghcr.io/prostartbusinesssolutions/pureai:${PUREAI_TAG:-latest}
    container_name: pureai
    restart: unless-stopped
    pull_policy: always
    ports:
      - "${PUREAI_PORT:-5074}:5074"
    environment:
      TZ: Europe/London
      UMASK: "022"            # only the PureAI user can write to /PureAI/Data
      PUID: ${PUID:-10001}
      PGID: ${PGID:-10001}
    volumes:
      - /PureAI/Data:/data
      - /PureAI/Projects:/projects
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    ulimits:
      memlock: -1             # lets llama.cpp pin memory
    stop_grace_period: 45s    # time for the model servers to shut down cleanly
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://127.0.0.1:5074/api/v1/health >/dev/null || exit 1"]
      interval: 30s
      timeout: 5s
      start_period: 90s
      retries: 3
```

`latest` always means the newest release, so a normal update is just **Pull and redeploy**. To pin a version, set `PUREAI_TAG` in the stack's environment variables.

## 3. Releasing a version

1. Get `main` to the state you want to release, and push it.
2. Set `version` in `manifest.json` to the new version (for example `0.11.0.1`). Commit and push it.
3. Tag the commit and push the tag:

       git tag v0.11.0.1
       git push origin v0.11.0.1

4. Back up `/PureAI/Data` on the Spark (see Backups).
5. Build and push the image:
   - **GitHub Actions (the normal route).** The tag starts `docker.yml`. It builds on a native ARM64 runner and pushes `pureai:0.11.0.1` and `pureai:latest`.
   - **A developer laptop (the fallback).** Start Docker Desktop, then run these from the tagged checkout:

         git submodule update --init --depth 1 Submodules/SmartReader
         docker login ghcr.io -u <github-user>
         docker buildx build --platform linux/arm64 -t ghcr.io/prostartbusinesssolutions/pureai:0.11.0.1 -t ghcr.io/prostartbusinesssolutions/pureai:latest --push .

6. On the Spark, go to **Portainer → Stacks → pureai** and choose **Pull and redeploy**.
7. Check that:
   - the container shows as healthy;
   - the control panel's About screen shows the new version;
   - the first lines of the log show the version and the A·R·I credit.

## Rolling back

1. In the stack's environment variables, set `PUREAI_TAG` to the previous version, for example `0.11.0.1`.
2. Update the stack.
3. If the newer version changed anything in `/PureAI/Data`, restore the backup taken before the release.
4. Once a fixed release is out, set `PUREAI_TAG` back to `latest`.

## Bringing in A·R·I updates

```
git remote add upstream https://github.com/ari-labz/A.R.I.git    # once
git fetch upstream
git merge upstream/main          # or a specific A·R·I release commit
```

- **Merge, never rebase.** A normal merge keeps every upstream commit and its author. Never rebase, squash or re-author upstream commits (ADDITIONAL-TERMS.md, section 4).
- **Keep our entries.** If `README.md`, `NOTICE` or the About screen conflict, keep both credit entries: A·R·I's first, then PureAI's.
- **Then release as usual.** Build, test, and release as in step 3, for example as `v0.12.0.1`.

## Backups

Back up `/PureAI/Data` before each release, and on a schedule. It holds the accounts, settings, the memory vault and the keys. You can leave out `/PureAI/Data/Models`, because models can be downloaded again.

## Don't

- **Don't use Watchtower.** A·R·I's own compose file uses it, but here **Pull and redeploy** is the update.
- **Don't put a self-hosted Actions runner on the Spark.** This repository is public, so anyone's pull request could run code on it.
- **Don't turn on `release.yml` or `llama-cuda.yml`.**
- **Don't push PureAI images to a public registry.**
