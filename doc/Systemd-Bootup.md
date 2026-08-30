---
tags: init process
---

# Systemd Bootup

## System Manager Bootup

Bootup 時，作業系統映像上的 system manager 負責初始化系統運作所需的必要檔案系統、服務和驅動程式。

在 systemd 中，開機程序被拆分為多個離散步驟，這些步驟會作為 target unit 公開。

開機過程高度並行化，因此特定 target unit 的到達順序是不確定的，但仍遵循有限的排序結構。

當 systemd 啟動系統時，它將啟動所有 `default.target` 的依賴 unit（以及遞歸地啟動這些`default.target` 的依賴 unit 的 所有 依賴 unit ）。

通常情況下，`default.target` 只是 `graphic.target` 或 `multi-user.target` 的別名，具體取決於系統是配置為 graphical UI 還是僅配置為 text console。

為了強制要求拉取的 unit 之間保持 minimal ordering，可以使用一些 well-known 的 target unit，如 systemd.special(7) 中所述。

下圖概述了這些 well-known target unit 的結構及其在開機邏輯中的位置。

**箭頭**描述哪些 target unit is pulled，以及哪些 target unit 在其他 target unit 之前被排序。位於圖表 top 的 target unit 會比位於 bottom 的 target unit 更早啟動。

```
                             cryptsetup-pre.target veritysetup-pre.target
                                                  |
(various low-level                                v
 API VFS mounts:             (various cryptsetup/veritysetup devices...)
 mqueue, configfs,                                |    |
 debugfs, ...)                                    v    |
 |                                  cryptsetup.target  |
 |  (various swap                                 |    |    remote-fs-pre.target
 |   devices...)                                  |    |     |        |
 |    |                                           |    |     |        v
 |    v                       local-fs-pre.target |    |     |  (network file systems)
 |  swap.target                       |           |    v     v                 |
 |    |                               v           |  remote-cryptsetup.target  |
 |    |  (various low-level  (various mounts and  |  remote-veritysetup.target |
 |    |   services: udevd,    fsck services...)   |             |              |
 |    |   tmpfiles, random            |           |             |    remote-fs.target
 |    |   seed, sysctl, ...)          v           |             |              |
 |    |      |                 local-fs.target    |             | _____________/
 |    |      |                        |           |             |/
 \____|______|_______________   ______|___________/             |
                             \ /                                |
                              v                                 |
                       sysinit.target                           |
                              |                                 |
       ______________________/|\_____________________           |
      /              |        |      |               \          |
      |              |        |      |               |          |
      v              v        |      v               |          |
 (various       (various      |  (various            |          |
  timers...)      paths...)   |   sockets...)        |          |
      |              |        |      |               |          |
      v              v        |      v               |          |
timers.target  paths.target   |  sockets.target      |          |
      |              |        |      |               v          |
      v              \_______ | _____/         rescue.service   |
                             \|/                     |          |
                              v                      v          |
                          basic.target         rescue.target    |
                              |                                 |
                      ________v____________________             |
                     /              |              \            |
                     |              |              |            |
                     v              v              v            |
                 display-    (various system   (various system  |
             manager.service     services        services)      |
                     |         required for        |            |
                     |        graphical UIs)       v            v
                     |              |            multi-user.target
emergency.service    |              |              |
        |            \_____________ | _____________/
        v                          \|/
emergency.target                    v
                              graphical.target
```


本文重點在於常用的開機 target unit。這些 target unit 是理想的選擇，

例如可以透過將它們傳遞給 `systemd.unit=` kernel command line option（請參閱 systemd(1)）或將 `default.target` 符號連結到這些開機 target unit 來實現。

## 參考來源

[ freedesktop bootup manual ](https://www.freedesktop.org/software/systemd/man/latest/bootup.html#)