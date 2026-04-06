# aia-jail

A single-file Bash launcher that runs the [`aia`](https://github.com/MadBomber/aia) AI assistant CLI inside a Docker sandbox. Ruby and all gem dependencies stay off your host machine, and `aia` only has access to the directory from which `aia-jail` is launched plus any Docker containers defined within that same directory.

## How it works

On first run, `aia-jail` builds a Docker image from an embedded Dockerfile (Ruby 3.3 on Debian Bookworm with `aia` and common CLI tools pre-installed). Subsequent runs reuse the image. If you edit the script — for example to add packages — the image is automatically rebuilt via `md5sum` change detection.

### Docker-in-Docker (DinD) isolation

`aia-jail` spins up a Docker-in-Docker sidecar so that any `docker` or `docker compose` commands run inside the `aia` container talk to an **isolated Docker daemon**, not the host's. The sidecar:

- cannot see host containers
- cannot mount the host filesystem (beyond the current working directory)
- cannot access the host Docker socket

#### Per-directory isolation

The DinD sidecar container and its private bridge network are named with a 12-character hash of the current working directory:

```
aia-jail-dind-<hash>
aia-jail-net-<hash>
```

This means you can run `aia-jail` simultaneously from different directories without any name collisions.

#### Automatic cleanup

When `aia-jail` exits it automatically stops and removes both the DinD sidecar container and the private network via an `EXIT` trap. No leftover containers or networks accumulate on your host.

## Prerequisites

- Docker installed and running
- `jq` installed on the host (used for OpenRouter key auto-detection)

## Installation

Copy the script to somewhere on your `$PATH` and make it executable:

```bash
cp aia-jail ~/.local/bin/aia-jail
chmod +x ~/.local/bin/aia-jail
```

## Usage

```
aia-jail [OPTIONS] [-- ARGS...]

Options:
  -r, --rebuild    Force rebuild of the Docker image
  -s, --shell      Start a bash shell instead of aia
  -h, --help       Show this help message

Arguments after -- are passed through to aia.
```

### Examples

```bash
# Run aia interactively (builds image on first run)
aia-jail -- --chat

# Pass arguments directly to aia
aia-jail -- --model claude-3-5-sonnet "Tell me a joke"

# Open a shell inside the container
aia-jail --shell

# Force a full image rebuild
aia-jail --rebuild
```

If `--shell` is used while a container from the same working directory is already running, the script `docker exec`s into it rather than starting a new one (and the DinD cleanup trap is suppressed so the sidecar keeps running).

## Configuration

### API keys

Set at least one provider key before running:

| Variable | Provider |
|---|---|
| `ANTHROPIC_API_KEY` | Anthropic (Claude) |
| `OPENAI_API_KEY` | OpenAI (GPT) |
| `GOOGLE_API_KEY` | Google (Gemini) |
| `OPEN_ROUTER_API_KEY` | OpenRouter |


##### Special Hack
AIA doesn't store auth credentials and only reads from ENV vars (see table above).

For example, personally, I have OpenRouter creds stored in the opencode auth.json file. 
If `OPEN_ROUTER_API_KEY` is not set but `~/.local/share/opencode/auth.json` exists, the key is read from it automatically — so credentials are shared seamlessly with [OpenCode](https://opencode.ai) and with each new instance of `aia-jail`.

### Adding packages

To install extra packages into the image, edit the `CUSTOM_APT_PACKAGES` variable near the top of the script:

```bash
CUSTOM_APT_PACKAGES="git tmux sqlite3"
```

Save the file and the image will rebuild automatically on the next run.

## Volume mounts

The following host paths are mounted into every container:

| Host | Container | Purpose |
|---|---|---|
| `$(pwd)` | same path | Current working directory (path-mirrored for DinD compatibility) |
| `~/.prompts` | `/home/aia/.prompts` | aia prompt files |
| `~/.config/aia` | `/home/aia/.config/aia` | aia configuration |
| `~/.local/share/opencode` | `/home/aia/.local/share/opencode` | OpenCode data and auth |

> **Note:** The Docker socket (`/var/run/docker.sock`) is **not** mounted. All Docker operations go through the isolated DinD sidecar over TCP on the private bridge network.

The current directory is mounted at its exact host path so that any nested `docker` or `docker compose` commands launched from inside `aia` resolve volume paths correctly on the host. The same path is also mounted into the DinD sidecar for the same reason.

## Included tools

The Docker image ships with these CLI tools alongside `aia`:

- `ripgrep` — fast file content search
- `fzf` — fuzzy finder
- `jq` — JSON processor
- `gh` — GitHub CLI
- `docker` CLI and `docker compose` plugin
- `curl`, `build-essential`

## Testing Docker access

A `docker-compose.yml` and `docker-compose.test.yml` are included to verify that `aia-jail` can reach Docker containers and services defined in the same directory, using the isolated DinD daemon.

The compose file defines two services:

| Service | Purpose |
|---|---|
| `test-web` | A minimal Alpine HTTP server listening on port 8080 |
| `docker-access-test` | A test runner that checks volume mounts and compose service networking |

Open a shell inside the `aia-jail` container from the project directory:

```bash
aia-jail --shell
```

From inside the container, run the test suite:

```bash
docker compose run --rm docker-access-test
```

Expected output:

```
=== DinD Docker Compose Test ===

--- Test 1: Volume mount (project files visible) ---
[PASS] docker-compose.test.yml found via volume mount

--- Test 2: Compose service networking ---
[PASS] Got response from test-web: OK
```

The `docker-compose.test.yml` file is a sentinel used by the test runner to confirm that the host working directory is correctly mounted inside the DinD-managed container.

When you exit the shell, the DinD sidecar and private network are automatically cleaned up.

## License

See [LICENSE](LICENSE) if present, or check the repository for details.
