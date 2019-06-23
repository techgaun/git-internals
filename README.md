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

## Git references

- names that point to sha1 hashes.
- stored in directories inside `.git/refs`.
- `heads` contain branch references.
- `tags` contain tag references.
- `remotes` contain references on remote urls added.

## Git packfiles

- Git stores objects on disk in so called loose object format initially.
- It would be inefficient if git kept on storing entire content everytime we make change on a file.
- Git occasionally packs up several of these loose objects into a single binary file called packfile to save space and
be more efficient. This allows storing versions of objects in the form of deltas.

## Git gc/reflog/fsck

### gc

- performs cleanup and optimizes the repository.
- several housekeepings such as compressing file revisions, removing unreachable objects, packing refs and pruning
reflogs & stale working trees.
- As it relates to packfiles, it gathers up loose objects & places them in packfiles. Also, it consolidates packfiles
into a single large packfile as necessary.
- Auto gc happens once in a while as git deems necessary for example when you try to push to remote.

### reflog

- git records what repo's `HEAD` is everytime it changes which we call `reflog`.
- stored in `.git/logs` directory.
- can be useful to recover accidentally deleted branches.

### fsck

- Integrity check of your objects in the database.
- Often gives us the knowledge of dangling objects that can no longer be reached.
- Could be potentially useful in cases when we don't have reflogs.

## Working Example

We will run series of commands and see how things work under the hood based on the understanding from information above.
We've already created `git_demo` repository earlier while looking at the tree structure of `.git` directory.

```shell
# lets create a simple text file
$ echo "Hello World" > readme.md

# now we can see what would the sha1 hash of readme.md according to git
# the way it works is, a format of data is created as below and then sha1 hash for that is created
# <type_of_object> <size_of_object><nullbyte><content_of_object> | sha1_hash_function
$ git hash-object readme.md
557db03de997c86a4a028e1ebd3a1ceb225be238

# we can replicate what git did by doing something like below:
# blob is the type of object in this case
# as you can see below, the hash matches, simple ;)
$ echo -e "blob $(wc -m readme.md | cut -d' ' -f1)\000$(cat readme.md)" | sha1sum 
557db03de997c86a4a028e1ebd3a1ceb225be238  -

# we ran hash-object which is a git plumbing command
# however that doesn't add our content to object database until we instruct git to do so
# next we will do that
# open a new terminal (or tmux split) with the following command
$ watch -n 0.5 tree .git

# now we will hash the object and ask git to store it in git object database as well
# when we do so, git will create new directory .git/objects/55
# and create file with name `7db03de997c86a4a028e1ebd3a1ceb225be238`
# first two characters of sha1 hash form directory and rest the filenames
# thats how git organizes objects in its objects database.
$ git hash-object -w readme.md
557db03de997c86a4a028e1ebd3a1ceb225be238

# next we will cat the content of file
$ cat .git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238 
xK��OR04b�H����/�I�A�I

# above we see some unrecognizable text and thats not what we saved though
# git saves our content with header + nullbyte + content as we saw earlier
# we can use git cat-file plumbing command to look at the object we just created.
$ git cat-file -p 557db03de997c86a4a028e1ebd3a1ceb225be238
Hello World

# and now lets look at the type of the file
$ git cat-file -t 557db03de997c86a4a028e1ebd3a1ceb225be238
blob

# now the reason the raw content of object is some sort of gibberish
# is because its stored after running zlib compression
# lets see what it has in there
# as we see next, it stores type of object (blob) and content length (12)
# and actual content separated by nullbyte character.
$ cat .git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238 | zlib-flate -uncompress
blob 12Hello World
```
