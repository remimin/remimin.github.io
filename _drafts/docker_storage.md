
# Overlay

Linux hard link
linux中的硬链接可以看作是文件的别名，而且硬链接只能作用于文件，使用`ls -i`查看硬链接文件
及链接文件，可以发现inode值是一样的，也就是说指向的同一个文件。对硬链接做任何操作都与对文件
本身操作时相同的。

# devicemapper

devicemapper 

```commandline
# dmsetup targets
thin-pool        v1.19.0
thin             v1.19.0
mirror           v1.13.2
striped          v1.6.0
linear           v1.3.0
error            v1.5.0
```

```text
dmsetup ls
docker-253:0-134388667-pool     (253:2)
docker-253:0-134388667-0c2b68f5dcda6a3a928650c0ad6f75d3c144744dc3348ce4bb6df3017b4cf1af (253:3)
system-swap     (253:1)
system-root     (253:0)
```

```text
Storage Driver: devicemapper
 Pool Name: docker-253:0-134388667-pool
 Pool Blocksize: 65.54 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: xfs
 Data file: /dev/loop0
 Metadata file: /dev/loop1
 Data Space Used: 6.203 GB
 Data Space Total: 107.4 GB
 Data Space Available: 101.2 GB
 Metadata Space Used: 6.504 MB
 Metadata Space Total: 2.147 GB
 Metadata Space Available: 2.141 GB
 Thin Pool Minimum Free Space: 10.74 GB
 Udev Sync Supported: true
 Deferred Removal Enabled: true
 Deferred Deletion Enabled: true
 Deferred Deleted Device Count: 0
```

参考：
[IBM DW linux device mapper](https://www.ibm.com/developerworks/cn/linux/l-devmapper/index.html)

