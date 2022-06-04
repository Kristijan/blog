---
title: "Manually fixing dirty JFS2 filesystems"
date: "2012-06-19"
categories: 
  - "aix"
tags: 
  - "aix"
  - "filesystem"
  - "fsdb"
  - "jfs2"
---

I think at some point during a systems administrators life span, they see the below error message when trying to mount a filesystem.

```console
# mount /test
Replaying log for /dev/fslv01.
mount: /dev/fslv01 on /test: Unformatted or incompatible media
The superblock on /dev/fslv01 is dirty.  Run a full fsck to fix.
```

Great! Let's run the `fsck` command, cross our fingers, and hope that the superblock is repaired and we can mount the filesystem. However, say the dirty filesystem is 3 TB in size? Depending on the extent of the damage, running an `fsck` on a 3 TB filesystem can take quite some time (we're talking hours here!). Let's also paint a perfect picture and say that we're willing to let `fsck` run its course no matter how long it takes (This is the preffered method). What if though, the `fsck` fails, and we still can't mount the filesystem?

We have two options here:

1. Restore the filesystem from backup
2. Use the `fsdb` command to edit the superblock and mark the filesystem as clean.

Taken from the `fsdb` man page:

> "The fsdb command enables you to examine, alter, and debug a file system, specified by the FileSystem parameter. The command provides access to file system objects, such as blocks, i-nodes, or directories. You can use the fsdb command to examine and patch damaged file systems. Key components of a file system can be referenced symbolically. This feature simplifies the procedures for correcting control-block entries and for descending the file system tree."

At this stage, I'd like to point out that manually editing filesystem objects is dangerous. I take no responsibility for any downtime, loss of data, loss of limbs, and abuse from management. It's imperitive that you have a backup of the filesystem data, which in a worst case scenario, you can restore from.

The below is done on a test 10 GB JFS2 filesystem. After the example I explain a little more on exactly what we're modifying.

{% highlight console linenos %}
# fsdb /test
 
File System:                    /test
 
File System Size:               20970472        (512 byte blocks)
Aggregate Block Size:           4096
Allocation Group Size:          32768   (aggregate blocks)
 
> su
[1] s_magic:            'J2FS'          [18] s_fscklog:         1
[2] s_version:          2               [19] s_fsckloglen:      50
[3] s_size:     0x00000000013ffbe8      [20] s_bsize:           4096
[4] s_logdev:   0x8000000a00000003      [21] s_logserial:       0x0000000a
[5] s_l2bsize:          12              [22] s_logpxd.len:      0
[6] s_l2bfactor:        3               [23] s_logpxd.addr1:    0x00
[7] s_pbsize:           512             [24] s_logpxd.addr2:    0x00000000
[8] s_l2pbsize:         9                    s_logpxd.address:  0
[9] s_rsv:              Not Displayed   [25] s_fsckpxd.len:     131
[10] s_agsize:          0x00008000      [26] s_fsckpxd.addr1:   0x00
[11] s_flag:            0x00000100      [27] s_fsckpxd.addr2:   0x0027ff7d
                                             s_fsckpxd.address: 2621309
                                        [28] s_ait.len:         4
  J2_GROUPCOMMIT                        [29] s_ait.addr1:       0x00
                                        [30] s_ait.addr2:       0x0000000b
                                             s_ait.address:     11
[12] s_state:           0x00000002      [31] s_fpack:           'fslv01'
        FM_DIRTY                        [32] s_fname:           ''
[13] s_time.tj_sec: 0x000000004fdfdce1  [33] s_time.tj_nsec:    0x00000000
[14] s_ait2.len:        4               [34] s_xfsckpxd.len:    0
[15] s_ait2.addr1:      0x00            [35] s_xfsckpxd.addr1:  0x00
[16] s_ait2.addr2:      0x00000155      [36] s_xfsckpxd.addr2:  0x00000000
     s_ait2.address:    341                 s_xfsckpxd.address: 0
