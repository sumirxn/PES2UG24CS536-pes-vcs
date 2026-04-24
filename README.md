# PES-VCS: Version Control System from Scratch

**Name:** Sumiran  
**SRN:** PES2UG24CS536  
**Platform:** Ubuntu 22.04  

---

## Overview

This report documents the implementation of PES-VCS, a local version control system built from scratch in C. The system tracks file changes, stores snapshots efficiently using content-addressable storage, and supports commit history. The design is based directly on how Git works internally.

---

## Phase 1: Object Storage Foundation

### What was implemented

The object store is the foundation of PES-VCS. Every piece of data (file contents, directory listings, commits) is stored as an object named by its SHA-256 hash. Objects are stored under `.pes/objects/XX/YYYY...` where `XX` is the first two hex characters of the hash, which shards the directory to avoid too many files in one place.

Two functions were implemented in `object.c`:

`object_write` takes raw data and a type (blob, tree, or commit), builds a header of the form `"type size\0"`, combines it with the data, computes the SHA-256 hash of the full object, and writes it atomically to disk using a temp file followed by a rename. If the object already exists (same hash), it skips the write entirely for deduplication.

`object_read` takes a hash, reads the corresponding file from the object store, recomputes the hash and compares it to the expected value for integrity verification, parses the header to extract the type and size, and returns the raw data portion to the caller.

### Key concepts

Content-addressable storage means the name of every object is determined by its content. This gives deduplication (identical files stored once), integrity checking (hash mismatch means corruption), and immutability (changing content produces a different hash and therefore a different object).

Atomic writes using the temp-file-then-rename pattern ensure that a crash mid-write never leaves a corrupt object in the store. On POSIX systems, `rename()` is atomic.

### Screenshots

**Screenshot 1A:** Output of `./test_objects` showing all tests passing.

<img width="797" height="145" alt="Screenshot from 2026-04-23 09-39-16" src="https://github.com/user-attachments/assets/349c19e0-004a-4b95-a3dd-42347dc39e92" />


**Screenshot 1B:** Output of `find .pes/objects -type f` showing the sharded directory structure.

<img width="797" height="75" alt="Screenshot from 2026-04-23 09-39-51" src="https://github.com/user-attachments/assets/23d729df-e3e0-465b-8630-b0ec3b8e2bd6" />


---

## Phase 2: Tree Objects

### What was implemented

A tree object represents a directory snapshot. Each entry in a tree maps a name to either a blob (file) or another tree (subdirectory), along with the file mode.

`tree_from_index` was implemented in `tree.c`. It loads the current index and recursively builds a tree hierarchy. For flat files (no slash in the path), entries are added directly as blob entries. For nested paths like `src/main.c`, the function groups all entries sharing the same directory prefix, recurses to build a subtree for that directory, and adds the subtree as a directory entry with mode `040000`. Each level is serialized and written to the object store before returning its hash.

### Key concepts

Trees are recursive structures. A root tree points to blobs (files) and other trees (subdirectories). This mirrors how a real filesystem directory works. Because trees are also content-addressed, two commits that share unchanged subtrees will point to the same tree objects, saving space.

### Screenshots

**Screenshot 2A:** Output of `./test_tree` showing all tests passing.

<img width="619" height="92" alt="Screenshot from 2026-04-23 18-23-03" src="https://github.com/user-attachments/assets/b4963f20-b9c4-47bf-8fca-6f5460ff6441" />


**Screenshot 2B:** Output of `xxd` on a raw tree object showing the binary format.

<img width="1064" height="68" alt="image" src="https://github.com/user-attachments/assets/5b3aa96a-ff58-4c24-9364-22c6277728b4" />


---

## Phase 3: The Index (Staging Area)

### What was implemented

The index is a text file at `.pes/index` that tracks which files are staged for the next commit. Each line has the format:

```
<mode-octal> <64-char-hex-hash> <mtime-seconds> <size> <path>
```

Three functions were implemented in `index.c`:

`index_load` opens `.pes/index` and parses each line using `fscanf`. If the file does not exist, it initializes an empty index without returning an error, since no staged files is a valid state.

`index_save` sorts entries by path using `qsort`, writes them to a temp file, calls `fsync` to flush to disk, closes the file, and atomically renames the temp file over the real index file.

