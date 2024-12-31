---
layout: default
title:  "以非 root 用户使用 btrfs-progs"
date:   2024-10-01 00:00:00 +0800
categories:
---

btrfs-progs 是 BTRFS 文件系统的用户空间工具, 可以创建检查删除子卷等等.
但是有部分操作用不能由非 root 用户完成, 如 `btrfs subvolume show /xxx` (创建子卷可以, 但是显示信息不行, 显得非常奇怪).

这对一些备份脚本的编写造成了困扰, 比如测试一个入口是否是一个 BTRFS 的子卷, 可能需要这种 workaround:
```bash
[[ "$(stat -f -c %T "$1")" == btrfs && "$(stat -c %i "$1")" == 256 ]]
```


下面来看看是否有什么方法能绕过这个问题,
注意到 btrfs-progs 并没有直接抱怨自己为非 root 用户, 而是抱怨了一个类似 `strerror(EPERM)` 的错误:
```bash
$ btrfs subvolume show /home
ERROR: Could not search B-tree: Operation not permitted
```
所以在急着去看 btrfs-progs 的源码之前, 先用 `strace` 看看是什么 syscall 造成了这个错误:
```bash
...
ioctl(6, BTRFS_IOC_GET_SUBVOL_INFO, 0x7fffffffdec0) = 1
ioctl(6, BTRFS_IOC_TREE_SEARCH, {key={...}}) = -1 EPERM (Operation not permitted)
write(2, "ERROR: ", 7)                         = 7
write(2, "Could not search B-tree: Operati"..., 48
...
```
看来问题在于内核不喜欢我们进行 `BTRFS_IOC_TREE_SEARCH`,
下面去看看这个 ioctl 的内核代码, 容易在 `fs/btrfs/ioctl.c` 中找到:
```c
static noinline int btrfs_ioctl_tree_search(struct inode *inode,
					    void __user *argp)
{
	struct btrfs_ioctl_search_args __user *uargs = argp;
	struct btrfs_ioctl_search_key sk;
	int ret;
	u64 buf_size;

	if (!capable(CAP_SYS_ADMIN))
		return -EPERM;
```
看来很快就找到了一个问题, 作为普通用户, 显然是没有 `CAP_SYS_ADMIN` 的 capability 的.

为了解决这个问题, 简单的方法是使用 `setcap` 或者 `capsh` 给脚本设置增加 suid 等,
如果是 systemd 系统, 更好 (更安全?) 的方法可能是使用 `AmbientCapabilities`,
```systemd
AmbientCapabilities=CAP_SYS_ADMIN
```
或者可以根据情况增加 `CAP_CHOWN,CAP_DAC_OVERRIDE,CAP_FOWNER,CAP_SETFCAP` (可能会被 `btrfs receive` 等命令需要).
