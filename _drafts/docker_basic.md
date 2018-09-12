# k8s pod sandbox



# Pod

## Pause Pod

```commandline
# docker inspect ef71d56e5b19 |grep d27c81ab30
        "ResolvConfPath": "/var/lib/docker/containers/d27c81ab30f6de5f19e49a34db2e885c62761007f46cf1454e854c979ee4863f/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/d27c81ab30f6de5f19e49a34db2e885c62761007f46cf1454e854c979ee4863f/hostname",
            "NetworkMode": "container:d27c81ab30f6de5f19e49a34db2e885c62761007f46cf1454e854c979ee4863f",
            "IpcMode": "container:d27c81ab30f6de5f19e49a34db2e885c62761007f46cf1454e854c979ee4863f",
                "io.kubernetes.sandbox.id": "d27c81ab30f6de5f19e49a34db2e885c62761007f46cf1454e854c979ee4863f",
```

# Storage Driver
查看docker使用的storage driver

```commandline
$ docker info |grep -i storage -A3
Storage Driver: overlay2
 Backing Filesystem: xfs
 Supports d_type: true
 Native Overlay Diff: true
```

# Security

## SELinux
SELinux是linux kernel提供的安全模块
SElinux对进程和文件通过label的方式实现访问控制，Selinux是Role-Based Access Control (RBAC), Type Enforcement (TE),
 Multi-Level Security (MLS)的组合。

- user: Selinux user不是操作系统用户，每个操作系统用户在selinux开启情况下会对应一个selinux user，是多对一的映射关系。
通常情况下以"_u"为后缀
- Roles: SElinux User可以被赋予不同的roles，以"_r"为后缀
- Types: 以"_t"为后缀
- contexts: 每个进程和对象都会有context，用于判断进程是否有权限访问对象，"user:role:type:range"
```commandline
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c255
    user    :    role    :    type    :    range    
```

## 常用selinux命令

- 查看context

```commandline
$ id -Z
$ ls -Z /bin/bash
$ ps -Z
```

- 修改context
```commandline
$ touch /tmp/myfile
$ ls -Z /tmp/myfile
unconfined_u:object_r:user_tmp_t:s0 /tmp/myfile
$ chcon -t user_home_t /tmp/myfile
$ ls -Z /tmp/myfile
unconfined_u:object_r:user_home_t:s0 /tmp/myfile
```
