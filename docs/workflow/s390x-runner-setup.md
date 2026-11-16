# s390x GitHub Actions self-hosted runner

The `s390x CI` workflow (`.github/workflows/s390x-ci.yml`) runs on a self-hosted
runner with labels `self-hosted, linux, s390x`. GitHub does not provide s390x
hosted runners, so you need a real s390x Linux machine (LinuxONE / z VM).

## 1. Prerequisites on the s390x host

- Linux s390x (Ubuntu 22.04+/RHEL 9+), 16+ GB RAM, 100+ GB disk
- `git`, `cmake`, `clang`, `lld`, `make`, `python3`, `libicu-dev`, `libssl-dev`,
  `libkrb5-dev`, `zlib1g-dev`, `liblttng-ust-dev`, `curl`, `jq`
- A **s390x .NET SDK** (the runtime's `build.sh` bootstrap SDK has no s390x build;
  install a locally built/distro one and set `DOTNET_ROOT`/`PATH` for the runner user)
- `src/s390tests/test.sh` also needs `dotnet` on PATH (`dotnet new console`)
- A dedicated non-root user, e.g. `runner`

## 2. Build the runner with IBM/action-runner-image-pz

GitHub publishes no linux-s390x runner binary. IBM's
[action-runner-image-pz](https://github.com/IBM/action-runner-image-pz) builds the
runner from source with s390x patches (`patches/runner-sdk8-s390x.patch`, falling
back to `patches/runner-main-sdk8-s390x.patch`) and sets up the host or container.

On the s390x host (Ubuntu 22.04/24.04 recommended; sudo required):

```bash
git clone https://github.com/IBM/action-runner-image-pz.git
cd action-runner-image-pz
export GITHUB_TOKEN=<a-github-token>   # optional, avoids API rate limits
bash run.sh                            # interactive: pick environment, OS, Minimal/Complete
```

Non-interactive example (complete Ubuntu 24.04 Incus container):

```bash
sudo ./scripts/incus_container.sh ubuntu 24.04 "" "" complete --use-cloud-img
```

Choose the **VM (host machine)** environment (`scripts/vm.sh`) if you want the runner
directly on the host, which is simplest for a long runtime build. Use **Complete**
setup to get the toolchain (cmake, clang, etc.). Note that the s390x .NET SDK
availability differs by OS; verify `dotnet --version` afterwards because
`src/s390tests/test.sh` and the runtime build need it.

## 3. Register the runner on your fork

The IBM README does not cover registration. Use the standard flow with the binary
the scripts built (check the script output / runner directory for its location):

1. Get a token: https://github.com/shreyarao4/runtime/settings/actions/runners/new
2. In the runner directory, as a non-root user:

```bash
./config.sh --url https://github.com/shreyarao4/runtime \
            --token <TOKEN_FROM_GITHUB> \
            --labels s390x,linux --name s390x-runner-1 --unattended
sudo ./svc.sh install <user>
sudo ./svc.sh start
```

Upgrade later by re-running `bash run.sh` (VM mode keeps registration; rebuild
container/Incus setups instead).

## 4. Security

Self-hosted runners execute PR code. Keep the fork **private**, or require
approval for fork PR workflows (Settings -> Actions -> General -> "Require approval
for all outside collaborators"). Never use this runner on a public repo
that accepts untrusted PRs.

## 5. Porting to giritrivedi/runtime

Copy `.github/workflows/s390x-ci.yml` unchanged; register a runner (or runner
group) on that repo/org with the same labels.
