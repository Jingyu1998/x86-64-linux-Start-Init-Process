---
tags: init process
---

# 研究 hedgedoc 可用所需要的最小服務集合

## 研究背景

目前的系統：
* Ubuntu + systemd
* HedgeDoc 運行於 Docker container
* 使用 Tailscale IP 存取 HedgeDoc
* 目前正常 boot 會進入 `graphical.target`

## 定義 hedgedoc 可以使用的狀態

hedgedoc 可以使用的狀態，定義如下
- 系統完成 boot 後，可以透過已設定的 Tailscale IP，從瀏覽器進入 HedgeDoc 入口畫面，並且可以登入、閱讀及修改筆記。

## hedgedoc 需要什麼服務

hedgedoc 可以使用，需要
* Docker 可用
* Tailscale 可用

`docker.service` 是 active 狀態，代表 Docker 可用
`tailscaled.service` 是 active 狀態，代表 Tailscale 可用

因此，hedgedoc 可以使用的狀態，所需要的 service 
- `docker.service`
- `tailscaled.service`

## 實驗確認 Docker 和 Tailscale 的必要性

個別執行以下指令
* `systemctl stop docker.service` 
* `systemctl stop tailscaled.service` 

停止任何一項服務，皆無法透過已設定的 Tailscale IP，從瀏覽器進入 HedgeDoc 入口畫面。

## docker service 的 dependency

以不同觀測角度，研究 dependency
- 明確寫在 Unit file 的 dependency
- systemd 解析後的 dependency

### 明確寫在 Unit file 的 dependency

```ini
# /usr/lib/systemd/system/docker.service
[Unit]
Wants=network-online.target containerd.service
Requires=docker.socket
```

### systemd 解析後的 dependency

除了 Unit file 明確宣告的 dependency 外， systemd 會根據其他 Unit 設定、implicit dependencies、default dependencies，建立完整的 dependency。 

`systemctl list-dependencies docker.service --no-pager`

```
● ├─containerd.service
● ├─docker.socket
● ├─system.slice
● ├─network-online.target
● └─sysinit.target
```

## docker service 的 ordering

以不同觀測角度，研究 ordering
- 明確寫在 Unit file 的 ordering
- systemd 解析後的 ordering

### 明確寫在 Unit file 的 ordering

```ini
# /usr/lib/systemd/system/docker.service
[Unit]
After=network-online.target nss-lookup.target docker.socket firewalld.service containerd.service time-set.target
```

### systemd 解析後的 ordering

除了 Unit file 明確宣告的 ordering 外， systemd 會根據其他 Unit 設定、implicit ordering、default ordering，建立完整的 ordering。 

`systemctl list-dependencies docker.service --no-pager --after`

```
● ├─containerd.service
● ├─docker.socket
○ ├─firewalld.service
● ├─system.slice
● ├─systemd-journald.socket
● ├─basic.target
...
● ├─network-online.target
○ │ ├─cloud-init.service
● │ ├─NetworkManager-wait-online.service
○ │ ├─systemd-networkd-wait-online.service
● │ └─network.target
○ │   ├─netplan-ovs-cleanup.service
● │   ├─NetworkManager.service
○ │   ├─systemd-networkd.service
● │   ├─systemd-resolved.service
● │   ├─wpa_supplicant.service
● │   └─network-pre.target
○ │     ├─cloud-init-local.service
● │     └─ufw.service
...
● ├─nss-lookup.target
● │ └─systemd-resolved.service
...
● ├─sysinit.target
...
```

## tailscale service 的 dependency

以不同觀測角度，研究 dependency
- 明確寫在 Unit file 的 dependency
- systemd 解析後的 dependency

### 明確寫在 Unit file 的 dependency

```ini
# /usr/lib/systemd/system/tailscaled.service
[Unit]
Wants=network-pre.target
```

### systemd 解析後的 dependency

除了 Unit file 明確宣告的 dependency 外， systemd 會根據其他 Unit 設定、implicit dependencies、default dependencies，建立完整的 dependency。 

`systemctl list-dependencies tailscaled.service --no-pager`

```
● ├─-.mount
● ├─system.slice
● ├─network-pre.target
● └─sysinit.target
```

## tailscale service 的 ordering

以不同觀測角度，研究 ordering
- 明確寫在 Unit file 的 ordering
- systemd 解析後的 ordering

### 明確寫在 Unit file 的 ordering

```ini
# /usr/lib/systemd/system/tailscaled.service
[Unit]
After=network-pre.target NetworkManager.service systemd-resolved.service
```

### systemd 解析後的 ordering

