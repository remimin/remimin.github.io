Thread Starving Reason

# mysql io thread

1. xfs vs. ext4
原因：大磁盘格式化超时

 


# ceph cli

ceph使用librados

```commandline
import rados, rbd

cluster = rados.Rados(conffile="/etc/ceph/ceph.conf")
cluster.connect()
# 访问pool 创建ioctx
ioctx = cluster.open_ioctx("pool-name")
# librbd 访问对象
r_lib = rbd.RBD()
# 查看所有的rbd 
r_lib.list(ioctx)
```

## rbd format
rbd format 1 已经过期



## rbd Features

配置项为rbd_default_features = [3 or 61],这个值是由几个属性加起来的：

only applies to format 2 images
+1 for layering,
+2 for stripingv2,
+4 for exclusive lock,
+8 for object map
+16 for fast-diff,
+32 for deep-flatten,
+64 for journaling

layering: layering support -- COW支持
striping: striping v2 support -- 条带化
exclusive-lock: exclusive locking support
object-map: object map support (requires exclusive-lock)
fast-diff: fast diff calculations (requires object-map)
deep-flatten: snapshot flatten support
journaling: journaled IO support (requires exclusive-lock)
data-pool: erasure coded pool support

所以61=1+4+8+16+32就是layering | exclusive lock | object map |fast-diff |deep-flatten
这些属性的大合集,需要哪个不需要哪个，做个简单的加法配置好rbd_default_features就可以了。

## rbd cache

qemu cache="writeback" 对应ceph rbd cache = true

librbd client cache默认是开启的

```commandline
>>> cluster.conf_get("rbd_cache")
'true'
>>> cluster.conf_get("rbd_cache_size")
'33554432'
>>> cluster.conf_get("rbd_cache_max_dirty")
'25165824'
>>> cluster.conf_get("rbd_cache_target_dirty")
'16777216'
```

```commandline
Writeback:

rbd_cache = true

Writethrough: #写入osd，从cache读取

rbd_cache = true
rbd_cache_max_dirty = 0

None:
rbd_cache = false
```
