# OpenGauss: A Practical Install & Autoformalization Guide (macOS first)

An unofficial, battle-tested walkthrough for installing [Math, Inc.'s OpenGauss](https://github.com/math-inc/OpenGauss) on your own computer and using it to turn mathematics (LaTeX papers, theorems, proofs) into machine-checked **Lean 4 + Mathlib** code.

The official one-line installer currently fails for new users. This guide documents every problem hit during a real install on a Mac and gives the exact fix for each, in the order you will meet them.

> **Not affiliated with Math, Inc.** This is a community guide. OpenGauss changes quickly; if a step no longer matches what you see, check the [OpenGauss repo](https://github.com/math-inc/OpenGauss) and its issues.

---

## Contents

1. [What OpenGauss is (and isn't)](#1-what-opengauss-is-and-isnt)
2. [What you need](#2-what-you-need)
3. [macOS installation, step by step](#3-macos-installation-step-by-step)
4. [Linux and Windows notes](#4-linux-and-windows-notes)
5. [Autoformalizing your own mathematics](#5-autoformalizing-your-own-mathematics)
6. [Checking that a formalization is trustworthy](#6-checking-that-a-formalization-is-trustworthy)
7. [Troubleshooting](#7-troubleshooting)
8. [Tested versions](#8-tested-versions)

---

## 1. What OpenGauss is (and isn't)

- **OpenGauss is an orchestrator.** It sets up a Lean + Mathlib workspace and drives AI coding agents (Claude Code by default, or OpenAI Codex) through Lean workflows from the open-source [`lean4-skills`](https://github.com/cameronfreer/lean4-skills) plugin: `prove`, `draft`, `formalize`, `autoprove`, `autoformalize`, `review`, and more.
- **It is not Math, Inc.'s proprietary *Gauss* model** (the one behind the strong Prime Number Theorem and sphere-packing formalizations). OpenGauss's output quality depends on the agent you connect, and so does the cost.
- Everything the agent produces is checked by the Lean kernel. Whether the formal *statement* matches your mathematics is still **your** job; see [Section 6](#6-checking-that-a-formalization-is-trustworthy).

There are two ways to use it once installed:

| Route | How | When to use |
|---|---|---|
| **A. Claude Code directly** (recommended to start) | Run `claude` in your workspace and type `/lean4:draft`, `/lean4:prove`, ... | Simplest; one agent at a time; asks before risky actions |
| **B. The `gauss` CLI** | Run `gauss` and type `/prove`, `/autoformalize`, `/swarm` | Several agents in parallel; needs a one-time "bypass permissions" acceptance |

---

## 2. What you need

- A Mac with **Apple Silicon (M1 or newer)** or Intel. Tested on Apple Silicon, macOS 14.
- **~8 GB free disk space.** The Lean toolchain is about 600 MB and the Mathlib cache several GB.
- **30–60 minutes**, mostly downloads.
- A way to run the AI agent, one of:
  - a **Claude plan that includes Claude Code**, or an **Anthropic API key**, or
  - an **OpenAI Codex** login.
- Basic comfort pasting commands into **Terminal** (Applications → Utilities → Terminal).

> **A rule that will save you a lot of time:** in every interactive prompt in this guide, the highlighted line (`❯` or `→`) is what **Enter** selects. Several prompts highlight **"No, exit"** by default. Always move to your choice with the **arrow keys** first, *then* press Enter.

---

## 3. macOS installation, step by step

### Step 0: Make sure Terminal is not running under Rosetta (Apple Silicon only)

```bash
uname -m
```

- `arm64` → good, continue.
- `x86_64` on an M-series Mac → Terminal is running under Rosetta. Quit Terminal, select it in Finder → **Get Info** → untick **"Open using Rosetta"**, reopen it, and check again.

### Step 1: Install the basic tools

Command Line Tools (gives you `git`):

```bash
xcode-select --install
```

(If it says they're already installed, that's fine.)

[Homebrew](https://brew.sh), if you don't have it yet:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Then `ripgrep`, which the Lean workflows use for searching:

```bash
brew install ripgrep
```

> Homebrew may warn that your macOS version is "Tier 3" or unsupported and build some packages from source. That's slow but harmless.

### Step 2: Install Claude Code natively, before OpenGauss

This avoids a known problem: the OpenGauss installer may install Claude Code through npm, which can leave a broken text placeholder at `~/.local/bin/claude` instead of the real program. Installing the native build first means the installer detects it and skips that.

```bash
curl -fsSL https://claude.ai/install.sh | bash
claude --version
```

You should see a version number such as `2.1.x (Claude Code)`. Then confirm it's a real Apple Silicon binary:

```bash
file ~/.local/bin/claude
```

✅ Expected: `Mach-O 64-bit executable arm64`.
❌ If it says `ASCII text`, or points into `node_modules`, see [Troubleshooting](#claude-exec-format-error).

Log in once (any folder is fine):

```bash
claude
```

Follow the login prompts, then type `/exit`.

> Using **Codex** instead? The installer also sets up `codex`. Log in with `codex` and see the backend note in [Step 7](#step-7-optional-set-up-the-gauss-cli).

### Step 3: Download OpenGauss

```bash
cd ~
git clone https://github.com/math-inc/OpenGauss.git
cd ~/OpenGauss
```

> ⚠️ Run `git clone` **only once**, from your home folder. Running it again from inside `~/OpenGauss` creates a nested `~/OpenGauss/OpenGauss` copy. If that happens, delete it: `rm -rf ~/OpenGauss/OpenGauss`.

### Step 4: Run the installer that works

The README's `./scripts/install.sh` currently fails with:

```
Error: Template YAML file does not exist ... failed to fetch shared template YAML from https://morph.new/opengauss/yaml: 404 Not Found
```

That wrapper downloads a setup template that is no longer online. The real installer ships in the repo, so run it directly:

```bash
cd ~/OpenGauss
./scripts/install-internal.sh --with-workspace --run-setup
```

- `--with-workspace` creates a Lean + Mathlib project at `~/GaussWorkspace`.
- `--run-setup` launches a short setup wizard at the end.

This takes several minutes. A successful run ends with a box saying **"✓ Open Gauss Is Ready"**, plus lines like:

```
✓ Managed Lean workflow assets ready: lean4-skills ...
✓ Managed /prove staging verified: /Users/<you>/GaussWorkspace
```

**About the setup wizard that follows:**

- It asks for an **inference provider** and may default to **OpenRouter**. You **don't need an OpenRouter key.** Leave it blank and press Enter, or press **Ctrl+C** to leave the wizard. Nothing installed gets undone.
- On some Macs the wizard's arrow-key menus never appear and it prints `Skipped (keeping current)`. That's fine; see [Step 7](#step-7-optional-set-up-the-gauss-cli).
- Skip optional keys (Firecrawl, image generation and so on) by pressing Enter.

Reload your shell so the `gauss` command is found:

```bash
source ~/.zshrc
```

### Step 5: Download Mathlib (don't skip this)

The installer may report "Lean workspace created" **without actually downloading Mathlib**. Lean will then hang on anything that imports it. Fetch the prebuilt Mathlib files:

```bash
cd ~/GaussWorkspace
lake exe cache get
lake build
```

✅ Expected near the end:

```
Downloaded: 9104 file(s) ...
Decompressed 9104 file(s)
Build completed successfully
```

(The exact file count varies with the Mathlib version.)

> ⚠️ If instead you see Lake **compiling thousands of Mathlib files** one by one, press **Ctrl+C**. Building Mathlib from source takes hours. It means the Lean toolchain and Mathlib versions don't match; see [Troubleshooting](#mathlib-compiles-from-source).

### Step 6: Give the workspace its own git repo, then trust it in Claude Code

**6a. Check git.** If your home folder happens to be a git repository, commits from the workspace will land there by mistake. Check:

```bash
cd ~/GaussWorkspace
git rev-parse --show-toplevel 2>/dev/null || echo "not a git repo"
```

If it prints `/Users/<you>` (your home folder) or `not a git repo`, give the workspace its own repository:

```bash
git init
git add -A
git commit -m "Initial Gauss Lean workspace"
```

> Check that `.lake` is in `.gitignore` first (`cat .gitignore`), so you don't commit gigabytes of Mathlib.

**6b. Trust the folder in Claude Code.**

```bash
cd ~/GaussWorkspace
claude
```

You'll see one or both of these prompts:

1. **"Settings Warning … Invalid permission rule `mcp__*` was skipped"**: choose **1. Continue** (the default). It's harmless.
2. **"Quick safety check: Is this a project you created or one you trust?"**: the default is **"No, exit"**. Press **↓** to **"Yes, I trust this folder"**, then Enter.

It also tells you the folder pre-approves Write, Edit, Bash, WebSearch, WebFetch and Read. That's OpenGauss's configuration, so agents can edit Lean files and run `lake` without asking each time. Keep only Lean work in this folder.

### Step 7: Smoke test

Still inside Claude Code in `~/GaussWorkspace`, type:

```
/lean4:prove 1+1=2
```

✅ Expected: within a minute or two it adds something like

```lean
theorem one_add_one_eq_two : (1 : ℕ) + 1 = 2 := by
  rfl
```

and reports that it compiles. The first `import Mathlib` can take about 2 minutes while Mathlib loads from disk; later runs are faster.

**Installation is now complete for Route A.** Next is [Section 5](#5-autoformalizing-your-own-mathematics).

### Step 7 (optional): Set up the `gauss` CLI

Only needed if you want Route B: the `gauss` interface and parallel agents via `/swarm`.

**7a. The chat provider.** The `/prove`, `/draft`, `/formalize` and `/autoformalize` commands in `gauss` run on **Claude Code**, so they work without anything extra. Gauss's own free-text chat, however, needs a provider. If you want it, try:

```bash
gauss setup model
```

and choose **"Anthropic (… Claude Code subscription)"** or **"Login with OpenAI Codex"** with the arrow keys. If the menu doesn't appear, see [Troubleshooting](#gauss-wizard-menus-dont-appear). You can ignore the chat and use only the slash commands.

**7b. Accept "bypass permissions" mode once.** Gauss starts its workers with Claude Code's *Bypass Permissions* mode so they can run unattended. The first time, Claude Code asks you to confirm, with **"No, exit"** as the default, and the worker silently fails with `exit 1`.

> **Understand the risk before accepting.** In this mode the agent runs shell commands and edits files **anywhere your user account can reach**, without asking. Claude Code recommends it only inside a sandbox or VM. If you accept: keep important work committed and pushed, and keep `~/GaussWorkspace` dedicated to Lean.

To accept once:

```bash
cd ~/GaussWorkspace
claude --dangerously-skip-permissions
```

Press **↓** to **"Yes, I accept"**, Enter, then `/exit`.

**7c. Test it:**

```bash
cd ~/GaussWorkspace
gauss
```

At the `›` prompt:

```
/prove 1+1=2
/swarm
```

`/swarm` lists running agents; `/swarm attach <task-id>` shows one working (detach with **Ctrl+]**).

> In the `gauss` status bar, the timer (e.g. `14m`) is how long the session has been open, **not** a running job. If the `›` prompt is empty, Gauss is idle and waiting for you.

---

## 4. Linux and Windows notes

> ⚠️ The steps below follow the same logic as the macOS walkthrough but **have not been tested end to end** by this guide. Corrections are welcome.

### Linux (Ubuntu/Debian)

Same as macOS, with these differences:

- Use `sudo apt install git ripgrep curl` instead of Homebrew.
- Skip Step 0 (Rosetta).
- `install-internal.sh` may run `apt-get` itself to install system packages; it will ask for your password. Pass `--skip-system-packages` if you'd rather manage them yourself.
- Everything from Step 4 onward is the same.

### Windows (via WSL2)

OpenGauss runs on Windows inside **WSL2** (Linux on Windows).

1. In **PowerShell (as Administrator)**: `wsl --install -d Ubuntu`, then restart if asked.
2. Open **Ubuntu** from the Start menu and create a Linux user.
3. Inside Ubuntu, follow the **Linux** steps above. Clone into your Linux home (`~/OpenGauss`), **not** into `/mnt/c/...`, because it is much faster and avoids permission problems.
4. Use `./scripts/install-internal.sh --with-workspace --run-setup` (Step 4) instead of `install.ps1`, since `install.ps1` also goes through the broken template download.

---

## 5. Autoformalizing your own mathematics

Always run these from Claude Code started in your workspace:

```bash
cd ~/GaussWorkspace
claude
```

(Using `gauss` instead? Drop the `lean4:` prefix: `/draft`, `/prove`, `/autoformalize`.)

### 5.1 The commands

| Command | What it does |
|---|---|
| `/lean4:draft <topic or file>` | Writes Lean **statements** (definitions and theorem signatures) with `sorry` proofs |
| `/lean4:prove <goal>` | Guided proving: works with you step by step |
| `/lean4:autoprove` | Autonomous proving: tries to fill every `sorry` in scope |
| `/lean4:formalize <topic or file>` | Interactive formalization of statements and proofs |
| `/lean4:autoformalize <file>` | Autonomous end-to-end formalization of a document |
| `/lean4:review` | Read-only review of existing Lean proofs |

### 5.2 Recommended workflow: statements first, proofs second

The most common failure in autoformalization is not a wrong proof (Lean rejects those) but a **proof of the wrong statement**. So split the work.

**1. Put your source in the workspace.** LaTeX (`.tex`) works better than PDF.

```bash
cp ~/Documents/mypaper.tex ~/GaussWorkspace/
```

**2. Draft statements for one result first.**

```
/lean4:draft Read mypaper.tex and draft Lean 4 statements, with sorry proofs, for Theorem 1 and the definitions it depends on. Keep every hypothesis exactly as stated in the paper. Do not strengthen hypotheses or weaken conclusions. Put the result in GaussWorkspace/MyPaper.lean and import it from GaussWorkspace.lean.
```

**3. Review the statements yourself** using the checklist in [Section 6](#6-checking-that-a-formalization-is-trustworthy). Ask for corrections in plain language, for example "the paper requires n ≥ 2, not n ≥ 1".

**4. Prove them.**

```
/lean4:autoprove Fill the sorries in GaussWorkspace/MyPaper.lean without changing any theorem statements.
```

**5. Verify and commit.**

```
Run lake build, then #print axioms on every main theorem in GaussWorkspace/MyPaper.lean, and commit.
```

**6. Scale up** only once one theorem works end to end:

```
/lean4:autoformalize mypaper.tex
```

### 5.3 Prompting tips

- **Name the exact result**: "Theorem 2.3 and Lemma 2.1" rather than "the main results".
- **Tell it what not to change**: "do not modify existing theorem statements" protects work you've already checked.
- **Give it the hard part.** If your proof uses a nonstandard trick, describe it in a sentence or two; agents do much better with a proof sketch.
- **Avoid importing all of Mathlib in finished files**: ask it to import only the modules needed. Full `import Mathlib` is slow to load.

### 5.4 What formalizes well and what doesn't

| Usually works | Usually needs extra work or can't be done |
|---|---|
| Elementary number theory, congruences, divisibility | Results Mathlib doesn't have yet (they become `axiom`s or hypotheses) |
| Inequalities, finite sums and products, induction | "By computer search, there are no solutions below 10¹⁸": large computations don't move into Lean |
| Combinatorics with explicit bounds | Arguments relying on CAS output (Magma, PARI, Sage) for ranks, class groups and so on |
| Algebra with existing Mathlib structures | Long chains of results from other papers not yet in Mathlib |

Small finite checks can be done with `decide`; `native_decide` is faster but adds trust in the compiler. Keep track of which you used.

---

## 6. Checking that a formalization is trustworthy

Lean guarantees the proof proves *the formal statement*. It cannot tell you whether that statement is *your* theorem. Before you rely on any result:

**1. Check for gaps and extra axioms.** For each main theorem, add or ask for:

```lean
#print axioms my_main_theorem
```

✅ Acceptable: `propext`, `Classical.choice`, `Quot.sound` (or no axioms at all).
❌ Red flags: `sorryAx` (an unfinished proof), any custom `axiom` declared in your files, `Lean.ofReduceBool` (this comes from `native_decide`; acceptable only if you chose it deliberately).

Also search your files for leftovers:

```bash
cd ~/GaussWorkspace
grep -rn "sorry\|^axiom\|admit" GaussWorkspace/
```

**2. Read every theorem statement against the paper.** Typical silent changes:

- an extra or stronger hypothesis (e.g. `0 < n` became `2 < n`, or a prime added that wasn't in the paper)
- a weaker conclusion (`∃` instead of `∃!`, or one direction of an `↔`)
- the wrong number system (`ℕ` subtraction truncates at 0, and `ℕ`/`ℤ` division rounds)
- a definition that differs from yours in an edge case (zero, empty set, degenerate objects)

**3. Have a second agent review it.**

```
/lean4:review Compare each theorem statement in GaussWorkspace/MyPaper.lean against mypaper.tex and list any mismatch in hypotheses, conclusions, quantifiers or number types.
```

---

## 7. Troubleshooting

<a id="installer-404"></a>
**`Template YAML file does not exist ... 404 Not Found`**
The `install.sh` wrapper can't download its template. Use `./scripts/install-internal.sh --with-workspace --run-setup` ([Step 4](#step-4-run-the-installer-that-works)).

<a id="claude-exec-format-error"></a>
**`OSError: [Errno 8] Exec format error: '.../.local/bin/claude'`**
`~/.local/bin/claude` is a broken npm placeholder, not a real program. Check with `file ~/.local/bin/claude`. If it shows `ASCII text`, fix it:

```bash
rm -f ~/.local/bin/claude
curl -fsSL https://claude.ai/install.sh | bash
file ~/.local/bin/claude      # should now say Mach-O 64-bit executable arm64
```

Then rerun the installer from [Step 4](#step-4-run-the-installer-that-works). It's safe to rerun and won't download Mathlib again.

<a id="gauss-wizard-menus-dont-appear"></a>
**`gauss setup` / `gauss setup model` prints `Skipped (keeping current)` without showing a menu**
The arrow-key menu failed to draw. Try `TERM=xterm-256color gauss setup model`. If it still doesn't show, it doesn't matter for formalization: the workflow commands use Claude Code, not this provider.

**`gauss model` asks for an OpenRouter API key**
You don't need one. Press Enter to cancel.

**A `gauss` task fails immediately with `exit 1`**
Almost always a Claude Code prompt defaulted to "No, exit". Do [Step 6b](#step-6-give-the-workspace-its-own-git-repo-then-trust-it-in-claude-code) (trust the folder) and [Step 7b](#step-7-optional-set-up-the-gauss-cli) (bypass permissions), then retry.

**Lean hangs on a tiny proof; Claude says "Mathlib isn't downloaded" or "no .lake folder"**
Run [Step 5](#step-5-download-mathlib-dont-skip-this): `lake exe cache get && lake build` inside `~/GaussWorkspace`. When checking files by hand, use `lake env lean File.lean`, not bare `lean`.

<a id="mathlib-compiles-from-source"></a>
**`lake build` starts compiling Mathlib from source**
Stop it (Ctrl+C). The toolchain doesn't match the Mathlib revision. Run `cat lean-toolchain`, then `lake update` followed by `lake exe cache get`. Only run `lake build` once `cache get` reports files downloaded.

**Settings warning about `mcp__*`**
Harmless. Choose **Continue**. Newer Claude Code rejects that wildcard; the other permissions still apply.

**Claude Code says "N MCP servers need authentication"**
These are usually connectors on your Claude account, not the Lean tools (`lean-lsp-mcp` runs locally and needs no login). Ignore it unless you need those connectors.

**`gauss doctor` warnings you can ignore** for formalization: npm audit vulnerabilities in browser tools, `tinker-atropos` not installed, missing `FIRECRAWL_API_KEY` / `GITHUB_TOKEN` / `FAL_KEY`, and the project template source.

**Keeping things updated**

```bash
cd ~/OpenGauss && git pull && ./scripts/install-internal.sh --with-workspace
cd ~/GaussWorkspace && lake update && lake exe cache get
```

---

## 8. Tested versions

Tested November 2026 on macOS 14, Apple Silicon (M1 Pro):

| Component | Version |
|---|---|
| OpenGauss (`gauss-agent`) | 0.2.2 |
| Claude Code | 2.1.322 (native install) |
| lean4-skills Claude plugin | 4.12.0 |
| Lean toolchain | v4.36.0-rc2 (selected by Mathlib master) |
| Mathlib | master @ `81b8ee0` |
| uv / Node.js | 0.9.18 / v25.8.2 |

---

### Contributing

Found a step that changed, or got it working on Linux or Windows? Open an issue or a pull request with the exact error text and what fixed it.