除了 Unit file 明確宣告的 ordering 外， systemd 會根據其他 Unit 設定、implicit ordering、default ordering，建立完整的 ordering。 

`systemctl list-dependencies tailscaled.service --no-pager --after`

```
● ├─-.mount
● ├─NetworkManager.service
● ├─system.slice
● ├─systemd-journald.socket
● ├─systemd-remount-fs.service
● ├─systemd-resolved.service
● ├─basic.target
● ├─network-pre.target
○ │ ├─cloud-init-local.service
● │ └─ufw.service
● └─sysinit.target
```

## 目前的 boot architecture

目前正常 boot 會進入 `graphical.target`。
`graphical.target` 依賴 `multi-user.target`

`docker.service` 與 `tailscaled.service` 透過 `WantedBy=multi-user.target` 被納入 `multi-user.target` 的 activation dependency。


```ini
# /usr/lib/systemd/system/docker.service
[Install]
WantedBy=multi-user.target
```

```ini
# /usr/lib/systemd/system/tailscaled.service
[Install]
WantedBy=multi-user.target
```

也就是 
* `multi-user.target` 依賴 `docker.service` 
* `multi-user.target` 依賴 `tailscaled.service`

依賴關係如下圖所示

```
docker.service tailscaled.service
    |                |
    |                |
    V                V
    multi-user.target
            |
            |
            V
    graphical.target
```

## 尋找較早的 boot synchronization point

根據[ freedesktop bootup manual ](https://www.freedesktop.org/software/systemd/man/latest/bootup.html#)

```
sysinit.target
      ↓
basic.target
      ↓
multi-user.target
      ↓
graphical.target
```

bootup 時，有四個重要的 synchronization point。`basic.target` 是` multi-user.target` 之前的重要 synchronization point。

因此提出以下問題：
> 當 `basic.target` reached 時，HedgeDoc 可用所需要的 units 已經被啟動到什麼程度？

如果 `basic.target` 的 dependency graph 已經涵蓋 HedgeDoc 可用所需要的所有 units，且這些 units 在 `basic.target` reached 時都已達到可用狀態，那麼 `basic.target` 可以作為 HedgeDoc 可用的 synchronization point。

但如果 `basic.target` 尚未涵蓋 HedgeDoc 可用所需要的全部 units，則需要進一步找出 `basic.target` 之後缺少的 units，並建立一個新的 target 描述 HedgeDoc 可用所需的 system state。

## 分析 basic target 能涵蓋哪些 dependency

basic target 由 systemd 解析的 dependency

`systemctl list-dependencies basic.target --no-pager`

```
● ├─-.mount V
...
● ├─slices.target
● │ ├─system.slice V
...
● ├─sockets.target
● │ ├─docker.socket V
...
● └─sysinit.target V
```

### 與 Docker dependency 比較

docker service 由 systemd 解析的 dependency

```
● ├─containerd.service
● ├─docker.socket
● ├─system.slice
● ├─network-online.target
● └─sysinit.target
```

| Dependency              | `basic.target` 已涵蓋？ |
| ----------------------- | ------------------- |
| `docker.socket`         | ✓                   |
| `system.slice`          | ✓                   |
| `sysinit.target`        | ✓                   |
| `containerd.service`    | ✗                   |
| `network-online.target` | ✗                   |

### 與 Tailscale dependency 比較

tailscale service 由 systemd 解析的 dependency

```
● ├─-.mount
● ├─system.slice
● ├─network-pre.target
● └─sysinit.target
```

| Dependency           | `basic.target` 已涵蓋？ |
| -------------------- | ------------------- |
| `system.slice`       | ✓                   |
| `sysinit.target`     | ✓                   |
| `network-pre.target` | ✗                   |

### 小結

抵達 `basic.target` 並不足以涵蓋 Docker 與 Tailscale 啟動所需的全部 dependency。
要讓 Hedgedoc 可用，還需要把 `basic.target` 所未涵蓋、且 HedgeDoc 可用所需的 units 納入 dependency graph。

## 設計原則

我不一定要進入 `multi-user.target`。我只需要 HedgeDoc 可用所需的最小 system state。

既然 `basic.target` 並不足以涵蓋 Docker 與 Tailscale 啟動所需的全部 dependency。

那我就建立一個新的 `hedgedoc.target`，描述 HedgeDoc 可用所需的最小 dependency graph。
同時讓 `multi-user.target` 依賴 `hedgedoc.target`，把 `hedgedoc.target` 納入 `graphical.target` 正常啟動流程裡。

## 實作

[實作 HedgeDoc 可用的最小 Boot State](http://100.71.125.87:3000/fryK7sW-Ss6zY_jesQj_rw)