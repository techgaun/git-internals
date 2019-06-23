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
# -p means pretty print the content of that object
$ git cat-file -p 557db03de997c86a4a028e1ebd3a1ceb225be238
Hello World

# and now lets look at the type of the file
# -t means print the type of that object
$ git cat-file -t 557db03de997c86a4a028e1ebd3a1ceb225be238
blob

# now the reason the raw content of object is some sort of gibberish
# is because its stored after running zlib compression
# lets see what it has in there
# as we see next, it stores type of object (blob) and content length (12)
# and actual content separated by nullbyte character.
$ cat .git/objects/55/7db03de997c86a4a028e1ebd3a1ceb225be238 | zlib-flate -uncompress
blob 12Hello World

# now lets see an example of creating a tree out of the object we added
# we have added the object to git object database but it has no idea about
# where and how that should exist in our repository
# before we do that, lets look at our git status
$ g status -s
?? readme.md

# so there's an untracked file which we will add to git's staging area
# we normally do that via git add readme.md for example
# this time, we will use git update-index plumbing command
# which updates .git/index file (the file that holds staging info)
$ git update-index --add readme.md

# if we run git status, that matches with the fact that readme.md is now in staging area
# the ?? from previous status has changed to A now :)
$ git status -s
A  readme.md

# now we can take a look at our staging area more deeply
# 100644 - 100 means blob (040 means tree) and 644 represents permission
# permission in git looks like unix permissions except its much more limited
$ git ls-files --stage
100644 557db03de997c86a4a028e1ebd3a1ceb225be238 0	readme.md

# now we can create a new tree with the current state of index
# note that current state of index has readme.md file in staging area
# for this, we use git write-tree plumbing command
# and we get a hash back
$ git write-tree
3a3aff7fa9639da674465c43fac565c1291f952b

# we can use cat-file to look into the content & type of object that hash represents
$ git cat-file -p 3a3aff7fa9639da674465c43fac565c1291f952b
100644 blob 557db03de997c86a4a028e1ebd3a1ceb225be238	readme.md

$ git cat-file -t 3a3aff7fa9639da674465c43fac565c1291f952b
tree

# now that we have a tree object that holds a blob object we want to be committed
# we can use git commit-tree plumbing command to create new commit object
# with the tree object we just created
$ echo "initial commit" | git commit-tree 3a3aff7fa9639da674465c43fac565c1291f952b
86aa1cb0eec333b600d5b8c23c9c95d4983d5e6d

# now lets look at the log of current branch
# we should see new commit we just made
# but for some reason, we don't :(
$ git log --oneline
fatal: your current branch 'master' does not have any commits yet

# so where did that commit go then
# if you search for that object in .git/objects, we do see .git/objects/86/aa1cb0eec333b600d5b8c23c9c95d4983d5e6d
# then why didn't it show up on the git log?
# lets see what data we have in that object
$ cat .git/objects/86/aa1cb0eec333b600d5b8c23c9c95d4983d5e6d | zlib-flate -uncompress
commit 181tree 3a3aff7fa9639da674465c43fac565c1291f952b
author techgaun <coolsamar207@gmail.com> 1561256669 -0500
committer techgaun <coolsamar207@gmail.com> 1561256669 -0500

initial commit

# so we have the data such as tree the commit object was created from,
# author, committer and finally commit message
# now we come back to the same question we had
# why did the git log not show that commit?
# the reason is that this commit is not associated to the current branch
# we only created the commit object so far
# now we can do that using git update-ref plumbing command
# which updates .git/refs/heads/master file among other things
# we could have done: echo 86aa1cb0eec333b600d5b8c23c9c95d4983d5e6d > .git/refs/heads/master
# but git does it in a safer way while handling other side effects as necessary
$ git update-ref refs/heads/master 86aa1cb0eec333b600d5b8c23c9c95d4983d5e6d

# now lets see what happens with git log
# as you will see next, our commit is now part of master branch. Voila!
# we just made a commit to git without using normal commands we are used to with
$ git log --oneline
86aa1cb (HEAD -> master) initial commit

# now lets create another commit with the earlier commit id as the parent
# we repeat same stuff again, this time we create file with much larger content
$ printf 'Hello World.%.0s' {1..1000} > new.md

# lets check the status real quick
$ git status -s
?? new.md

# now lets add that file to staging area
# note that we will skip hash-object this time
# and the reason why it still works is because
# update-index goes through the process of hashing all the objects
# while adding them to the staging area
$ git update-index --add new.md

# and if we check status, we see its added to the staging area
$ git status -s
A  new.md

$ git write-tree
c4996cfea245445e4bdb0561bf18e29436568e58

