# Garbage collection tree

## Summary

This is meant to be the tree where all extent freeing happens.  In the case of
more complicated metadata free'ing operations we may not actually free anything.
For data this will allow us to spread the pain of deleting large extents and
large chunks of checksums across multiple transactions.

As this tree is indexed on bytenr it will be part of the global roots ID scheme
so we can spread the lock contention pain out.

## On disk changes

At first there will be 4 different key types

```
struct btrfs_key {
	.objectid = bytenr,
	.type = BTRFS_QGROUP_UPDATE_ITEM_KEY,
	.offset = owner,
};

struct btrfs_key {
	.objectid = bytenr,
	.type = BTRFS_FREE_METADATA_ITEM_KEY,
	.offset = owner,
};

struct btrfs_key {
	.objectid = bytenr,
	.type = BTRFS_FREE_EXTENT_ITEM_KEY,
	.offset = num_bytes,
};

struct btrfs_key {
	.objectid = block group objectid,
	.type = BTRFS_FREE_BLOCK_GROUP_ITEM_KEY,
	.offset = 0,
};
```

For the `BTRFS_QGROUP_UPDATE_ITEM_KEY` this will be set if qgroups are enabled
and if our `drop_count` == the minimum references - 1.  This will go and do the
conversion from shared to exclusive for the remaining root.

For the `BTRFS_FREE_METADATA_ITEM_KEY`, if the `drop_count` doesn't == the
maximum reference count we will go look up any chained snapshots to see if they
refer to the block anymore.  If they do then we will delete our item and move
on, otherwise we'll free the extent and delete the drop item.

For the `BTRFS_FREE_EXTENT_ITEM_KEY` we will delete the checksums, delete the
extent item, and then add the free space to the free space tree.

For `BTRFS_FREE_BLOCK_GROUP_ITEM_KEY` we will delete the remap objects in the
remap tree for the given block group, and then delete the block group item
itself.
