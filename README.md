# cworktree

Create a git worktree under `<repo-root>/.claude/worktrees/<name>` — and, optionally,
launch [Claude Code](https://claude.com/claude-code) in it.

A single dependency-free bash script. It works from anywhere inside a repository,
including from another worktree: the path is always resolved against the **main**
repository root, so nested worktree trees never appear.

The default layout (`.claude/worktrees/`) is the one Claude Code uses itself, so
`claude --worktree <name>` reuses the worktree created here instead of creating a
second one. Names with slashes are flattened the same way it flattens them —
`feat/xy/login` lives in `.claude/worktrees/feat+xy+login`, never in a
`feat/xy/` subtree — because Claude Code only looks *directly* under that
directory. The branch keeps its slashes.

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
plus tab completion (the wrapper is also what makes `--cd -c` cd first and start
claude afterwards):

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
cworktree --checkout feature/x            # attach an existing branch
cworktree fix-login --claude              # ... then launch Claude Code in it
cworktree fix-login --effort high         # ... with a high effort level
cworktree fix-login --max                 # ... same as --effort max
cworktree fix-login -c -- fix the login   # ... with a prompt for the session
cworktree fix-login --cd                  # cd into the worktree
cworktree fix-login --cd -c               # cd into it, then launch claude there
cworktree --cd ..                         # cd back to the repo root
cworktree fix-login -D                    # remove the worktree again
cworktree fix-login review -D             # ... or several worktrees at once
cworktree -D                              # prune worktrees deleted by hand
cworktree -l                              # list worktrees + their HEAD commit
```

Options may come before or after the branch name, and `--cd`, `-D`/`--delete`
and `--checkout` take their name from the positional argument when they are
not given one directly, so `cworktree fix-login --cd` and `cworktree --cd
fix-login` mean the same thing — likewise `cworktree --checkout feature/x` and
`cworktree feature/x --checkout`. `-D`/`--delete` goes on taking names: every
positional argument on the line is one more worktree to remove.

A worktree's name is always its branch's — the two are never configured
apart. By default a new branch is created from `HEAD`; if the base branch
looks like `<remote>/<branch>` but is unknown, cworktree fetches it once
before giving up. `--checkout` attaches an existing branch instead of
creating one; `--detach` checks out a commit with no branch at all, so the
worktree's name is then just a name of its own.

A branch name may contain `/`, but the worktree directory it gets is one
level deep: every `/` becomes a `+`, so `feat/xy/login` is
`.claude/worktrees/feat+xy+login` with the branch still called
`feat/xy/login`. That is exactly what Claude Code does with a `--worktree`
name, and it is what lets it find this worktree; a nested
`.claude/worktrees/feat/xy/login` would be invisible to it, and
`claude --worktree feat/xy/login` would build a second worktree of its own
alongside. Either spelling addresses the same worktree afterwards, so
`--cd feat/xy/login` and `--cd feat+xy+login` (the name `--list` prints) both
work.

The worktree path is the only thing printed on stdout (everything else goes to
stderr), so it composes: `cd "$(cworktree scratch)"`, or `cworktree scratch -p`
to get the path without creating anything.

The base directory is added to `.git/info/exclude` (not `.gitignore`, so nothing
in your repository changes) unless git already ignores it, or `--no-exclude` is
given.

### Options

| Option | Meaning |
| --- | --- |
| `-c, --claude [args]` | after a successful create, launch Claude Code in the worktree |
| `-e, --effort <level>` | pass `--effort <level>` to `claude` (`low`, `medium`, `high`, `xhigh`, `max`); implies `-c`. Also as its own flag: `--low`, `--medium`, `--high`, `--xhigh`, `--max` |
| `-B, --reset-branch` | reset the branch to the base branch if it already exists |
| `--checkout <name>` | attach an existing branch instead of creating one; the worktree is named after it, exactly like a new branch |
| `--detach` | detached HEAD at the start point, no branch |
| `-f, --force` | pass `--force` to `git worktree add`, or to `git worktree remove` with `-D`/`--delete` (twice for a locked worktree) |
| `-d, --dir <path>` | base dir for worktrees, relative to the repo root |
| `--no-exclude` | do not touch `.git/info/exclude` |
| `--no-fetch` | do not auto-fetch when a remote branch is unknown |
| `-p, --path` | print the worktree path and exit |
| `-n, --dry-run` | show what would happen, change nothing |
| `--cd [name]` | print the path of an existing worktree so the shell wrapper can cd into it; `--cd ..` is the repo root. With `-c`/`--claude` the shell cds there first and launches claude afterwards |
| `-D, --delete [name...]` | remove the worktree (`git worktree remove`); several names remove several worktrees; without a name, prune the bookkeeping of worktrees whose directory is gone (`git worktree prune`) |
| `-l, --list` | list the worktrees under the base dir, one line each: `git log --oneline -1 --decorate` of that worktree's HEAD, so the decoration reads `(HEAD -> <its own branch>)`. When the listing holds a single worktree it is shown the way git itself would — that same line followed by `git status --short`, run in the worktree, colours included. At a terminal only: piped or captured it prints bare names, one per line, so it stays usable for scripting and tab completion |
| `--shell-init [sh]` | see above |
| `--` | everything after this is joined into a single string and passed to `claude` as its `[prompt]`; implies `-c` |

> `-D` used to be the short form of `--dir`, which is now `-d`/`--dir`.
> `-b`/`--branch` has been removed: a worktree is always named after its
> branch, so there is no longer a separate name to give it.

Re-running the same command is a no-op if the worktree already exists (with
`--claude`, it just starts a new session there).

## Launching Claude Code

`-c`/`--claude` runs, in the current terminal:

```
claude --worktree <name> --name <name> [args]
```

The name is the one that resolves to the worktree just created, so the session
runs in it instead of in a new one: for `feat+xy+login` on disk that is
`--worktree feat/xy/login`, since Claude Code maps the slashes back to pluses
itself and rejects a `+` in a name given to it. The same name is passed to
`--name`, which sets the session's display name: an *unnamed* `--worktree`
session is one Claude Code deletes on a clean exit — worktree and branch both
— so naming it turns that into a prompt instead, and lets `claude --resume
<name>` find the same worktree again later. When the worktree's name *cannot*
be made to agree with what `claude --worktree` accepts — a base dir other than
`.claude/worktrees/`, a name longer than the 64 characters it allows, or a
worktree still nested by an older cworktree — cworktree says so and starts a
plain `claude` inside the worktree instead, with neither flag: Claude Code
never created that worktree, so it never auto-deletes it either.

No claude flags are hardcoded, apart from `--worktree`/`--name` and
`--effort <level>` when you ask for one (`-e`/`--effort`, or the shorthands
`--low` … `--max`). Asking for an effort level implies `-c`, since it is only
useful with a session; it is inserted before the arguments below, so anything
you pass yourself comes later on the command line — including your own
`--name`, if you would rather override the default.

Extra arguments come from, in order of precedence:

1. **`-c '<args>'`** — a single string, parsed like a shell command line, so
   quoting works: `cworktree fix-login -c '--chrome -p "look at the tests"'`.
   The string is only recognised as claude arguments when it starts with `-` and
   is not a cworktree option; for anything else write `--claude='<args>'`.
2. **`$CWORKTREE_CLAUDE_ARGS`** — your default, used when `-c` is given without
   arguments of its own.
3. **After `--`** — everything left is joined with spaces into *one* argument and
   appended last, which is what `claude` takes as its `[prompt]`:
   `cworktree fix-login -- look at the failing tests`. Since it is a single
   argument, put claude *flags* in `-c '<args>'` rather than after `--`.

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
| `CWORKTREE_DIR` | default base dir for worktrees (see `-d`/`--dir`) |

## Navigating and cleaning up

`--cd` prints the path of an existing worktree; with the shell wrapper installed
your shell moves there. It never creates anything on its own — a typo is an
error, not a new worktree — but combined with `-c`/`--claude` (which does create
one when it is missing) the shell cds in first and then starts the session, so
you are left in the worktree when claude exits:

```sh
cworktree fix-login --cd -c --high     # create if needed, cd there, then claude
cworktree --cd ..                      # back to the repo root
```

`-D`/`--delete` removes a worktree with `git worktree remove`; add `-f` when it
has uncommitted or untracked files (twice when it is locked). The branch is left
alone. Empty parent directories are cleaned up as well, which is how a worktree
that an older cworktree nested is migrated: remove it, then create it again to
get the flat directory Claude Code can reuse. Without a name it runs `git worktree prune`, which cleans up after
worktree directories that were deleted by hand:

```sh
cworktree fix-login -D                 # git worktree remove <path>
cworktree fix-login -D -f              # ... even when it is dirty
cworktree -D                           # git worktree prune
cworktree -D -n                        # show what that would do
```

Any number of names may be given, in either order — `cworktree a b c -D` and
`cworktree -D a b c` are the same list. Nothing is removed until every name has
resolved to a registered worktree, so a typo at the end of the list cannot take
the beginning of it with it, and the same worktree named twice is removed once.
After that the list is worked through to the end: one worktree that refuses to
go — dirty, or locked — leaves the others removed and only shows up in the exit
status.

```sh
cworktree a b c -D                     # remove three worktrees
cworktree -D a b c                     # ... the same list
cworktree -D a b c -f                  # ... even the dirty ones
cworktree -D a b c -n                  # show what that would do
```

## License

[MIT](LICENSE)
