PES-VCS — Version Control System

_____________________________________________________________________________

Name:  Shweta Mailaragouda Patil

SRN: PES2UG25CS825

______________________________________________________________________________



Phase 1: Object Storage

What I implemented
object_write: Builds a header (blob/tree/commit <size>\0), combines it with data, computes SHA-256 hash, writes atomically using temp file + rename, shards into .pes/objects/XX/ directories.
object_read: Reads object file, verifies integrity by recomputing hash, parses type from header, returns data portion.

**📸 Screenshot 1A:** Output of `./test_objects` showing all tests passing.

<img width="944" height="94" alt="image" src="https://github.com/user-attachments/assets/01bb81fd-c783-4f4a-b172-b67e4880582b" />


**📸 Screenshot 1B:** `find .pes/objects -type f` showing the sharded directory structure.

<img width="943" height="160" alt="image" src="https://github.com/user-attachments/assets/a84110cc-b7f1-4346-95b3-bc2053bd2c83" />


---

## Phase 2: Tree Objects

What I implemented
tree_from_index: Loads the index, builds a Tree struct from staged entries, serializes it to binary format and writes it to the object store.

**📸 Screenshot 2A:** Output of `./test_tree` showing all tests passing.

<img width="946" height="146" alt="image" src="https://github.com/user-attachments/assets/d990791b-ab46-4b8c-8402-5a775deb9958" />


**📸 Screenshot 2B:** Pick a tree object from `find .pes/objects -type f` and run `xxd .pes/objects/XX/YYY... | head -20` to show the raw binary format.

<img width="940" height="77" alt="image" src="https://github.com/user-attachments/assets/4312b91f-2a9a-46eb-b8b3-9d1a1a5f203c" />

---

## Phase 3: The Index (Staging Area)


What I implemented
index_load: Opens .pes/index, reads each line parsing mode, hash, mtime, size and path. If file doesn't exist, initializes empty index.
index_save: Sorts entries by path, writes to temp file atomically using fsync + rename.
index_add: Reads file contents, writes blob to object store, gets metadata via stat(), updates or adds index entry.

**📸 Screenshot 3A:** Run `./pes init`, `./pes add file1.txt file2.txt`, `./pes status` — show the output.

<img width="944" height="202" alt="image" src="https://github.com/user-attachments/assets/be46dbca-9beb-4852-9cce-3a0239355a57" />


**📸 Screenshot 3B:** `cat .pes/index` showing the text-format index with your entries.

<img width="944" height="61" alt="image" src="https://github.com/user-attachments/assets/809711b7-09ea-4eb4-8385-6ac9b7488960" />

---

## Phase 4: Commits and History

What I implemented
commit_create: Builds tree from index, reads parent from HEAD (if exists), sets author/timestamp/message, serializes commit, writes to object store, updates HEAD.
**📸 Screenshot 4A:** Output of `./pes log` showing three commits with hashes, authors, timestamps, and messages.

<img width="940" height="586" alt="image" src="https://github.com/user-attachments/assets/71444272-6a88-4d85-ac04-9ed6a5c89d1e" />


**📸 Screenshot 4B:** `find .pes -type f | sort` showing object store growth after three commits.

<img width="946" height="208" alt="image" src="https://github.com/user-attachments/assets/c6790ef1-3a2a-4586-8e13-91a6b7bb5478" />


**📸 Screenshot 4C:** `cat .pes/refs/heads/main` and `cat .pes/HEAD` showing the reference chain.

<img width="944" height="46" alt="image" src="https://github.com/user-attachments/assets/9c156501-bdba-4fda-90b7-25b04756a897" />


---

## Phase 5 & 6: Analysis-Only Questions


Q5.1 — How would pes checkout work?
To implement pes checkout <branch>, the following changes are needed in .pes/:

1.Read .pes/refs/heads/<branch> to get the target commit hash.
2.Update .pes/HEAD to contain ref: refs/heads/<branch>.
3.Read the target commit object, get its tree hash.
4.Walk the tree recursively and restore every file in the working directory to match that tree's blobs.
What makes it complex: if the user has modified files in the working directory that differ between the current and target branch, we must either refuse (to avoid data loss) or merge carefully. You also need to handle new files that exist in the target branch but not the current one, and delete files that exist in the current branch but not the target.

Q5.2 — Detecting dirty working directory conflicts
For each entry in the index:

1.Use stat() on the working directory file to get its current mtime and size.
2.Compare these against the stored mtime_sec and size in the index entry.
3.If they differ, the file has been modified since it was staged — this is a conflict.
4.Additionally, check if the file exists in the target branch's tree by reading that tree object from the object store. If the blob hash differs between the current index and the target tree, and the working file is also dirty, refuse checkout.
This approach avoids re-hashing every file (expensive) by using metadata as a fast-path check, exactly like Git's index design.

Q5.3 — Detached HEAD
When HEAD contains a commit hash directly (instead of ref: refs/heads/main), you are in detached HEAD state. New commits are created and chained correctly, but no branch pointer is updated — so the commits are only reachable by their hash. If you switch branches, those commits become unreachable and will eventually be deleted by garbage collection.

To recover: run git branch new-branch <hash> using the hash of the last commit you made in detached HEAD state. This creates a branch pointing to that commit, making the whole chain reachable again.

Phase 6: Garbage Collection Analysis
Q6.1 — Finding and deleting unreachable objects
Algorithm:

1.Start from all branch refs in .pes/refs/heads/ — collect every commit hash.
2.For each commit: mark it reachable, then follow its tree pointer and mark that reachable.
3.For each tree: recursively walk all entries — mark every blob and subtree reachable.
4.Follow each commit's parent pointer and repeat until no parent exists.
5.Walk all files in .pes/objects/ — any hash NOT in the reachable set can be deleted.
Data structure: A hash set (like a C unordered_set or a boolean array indexed by hash prefix) to track reachable hashes efficiently. Lookup and insert are O(1) average.

Estimate for 100,000 commits + 50 branches: Each commit points to a tree, and each tree might have ~10–50 entries. Roughly: 100,000 commits + 100,000 trees + ~500,000 blobs = around 600,000–700,000 objects to visit.

Q6.2 — Race condition between GC and commit
Race condition:

1.A commit operation writes a new blob to the object store but hasn't yet written the commit object pointing to it.
2.GC runs at this exact moment — it walks all reachable objects from existing refs, doesn't find the new blob (no commit points to it yet), and deletes it.
3.The commit operation finishes and writes a commit pointing to the now-deleted blob — the repository is now corrupt.
How Git avoids this: Git uses a "grace period" — objects newer than a certain age (default 2 weeks) are never deleted by GC, even if they appear unreachable. This gives any in-progress operation time to finish and create references to new objects before GC can touch them.

