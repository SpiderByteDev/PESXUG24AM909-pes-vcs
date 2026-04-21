# PES-VCS — Version Control System Lab Report
**Name:** Ayush  
**SRN:** PES1UG24AM909  
**Repository:** PESXUG24AM909-pes-vcs

---

## Table of Contents
1. [Phase 1 — Object Storage](#phase-1--object-storage)
2. [Phase 2 — Tree Objects](#phase-2--tree-objects)
3. [Phase 3 — Index / Staging Area](#phase-3--index--staging-area)
4. [Phase 4 — Commits and History](#phase-4--commits-and-history)
5. [Integration Test](#integration-test)
6. [Analysis Questions — Branching](#analysis-questions--branching)
7. [Analysis Questions — Garbage Collection](#analysis-questions--garbage-collection)

---

## Phase 1 — Object Storage

### Screenshot 1A — `./test_objects` passing
![1A](screenshots/Screenshot%202026-04-21%20122341.png)

### Screenshot 1B — Sharded object directory
![1B](screenshots/Screenshot%202026-04-21%20122508.png)

---

## Phase 2 — Tree Objects

### Screenshot 2A — `./test_tree` passing
![2A](screenshots/Screenshot%202026-04-21%20122706.png)

### Screenshot 2B — Raw binary tree object (`xxd`)
![2B](screenshots/Screenshot%202026-04-21%20124906.png)

---

## Phase 3 — Index / Staging Area

### Screenshot 3A — `pes init` → `pes add` → `pes status`
![3A](screenshots/Screenshot%202026-04-21%20125304.png)

### Screenshot 3B — `cat .pes/index`
![3B](screenshots/Screenshot%202026-04-21%20125355.png)

---

## Phase 4 — Commits and History

### Screenshot 4A — `pes log` with three commits
![4A](screenshots/Screenshot%202026-04-21%20125517.png)
![4a](screenshots/Screenshot%202026-04-21%20125529.png)

### Screenshot 4B — `find .pes -type f | sort` showing object growth
![4B](screenshots/Screenshot%202026-04-21%20125616.png)


### Screenshot 4C — `cat .pes/refs/heads/main` and `cat .pes/HEAD`
![4c](screenshots/Screenshot%202026-04-21%20125704.png)


---

## Integration Test
![integration-1](screenshots/Screenshot%202026-04-21%20131356.png)
![integration-2](screenshots/Screenshot%202026-04-21%20131414.png)


### Screenshot — `make test-integration`


---

## Analysis Questions — Branching

### Q5.1 — How would you implement `pes checkout <branch>`?

At its core, a branch in PES-VCS is just a plain text file sitting inside `.pes/refs/heads/` that contains a commit hash. So creating a branch is literally just creating a file — which is kind of elegant when you think about it.

To implement `pes checkout <branch>`, here's what needs to happen:

1. **Read the branch file** at `.pes/refs/heads/<branch>` to get the target commit hash. If it doesn't exist, error out.
2. **Read the commit object** to get its tree hash.
3. **Recursively walk the tree** and restore every file to the working directory — writing file contents from the blob objects back to disk.
4. **Update `.pes/HEAD`** to contain `ref: refs/heads/<branch>` so future commits go to the right branch.
5. **Rebuild the index** to match the checked-out tree, so `pes status` reports correctly after the switch.

What makes this genuinely tricky is handling the working directory cleanly. You can't just blast new files in — you also need to **delete files** that exist in the current branch but not the target one. And if there are subdirectories involved, you have to recurse through the whole tree structure. It's a lot more than just swapping a pointer.

---

### Q5.2 — How do you detect a "dirty working directory" conflict?

Before actually switching branches, you need to check whether the user has unsaved changes that would be silently overwritten. Here's how you'd catch that using only what's already available in the index and object store:

1. **Check for unstaged modifications**: For each entry in the current index, compare its stored `mtime` and `size` against the actual file on disk. If either differs, the file has been modified since it was last staged — that's a dirty file.
2. **Check for cross-branch conflicts**: For each dirty file found above, look up what blob hash the *target branch* has for that same path (by reading the target commit's tree). If the target branch has a different version of that file than what's currently staged, and the working file is dirty on top of that — you have a real conflict.
3. **If any conflict exists, abort** and print something like: `error: your changes to 'file.txt' would be overwritten by checkout`.

The nice thing is you don't need any extra data — the index already stores enough metadata (`mtime`, `size`, blob hash) to detect all of this without re-reading file contents.

---

### Q5.3 — What is "Detached HEAD" and how do you recover from it?

Normally, `.pes/HEAD` contains something like `ref: refs/heads/main` — meaning HEAD points to a *branch*, not a commit directly. When you're in detached HEAD state, HEAD instead contains a raw commit hash like `d748cb7b...`. There's no branch involved.

If you make commits while in this state, they do get written to the object store correctly, and HEAD gets updated to each new commit hash. So the commits *exist* — they're just not attached to any branch. The moment you switch to another branch, nothing points to those commits anymore. They're like islands with no path to them, and eventually GC would clean them up.

To recover, you'd need the commit hash — which you might still have in your terminal's scroll history or remember from the output. Then you can manually create a branch pointing to it:

```bash
echo "<your-commit-hash>" > .pes/refs/heads/recovery
```

Then checkout that branch normally. Git also has `git reflog` for this exact situation, which keeps a log of everywhere HEAD has pointed — that's the real safety net.

---

## Analysis Questions — Garbage Collection

### Q6.1 — How would you find and delete unreachable objects?

Over time, objects accumulate in `.pes/objects/` that nothing points to anymore — maybe from a commit that was never finished, or a file that was staged and then unstaged. These are "unreachable" objects and they just waste space.

Here's a clean algorithm to find and remove them:

1. **Collect all roots**: Start by reading every file in `.pes/refs/heads/` to get all branch tip commit hashes. These are your starting points.
2. **Mark reachable objects**: For each commit, mark it as reachable, then follow its `tree` pointer (mark the tree), then recurse into all subtrees and blobs (mark each of those). Follow each commit's `parent` pointer and repeat until you hit the first commit.
3. **Sweep unreachable objects**: Walk every file under `.pes/objects/`. Any object whose hash is NOT in your reachable set gets deleted.

The right data structure for the reachable set is a **hash set** — you need fast O(1) lookups when checking whether a given object hash has been marked. A sorted array with binary search would also work.

For a repo with 100,000 commits and 50 branches: assuming an average of ~3 objects per commit (1 commit + 1 root tree + ~1 blob), you'd be visiting roughly **300,000 objects** during the mark phase. That's very manageable — fits easily in memory as a hash set.

---

### Q6.2 — Why is it dangerous to run GC at the same time as a commit?

This is a classic time-of-check vs time-of-use race condition, and it's subtle enough to cause real data corruption.

Here's exactly how it can go wrong:

1. `pes add` writes a new blob object to `.pes/objects/` — the blob exists on disk.
2. GC starts its scan *right at this moment*. It sees the blob but no commit references it yet, so it marks it **unreachable** and deletes it.
3. `pes commit` now runs and tries to build a tree that references that blob hash — but the file is gone. The commit either fails or, worse, writes a commit object that points to a non-existent blob, silently corrupting the repository.

Git avoids this with two strategies:
- **Grace period**: Git's GC only prunes objects older than 2 weeks by default (`gc.pruneExpire = 2.weeks.ago`). A freshly written blob will never be deleted, giving any in-progress operation plenty of time to finish referencing it.
- **Lock file**: Git writes a `gc.pid` lock file before starting GC, preventing two GC processes from running simultaneously and competing over the same objects.

The grace period is the key insight — it decouples "object exists" from "object is referenced," which is what makes safe concurrent operation possible.

---

*Report generated for PES University — Operating Systems Lab*
