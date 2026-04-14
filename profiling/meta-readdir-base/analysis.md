# meta-readdir-base: yolofs 6.5x slower than overlayfs

yolofs-no-perm: 35,696 µs/op
overlayfs:     5,457 µs/op

## Root cause

`yolofs_readdir` (kmod/file.c:456) resets `lower_file->f_pos = 0` before
every `iterate_dir` call. With 10k entries and a ~32KB getdents buffer,
userspace makes ~88 getdents64 syscalls to drain the directory. Each
syscall re-enters `yolofs_readdir`, which re-reads the entire ext4
directory from the start — ext4 rebuilds its htree rb-tree each time —
then skips already-emitted entries via the `off < ctx->pos` check in
`yolofs_fill_base`. This is O(n²) in ext4 htree work.

Overlayfs calls `iterate_dir` on the lower file directly, letting ext4
resume from `f_pos`. That's O(n).

## Evidence from breakdown

ext4 internals account for nearly all the gap. `yolofs_find_dirent` (the
per-entry dedup check against the empty dirent table) is only 389 µs —
negligible. The cost is ext4 repeating work:

    ext4_htree_fill_tree    yolofs: 21,897 µs   ovl: 2,628 µs   (8.3x)
    ext4fs_dirhash          yolofs:  7,974 µs   ovl:   947 µs   (8.4x)
    ext4_htree_store_dirent yolofs:  8,976 µs   ovl: 1,095 µs   (8.2x)
    free_rb_tree_fname      yolofs:  4,806 µs   ovl:   906 µs   (5.3x)
    call_filldir            yolofs:  5,011 µs   ovl: 1,017 µs   (4.9x)

## Fix

For directories with no dirents (`!YOLOFS_I(dir)->de_buckets`), skip the
two-phase merge and fall through to the passthrough path, same as when
staging is off. This avoids the f_pos reset entirely.

For directories with dirents (the merge path), track a persistent
`base_pos` per open file instead of resetting to 0, so ext4 can resume
where it left off.
