---
layout: post
title: "How to Construct Git Objets Manually"
description: "How to Construct Git Objets Manually"
category: "Programming" 
comments: true
tags: [Git, Git Internals]
---

This post is an attempt to get a better sense of how Git store objects. I am going to build a simple legal Git repository manually.  
This article assumes basic knowledge about Git internals. I recommend you read the [Git document about it's internals](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects), the document shows how to interact with Git by low level powerful commands, e.g., `cat-file`, `write-tree`, `update-index`. 
## The Structure of .git Directory
First Let's create a simple Git directory to imitate(the Git version is 2.6.3).

```
[Steed:~/test/git]$ git init sample
Initialized empty Git repository in /Users/Steed/test/git/sample/.git/
[Steed:~/test/git]$ cd sample/
[Steed:~/test/git/sample]$ echo 'foo' > file1 && echo 'bar' > file2
[Steed:~/test/git/sample]$ git add -A
[Steed:~/test/git/sample]$ git commit -m "First commit"
[master (root-commit) 2cb7c65] First commit
 2 files changed, 2 insertions(+)
 create mode 100644 file1
 create mode 100644 file2
```
This repository only contains two files and one commit, let's take a look into the .git directory.

```
[Steed:~/test/git/sample]$ tree .git/
.git/
├── COMMIT_EDITMSG
├── HEAD
├── config
├── description
├── hooks
│   ├── applypatch-msg.sample
│   ├── commit-msg.sample
│   ├── post-update.sample
│   ├── pre-applypatch.sample
│   ├── pre-commit.sample
│   ├── pre-push.sample
│   ├── pre-rebase.sample
│   ├── prepare-commit-msg.sample
│   └── update.sample
├── index
├── info
│   └── exclude
├── logs
│   ├── HEAD
│   └── refs
│       └── heads
│           └── master
├── objects
│   ├── 25
│   │   └── 7cc5642cb1a054f08cc83f2d943e56fd3ebe99
│   ├── 2c
│   │   └── b7c65d3f594d1b597258aeda68759b4ae7dab3
│   ├── 57
│   │   └── 16ca5987cbf97d6bb54920bea6adde242d87e6
│   ├── f9
│   │   └── c36476895b0f9a475dfbaeb492332c63c148ec
│   ├── info
│   └── pack
└── refs
    ├── heads
    │   └── master
    └── tags
```
Let’s go over some of the important files.  

* `COMMIT_EDITMSG `: This is the last commit’s message.
* `HEAD`: Notice that the uppercase HEAD is different from the lowercase head, mostly it’s probably refs/heads/master or other branch head, sometimes it points to a commit that's not associated with any branches, which is called a detached HEAD.
* `config`: Configuration file of repository.
* `index`: A binary file containing current index files for the repository, including the staging and committed files.
* `hooks`: These files are custom scripts that are executed at certain times when working with Git, such as before or after a commit.
* `logs`: Contains history for different branches.
* `objects`: Git internal warehouse of objects.
* `refs/heads`: Records the head commit object of each branch.

There are four objects under `.git/objects` directory, containing two blob objects, one tree object and one commit object, we will reproduce them manually later.
## Create the Blob Object
Let's init a empty Git directory with same two files.

```
[Steed:~/test/git]$ cd manually
[Steed:~/test/git/manually]$ echo 'foo' > file1 && echo 'bar' > file2
[Steed:~/test/git/manually]$ ls -aF1
./
../
.git/
file1
file2
```
The form of a blob object:

```
blob [content size]\0[raw content]
```
Git constructs every a object started with a header, in this case a blob, the header starts with 'blob' plaintext, then the size of raw file and finally a null byte. The last part is the raw file content. Git hashes the content to get the sha1 value of the object. Then Git compress the content with zlib and store it into the file with the path according to the sha1.
I write a simple Python script `blob.py` under the `manually` folder to generate the blob objects, which accepts the raw files as argument, here's the code:

```python
import hashlib
import sys
import zlib
from pathlib import Path

def write_object(sha1, store):
    path = ".git/objects/" + sha1[0:2] + "/" + sha1[2:]
    p = Path(path)
    Path(p.parent).mkdir(parents=True, exist_ok=True)
    p = p.write_bytes(store)

for raw_file in sys.argv[1:]:
    raw_content = Path(raw_file).read_text()
    # Set the start string
    header = "blob {}\0".format(len(raw_content))
    blob = (header + raw_content).encode("ascii")
    # Calculate the sha1 value
    sha1 = hashlib.sha1(blob).hexdigest()
    store = zlib.compress(blob)
    write_object(sha1, store)
```
Run the script and use `git cat-file` command to validate the objects.

```
[Steed:~/test/git/manually]$ python blob.py file1 file2
[Steed:~/test/git/manually]$ find .git/objects -type f
.git/objects/25/7cc5642cb1a054f08cc83f2d943e56fd3ebe99
.git/objects/57/16ca5987cbf97d6bb54920bea6adde242d87e6
[Steed:~/test/git/manually]$ git cat-file -p 257cc5
foo
[Steed:~/test/git/manually]$ git cat-file -p 5716ca
bar
```
## Create the Tree Object
Tree object is a little confusing. We could use `git ls-tree` command to print the content of a tree commit in a readable format, let's run it under the former sample project to see the tree object content.

```
[Steed:~/test/git/sample]$ git ls-tree 2cb7c
100644 blob 257cc5642cb1a054f08cc83f2d943e56fd3ebe99	file1
100644 blob 5716ca5987cbf97d6bb54920bea6adde242d87e6	file2
```
Each line of the output contains the file mode, object type, object sha1 and the file name. Notice that the character between sha1 and file name is tab. But this's not how a tree object is saved, Git does more processing.
The format of a tree storage object:

```
tree [content size]\0[mode] [Entries having reference to other trees and blobs]]
```
The format of each entry having references to other trees and blobs:

```
[mode] [file/folder name]\0[sha1 of object]
```
Note that Git doesn't store sha1 value in plaintext, they are packed down to 20 bytes, each two-character pair is merged into a single hex value. At last, Git compresses the data before storing it physically just like blob object.
Here is a script using the raw output of `git ls-tree` command to generate the tree object:

```python
import binascii
import hashlib
import zlib
from pathlib import Path

def write_object(sha1, store):
    path = ".git/objects/" + sha1[0:2] + "/" + sha1[2:]
    p = Path(path)
    Path(p.parent).mkdir(parents=True, exist_ok=True)
    p = p.write_bytes(store)

store = b''

tree_data = '''\
100644 blob 257cc5642cb1a054f08cc83f2d943e56fd3ebe99	file1
100644 blob 5716ca5987cbf97d6bb54920bea6adde242d87e6	file2\
'''

for line in tree_data.split("\n"):
    mode, object_type, tail = line.split(" ")
    # The character betwween sha1 value and file name is tab
    sha1, file_name = tail.split("\t")
    store += (mode + " " + file_name).encode("ascii")
    store += b"\0" + binascii.unhexlify(sha1)


header = "tree {}\0".format(len(store))
store = header.encode("ascii") + store
# Calculate the sha1 value
tree_sha1 = hashlib.sha1(store).hexdigest()

store = zlib.compress(store)
write_object(tree_sha1, store)
```

## Create the Commit Object
The stored format of commit object is almost the same with the `git cat-file` command output. Let's run it under the former sample repository:

```
[Steed:~/test/git/sample]$ git cat-file -p 2cb7c
tree f9c36476895b0f9a475dfbaeb492332c63c148ec
author bittenApple <mailofmj@163.com> 1483717925 +0800
committer bittenApple <mailofmj@163.com> 1483717925 +0800

First commit
```
The output is

* The tree commit
* (There should be one or two parent commit sha1 if not first commit)
* The author info (Including the name, email, timestamp, even the timezone info!)
* The committer info
* The commit message

Git add the header for a commit object:

```
commit [content size]\0[the aforementioned output]
```
Here is the script to generate the commit object:

```python
import hashlib
import zlib
from pathlib import Path

def write_object(sha1, store):
    path = ".git/objects/" + sha1[0:2] + "/" + sha1[2:]
    p = Path(path)
    Path(p.parent).mkdir(parents=True, exist_ok=True)
    p = p.write_bytes(store)

commit_data = '''\
tree f9c36476895b0f9a475dfbaeb492332c63c148ec
author bittenApple <mailofmj@163.com> 1483717925 +0800
committer bittenApple <mailofmj@163.com> 1483717925 +0800

First commit
'''

header = "commit {}\0".format(len(commit_data))
commit = (header + commit_data).encode("ascii")
sha1 = hashlib.sha1(commit).hexdigest()
store = zlib.compress(commit)
write_object(sha1, store)
```
Put it under `manully` repository and run it, then all objects to imitate are generated.

```
[Steed:~/test/git/manually]$ python commit.py
[Steed:~/test/git/manually]$ find .git/objects -type f
.git/objects/25/7cc5642cb1a054f08cc83f2d943e56fd3ebe99
.git/objects/2c/b7c65d3f594d1b597258aeda68759b4ae7dab3
.git/objects/57/16ca5987cbf97d6bb54920bea6adde242d87e6
.git/objects/f9/c36476895b0f9a475dfbaeb492332c63c148ec
```
Besides, we should fix the content of `.git/refs/heads/master` file, which tells Git the head commit of master, and reset the Git index info.

```
[Steed:~/test/git/manually]$ mkdir -p .git/refs/heads
[Steed:~/test/git/manually]$ echo 2cb7c65d3f594d1b597258aeda68759b4ae7dab3 > .git/refs/heads/master
[Steed:~/test/git/manually]$ git reset --hard HEAD
HEAD is now at 2cb7c65 First commit

```
Let's run `git status` and `git log` commands and Git looks fine(the python scripts are still untracked)

```
[Steed:~/test/git/manually]$ git log
commit 2cb7c65d3f594d1b597258aeda68759b4ae7dab3
Author: bittenApple <mailofmj@163.com>
Date:   Fri Jan 6 23:52:05 2017 +0800

    First commit
[Steed:~/test/git/manually]$ git status
On branch master
Untracked files:
  (use "git add <file>..." to include in what will be committed)

	blob.py
	commit.py
	tree.py
```

Hope this post will help you have a better understand of the format of different Git objects.

## References
[https://git-scm.com/book/en/v2/Git-Internals-Git-Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)  
[http://stackoverflow.com/questions/14790681/what-is-the-internal-format-of-a-git-tree-object](http://stackoverflow.com/questions/14790681/what-is-the-internal-format-of-a-git-tree-object) 
