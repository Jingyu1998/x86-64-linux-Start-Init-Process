---
tags: init process
---

# Sysinit Target Unit

## bootup picture

```
                             cryptsetup-pre.target veritysetup-pre.target
                                                  |
(various low-level                                v
 API VFS mounts:             (various cryptsetup/veritysetup devices...)
 mqueue, configfs,                                | 
 debugfs, ...)                                    v 
 |                                  cryptsetup.target
 |  (various swap                                 |
 |   devices...)                                  |
 |    |                                           |
 |    v                       local-fs-pre.target |
 |  swap.target                       |           |
 |    |                               v           |
 |    |  (various low-level  (various mounts and  |
 |    |   services: udevd,    fsck services...)   |
 |    |   tmpfiles, random            |           |
 |    |   seed, sysctl, ...)          v           |
 |    |      |                 local-fs.target    |
 |    |      |                        |           |
 \____|______|_______________   ______|___________/
                             \ /
                              v 
                       sysinit.target
```

## 此 target 代表什麼 synchronization point？

`sysinit.target` 是 **systemd 完成基本 system initialization** 後，供**後續一般 system services 使用的 synchronization point**

systemd 會自動對**一般 Service** 加入

```ini
Requires=sysinit.target
After=sysinit.target
```

除非該 Service 設定 `DefaultDependencies=no`

一般 Service 預設**不能**在 `sysinit.target` 之前完成 startup。

## 到達此 target 時，哪些重要 Units 已經完成？

### Local filesystem initialization 已經完成

當 `local-fs.target` reached 時，systemd 所管理的 local filesystem mounts 已經依照其 ordering 完成。

這通常包括：
* root filesystem `/`
* `/etc/fstab` 中需要在 boot 時 mount 的 local filesystems

### Swap initialization 已經完成

`swap.target` reached → systemd 所管理的 swap partitions/files 已經完成 activation。

### Low-level system initialization 已經完成

* `systemd-udevd`: 負責 userspace device management。
* `systemd-tmpfiles`: 負責建立、清理及初始化部分 runtime / temporary files。
* `systemd-sysctl`: 載入 kernel `/proc/sys` 相關設定。

## 哪些後續 Units 會依賴此 target

所有使用預設 `DefaultDependencies=yes` 的 system Service，systemd 都會自動加入 `Requires=sysinit.target` 和 `After=sysinit.target`。

## 參考來源

[ freedesktop bootup manual ](https://www.freedesktop.org/software/systemd/man/latest/bootup.html#)
[ arch linux 的 systemd.special manual ](https://man.archlinux.org/man/systemd.special.7.en)