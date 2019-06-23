# git-internals

> An overview of git internals

This repo consists of the talk given at PayLease's Show and Tell on 06/21/2019.

Git has a content-addressable filesystem as the layer which acts as KV store in a way.
You give some content to git and git gives you a 40 character sha1 hash. You can then
use the sha1 hash in the future to talk with git about that content.

## [Slides](slides.md) - Use with `mdp slides.md`

## Walkthrough

### Git aliases/Shell aliases/Global gitignore

- [My gitconfig](https://github.com/techgaun/dotfiles/blob/79cad9d116bdff6d05a16806668df72bd50af3c0/.gitconfig#L11-L43)
- [My global gitignore](https://github.com/techgaun/dotfiles/blob/79cad9d116bdff6d05a16806668df72bd50af3c0/.gitignore)
- [My git shell
alias](https://github.com/techgaun/dotfiles/blob/79cad9d116bdff6d05a16806668df72bd50af3c0/.bash_aliases#L94-L97) with
[autocompletion](https://github.com/techgaun/dotfiles/blob/79cad9d116bdff6d05a16806668df72bd50af3c0/.bashrc.defaults#L14-L20)

### .git directory

We create a git repository first and look at the initial tree structure of .git.
Git repo is a directory with .git sub-directory with relevant metadata.

```shell
$ git init git_demo
Initialized empty Git repository in /tmp/git_demo/.git/

$ cd git_demo

$ tree .git
.git
├── branches
├── config
├── description
├── HEAD
├── hooks
│   ├── applypatch-msg.sample
│   ├── commit-msg.sample
│   ├── fsmonitor-watchman.sample
│   ├── post-update.sample
│   ├── pre-applypatch.sample
│   ├── pre-commit.sample
│   ├── prepare-commit-msg.sample
│   ├── pre-push.sample
│   ├── pre-rebase.sample
│   ├── pre-receive.sample
│   └── update.sample
├── info
│   └── exclude
├── objects
│   ├── info
│   └── pack
└── refs
    ├── heads
    └── tags

9 directories, 15 files
```

- `.git/config` holds local git configuration that applies to the repo we are in.
- `.git/description` holds description that is shown by gitweb.
- `.git/HEAD` holds pointer/reference to what branch/tag/commit id we are at.
- `.git/hooks` holds sample hooks initially and you can create your own.
- `.git/info/exclude` holds repo level gitignore that doesn't go in repo's .gitignore.
- `.git/objects` holds all kind of objects git stores.
- `.git/refs` holds all kind of references git makes use of (branch/tag/stash, etc.).
- `.git/logs` doesn't exist initially but gets created as you travel through your git repo. It holds all the logs that
show up on `git reflog` subcommand.
- `.git/index` doesn't exist initially but holds information about the staging area.

## [Git hooks](https://githooks.com/)

- scripts that executes before or after certain events such as: `commit`, `push`, `receive`, etc.
- `pre-commit` - usage could be something like running lint or unit tests on files changed. Exits without making commit
if the `pre-commit` hook returns non-zero exit code.
- `post-receive` - usage could be for pushing code to the production.
- to enable hooks, overwrite or create one of the scripts in `.git/hooks` and make it executable.

## Git plumbing vs porcelain commands

- Most of the commands we use on our day to day interaction with git are porcelain commands that are much simpler to
use. Think of them as the frontend for git with simplified interface.
- There are another sets of commands that are lower level and can be composed together to form the porcelain commands.
These commands are called plumbing commands.
- As we explore further, we will look at some of the plumbing commands here in a bit with example.
- Some of the plumbing commands we will look at are `hash-object`, `update-index`, `write-tree`, `commit-tree` and
`cat-file`.

## Git objects

4 types of Objects:

- `blob` (binary large object) - the data we want git to store and version
- `tree` - pointers to file names, contents & other trees. A git tree object creates hierarchy between files and
directories in a git repository.
- `commit` - tree of changes together with some additional metadata (like author, commit message, committer, etc.). It
represents snapshot of the state of the repository.
- `tag` - For annotated tags which contains hash of tagged object (usually commits are tagged).
