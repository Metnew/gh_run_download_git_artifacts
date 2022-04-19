# Cli: `gh run download` implementation allows overwriting git repository configuration upon artifacts downloading

Hi there!

https://github.com/Metnew

## Writing to `<root>/.git/` with `gh run download`

```
USAGE
  gh run download [<run-id>] [flags]

FLAGS
  -D, --dir string         The directory to download artifacts into (default ".")
```

By default, `gh run download` command puts artifacts in the current dir. It can't overwrite files though, only create new ones. There are also no restrictions on the name/path of the files to be written (only path traversal check).

It means, it's possible to craft an artifact that creates new files in `<root>/.git/*` upon `gh run download`. 

## Achieving code execution with `.git/commondir`

[What's `.git/commondir`?](https://git-scm.com/docs/gitrepository-layout#Documentation/gitrepository-layout.txt-commondir)
> **commondir**
> If this file exists, $GIT_COMMON_DIR (see git[1]) will be set to the path specified in this file if it is not explicitly set. If the specified path is relative, it is relative to $GIT_DIR. The repository with commondir is incomplete without the repository pointed by "commondir".

With `.git/commondir` it's possible to re-define `$GIT_COMMON_DIR` for the current repository and therefore supply an attacker-crafted `.gitconfig` for the repo.
This effectively allows the attacker to run arbitrary commands once user (or any of user's devtools) run any git command (e.g., git fetch or similar).

**Step-by-step**

1. `gh run download`
2. the downloaded artifact has fileee `.git/commondir` pointing to `./poc` and `.git/poc` git repository with malicious `.gitconfig`
3. gh cli writes artifact files to `.git`
3. now the actual gitconfig in effect isn't `/.git/config`, but `.git/poc/config`
3. user tooling interacts with the repository (regular `git diff` in github desktop) OR user runs some git command (e.g., `git fetch`)
4. code execution!

> A malicious payload can be achieved through gitconfig properties (core.gitproxy, core.sshCommand, credential.helper), git hooks (see below), git filters.

## Achieving code execution with git hooks

> it's possible to leverage `commondir` scenario to re-define `core.hooksPath` and make executable hooks a part of the repository. In this case, they'll have `+x` flag, and rce via git hooks will be accomplishable. 

There is a well-known issue of `actions/upload` [reseting file permissions](https://github.com/actions/upload-artifact/issues/38) of artifacts.
> Funny enough, adding `+x` to all artifacts initially was a (default behaviour)[https://github.com/actions/upload-artifact/issues/20].

Thus, at this moment the malicious hooks written into `.git/hooks` are not executable :( 
But this scenario will be actual as soon as the above-mentioned issue is fixed.

> I can't imagine someone managing to have all of possible client-side git hooks (even non-standard!) configured, so there always be a place for a malicious one (+ git-p4* hooks). 
```
applypatch-msg pre-commit commit-msg    pre-merge fsmonitor-watchman    pre-merge-commit post-applypatch       pre-push
post-checkout  pre-rebase post-index-change  pre-receive post-merge prepare-commit-msg
post-rewrite  push-to-checkout post-update  reference-transaction pre-applypatch sendemail-validate pre-auto-gc update
```
In this scenario malicious code will be executed upon running any porcelain git command:  `git fetch`, `pull`, `push`, `checkout`, `commit`, `merge`, `rebase`, `am`, `add`, `rm`.