`index_add` reads a file from the working directory, writes its contents as a blob object, reads the file metadata (mode, size, mtime) using `lstat`, and updates or creates the corresponding index entry before saving.

### Key concepts

The index acts as a preparation area between the working directory and the repository. Files must be explicitly staged before they appear in a commit. The text format makes it human-readable and easy to debug with `cat .pes/index`.

Atomic writes are used here too. If a crash happens during `index_save`, the old index file is left intact and the temp file is discarded.

### Screenshots

**Screenshot 3A:** Output of `./pes init`, `./pes add file1.txt file2.txt`, and `./pes status`.

<img width="587" height="510" alt="Screenshot from 2026-04-24 10-14-07" src="https://github.com/user-attachments/assets/b31ac562-603f-4cdf-9ba6-7b508be6dfdb" />


**Screenshot 3B:** Output of `cat .pes/index` showing the text-format index.

<img width="844" height="64" alt="Screenshot from 2026-04-24 10-14-29" src="https://github.com/user-attachments/assets/b0d61c86-0260-48ff-ab5f-fd409a9f4910" />


---

## Phase 4: Commits and History

### What was implemented

`commit_create` was implemented in `commit.c`. It performs the following steps in order:

1. Calls `tree_from_index` to build a tree from the staged files and get the root tree hash.
2. Calls `head_read` to get the current HEAD commit hash as the parent. For the first commit, this fails and `has_parent` is set to 0.
3. Reads the author string from the `PES_AUTHOR` environment variable using `pes_author`.
4. Records the current Unix timestamp using `time(NULL)`.
5. Serializes the commit struct to text using `commit_serialize`.
6. Writes the serialized text as a commit object using `object_write`.
7. Updates HEAD to point to the new commit using `head_update`.

### Key concepts

Commits form a linked list on disk. Each commit stores the hash of its parent, so walking history means repeatedly reading the parent field until a commit with no parent is reached. This is how `pes log` works.

The branch file (`.pes/refs/heads/main`) stores only the hash of the latest commit. HEAD stores a reference to the branch file (`ref: refs/heads/main`). When a new commit is created, only the branch file is updated. HEAD automatically follows because it points to the branch.

### Screenshots

**Screenshot 4A:** Output of `./pes log` showing three commits with hashes, authors, timestamps, and messages.

<img width="844" height="490" alt="Screenshot from 2026-04-24 10-19-59" src="https://github.com/user-attachments/assets/d47c36d6-4b40-462e-8bba-340a07ccdfe9" />


**Screenshot 4B:** Output of `find .pes -type f | sort` showing object store growth after three commits.

<img width="844" height="252" alt="Screenshot from 2026-04-24 10-21-17" src="https://github.com/user-attachments/assets/57b0521a-1265-4765-bc7f-282896a017e7" />


**Screenshot 4C:** Output of `cat .pes/refs/heads/main` and `cat .pes/HEAD` showing the reference chain.

<img width="844" height="75" alt="Screenshot from 2026-04-24 10-21-36" src="https://github.com/user-attachments/assets/f866b75b-620c-412d-afeb-783d84a598ca" />


**Integration test:** Output of `make test-integration` showing all tests passing.

<img width="579" height="868" alt="Screenshot from 2026-04-24 10-23-44" src="https://github.com/user-attachments/assets/c792fe71-09f7-4314-92a5-2441a8151b37" /> <br>
<img width="579" height="411" alt="Screenshot from 2026-04-24 10-24-02" src="https://github.com/user-attachments/assets/97b55805-099f-42e3-99e2-222c7282919b" />



---

## Phase 5: Branching and Checkout (Analysis)

### Q5.1: How would you implement `pes checkout <branch>`?

A branch in PES-VCS is a file at `.pes/refs/heads/<branch>` containing a commit hash. Creating a branch means creating that file. Switching to a branch involves the following steps:

First, read the target branch file to get the commit hash. Then read the commit object to get its tree hash. Walk the tree recursively to get the full set of files and their blob hashes that the target branch represents.

The files that need to change in `.pes/` are:

