# Drop and snapshot trees

## Summary

Instead of doing traditional reference counting for shareable trees, we will
instead track whether not individual blocks are shared via the snapshot tree and
the drop trees.

Upon snapshot we will insert an item to map the origin and snapshot
`objectid`'s, as well as their `generation` and `snapshot id`.  The root item of
the origin will be updated with the last snapshot `generation` and `snapshot id`
and this will be used to determine if a block is shared.

When free'ing a shared block we will simply increment that blocks drop count in
the drop tree.  Once the drop count == the theoretical minimum reference count
for that block we will insert a free object into the garbage collection tree to
do the work to see if the block can be freed.

Data will work the same way it current does, if we COW down from the owning root
then it will unconditionally use the `FULL_BACKREF` logic on the leaf and add
extent references for the bytenr of the leaf.  When the leaf is free'd via the
GC tree we will drop the extent references for that bytenr.

## `snapshot_id`

The `btrfs_header` will now include a `snapshot_id`, which will be taken from
the owning root.  Additionally, the `btrfs_file_extent_item` will also grow a
`snapshot_id`.  On every snapshot, the `snapshot_id` will be incremented.  This
will allow us to easily determine the possible references of each block and file
extent.

This also will allow us to implement transaction commit-less snapshotting.
Because each block will be updated with their own `snapshot_id` we can easily
track modifications within the same transaction to a root, and thus allow us to
decouple the work of snapshot creation from the transaction commit.

## Snapshot tree

On snapshot we will no longer push references for metadata, as metadata will not
be reference counted.  Instead a new item will be inserted into the snapshot
tree at snapshot creation time

```
struct btrfs_key {
	.objectid = target_subvol_id,
	.type = BTRFS_SNAPSHOT_ITEM_KEY,
	.offset = snapshot_id,
};

struct btrfs_snapshot_item {
	/* The generation when the snapshot was taken. */
	u64 generation;

	/* The snapshot_id of the source root when the snapshot was taken */
	u64 snapshot_id;

	/*
	 * The number of metadata bytes in the source subvolume at snapshot
	 * time, this will be used to determine when we can delete this item.
	 */
	u64 shared_bytes;
};

struct btrfs_key {
	.objectid = snapshot_id,
	.type = BTRFS_SNAPSHOT_REF_ITEM_KEY,
	.offset = target_subvol_id,
};
```

This will allow us to find who references any block based on it's
`btrfs_header_owner`, `generation`, and `snapshot_id`.  `should_cow_block` will
be changed to be essentially

```
if (btrfs_header_generation(buf) < trans->transid)
	return 1;
if (btrfs_header_snapshot_id(buf) <= root->last_snapshot_id)
	return 1;
```

We will no longer need to rely on `BTRFS_ROOT_FORCE_COW` as any modification
that occurs after the snapshot operation will require a COW.  `snapshot_id` for
new blocks will be set to `root->last_snapshot_id + 1`.

The following is an example snapshot graph

             gen  1
    ┌───┐    snap 1   ┌───┐
    │ A │◄────────────┤ B │
    └─▲─┘             └───┘
      │
      │     gen  10           gen  10
      │     snap 2    ┌───┐   snap  1  ┌───┐
      └───────────────┤ C │◄───────────┤ D │
                      └───┘            └───┘

If a block owned by A had a `generation` of 1 and a `snapshot_id` of 1 then we
know that B, C, and D potentially have references to that block.  D is
questionable, because we could have COW'ed that block in C before we took the
snapshot of D.  In order to determine if it points at our block we must look it
up and check.  This will be handled by the garbage collection tree.

The `BTRFS_SNAPSHOT_REF_ITEM_KEY` will simply allow us to find the original
subvolume a snapshot was taken of, this will be used for snapshot deletion.

## Drop trees

The drop tree is what will be used to determine if a block can potentially be
freed.  At `update_ref_for_cow` time we will determine if the block is shared.
If the block is shared we will modify the drop count for the block.  Drop items
will look like this

```
struct btrfs_key {
	.objectid = bytenr,
	.type = BTRFS_DROP_ITEM_KEY,
	.offset = owner,
};

struct btrfs_drop_item {
	/* The number of trees that no longer reference this block. */
	u64 drop_count;

	/* The first key in this node/leaf. */
	struct btrfs_disk_key first_key;
};
```

At every `update_ref_for_cow` on a shared block we will increase the
`drop_count` for the block we are COW'ing.

