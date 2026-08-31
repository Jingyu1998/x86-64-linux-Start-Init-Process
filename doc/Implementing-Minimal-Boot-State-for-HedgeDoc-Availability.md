---
tags: init process
---

# 實作 HedgeDoc 可用的最小 Boot State

## 實作目標

[研究 hedgedoc 可用所需要的最小服務集合-設計原則](./Implementing-Minimal-Boot-State-for-HedgeDoc-Availability.md)

上述文件的設計原則是建立一個描述「HedgeDoc 可用」所需最小 dependency graph 的 `hedgedoc.target`。

後續實驗透過 kernel command line `systemd.unit=hedgedoc.target`，讓 `hedgedoc.target` 取代 `default.target` 啟動。實驗確認，即使未 reached `multi-user.target`，仍可透過 Tailscale IP 存取 HedgeDoc，但此時無法登入 Linux 系統。

因此，本次實作將需求擴充為：開機抵達 `hedgedoc.target` 時，除了提供 HedgeDoc 服務，也能保留一般 Linux 開機環境的基本互動能力，讓使用者可以登入 Linux 並透過 shell 執行指令。

除了上述需求外，本實作還有以下兩個目標：
* 納入正常開機流程：`hedgedoc.target` 不只能透過 `systemd.unit=hedgedoc.target` 作為一次性的 boot target，也能納入正常的 systemd boot flow。
* 建立獨立的 Target State：將「HedgeDoc 可用 + Linux user login」描述成一個獨立的 system state，並能透過 target isolation 在不同 system state 之間切換。

因此，最終的實作目標是：

> 建立一個描述「HedgeDoc 可用 + Linux user login」所需最小 system state 的 hedgedoc.target，使其能作為獨立的 boot target、納入正常的 systemd boot flow，並能透過 target isolation 在不同 system state 之間切換。

## 第一版 hedgedoc target

```ini
# /etc/systemd/system/hedgedoc.target
[Unit]
Description=HedgeDoc Availability Target
Wants=docker.service tailscaled.service basic.target
```

* 使用 `Wants=` 將 HedgeDoc 所需的 `docker.service` 與 `tailscaled.service` 納入 `hedgedoc.target` 的 activation transaction。
* 使用 `Wants=` 將 `basic.target`  納入 `hedgedoc.target` 的 activation transaction。

### 準備實驗一環境

重讀設定檔: 
`systemctl daemon-reload`

手動關閉 docker.service 和 tailscaled.service
`systemctl stop docker.service`
`systemctl stop tailscaled.service `

docker.service 的狀態

`systemctl status docker.service --no-pager`

```
○ docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: inactive (dead) since Thu 2026-08-27 11:50:11 CST; 33s ago
```

tailscaled.service 的狀態

`systemctl status tailscaled.service --no-pager`

```
○ tailscaled.service - Tailscale node agent
     Loaded: loaded (/usr/lib/systemd/system/tailscaled.service; enabled; preset: enabled)
     Active: inactive (dead) since Thu 2026-08-27 11:50:27 CST; 40s ago
```

### 實驗一: 手動啟用 hedgedoc target 

手動開啟 hedgedoc target
`systemctl start hedgedoc.target`

hedgedoc target 的狀態
`systemctl status hedgedoc.target --no-pager`

```
● hedgedoc.target - HedgeDoc Availability Target
     Loaded: loaded (/etc/systemd/system/hedgedoc.target; static)
     Active: active since Thu 2026-08-27 11:53:37 CST; 22s ago
```

docker.service 的狀態

```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-08-27 11:53:37 CST; 36s ago
```

tailscaled.service 的狀態

```
● tailscaled.service - Tailscale node agent
     Loaded: loaded (/usr/lib/systemd/system/tailscaled.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-08-27 11:53:36 CST; 2min 2s ago
```

### 實驗一結論

手動啟用 hedgedoc target，會自動啟用 docker service、tailscaled service。
此時我可以透過 Tailscale IP 存取 HedgeDoc。

### 實驗二: hedgedoc target 是否為適合實際使用的 Linux boot state

手動在 kernel command line 加入 `systemd.unit=hedgedoc.target`，並且重開機。

### 實驗二結果

開機 log 停在
`[  OK  ] Reached target hedgedoc.target - HedgeDoc Availability Target.`

我可以透過 Tailscale IP 存取 HedgeDoc。
但是現在沒有出現 login screen，此時系統不具有像一般 Linux 開機環境的基本互動能力。
所以此時的 `hedgedoc.target`，還不適合作為實際使用的 Linux boot state。

## 第二版 hedgedoc target- 加入 Linux Login Screen

### 問題

第一版已經可以讓 HedgeDoc 使用，但還沒有確保 Linux login screen 存在。

### Unit file

```ini
# /etc/systemd/system/hedgedoc.target
[Unit]
Description=HedgeDoc Availability Target
Wants=docker.service tailscaled.service basic.target
Wants=getty.target
```

* 使用 `Wants=` 將登入所需的 `getty.target` 納入 `hedgedoc.target` 的 activation transaction。

### 實驗

手動在 kernel command line 加入 `systemd.unit=hedgedoc.target`，並且重開機。

### 實驗預期

在 terminal 上，看到 Linux login screen，並且可以登入 Linux。

### 實驗結果

開機 log 停在
```
[  OK  ] Reached target hedgedoc.target - HedgeDoc Availability Target.

Ubuntu 24.04.3 LTS xiaoyu-VMware-Virtual-Platform ttyS0

xiaoyu-VMware-Virtual-Platform login: xiaoyu
"System is booting up. Unprivileged users are not permitted to log in yet. Please come back later. For technical details, see pam_nologin(8)."

Login incorrect
```

