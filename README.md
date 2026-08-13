# cworktree

Create a git worktree under `<repo-root>/.claude/worktrees/<name>` — and, optionally,
launch [Claude Code](https://claude.com/claude-code) in it.

A single dependency-free bash script. It works from anywhere inside a repository,
including from another worktree: the path is always resolved against the **main**
repository root, so nested worktree trees never appear.

The default layout (`.claude/worktrees/`) is the one Claude Code uses itself, so
`claude --worktree <name>` reuses the worktree created here instead of creating a
second one.

## Install

```sh
curl -fsSLO https://raw.githubusercontent.com/DefeatMan/cworktree/main/cworktree
chmod +x cworktree
mv cworktree ~/.local/bin/    # anywhere on your $PATH
```

Requires `bash` (3.2+, so the stock macOS bash is fine) and `git` 2.9 or newer.

### Shell integration (optional, needed for `--cd`)

A child process cannot change its parent's directory, so `--cd` only *prints* a
path and a small shell wrapper does the `cd`. `--shell-init` emits that wrapper
plus tab completion:

```sh
# ~/.bashrc
eval "$(cworktree --shell-init bash)"

# ~/.zshrc  (after compinit)
eval "$(cworktree --shell-init zsh)"
```

## Usage

```sh
cworktree fix-login                       # new branch fix-login from HEAD
cworktree fix-login origin/main           # new branch tracking origin/main
cworktree hotfix v1.2.3 --detach          # detached worktree at tag v1.2.3
cworktree review --checkout feature/x     # check out an existing branch
cworktree fix-login --claude              # ... then launch Claude Code in it
cworktree fix-login --effort high         # ... with a high effort level
cworktree fix-login --max                 # ... same as --effort max
cworktree --cd fix-login                  # cd into the worktree
cworktree --cd ..                         # cd back to the repo root
cworktree --list                          # list worktrees under the base dir
```

By default a new branch named after the worktree is created from `HEAD`. If the
start point looks like `<remote>/<branch>` but is unknown, cworktree fetches it
once before giving up. The worktree path is the only thing printed on stdout
(everything else goes to stderr), so it composes: `cd "$(cworktree scratch)"`,
or `cworktree scratch -p` to get the path without creating anything.

The base directory is added to `.git/info/exclude` (not `.gitignore`, so nothing
in your repository changes) unless git already ignores it, or `--no-exclude` is
given.

### Options

| Option | Meaning |
| --- | --- |
| `-c, --claude [args]` | after a successful create, launch Claude Code in the worktree |
| `-e, --effort <level>` | pass `--effort <level>` to `claude` (`low`, `medium`, `high`, `xhigh`, `max`); implies `-c`. Also as its own flag: `--low`, `--medium`, `--high`, `--xhigh`, `--max` |
| `-b, --branch <name>` | name for the new branch (default: the worktree name) |
| `-B, --reset-branch` | reset the branch to the start point if it already exists |
| `--checkout` | check out the given existing branch instead of creating one |
| `--detach` | detached HEAD at the start point, no branch |
| `-f, --force` | pass `--force` to `git worktree add` |
| `-D, --dir <path>` | base dir for worktrees, relative to the repo root |
| `--no-exclude` | do not touch `.git/info/exclude` |
| `--no-fetch` | do not auto-fetch when a remote branch is unknown |
| `-p, --path` | print the worktree path and exit |
| `-n, --dry-run` | show what would happen, change nothing |
| `--cd <name>`, `--list`, `--shell-init [sh]` | see above |
| `--` | everything after this is passed through to `claude` |

Re-running the same command is a no-op if the worktree already exists (with
`--claude`, it just starts a new session there).

## Launching Claude Code

`-c`/`--claude` runs, in the current terminal:

```
claude --worktree <worktree-name> [args]
```

No claude flags are hardcoded, apart from `--effort <level>` when you ask for one
(`-e`/`--effort`, or the shorthands `--low` … `--max`). Asking for an effort level
implies `-c`, since it is only useful with a session; it is inserted before the
arguments below, so anything you pass yourself comes later on the command line.

Extra arguments come from, in order of precedence:

1. **`-c '<args>'`** — a single string, parsed like a shell command line, so
   quoting works: `cworktree fix-login -c '--chrome -p "look at the tests"'`.
   The string is only recognised as claude arguments when it starts with `-` and
   is not a cworktree option; for anything else write `--claude='<args>'`.
2. **`$CWORKTREE_CLAUDE_ARGS`** — your default, used when `-c` is given without
   arguments of its own.
3. **After `--`** — always appended last: `cworktree fix-login -c -- --resume`.

```sh
# ~/.bashrc or ~/.zshrc
export CWORKTREE_CLAUDE_ARGS='--chrome'
```

> `--dangerously-skip-permissions` is a convenient thing to put in
> `$CWORKTREE_CLAUDE_ARGS` for throwaway worktrees, but it disables every
> permission prompt for that session. Opt into it deliberately.

### Environment

| Variable | Meaning |
| --- | --- |
| `CWORKTREE_CLAUDE_ARGS` | default arguments for `claude` with `-c`/`--claude` |
| `CWORKTREE_DIR` | default base dir for worktrees (see `-D`/`--dir`) |

## License

[MIT](LICENSE)
