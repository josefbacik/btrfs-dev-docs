# Multiple global root sets

## Summary

"Global roots" refer to roots that we use for the entire file system.  This
consists of the extent trees, free space trees, drop trees, garbage collection
trees, and checksum trees.  These trees hit lock contention from multiple
threads updating `bytenr` offsets in these trees.  In order to reduce the lock
contention we'll create multiple sets of these trees and spread the load across
all of them.

## On-disk changes

The `btrfs_root_item` key will now look like this

```
struct btrfs_key {
	.objectid = <global root id, ie BTRFS_EXTENT_TREE_OBJECTID>,
	.type = BTRFS_ROOT_ITEM_KEY,
	.offset = global_root_id,
};
```

For example, if `nr_cpus` is 4, we will have `global_root_id` 0 through 3, and
we will have all of these roots with those root id's.  To use them, we will set
each block group to their own set of global root id's.  Currently the
`btrfs_block_group_item` sets the `chunk_objectid` to
`BTRFS_CHUNK_TREE_OBJECTID` unconditionally, so we will use this to specify the
global root id.  Then if we modify the checksum for a bytenr in that block
group, we will use the csum tree that corresponds with that block groups global
root id.

The global root id can be assigned in any way we feel like, but initially will
simply be round robin.
