---
tags: init process
---

# Basic Target Unit

## bootup picture

```
                       sysinit.target 
                              |       
                     ________/|\______
                     |        |      |
                     |        |      |
                     v        |      v
                (various      |  (various
                  paths...)   |   sockets...)
                     |        |      |
                     v        |      v
               paths.target   |  sockets.target
                     |        |      |
                     \_______ | _____/
                             \|/
                              v
                          basic.target  
```

## 此 target 代表什麼 synchronization point？

Reach `basic.target` 時，系統已經具備啟動一般 system services 所需的基本 userspace 環境。

`basic.target` 作為 later boot process 的 synchronization point

systemd 會自動對一般 service 加上 `After=basic.target`，除非該 service 設定 `DefaultDependencies=no`。

`basic.target` 會自動成為一般 services 的 `After=` dependency。

## 到達此 target 時，哪些重要 Units 已經完成？

Usually, this should pull-in all **local mount points** plus `/var`, `/tmp` and `/var/tmp`, swap devices, sockets, timers, path units and other basic initialization necessary for general purpose daemons.

### Sysinit.target

`local-fs.target`
基本 filesystem mounts 已經完成，例如：
* `/`
* `/var`
* `/tmp`
* `/var/tmp`

`Swap.target`
swap devices / swap files 已處理。

### Socket.target

systemd 所管理的 socket units 已建立。

### Timer.target

boot 後應該保持 active 的 timer units 已建立。

### Paths.target 

boot 後應該保持 active 的 path units 已建立。

## 哪些後續 Units 會依賴此 target

一般 service units 預設都會有 `After=basic.target`。

## 參考來源

[ freedesktop bootup manual ](https://www.freedesktop.org/software/systemd/man/latest/bootup.html#)
[ arch linux 的 systemd.special manual ](https://man.archlinux.org/man/systemd.special.7.en)