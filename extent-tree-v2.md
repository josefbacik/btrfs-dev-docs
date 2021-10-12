# Extent tree v2 design document

## Problems that we are solving

The original design of Btrfs has had some drawbacks that have become more and
more of a pain to deal with.  Thus this re-design of a few of the core aspects
of Btrfs in order to address some of these problems.  The things we are trying
to solve are

1. Lock contention on the global roots (extent root, checksum root).
2. Recursive updates of the extent root in order to track all metadata
   allocations.
3. Delayed ref explosion when updating reference counts of shared metadata
   blocks.
4. Long running operations at transaction commit time.

In fixing these relatively large problems, the design to solve these issues
means that we also have to drastically change a few of the core features of the
file system in order for them to continue to work, so as a side-effect the
design requires us to

1. Completely change relocation, so it will be a more straightforward operation
   that is less complicated and error prone.
2. Allow us to easily determine what roots refer to which blocks without the
   need for backref lookups.

The following sections will describe how specifically we mean to address these
problems, with links to the design documents for each individual solution.

## Lock contention on the global roots

The solution for this is straightfoward, [multiple sets of global
roots](global-roots.md).  This is N copies of the csum, extent, drop, and free
space trees.  The N will be user configurable, but will simply default to
`nr_cpus` if not specified.  Each block group will be assigned a specific
`global_root_id` and the extents associated with that block group will go into
that set of global roots.  This will allow us to spread the lock contention on
the root node of the global roots across the file system better.

## Recursive updates of the extent root

To solve this we will no longer be tracking metadata in the extent tree's.  The
extent trees will only track data reference counts.  This is relatively simply
for `cowonly` (non-reference counted) trees, as we simply reclaim the space on
free as their blocks cannot be shared.

Reference-counted trees will be more complicated.  Free'ing those blocks will be
handled via [drop trees](drop-and-snapshot-trees.md).

## Delayed ref explosions

At `update_ref_for_cow` time we must push reference counts for every child a
shared block refers to in order to maintain the correct reference counts.  In
the worst case, this can be around 500 delayed refs per level, excluding the
root level and the leaf level.  These refs must be run before the transaction
commit occurs, which is a source of latency when modifying shared blocks.

To solve this problem we're introducing [drop trees and a snapshot
tree](drop-and-snapshot-trees.md) in order to change how we track the free'ing
of shared metadata blocks.

## Long running operations at transaction commit time

Delayed ref explosion also directly ties into transaction commit latency.  Any
delayed refs that are generated during the transaction must be run before we can
commit the transaction.  This is done in order to make sure the reference
counting is consistent for every transaction commit.

Instead of doing this however, we're going to move to a [garbage collection
tree](garbage-collection-tree.md) in order to handle the free'ing of shared
metadata extents and free'ing of data extents.

This will allow us to simply process the more complicated free operations at
some arbitrary point in the future rather than tying it to the transaction that
the free occurred in.

## Relocation rework

The current relocation requires all metadata to be tracked by the extent tree,
this is how it finds the space that needs to be relocated.  The actual mechanics
of the relocation consist of moving a relocated block, and then modifying all
paths that point to that block to point at the new block.  This operation has to
be done all in one shot, so if you have 1000 snapshots pointing at a relocated
block, we have to update all 1000 paths to this block in that transaction.  This
further explodes out the delayed ref operations and can cause transaction
commits to take many minutes to complete.

Because we won't be reference counting metadata any more this makes the old
style relocation problematic, and so we're taking this opportunity to fix this
problem at the same time, partly because we have to anyway.

This will be done with a new [remap tree](remap-tree.md), which will simply copy
used extent ranges to new block groups, and insert their new location into the
remap tree.  Block groups that are remapped will be flagged and any reads to
their logical addresses will be remapped to the new logical address.  This will
give us a lot more flexibility on how we can relocate space, and will avoid this
delayed refs problem because we won't be modifying all snapshots to point at the
new location, all the metadata will use the original logical address and we will
simply translate to the new location.

In order to facilitate this rework we will need to separate out the
`btrfs_block_group_item`s into their own special block group root.  This is also
discussed in the remap tree document.

## Easier qgroup accounting

Currently the way we handle qgroup accounting for metadata is to lookup all of
the roots that pointed at a block for the previous commit (T-1) and then look
them up for the current commit (T) and then update the counters for the roots
that lost references to that block.  These lookups are very expensive, and have
to be done at transaction commit time, which introduces more latency at
transaction commit if you have qgroups enabled.

Instead we can use the snapshot tree and drop trees to determine who references
that block without doing backref lookups.  We can also delay the more
complicated operations (freeing from N->1 references, which converts the
accounting from simply referenced to exclusive) to other transactions via the
garbage collection tree.

## List of on disk changes with their respective design docs

- New snapshot ID in both the `btrfs_header` and the `btrfs_file_extent_item` to
  help with snapshot tracking.  [Described here](drop-and-snapshot-trees.md).
- New [snapshot and drop trees](drop-and-snapshot-trees.md) to track snapshot
  relationships and block counts.
- New [garbage collection trees](garbage-collection-tree.md) to offload the more
  complicated free work that could induce latency at transaction commit time.
- New [remap tree](remap-tree.md) to remap logical blocks to new locations
  without needing to update any of the referring metadata.
- [Multiple global roots](global-roots.md) to cut down on fs wide lock
  contention.
- New [mapping block group type](remap-tree.md) to store the block group remap
  tree as it requires special handling.
- New [block group tree](remap-tree.md) to facilitate the new block group remap
  scheme, as well as improve mount times for very large file systems.