輸入 username 後得到以下 error

```
"System is booting up. Unprivileged users are not permitted to log in yet.
Please come back later. For technical details, see pam_nologin(8)."
```

在 terminal 上，
* 可以看到 Linux login screen，並顯示 login prompt
* **但是仍無法登入 linux**

### 問題分析

`getty.target` 只解決了「提供 login prompt」的問題，並不代表 systemd 已經允許一般使用者登入。

## 第三版 hedgedoc target- 加入 systemd-user-sessions.service

### 問題分析

error log 提到 `Unprivileged users are not permitted to log in yet`

我查詢 systemd 的 manual，找到 `systemd-user-sessions.service`。
這個 service 的功能是允許 user 在 boot 後 login，禁止 user 在 shutdown 時 login。

[ Arch linux 的 systemd-user-sessions.service manual ](https://man.archlinux.org/man/systemd-user-sessions.service.8.en)

此外也確認此 service 在正常的 boot 流程中，是由 `multi-user.target` 拉起來。因此使用 `hedgedoc.target` 開機時，有可能沒有拉起來 `systemd-user-sessions.service`。

### Unit file

```ini
# /etc/systemd/system/hedgedoc.target
[Unit]
Description=HedgeDoc Availability Target
Wants=docker.service tailscaled.service basic.target
Wants=getty.target systemd-user-sessions.service
```

* 使用 `Wants=` 將登入所需的 `systemd-user-sessions.service` 納入 `hedgedoc.target` 的 activation transaction。

### 實驗

手動在 kernel command line 加入 `systemd.unit=hedgedoc.target`，並且重開機。

### 實驗預期

在 terminal 上，看到 Linux login screen，並且可以登入 Linux。

### 實驗結果

成功登入linux

## 第四版 hedgedoc target- 將其納入正常 Boot Flow

```ini
# /etc/systemd/system/hedgedoc.target
[Unit]
Description=HedgeDoc Availability Target
Wants=docker.service tailscaled.service basic.target
Wants=getty.target systemd-user-sessions.service
Before=multi-user.target

[Install]
RequiredBy=multi-user.target
```

* 加入 `Before=multi-user.target` 讓 `hedgedoc.target` 在與 `multi-user.target` 同一個 transaction 時，排在 `multi-user.target` 之前。
* 加入 `RequiredBy=multi-user.target`，讓 `multi-user.target` 將 `hedgedoc.target` 拉起來。

### 實驗

不再使用 `systemd.unit=hedgedoc.target`，恢復正常 `default.target → graphical.target`，並且重開機。

### 實驗預期

以下 target 開機後皆為 active 狀態 
* hedgedoc.target
* multi-user.target
* graphical.target

### 實驗結果

```
● hedgedoc.target - HedgeDoc Availability Target
     Loaded: loaded (/etc/systemd/system/hedgedoc.target; enabled; preset: enab>
     Active: active since Sat 2026-08-29 18:07:23 CST; 45s ago
```

```
● multi-user.target - Multi-User System
     Loaded: loaded (/usr/lib/systemd/system/multi-user.target; static)
     Active: active since Sat 2026-08-29 18:07:23 CST; 54s ago
```

```
● graphical.target - Graphical Interface
     Loaded: loaded (/usr/lib/systemd/system/graphical.target; indirect; preset>
     Active: active since Sat 2026-08-29 18:07:23 CST; 1min 15s ago
```

## 最終版 hedgedoc target- 啟用 Isolation

### Unit file

```ini
# /etc/systemd/system/hedgedoc.target
[Unit]
Description=HedgeDoc Availability Target
Wants=docker.service tailscaled.service basic.target
Wants=getty.target systemd-user-sessions.service
Before=multi-user.target
AllowIsolate=yes

[Install]
RequiredBy=multi-user.target
```

* 加入 `AllowIsolate=yes` 允許 `hedgedoc.target` 成為 `systemctl isolate` 的目標。

### 實驗

正常重開機，抵達 `graphical.target`。
登入後，執行 `systemctl isolate hedgedoc.target`

### 實驗預期

切換成 `hedgedoc.target` 時，`hedgedoc.target` 不需要的 unit 將被停止。
例如, 以下 target 的狀態會變成 inactive。
* `multi-user.target`
* `hedgedoc.target`

### 實驗結果

```
○ multi-user.target - Multi-User System
     Loaded: loaded (/usr/lib/systemd/system/multi-user.target; static)
     Active: inactive (dead) since Sat 2026-08-29 18:57:23 CST; 1min 8s ago
```

```
○ graphical.target - Graphical Interface
     Loaded: loaded (/usr/lib/systemd/system/graphical.target; indirect; preset>
     Active: inactive (dead) since Sat 2026-08-29 18:57:23 CST; 1min 53s ago
```

`hedgedoc.target` 不只是 boot target，也可以作為一個獨立的 target state，透過 `systemctl isolate` 與其他 target state 之間切換。

## 實作結果與結論 

| 能力                          | `multi-user.target` | `hedgedoc.target` |
| --------------------------- | ------------------: | ----------------: |
| Docker                      |                   ✓ |                 ✓ |
| Tailscale                   |                   ✓ |                 ✓ |
| HedgeDoc                    |                   ✓ |                 ✓ |
| Login screen                |                   ✓ |                 ✓ |
| 一般使用者登入                     |                   ✓ |                 ✓ |
| `multi-user.target` reached |                   ✓ |               不需要 |
| Graphical environment       |                   ✓ |               不需要 |

本實作成功建立一個描述「HedgeDoc 可用 + Linux user login」所需最小 system state 的 hedgedoc.target，使其能作為獨立的 boot target、納入正常的 systemd boot flow，並能透過 target isolation 在不同 system state 之間切換。