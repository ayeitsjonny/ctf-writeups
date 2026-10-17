# One More Commit Before Kopitiam

| | |
|---|---|
| **Category** | Forensics |
| **Points** | 478 |
| **Solves** | 10 |
| **Author** | Anonymous |
| **Files** | `kopitiam.bundle`, `build_receipt.txt` |

## Challenge description

> An incident note vanished with its Git branch. A build receipt still points to something the visible history cannot reach.

## TL;DR

The bundle only advertises `main`, but its packfile still contains a commit that no branch points to. The build receipt gives the prefix of that commit's hash. Reading the commit's `incident_note.txt` gives the flag.

**Flag:** `CYBLTZ2026{r3fL0g_b3f0r3_br34kf457}`

## Solution

### 1. Look at the receipt

```text
overnight object receipt: d3143b8dfed6
The branch label was removed after the bundle was sealed.
```

Two clues:

- `d3143b8dfed6` looks like an abbreviated Git object ID.
- The branch that pointed to it was deleted, so the object is unreachable from any ref.

### 2. Inspect the bundle

A bundle is a packfile plus a list of refs. Listing the refs shows only one:

```console
$ git bundle list-heads kopitiam.bundle
9a32d4e341b83ea0becd4e9b81a02cc46368c838 refs/heads/main
9a32d4e341b83ea0becd4e9b81a02cc46368c838 HEAD
```

Cloning it and walking the history shows only two commits:

```console
$ git clone kopitiam.bundle repo && cd repo
$ git log --oneline --all --graph
* 9a32d4e publish clean status
* 4f13820 start overnight build
```

Neither commit contains an incident note, which matches the challenge text: the visible history cannot reach it.

### 3. Enumerate every object in the pack

`git log` only follows refs. To see everything stored in the pack, ask the object database directly:

```console
$ git cat-file --batch-all-objects --batch-check
...
4b4b94f5adfdefa073b50e846520eb79dd2bf5e9 commit 281
4f1382099b55c0f1f7771dada47eb590e1c6e8d5 commit 212
9a32d4e341b83ea0becd4e9b81a02cc46368c838 commit 259
d3143b8dfed6936c6f448dc8033f5b3f6cfb0ff9 commit 271
...
```

The pack holds 12 objects, including four commits, but `git log --all` only showed two. The receipt prefix `d3143b8dfed6` matches the full hash `d3143b8dfed6936c6f448dc8033f5b3f6cfb0ff9`, so it identifies the hidden commit.

### 4. Read the hidden commit

```console
$ git cat-file -p d3143b8dfed6
tree 389387a4259b2de904f70430c37d1c3555070746
parent 4b4b94f5adfdefa073b50e846520eb79dd2bf5e9
author Afterlight Builder <builder@afterlight.local> 1788108688 +0800
committer Afterlight Builder <builder@afterlight.local> 1788108688 +0800

record incident note before kopi
```

```console
$ git ls-tree -r d3143b8dfed6
100644 blob 8048232a9d164ad230a609655f2d6868277f325f	README.md
100644 blob f0a16c46dc905424c241a47b4ea32c17dccbb7b0	incident_note.txt

$ git cat-file -p f0a16c46dc905424c241a47b4ea32c17dccbb7b0
Build 7319 recovered token: CYBLTZ2026{r3fL0g_b3f0r3_br34kf457}
```

### 5. The decoy

The hidden commit's parent, `4b4b94f`, is also unreachable. It contains an earlier draft of the same file:

```console
$ git cat-file -p 4b4b94f5:incident_note.txt
Build 7319 recovered token: PENDING-SUPERVISOR-SEAL
Status: draft; do not publish this receipt.
```

This is a placeholder. The receipt points at `d3143b8dfed6`, the sealed version, not at this draft.

## Takeaways

- Deleting a branch only removes a ref. The commits stay in the object store until garbage collection prunes them, and they travel inside a bundle's packfile.
- `git log --all` shows only what refs can reach. Use `git cat-file --batch-all-objects --batch-check` (or `git fsck --lost-found`) to see everything in the object database.
- Abbreviated hashes in artifacts such as receipts and logs can be expanded with `git rev-parse --disambiguate=<prefix>`.

## Commands used

```bash
git bundle list-heads kopitiam.bundle
git clone kopitiam.bundle repo && cd repo
git log --oneline --all --graph
git cat-file --batch-all-objects --batch-check
git rev-parse --disambiguate=d3143b8dfed6
git cat-file -p d3143b8dfed6
git ls-tree -r d3143b8dfed6
git cat-file -p f0a16c46dc905424c241a47b4ea32c17dccbb7b0
```
