# PES Version Control System (PES-VCS)

**Name:** Advika Raj  
**SRN:** PES1UG24CS906  
**Repository:** [PES1UG24CS906-pes-vcs](https://github.com/Lilac3010/PES1UG24CS906-pes-vcs)  
**Section:** E

---

## Table of Contents

- [Phase 1](#phase-1)
- [Phase 2](#phase-2)
- [Phase 3](#phase-3)
- [Phase 4](#phase-4)
- [Questions](#questions)
  - [Q5.1 — Implementing `pes checkout <branch>`](#q51--implementing-pes-checkout-branch)
  - [Q5.2 — Detecting a Dirty Working Directory](#q52--detecting-a-dirty-working-directory)
  - [Q5.3 — Detached HEAD](#q53--detached-head)
  - [Q6.1 — Finding and Deleting Unreachable Objects](#q61--finding-and-deleting-unreachable-objects)
  - [Q6.2 — Race Condition Between GC and Commit](#q62--race-condition-between-gc-and-commit)

---

## Phase 1

### 1A


### 1B


---

## Phase 2

### 2A


### 2B


---

## Phase 3

### 3A and 3B


---

## Phase 4

### 4A


### 4B


### 4C


---

## Questions

### Q5.1 — Implementing `pes checkout <branch>`

To implement `pes checkout`, three components inside `.pes/` must be updated: **HEAD**, **working directory**, and **index**.

#### 1. Update HEAD

The HEAD must point to the target branch by rewriting `.pes/HEAD` to:

```
ref: refs/heads/<branch>
```

The actual branch file remains unchanged; only the reference is updated.

#### 2. Update the Working Directory

The working directory must match the snapshot of the target branch. This involves:

- Reading the commit object of the branch tip
- Extracting its tree hash
- Traversing the tree recursively
- Writing each blob's content back to disk

File handling rules:
- **Not present in the new tree** → deleted
- **Different from current** → overwritten

#### 3. Update the Index

The index must be updated to match the new tree exactly. Each entry should include:

- `mode`
- `hash`
- modification time (`mtime`)
- `size`
- file path

#### Handling File States

| File State | Action |
|---|---|
| Untracked | Preserved |
| Tracked and unchanged | Safe to overwrite |
| Modified (dirty) | Checkout should be **refused** |

#### Order of Operations

To avoid inconsistencies, checkout should:
1. Update files first
2. Then update HEAD

This ensures that even if a crash occurs mid-operation, the repository remains logically consistent.

---

### Q5.2 — Detecting a Dirty Working Directory

A dirty working directory can be detected using only the **index** and **object store**.

#### Detection Algorithm

For each file tracked in the index:

1. Use `stat()` to compare file metadata (`size` and `mtime`)
2. If metadata differs → file **may** be modified
3. Read the file and compute its **SHA-256 hash**
4. Compare it with the stored hash in the index

#### Outcomes

| Condition | Result |
|---|---|
| Hash differs | File is **modified** |
| File missing on disk | Treated as **deleted** (dirty) |

#### Checkout Behavior

- If a file is modified locally and differs from the target branch → **refuse checkout**
- If both branches have the same version of the file → safe, but usually still refused for safety

This method works entirely locally using `.pes/index` and `.pes/objects`.

---

### Q5.3 — Detached HEAD

A **detached HEAD** occurs when `.pes/HEAD` stores a commit hash directly instead of a branch reference.

```
# Normal HEAD
ref: refs/heads/main

# Detached HEAD
a3f1c9b2d4e6f8a0b1c2d3e4f5a6b7c8d9e0f1a2
```

#### Behavior When Committing in Detached HEAD State

- A new commit is created normally
- HEAD is updated with the new commit hash
- **No branch is updated**

#### The Problem

If the user switches branches, the detached commit becomes **unreachable**:
- It still exists in `.pes/objects`
- But no reference points to it
- It may eventually be garbage collected

#### Recovery

If the commit hash is known, create a new branch pointing to it:

```bash
# Write hash to a new branch file
echo <commit_hash> > .pes/refs/heads/<new-branch>

# Then checkout that branch
pes checkout <new-branch>
```

> **Note:** Unlike Git, PES-VCS does not have a `reflog`, so recovery depends entirely on remembering or having noted the commit hash.

---

### Q6.1 — Finding and Deleting Unreachable Objects

Unreachable objects are those not referenced by any branch. This is solved using a **mark-and-sweep** algorithm.

#### Mark Phase

1. Start from all branch references in `.pes/refs/heads/*`
2. Include HEAD if detached
3. Traverse the full object graph:
   - commits → trees → blobs
4. Store all reachable hashes in a **hash set**

#### Sweep Phase

1. Traverse all files in `.pes/objects/`
2. If an object hash is **not** in the reachable set → delete it

#### Why a Hash Set?

- Fast lookup: **O(1)** average time
- Efficient for large repositories
- Minimal memory overhead relative to object count

#### Scalability

For large repositories (~100k commits, millions of objects), the operation remains feasible in seconds to minutes, since each object is visited at most once.

---

### Q6.2 — Race Condition Between GC and Commit

A race condition occurs when **garbage collection (GC)** runs concurrently with a new **commit being created**.

#### The Problem Scenario

```
T1: GC computes the set of reachable objects
T2: New commit is created; new objects written to .pes/objects/
T3: GC sweeps — new objects are NOT in the reachable set → deleted
```

This results in:
- Broken commits
- Potential repository corruption

#### Solutions

**Option 1: Grace Period (Git's approach)**

- Unreachable objects are kept for ~2 weeks before deletion
- If an object is referenced again before expiry, it is preserved
- Practical and scalable
- Temporarily allows storage bloat

**Option 2: Locking**

- Acquire an exclusive lock (e.g., `flock()`) on the repository during GC
- Prevent any commits from occurring while GC runs
- Guarantees safety
- Blocks users and reduces concurrency

#### Tradeoff Summary

| Approach | Safety | User Impact |
|---|---|---|
| Grace period | High (with small risk window) | Non-blocking |
| Locking | Guaranteed | Blocks during GC |
