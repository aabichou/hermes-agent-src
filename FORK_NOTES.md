# Fork notes — `aabichou/hermes-agent-src`

Single-purpose fork of [`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent)
to fix a Python packaging bug that prevents `hermes dashboard` from
working when the package is installed from a git ref instead of as an
editable checkout.

This fork **only** carries packaging changes — the runtime, CLI, and
plugin code are unchanged from the upstream commit at the branch base.

## The bug

`pyproject.toml`'s `[tool.setuptools.packages.find]` include list
(line 240 on the patched branch base):

```toml
include = ["agent", "agent.*", "tools", "tools.*", "hermes_cli",
           "gateway", "gateway.*", "tui_gateway", "tui_gateway.*",
           "cron", "acp_adapter", "plugins", "plugins.*",
           "providers", "providers.*"]
```

`hermes_cli` appears top-level only, **not** as `hermes_cli.*`. So
`find_packages` matches only `hermes_cli/__init__.py` and skips every
subpackage — including `hermes_cli/dashboard_auth/`, which
`hermes_cli/web_server.py` imports at module top level:

```python
# hermes_cli/web_server.py:4822
from hermes_cli.dashboard_auth.routes import router as _dashboard_auth_router
```

Editable installs (`pip install -e .` / `uv pip install -e .`) work
because `find_packages` isn't consulted — the source tree itself is on
`sys.path`. Upstream's own `Dockerfile` uses an editable install
(`RUN uv pip install --no-cache-dir --no-deps -e "."`) so the bug
never trips for them.

Wheel installs from `git+...` (the standard `pip install hermes-agent@git+...`
path) crash on first dashboard request with:

```
ModuleNotFoundError: No module named 'hermes_cli.dashboard_auth'
```

## The patch

```diff
 [tool.setuptools.packages.find]
-include = ["agent", "agent.*", "tools", "tools.*", "hermes_cli", "gateway", "gateway.*", "tui_gateway", "tui_gateway.*", "cron", "acp_adapter", "plugins", "plugins.*", "providers", "providers.*"]
+include = ["agent", "agent.*", "tools", "tools.*", "hermes_cli", "hermes_cli.*", "gateway", "gateway.*", "tui_gateway", "tui_gateway.*", "cron", "acp_adapter", "acp_adapter.*", "plugins", "plugins.*", "providers", "providers.*"]
```

Two additions:
- `"hermes_cli.*"` — fixes the dashboard crash by including
  `hermes_cli/dashboard_auth/`, `hermes_cli/plugins/`, and any other
  subpackage upstream adds in the future.
- `"acp_adapter.*"` — same potential bug for the ACP adapter
  (it ships subdirectories too); added defensively even though no
  current consumer trips it.

## Verifying

```bash
# In a fresh venv:
uv pip install "hermes-agent[web] @ git+https://github.com/aabichou/hermes-agent-src@fix/packages-find-hermes-cli-subpackages"
python -c "from hermes_cli.dashboard_auth.routes import router; print('ok')"
# → ok
```

For comparison, the same command against upstream emits
`ModuleNotFoundError`.

## Consumed by

- [`aabichou/hermes-operator`](https://github.com/aabichou/hermes-operator) —
  `images/hermes-agent/pyproject.toml` pins this fork branch.
- The image built by that operator is deployed in the `homelab-k3s`
  repo at `clusters/tunis/hermes/`.

## Upstreaming

Worth a PR to `NousResearch/hermes-agent`. The change is one line,
risk-free for editable consumers (no behavior change — `find_packages`
already discovers these dirs in editable mode), and unblocks every
non-editable consumer.
