# Upstream issue draft for math-inc/OpenGauss

Draft of the bug report filed (or to be filed) at https://github.com/math-inc/OpenGauss/issues, summarising the install problems this guide works around.

---

**Title:** `install.sh fails with 404 on template; local install has several follow-on issues (macOS)`

## Summary

On a fresh macOS clone, `./scripts/install.sh` fails before installing anything because the shared Morph template no longer resolves. Running `./scripts/install-internal.sh` directly works, but surfaced a few more problems along the way. Details and the workarounds I used are below.

**Environment:** macOS 14 (Apple Silicon, M1 Pro), OpenGauss `main` (gauss-agent 0.2.2), Claude Code 2.1.322, Nov 2026.

## 1. `install.sh` → 404 on the shared template

```
Running Open Gauss installer flow locally from target: opengauss
Error: Template YAML file does not exist: /Users/<me>/OpenGauss/opengauss. Also failed to fetch shared template YAML from https://morph.new/opengauss/yaml: 404 Not Found
```

The pinned target from the README also returns 404:

```
$ curl -s -o /dev/null -w "%{http_code}\n" https://morph.new/opengauss-0-2-2/yaml
404
```

**Workaround:** `./scripts/install-internal.sh --with-workspace --run-setup`. Pointing `OPEN_GAUSS_INSTALL_TARGET` at `.github/morph/opengauss-template.yaml` may also work, but I didn't test it.

## 2. npm-installed `claude` is a non-executable placeholder

The installer's Claude Code install left `~/.local/bin/claude` as a symlink to a text file, so the managed-workflow prewarm crashed:

```
$ file ~/.local/bin/claude
/Users/<me>/.local/bin/claude: ASCII text
$ ls -l ~/.local/bin/claude
... ~/.local/bin/claude -> ../lib/node_modules/@anthropic-ai/claude-code/bin/claude.exe

gauss_cli.autoformalize.AutoformalizeStagingError: Failed to register the Lean4 Claude marketplace in the user profile: [Errno 8] Exec format error: '/Users/<me>/.local/bin/claude'
```

**Workaround:** remove it, install natively (`curl -fsSL https://claude.ai/install.sh | bash`), and rerun the installer. Prewarm and `/prove` staging then succeed. A check like `file "$(command -v claude)"` before prewarm would catch this.

## 3. "Lean workspace created" but Mathlib was not downloaded

With `--with-workspace`, the installer printed `info: downloading mathlib lean-toolchain file` and `✓ Lean workspace created`, but there was no `.lake` and no Mathlib oleans. The first `/prove 1+1=2` hung for minutes.

**Workaround:** `cd ~/GaussWorkspace && lake exe cache get && lake build`. This downloaded 9104 files and the build succeeded. It would help if the installer ran `lake exe cache get` itself, or at least reported that it hadn't.

## 4. Smaller issues

- **Setup wizard menus don't render.** `gauss setup` and `gauss setup model` print `Skipped (keeping current)` without showing the provider menu, so the provider stays on OpenRouter and `gauss model` only offers an OpenRouter key prompt.
- **Managed workers fail with `exit 1` on Claude Code's first-run prompts.** Both the folder-trust prompt and the Bypass Permissions confirmation default to "No, exit", and the swarm task dies silently. Pre-accepting them by running `claude` / `claude --dangerously-skip-permissions` once in the workspace fixes it. A note in the docs, or a detection with a clear message, would save people a lot of confusion.
- **`.claude/settings.local.json` contains `"mcp__*"`**, which current Claude Code rejects ("Wildcard tool name `mcp__*` is not supported in allow rules"). It probably needs to be scoped, for example `mcp__lean-lsp-mcp__*`.
- **The workspace isn't its own git repo.** If `~` happens to be a git repo, workspace commits land there. `git init` in the workspace during install would avoid that.

## Community guide

I wrote up the full working sequence, with fixes, here for anyone hitting the same problems:
https://github.com/JAgbanwa/opengauss-install-guide

Happy to test a fix or open a PR for any of these.
