# navi-core reference — standalone installation

For operators installing navi directly (not needed when running under
navi-mcp, where install + keys are handled for you). Covers Python/Docker
setup, API key entry, key-permission scoping, version detection, and
standalone post-upgrade recovery.

## Standalone installation

**This section is for operators installing navi directly, not for end-users
running under navi-mcp.** Under navi-mcp, these steps have already been
done for you.

### Python requirement

Navi requires **Python 3.12 or higher**. This is the most common first-run blocker.

```bash
# Check Python version first
python3 --version

# If below 3.12, install via pyenv (recommended — avoids system Python conflicts)
curl https://pyenv.run | bash
pyenv install 3.12
pyenv global 3.12

# Or via Homebrew on macOS
brew install python@3.12
echo 'export PATH="/usr/local/opt/python@3.12/bin:$PATH"' >> ~/.zshrc

# Or via apt on Ubuntu/Debian
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update && sudo apt install python3.12 python3.12-pip

# Then install navi
pip3 install navi-hostio
```

### Docker — build from source (recommended over the published image)

The published Docker image on Docker Hub is outdated. Build from the latest
GitHub source instead:

```bash
# Clone the latest navi source
git clone https://github.com/packetchaos/navi.git
cd navi

# Build a fresh image from the current Dockerfile
docker build -t navi:latest .

# Run navi in a container — mount a local directory to persist navi.db
docker run -it \
  -v $(pwd)/navi-data:/home/navi \
  navi:latest \
  navi config keys --a <ACCESS_KEY> --s <SECRET_KEY>
```

Mounting a local volume (`-v`) is critical — otherwise the navi.db is lost
when the container stops.

### API keys — set these BEFORE running any other command

**The most common mistake**: running `navi config update full` or any other
command before setting API keys. Navi will appear to run but return no data.

```bash
navi config keys --a <ACCESS_KEY> --s <SECRET_KEY>
```

API keys must be set first. Every other command depends on them.

### API key permissions matter

Navi can only see what your API key can see. If the API key is scoped to a
subset of assets (e.g. a specific network or business unit), `navi config
update full` will only download data for those assets. This is intentional
and useful for scoped workloads — but it is also the cause of "why am I
missing assets?" questions.

**"Zero chunks" can have several causes** — one of them is the API key
having no permission to the data being requested. (Other causes: a lookback
window too short to contain new findings, scanners that have stopped
producing data.) Check the key first when investigating:

- Does the key have access to the assets/vulnerabilities you expect?
- Is the key scoped to ALL assets, or a subset?
- If you need full environment coverage, the key must have permissions on
  ALL assets.

For the full diagnostic ladder when this happens — including the
`--days 365` backfill pattern and scanner-health check — see
navi-troubleshooting's "Zero chunks on update commands".

### Check version after install

```bash
navi explore info version
```

Under navi-mcp, the same check is available as
`navi_explore_info(subcommand="version")`.

- **8.2+**: use `navi enrich tag`, `navi explore data query`, `navi explore uuid`
  (single-asset detail; note it is `explore uuid <IP_or_UUID>`, NOT `explore asset`)
- **Pre-8.2**: use `navi tag`, `navi find query`

Default to 8.2+ syntax unless the user confirms an older version.

### After upgrading navi to a new version (standalone)

Navi version upgrades cause a database schema mismatch. When this happens:

```bash
# 1. Delete the old database
rm navi.db

# 2. Re-enter API keys (they are stored in navi.db)
navi config keys --a <ACCESS_KEY> --s <SECRET_KEY>

# 3. Re-sync data
navi config update full

# 4. Rebuild indexes (navi 8.5.31+) — config update full does not build them
#    and they were dropped when navi.db was deleted
navi config optimize
```

This is expected behaviour — not a bug. For MCP users, see the five-step
recovery under "Setup under navi-mcp" above.