# now lets inspect that tree
# we see that this tree contains complete snapshot of what we have in the git repo
$ git cat-file -p c4996cfea245445e4bdb0561bf18e29436568e58
100644 blob c7fc1d8f722cc984f6c90f4151de8b250eeb6343	new.md
100644 blob 557db03de997c86a4a028e1ebd3a1ceb225be238	readme.md

# and now lets make commit object with our newly created tree
# as you will see, we pass part of commit hash from first commit we made
# as you can see, we only passed 7 first characters of hash
# as long as git can resolve the part of hash into an object,
# we can use such short partials of sha1 hash
$ echo "Added new file" | git commit-tree c4996cfea245445e4bdb0561bf18e29436568e58 -p 86aa1cb
fed6ba87e445db5175c628cfecbbd0b83526a54a

# we can do cat-file on commit object as well
# note the parent line in this case
$ git cat-file -p fed6ba87e445db5175c628cfecbbd0b83526a54a
tree c4996cfea245445e4bdb0561bf18e29436568e58
parent 86aa1cb0eec333b600d5b8c23c9c95d4983d5e6d
author techgaun <coolsamar207@gmail.com> 1561258195 -0500
committer techgaun <coolsamar207@gmail.com> 1561258195 -0500

Added new file

# also, lets look at the type of commit object with cat-file
$ git cat-file -t fed6ba87e445db5175c628cfecbbd0b83526a54a
commit

# finally, lets update master ref to this commit
$ git update-ref refs/heads/master fed6ba87e445db5175c628cfecbbd0b83526a54a

# and lets check the git log one more time
# and we see things as expected
$ git log --oneline
fed6ba8 (HEAD -> master) Added new file
86aa1cb initial commit
```

## Other Examples

We will continue to operate on the above repository we created earlier

### gc and packfile

```shell
# lets look at the size of .git/objects once
# and as per the output below, we are at around 41K with our git object

$ du -b .git/objects/
4096	.git/objects/pack
4224	.git/objects/86
4096	.git/objects/info
4150	.git/objects/3a
4177	.git/objects/c4
4124	.git/objects/55
4227	.git/objects/0a
4255	.git/objects/fe
4207	.git/objects/c7
41652	.git/objects/

# and now lets look at the tree of .git directory after all the things we did
# hooks directory is not shown here to preserve space
$ tree .git
.git
├── branches
├── config
├── description
├── HEAD
├── hooks
├── index
├── info
│   └── exclude
├── logs
│   ├── HEAD
│   └── refs
│       └── heads
│           └── master
├── objects
│   ├── 0a
│   │   └── 9c3e68d37858d478ad2692e01126e6851d1c93
│   ├── 3a
│   │   └── 3aff7fa9639da674465c43fac565c1291f952b
│   ├── 55
│   │   └── 7db03de997c86a4a028e1ebd3a1ceb225be238
│   ├── 86
│   │   └── aa1cb0eec333b600d5b8c23c9c95d4983d5e6d
│   ├── c4
│   │   └── 996cfea245445e4bdb0561bf18e29436568e58
│   ├── c7
│   │   └── fc1d8f722cc984f6c90f4151de8b250eeb6343
│   ├── fe
│   │   └── d6ba87e445db5175c628cfecbbd0b83526a54a
│   ├── info
│   └── pack
└── refs
    ├── heads
    │   └── master
    └── tags

19 directories, 26 files

# Now lets see if we can optimize our repo like git promises to by running gc
$ git gc

# And once again, lets see the size of .git/objects
$ du -b .git/objects/
5855	.git/objects/pack
4150	.git/objects/info
4227	.git/objects/0a
18328	.git/objects/

# many of the objects are gone as we see above
# and our git object database is down to 18K
# if we look at the tree of .git repo, it will be different now
$ tree .git
.git
├── branches
├── config
├── description
├── HEAD
├── hooks
├── index
├── info
│   ├── exclude
│   └── refs
├── logs
│   ├── HEAD
│   └── refs
│       └── heads
│           └── master
├── objects
│   ├── 0a
│   │   └── 9c3e68d37858d478ad2692e01126e6851d1c93
│   ├── info
│   │   └── packs
│   └── pack
│       ├── pack-5dda0074f5c0745e99fad6c6d639ca69f009091e.idx
│       └── pack-5dda0074f5c0745e99fad6c6d639ca69f009091e.pack
├── packed-refs
└── refs
    ├── heads
    └── tags

13 directories, 24 files

# As we see above, we have .git/objects/pack with two files .idx and .pack
# git has optimized our repository and created packfile like we said earlier
# there's git show-index command to which you can pipe .idx file
# I leave that as homework for you to look into that and see what you will see in those
```