[17] s_xsize: 0x0000000000000000        [37] s_xlogpxd.len:     0
[40] feature_compat: 0x0000000000000001 [38] s_xlogpxd.addr1:   0x00
[41] feature_rdonly: 0x0000000000000000 [39] s_xlogpxd.addr2:   0x00000000
[42] feature_incompat: 0x0000000000000000    s_xlogpxd.address: 0
[43-49] <...snapshot info...>           [50] s_maxext:  0x00000000
display_super: [m]odify, [s]napshot info or e[x]it: m
Please enter: field-number value > 12 0x0
[1] s_magic:            'J2FS'          [18] s_fscklog:         1
[2] s_version:          2               [19] s_fsckloglen:      50
[3] s_size:     0x00000000013ffbe8      [20] s_bsize:           4096
[4] s_logdev:   0x8000000a00000003      [21] s_logserial:       0x0000000a
[5] s_l2bsize:          12              [22] s_logpxd.len:      0
[6] s_l2bfactor:        3               [23] s_logpxd.addr1:    0x00
[7] s_pbsize:           512             [24] s_logpxd.addr2:    0x00000000
[8] s_l2pbsize:         9                    s_logpxd.address:  0
[9] s_rsv:              Not Displayed   [25] s_fsckpxd.len:     131
[10] s_agsize:          0x00008000      [26] s_fsckpxd.addr1:   0x00
[11] s_flag:            0x00000100      [27] s_fsckpxd.addr2:   0x0027ff7d
                                             s_fsckpxd.address: 2621309
                                        [28] s_ait.len:         4
  J2_GROUPCOMMIT                        [29] s_ait.addr1:       0x00
                                        [30] s_ait.addr2:       0x0000000b
                                             s_ait.address:     11
[12] s_state:           0x00000000      [31] s_fpack:           'fslv01'
        FM_CLEAN                        [32] s_fname:           ''
[13] s_time.tj_sec: 0x000000004fdfdce1  [33] s_time.tj_nsec:    0x00000000
[14] s_ait2.len:        4               [34] s_xfsckpxd.len:    0
[15] s_ait2.addr1:      0x00            [35] s_xfsckpxd.addr1:  0x00
[16] s_ait2.addr2:      0x00000155      [36] s_xfsckpxd.addr2:  0x00000000
     s_ait2.address:    341                 s_xfsckpxd.address: 0
[17] s_xsize: 0x0000000000000000        [37] s_xlogpxd.len:     0
[40] feature_compat: 0x0000000000000001 [38] s_xlogpxd.addr1:   0x00
[41] feature_rdonly: 0x0000000000000000 [39] s_xlogpxd.addr2:   0x00000000
[42] feature_incompat: 0x0000000000000000    s_xlogpxd.address: 0
[43-49] <...snapshot info...>           [50] s_maxext:  0x00000000
display_super: [m]odify, [s]napshot info or e[x]it: x
> q
{% endhighlight %}

```console
# mount /test
# df -g /test
Filesystem    GB blocks      Free %Used    Iused %Iused Mounted on
/dev/fslv01       10.00     10.00    1%        4     1% /test
```

As you can see from the example, we're now able to mount the `/test` filesystem. We did this by telling the filesystem that it was "clean". A breakdown of the important lines follows.

| Line | Description                                                                                |
| ---- | ------------------------------------------------------------------------------------------ |
| 1    | Invokes fsdb on the `/test` filesystem                                                     |
| 9    | Shows the superblock                                                                       |
| 26   | Show field number `[12]` `s_state` is marked as `FM_DIRTY`, represented by the value `0x2` |
| 38   | Puts fsdb into modify mode                                                                 |
| 39   | Changes field number `[12]` `s_state` to `FM_CLEAN` by changing the value to `0x0`         |

In all cases, you'd want to run `fsck` to fix this issue. If `fsck` fails, restoring from a backup will at least ensure data integrity. I only show this as an alternative method for those that have run into a roadblock, don't have a backup, and need to salvage as much data on the filesystem as possible. Once again, use this at your own risk!
