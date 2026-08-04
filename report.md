# Lab 4: B+ Tree Index - Report

## 1. Design decisions

**Descent and duplicate keys.**
`findLeafPage()` walks the internal nodes and takes the left child of the first entry whose key satisfies `f <= key`.
The comparison is deliberately non-strict.
Using a strict `<` would send a search for a key sitting exactly on a boundary into the right-hand child, but duplicates of that key may still live in the left leaf, so the tuple would be missed.
The non-strict test returns the left-most leaf that can contain the key, which is what the lab requires and what `BTreeSearchIterator` assumes: it starts at that leaf and scans right until it sees a key strictly greater than the target.
A null key degenerates to the same branch and always descends the left-most child, giving the left-most leaf for a full scan.

**Split ratio.**
Both split methods move `n/2` items from the back of the page into a fresh right-hand page, obtained from `getEmptyPage()` so that pages freed by merges are reused.
Leaves copy the first key of the new right page up into the parent; internal nodes push the middle entry up, which is deleted from the left page with `deleteKeyAndRightChild()` so that its right child becomes the left-most child pointer of the new page.
Neither method inspects the incoming key while splitting.
The key is used only at the end, to decide which of the two halves to return, per the hint in the lab text.

**Redistribution as key rotation.**
The three steal methods move `(sibling - page)/2` items so that the two pages end up balanced.
For internal pages the move is a rotation rather than a copy: the parent's key comes *down* to become the new boundary entry in the receiving page, and the donor's outermost key goes *up* to replace it in the parent.
`stealFromRightInternalPage()` is the exact mirror of `stealFromLeftInternalPage()`, using the forward iterator and `deleteKeyAndLeftChild()` where the other uses the reverse iterator and `deleteKeyAndRightChild()`.
Writing them as mirrors rather than as one parameterised method kept each one readable, since the child pointer that has to be spliced differs on each side.

**Merging.**
`mergeLeafPages()` and `mergeInternalPages()` are the inverses of the splits.
The internal case has to reinstate the key that the split pushed up: before moving any entries, the parent's key is pulled down as a new entry bridging the left page's last child pointer and the right page's first, which is precisely the entry that `splitInternalPage()` removed.
Both methods then hand the emptied page back with `setEmptyPage()` and let `deleteParentEntry()` handle the recursive cases, including collapsing the root.

## 2. Non-trivial parts

The subtle part of the lab is not the tuple movement, it is the pointer maintenance around it.

Splitting a leaf touches four pages, not two: the original, the new page, the parent, and the original's former right neighbour, whose left-sibling pointer has to be redirected at the new page.
Merging has the same shape in reverse.
Missing that fourth page leaves a sibling chain that still looks locally consistent from one side, which is why `BTreeChecker` tests it from both.

For internal nodes, moving an entry also moves a subtree, so every child whose pointer changed pages needs its parent pointer rewritten.
`updateParentPointers()` is called after `splitInternalPage()`, both internal steals, and `mergeInternalPages()`.
It is not needed after a leaf operation because leaves have no children.

One consequence of `getParentWithEmptySlots()` is worth noting: it can split the parent, which may relocate the pointer to the page being split into the parent's new right half.
Both split methods therefore reassign `setParentId()` on both halves afterwards rather than assuming the parent is unchanged.

## 3. Changes to the API

None.
All work is inside the eight method bodies the lab identifies, plus the transaction-layer fix described below.
No signatures were altered and no helpers were added to `BTreeFile`.

## 4. A transaction-layer bug surfaced by this lab

`BTreeTest` failed while every other test passed.
Rather than guess, we ran the same workload twice through identical code paths, once with the inserter and deleter threads started concurrently and once with them run serially.
Serial was clean: `checkRep()` passed and all 41,000 tuples were present and reachable.
Concurrent broke: `checkRep()` failed on the sibling-pointer assertion, about 5,800 tuples were physically present in the file but unreachable through an index lookup, and about 200 were lost outright.
That isolated the fault to concurrency control rather than to the tree code.

The cause was in `BufferPool.transactionComplete()`.
On abort it discarded only pages whose dirty flag named the aborting transaction.
But pages are marked dirty only *after* `insertTuple()` / `deleteTuple()` returns, in a loop at the end.
A B+ tree split or merge modifies many pages as it goes and, under contention, frequently aborts part-way through, when a lock wait throws `TransactionAbortedException` from inside `splitLeafPage()`.
The marking loop never runs.
Half-applied structural edits, such as a leaf whose right-sibling pointer was updated while its neighbour's back-pointer was not, were therefore left in the buffer pool still flagged clean, survived the abort, and were later read and flushed by another transaction.

The fix is to discard every page the aborting transaction held a lock on, not only those already marked dirty.
Re-reading a page that was merely read is harmless; leaving a partially edited one is not.
This required one new accessor, `LockManager.pagesHeldBy()`.

A second, independent defect was fixed alongside it: `insertTuple()` and `deleteTuple()` mutated the `pages` map outside the monitor that every other accessor synchronises on, a data race on a non-thread-safe `HashMap`.

An earlier attempt marked pages dirty at write-lock acquisition time instead.
It made `BTreeTest` pass but broke three `BTreeFileDeleteTest` cases with "all pages are dirty; cannot evict", because `updateParentPointers()` legitimately touches more pages than the 50-page pool holds.
That change suppressed concurrency rather than fixing the defect, so it was reverted in favour of the abort-discard fix, which costs nothing in eviction pressure.

## 5. Missing or incomplete elements

Nothing in lab 4 is incomplete.
All exercises are implemented and all tests named in the lab text pass:
`BTreeFileReadTest` (7/7) and `BTreeScanTest` (4/4);
`BTreeFileInsertTest` unit (3/3) and system (5/5);
`BTreeFileDeleteTest` unit (7/7) and system (6/6);
`BTreeNextKeyLockingTest` (2/2), `BTreeDeadlockTest` (1/1) and `BTreeTest` (1/1).

Failures that remain in the wider suite all predate this lab and were confirmed against an unmodified checkout.
`IntHistogram` was unimplemented and has been filled in, so `IntHistogramTest` now passes (8/8).
Still outstanding, and belonging to other labs rather than to this one:

* `TableStatsTest` (3/3) and `JoinOptimizerTest` (5/5). `TableStats` and `JoinOptimizer` are still stubs; these are lab 3 query-optimizer exercises.
* `LogTest` (8 errors), which covers log-based recovery from lab 6.

One structural caveat is inherent rather than a defect.
When an internal page holding an even number of entries splits, the entries cannot divide evenly, because one is pushed up to the parent and `left + right = n - 1` is odd.
One side ends up an entry lighter than the other.
Biasing the split the other way only moves the imbalance, it does not remove it.
At realistic page sizes the margin is far from the half-full threshold, so the invariant is never at risk in practice.
