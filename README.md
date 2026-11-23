# telegram-bot-api-build

Reproducible, pinned builds of [tdlib/telegram-bot-api](https://github.com/tdlib/telegram-bot-api)
for x86_64 Linux (glibc), published as release binaries.

Upstream ships **no tags, no releases, and no prebuilt Linux binaries** — a local Bot API
server normally means compiling a fork of master on the machine that runs it. This repo
turns that into: one pinned commit, one runner, one SHA256 you can verify.

## Current pin

| | |
|---|---|
| tdlib commit | `5df614c91dab8662cfbb24a501ec89949fdcfebb` |
| Bot API | **10.5** ("Update version to 10.5.", 2026-11-21) |
| Runner | `ubuntu-24.04` (glibc 2.39) |
| Compiler | `g++` 13, `-DCMAKE_BUILD_TYPE=Release` |

The runner is pinned to `ubuntu-24.04`, not `ubuntu-latest`: the binary links glibc
dynamically, so a runner that silently rolls forward to 26.04 (glibc 2.41) produces a
binary that refuses to start on 24.04 hosts.

## Triggering a build

- **Manual** — Actions → `build-telegram-bot-api` → Run workflow. Leave `tdlib_ref` blank
  to use the pin, or pass a full commit SHA to build something else.
- **Monthly** — cron on the 1st, so the artefact does not rot against Ubuntu security updates.
- **On push to `main`** — only when the workflow file itself changes.

Each run publishes a tag `tdlib-<version>-<shortsha>` with:

```
telegram-bot-api                                  # raw binary
telegram-bot-api-<ver>-linux-x86_64.tar.gz
SHA256SUMS
BUILD-INFO.txt                                    # commit, compiler, build host
```

## Upgrading the pin

1. Find the release commit upstream:
   `git ls-remote https://github.com/tdlib/telegram-bot-api` — or search commits for
   `"Update version to"`.
2. Set `PINNED_TDLIB_SHA` in `.github/workflows/build.yml` **and** the table above.
3. Push — that commit touches the workflow path, so it builds immediately.
4. Confirm the `Verify version` step reports the version you expected. It parses
   `telegram-bot-api --version` and fails the job if it cannot.

Bumping the pin is the only way the version changes. Nothing floats.

## Installing on a systemd host

```sh
VER=10.5
TAG="tdlib-${VER}-5df614c"          # exact tag from the release page
BASE="https://github.com/aiskendi/telegram-bot-api-build/releases/download/${TAG}"

cd /tmp && rm -rf tba && mkdir tba && cd tba
curl -fsSLO "${BASE}/telegram-bot-api"
curl -fsSLO "${BASE}/SHA256SUMS"
sha256sum -c SHA256SUMS --ignore-missing        # must print "OK"

# back up the running binary, then swap
sudo cp -a /usr/local/bin/telegram-bot-api "/usr/local/bin/telegram-bot-api.bak.$(date +%Y%m%d_%H%M%S)"
sudo install -m 0755 -o root -g root telegram-bot-api /usr/local/bin/telegram-bot-api
/usr/local/bin/telegram-bot-api --version

sudo systemctl restart telegrambotapi.service
```

Rollback is the inverse: `sudo install -m 0755 telegram-bot-api.bak.<stamp> /usr/local/bin/telegram-bot-api`
then restart.

## Why not a container

A container does not make the same binary do less work, and Alpine's musl allocator is a
worse fit for a long-running threaded C++ daemon than glibc — the current one grows ~50 MB/h
and is recycled by a cgroup memory watchdog. The container's real value would be rollback,
which publish-a-SHA already gives you here. Target hosts stay on systemd and keep that
watchdog.