When we update the `drop_count` we will calculate the minimum reference count
for the block, and if our `drop_count` is equal to the minimum reference count
of the block then we insert an item into our garbage collection root to process
the free'ing of the block.  The garbage collection behavior is documented in the
[garbage collection trees](garbage-collection-tree.md) document.

## Minimum and maximum reference count calculations

As indicated above, we use the minimum reference count for our block based on
the dependency graph from the snapshot tree.  Take the following tree as our
example file system

       ┌───┐
       │ B ├┬───────────────┐
       ├───┴┤               │
       │    │ ┌───┐         │       ┌───┐
       │    │ │ A │         │       │ C │
      ┌┼────┼─┼───┴──┐      ├┬─────┬┴───┴─┐
      ││    │ │      │      ││     │      │
      ││    │ │      │      ││     │      │
    ┌─▼▼┐  ┌▼─▼┐  ┌──▼┐   ┌─▼▼┐  ┌─▼─┐  ┌─▼─┐
    │ 1 │  │ 2 │  │ 3 │   │ 4 │  │ 5 │  │ 6 │
    └───┘  └───┘  └───┘   └───┘  └───┘  └───┘

A - original subvolume
B - snapshot of A at generation 10, snapshot 1
C - snapshot of B at generation 20, snapshot 1

For this example we'll assume block 2 has a generation and snapshot id of 1.
We use the directed graph to know that our maximum reference count is 3, because
the generation is < 10 and < 20.  However, for this case, the minimum reference
count is 2.  This is because C is a snapshot of B, and so we can't know for
certain if 2 was every referenced by C without checking.

Block 3 is an example of this case.  Let's assume that block 3 also has a
generation and snapshot id of 1.  This gives it the same theoretical maximum
reference count of 3.  However at generation 20 we COW'ed that block from B, and
now it has a `drop_count` of 1.  Let's say that 4 is the COW of 3, and has a
generation of 20 and a snapshot id of 1.  At this point the snapshot C is taken.
By our directed graph we know that C could possibly reference 3, but we know it
doesn't in fact reference it.  This is what we use as the minimum reference
count.

First degree snapshots, that is direct snapshots of a subvolume, are guaranteed
reference counts based on their generation+snapshot id pair.  However chained
snapshots, that is a snapshot of a snapshot, may not actually refer to blocks in
the original subvolume.

This is the reason we require the garbage collection roots, as determining who
still refers to the block in this case is complicated.

The normal case where minimum reference count == maximum reference count means
we could bypass the garbage collection tree if we desired, because we know that
once our `drop_count` == our reference count that the block can be freed.  The
GC tree is only strictly necessary for the case where minimum reference count !=
maximum reference count.

## What about data?

Data is more complicated, as we can have things like reflink and bookend extents
create a special kind of chaos that is hard to reason out cleanly with the
snapshot tree and drop trees.

Instead we will keep the rules for data relatively the same in practice, with
just a few slight tweaks.

1. `update_ref_for_cow` will continue to call `btrfs_inc_ref` on shared leaf
   nodes.  This is to make sure that new roots get their appropriate references
   propagated.
2. `update_ref_for_cow` will call `btrfs_inc_ref` with `full_backref` set
   unconditionally when we COW from the owner root.
3. The `FULL_BACKREF` reference will be dropped at garbage collection time for
   that particular metadata block.

This will keep the data reference counting relatively unchanged from the extent
tree v1 setup.

We will be adding a `snapshot_id` to the `btrfs_file_extent_item` in order to
facilitate the mechanics of finding referencing roots, but it won't be used for
anything beyond that.

## Snapshot deletion

This is a con of the snapshot tree and drop trees design, we will have to walk
the entire tree to clean up all blocks we reference.  For unshared metadata
blocks they will simply be freed, for shared metadata blocks they will have
their drop count updated.  Data extents will have their references dropped if we
have a root reference for them, otherwise they'll remain untouched.

## Mechanics of the code and changes from v1

- `update_ref_for_cow` - if the block is shared, update the drop count for the
  block.  We no longer call `btrfs_inc_ref` for nodes, only for leaves.  We also
  no longer do the `FULL_BACKREF` conversion for nodes, we only add a
  `FULL_BACKREF` reference for any file extents in leaves.
- `update_ref_for_cow` will also update the `shared_bytes` counter on the
  snapshot item if we are not the owner, this will allow us to garbage collect
  the snapshot items themselves.
- Updating the `drop_count` on a block will calculate the min and max reference
  counts at the `drop_count` time, and if our `drop_count` matches the minimum
  reference count then we can insert a garbage collection item.
