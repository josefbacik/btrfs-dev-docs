# Remap tree and block group tree

## Summary

This tree will simply provide a translation layer for moving logical bytenrs
around the file system.  This is meant to replace the current relocation scheme
with something a little simpler to deal with.  It does bring with it some
complexities however

## `BTRFS_BLOCK_GROUP_MAPPING`

We stuff `BTRFS_BLOCK_GROUP_SYSTEM` mappings into the super block, because we
must be able to read the chunk root in order to get the logical->physical
mappings loaded so we can read all the rest of the metadata.  However because we
stuff these mappings into the super block we are limited to the number of SYSTEM
chunks we can have.  This doesn't matter in practice because it is just the
chunk mappings.

However the remap tree will be remapping all logical bytes, which will be much
more fine grained than the chunk map.  We have the same requirement however, we
must be able to load the remap tree before we read any metadata to make sure our
logical->logical mappings are in place.

We cannot remap any blocks in the remap root, as that would mean we would be
unable to read the remap tree without knowing where parts of it were remapped.
Enter `BTRFS_BLOCK_GROUP_MAPPING`, this block group will hold the remap root.
As such if we need to relocate a `BTRFS_BLOCK_GROUP_MAPPING` block group we'll
simply COW any blocks that fall inside that block group to relocate them.

We also need to be able to tell if a block group has been remapped, so we will
also have a new block group root that will hold all of the block group items,
and it will be stored in this block group type as well.

And finally because we need to be able to get to these roots without worrying
about remapped metadata blocks their location will be stored in the super block.

At mount time the chunk root will be read and cached, and then the block group
root, and then the remap root.  After that the rest of the file system will be
read as normal.

## Block group and remap trees

The block group root will have the same on disk scheme as the currently
`btrfs_block_group_item`s.  The remap tree will have the following keys and
items.

```
struct btrfs_key {
	.objectid = bytenr,
	.type = BTRFS_REMAP_ITEM_KEY,
	.offset = num_bytes,
};

struct btrfs_key {
	.objectid = new_bytenr,
	.type = BTRFS_REVERSE_REMAP_ITEM_KEY,
	.offset = num_bytes,
};

struct btrfs_remap_item {
	/* The new or old location. */
	u64 bytenr;
}
```

The `BTRFS_REVERSE_REMAP_ITEM_KEY` is used if we are relocating a block group
that has relocated extents inside of it.  We don't want to do multiple levels of
indirection, so we will use these items to first remap any previously remapped
extents into new block groups, and then process the remaining extents that need
to be remapped.

Once the remapped block group reaches a used count of 0 all of its corresponding
remap items can be removed from the remap tree.  This will occur at GC time via
the garbage collection tree.

## Relocation mechanics

The current relocation scheme revolves around creating special snapshots of
every root in the file system, searching down these roots for bytenrs that land
in the block group we are relocating, moving these blocks, and then updating all
roots that point to that block to point at the new block.

For data it is a two step process, first allocating extents that match the exact
length of the original extent, copying it into the new location, and then
updating the metadata to point at the new locations and merging it with the
original file system trees.

This creates a lot of churn for the file system, and can break snapshot sharing
in a lot of cases.  The data sharing is maintained, but the metadata sharing can
be broken in certain cases.

With remapping we will no longer mess with the trees themselves.  We will simply
find all allocated space within a block group, allocate a new location for each
extent and copy it into the new extent, and then insert a remap item for that
range in the block group.

Once the block group is completely remapped we will release the underlying
chunks for the block group, but leave the block group item itself with the flag
`BTRFS_BLOCK_GROUP_REMAPPED` set on the block group.

From this point on any reads that occur in this block group range will be
redirected to their new location on disk, and then submitted to the normal
mapping logic.

This will allow us to maintain complete sharing for the shared blocks, and avoid
creating special snapshots and the overhead that snapshots provide.
