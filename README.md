# git-internals

> An overview of git internals

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