- `HEAD` must be updated to contain `ref: refs/heads/<branch>` so it points to the new branch.
- The working directory files must be updated to match the target tree.

To update the working directory, for each file in the target tree, read the corresponding blob and write it to disk at the correct path. Files that exist in the current tree but not in the target tree must be deleted. Files that exist in both but have different blob hashes must be overwritten.

What makes this complex is handling conflicts, nested directories (trees must be walked recursively), file permissions, and ensuring that new directories are created and empty directories are removed as needed.

### Q5.2: How would you detect a dirty working directory conflict before checkout?

The index stores the mtime and size of each file at the time it was staged. The object store contains the blob for each staged file. A file in the working directory is considered dirty if either of the following is true:

1. Its current mtime or size differs from the values stored in the index entry, which indicates it has been modified since it was last staged.
2. Its blob hash (computed fresh) differs from the hash stored in the index entry.

To detect a conflict before checkout, for each file that differs between the current branch tree and the target branch tree, check whether the working directory version of that file is dirty using the method above. If any such file is dirty, refuse the checkout and report the conflicting paths to the user.

This approach uses only the index and the object store. No external tools or additional metadata are needed.

### Q5.3: What happens if you make commits in detached HEAD state, and how do you recover?

In detached HEAD state, `.pes/HEAD` contains a raw commit hash instead of a branch reference like `ref: refs/heads/main`. When a new commit is created in this state, `head_update` writes the new commit hash directly into `HEAD` rather than updating a branch file. The commits are valid objects in the object store and form a proper chain.

The problem is that no branch points to these commits. Once HEAD moves away (for example by checking out a branch), there is no reference keeping those commits reachable. A garbage collector would eventually delete them.

To recover the commits, the user needs to know the hash of the last commit made in detached HEAD state. If they recorded it (for example from the output of `pes commit`), they can create a new branch pointing to that hash by writing the hash directly into a new branch file:

```
echo "<commit-hash>" > .pes/refs/heads/recovery-branch
```

This makes the commit and all its ancestors reachable again through the new branch.

---

## Phase 6: Garbage Collection (Analysis)

### Q6.1: Algorithm to find and delete unreachable objects

The algorithm is a mark-and-sweep approach:

**Mark phase:** Start from every branch ref in `.pes/refs/heads/`. For each branch, read the commit it points to. From that commit, mark the commit hash, the tree hash, and recursively all blob and subtree hashes reachable from the tree. Then follow the parent pointer to the previous commit and repeat. Continue until every commit in every branch's history has been visited.

The data structure used to track reachable hashes is a hash set (for example a hash table or a sorted array with binary search). Lookup must be O(1) or O(log n) because the number of objects can be very large.

**Sweep phase:** Walk every file in `.pes/objects/` using `find` or `readdir`. For each file, compute the object hash from its path. If the hash is not in the reachable set, delete the file.

For a repository with 100,000 commits and 50 branches, assuming an average of 10 objects per commit (blobs, trees, and the commit itself), there would be roughly 1,000,000 objects to visit during the mark phase. In practice, many objects are shared across commits (unchanged files), so the reachable set would be smaller than 1,000,000 unique hashes. All objects in the store (potentially more than 1,000,000 including unreachable ones) would be visited during the sweep phase.

### Q6.2: Race condition between garbage collection and a concurrent commit

Consider the following sequence of events:

1. A commit operation calls `object_write` for a new blob and stores it in the object store. The blob is now on disk but no commit or tree points to it yet.
2. The garbage collector runs its mark phase at this exact moment. It walks all reachable objects starting from the branch refs. The new blob is not reachable yet because the commit that will reference it has not been written. The GC does not mark the blob.
3. The GC runs its sweep phase and deletes the blob because it is not in the reachable set.
4. The commit operation continues, writes the tree that references the blob, writes the commit that references the tree, and updates HEAD. The repository now contains a commit whose tree references a blob that no longer exists on disk. The repository is corrupt.

Git avoids this race condition in several ways. It keeps a grace period: objects newer than a certain age (typically two weeks) are never deleted by GC, even if they appear unreachable. This gives any in-progress operations time to complete and make their objects reachable before GC can touch them. Git also uses lock files during critical operations so that GC and commit do not run simultaneously on the same repository.

---
