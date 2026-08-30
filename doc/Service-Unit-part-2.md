---
tags: init process
---

# Service Unit- part 2

## Type Option

`Type=` 代表 systemd 要**如何判斷這個 Service 已經 started**

不同 `Type=` 的差異，主要就是：**systemd 在什麼時機認為「Service 已經 started」**。

這會直接影響：
```
Service A
    │
    ▼
Service A startup complete
    │
    ▼
Service B 可以繼續啟動
```

Type 的選項有 
* simple 
* exec
* forking
* oneshot
* dbus
* notify
* notify-reload
* idle

本篇筆記先針對 `simple`、`exec`、`oneshot`、`notify` 做介紹。

### Type=simple

當 main process 被 fork，Systemd 立即將 Service 視作已經 started。 

```
systemd
   │
   ▼
fork()
   │
   ▼
child process
   │
   ├── systemd 認為 Service started
   │
   ▼
execve()
   │
   ▼
actual service binary
```

> **在 `execve()` 之前，systemd 就認為 Service 已經 started**

如果 executable 根本不存在，`systemctl start` 對 `Type=simple` 的 Service  仍可能回報成功，因為 systemd 在 executable 真正被 `execve()` 執行之前就已經認定 service started。

### Type=exec

當 main service binary 被執行後，Systemd 立即將 Service 視作已經 started。

```
systemd
   │
   ▼
fork()
   │
   ▼
child process
   │
   │
   ▼
execve()
   │
   ├── systemd 認為 Service started
   ▼
actual service binary
```

當 main service binary 無法被成功呼叫時，`systemctl start` 對 `Type=exec` 的 Service 會回報失敗。

```
`fork()
  ↓
execve() fails
  ↓
systemd 得知 startup failed
```

因此，`Type=exec` 通常是比 `Type=simple` 更好的選擇。

### Type=oneshot

當 main process 退出後，systemd 才將 Service 視作已經 started。

```
systemd
   │
   ▼
ExecStart=
   │
   ▼
process
   │
   │ 
   ▼
process exits
   │
   ├── systemd 認為 Service started
   ▼
```

`Type=oneshot` 適合執行一個 action，而不是持續維持一個 daemon process。

另外，oneshot 可以有多個 `ExecStart=：`

```ini
Type=oneshot
ExecStart=/usr/bin/foo
ExecStart=/usr/bin/bar
```

而其他 Service Type 一般只能有一個 `ExecStart=`

### Type=notify

Service 自己通知 systemd：「我已經 started。」

```
systemd
   │
   ▼
start service
   │
   ▼
service initialization
   │
   │
   └── sd_notify("READY=1") ── 告訴 systemd Service started
                │
                ▼
             systemd
                │
                ▼
       start follow-up units
```

該 Service 發送 “READY=1” 通知訊息
所以 Service 本身要支援 systemd notification protocol
這個 Type 的優點是：Service 可以自己精確決定「我已經 started」的時間點

## ExecStart Option

`ExecStart=` 定義 **Service 啟動時要執行的 command。**

```ini
[Service]
ExecStart=/usr/bin/foo
```

當 systemd 啟動這個 Service 時，會執行：
```
systemd
   │
   │ ExecStart=/usr/bin/foo
   ▼
 fork()
   │
   ▼
child process ── systemd 視為此 process 是 Service 的 main process
```

除了 `Type=forking` 之外，`ExecStart=` 所啟動的 process 會被 systemd 視為 Service 的 main process。

### 比較 Type=oneshot 與其他 Type

除了 `Type=oneshot` 之外：Service **必須且只能設定一個** `ExecStart=`
例如:

```ini
[Service]
Type=exec
ExecStart=/usr/bin/foo
```

---

`Type=oneshot` 可以設定多個 `ExecStart=` ：

```ini
[Service]
Type=oneshot
ExecStart=/usr/bin/foo
ExecStart=/usr/bin/bar
ExecStart=/usr/bin/baz
```

這些 command 會按照 Unit file 中出現的順序**依序執行**：

```
ExecStart=/usr/bin/foo
        │
        ▼
ExecStart=/usr/bin/bar
        │
        ▼
ExecStart=/usr/bin/baz
```

### 如何清除先前的 ExecStart=

如果將 ExecStart= 設定為空字串：
```ini
ExecStart=
```

會將之前設定的 ExecStart= 清除。

例如:
```ini
ExecStart=/usr/bin/foo
ExecStart=
ExecStart=/usr/bin/bar
```

前面的 `/usr/bin/foo` 會被 reset，因此最後只剩 `/usr/bin/bar`。
這個功能主要會在 **Unit file override / drop-in** 中看到。

### 如果 Service 沒有設定 ExecStart= 

如果沒有設定 `ExecStart=`，Service 必須同時滿足：
* `RemainAfterExit=yes`
* 以及至少一個：`ExecStop=`

例如概念上：
```ini
[Service]
RemainAfterExit=yes
ExecStop=/usr/bin/foo-stop
```

如果 Service 同時沒有：
* `ExecStart=`
* `ExecStop=`
則 Service Unit 是無效的。

### ExecStartPre= ExecStartPost= Option

`ExecStartPre=`, `ExecStartPost=` 在 `ExecStart=` 命令之前或之後執行的附加命令。
`ExecStartPre=`, `ExecStartPost=` 語法與 `ExecStart=` 相同。
Systemd 允許使用多個 `ExecStartPre=`, `ExecStartPost=` 命令，與服務類型（即 `Type=`）無關，並且命令將按順序逐一執行。

`ExecStartPre=`, `ExecStartPost=` 的執行順序
```
ExecStartPre=
       ↓
ExecStart=
       ↓
ExecStartPost=
```

## ExecStop Option

`ExecStop=` 定義 Service 停止時要執行的 command，用來停止由 `ExecStart=` 啟動的 Service。

```ini
[Service]
ExecStart=/usr/bin/foo
ExecStop=/usr/bin/foo-stop
```

概念上：

```
systemd
   │
   │ start
   ▼
ExecStart=
   │
   ▼
Service process
   │
   │ stop requested
   ▼
ExecStop=
   │
   ▼
Service terminates
```

### ExecStop= 不是必要設定。

如果沒有指定：`ExecStop=`

當 systemd 要求 Service 停止時，systemd 會直接對 Service 的 process 發送：`KillSignal=`，或在 restart 情況下使用：`RestartKillSignal=` 指定的 signal。之後剩餘的 processes 會依照：`KillMode=` 進行處理。

因此：

```
有 ExecStop=
    │
    ▼
執行 ExecStop=
    │
    ▼
Service stop
    │
    ▼
依 KillMode= 處理剩餘 processes


沒有 ExecStop=
    │
    ▼
systemd 發送 KillSignal=
    │
    ▼
依 KillMode= 處理 processes
```

### ExecStop= 應該是同步操作

`ExecStop=` 不應該只是：發送一個 termination signal，然後自己立刻結束。

例如概念上：

```
ExecStop=
    │
    ▼
要求 Service terminate
    │
    ├── Service 還在 termination
    │
    ▼
ExecStop= command 已經結束
    │
    ▼
systemd 立即依 KillMode= / KillSignal=
處理剩餘 processes
```

這可能導致 Service 還沒完成正常 shutdown，就被 systemd 強制終止。

因此 manual 建議: `ExecStop=` 應該是 synchronous operation，而不是 asynchronous operation。 

也就是：
```
ExecStop=
    │
    ▼
要求 Service 停止
    │
    ▼
等待 Service 完成 termination
    │
    ▼
ExecStop= command 結束
    │
    ▼
systemd 完成 stop operation
```

因此可以簡化成：`ExecStop=` 應該等待 Service 已完成正常終止。

### ExecStop= 只有在 Service 成功啟動後才執行

`ExecStop=` 只會在 Service 曾經成功啟動後執行。

例如：
```
ExecStart=
    │
    ▼
startup success
    │
    ▼
Service started
    │
    ▼
stop requested
    │
    ▼
ExecStop= ✓
```

但是如果 Service 根本沒有成功啟動：

```
ExecStart=
    │
    ▼
startup failed
    │
    ▼
ExecStop= ✗
```

例如:
* ExecStart=
* ExecStartPre=
* ExecStartPost=

其中任一 command 失敗，導致 Service startup failed，`ExecStop=` 都不會被執行。

### ExecStopPost= Option

如果希望：不論 Service 是正常停止還是 startup 失敗，都執行 cleanup。
應該使用：`ExecStopPost=`

因此可以先建立：
```
正常啟動
   │
   ▼
Service started
   │
   ▼
stop
   │
   ├── ExecStop=
   │
   └── ExecStopPost=
```

而
```
startup failed
   │
   ▼
ExecStop= ✗
   │
   ▼
ExecStopPost= ✓
```

|                    | `ExecStop=`       | `ExecStopPost=`     |
| ------------------ | ----------------- | ------------------- |
| 主要目的              | 要求 Service 正常停止   | 停止後 cleanup      |
| Service 正常停止      | 執行                | 執行                  |
| Service startup 失敗 | 不執行               | 執行                  |

### Service restart 也會執行 ExecStop=

systemd 的 Service restart 可以理解成：

```
restart
  │
  ▼
stop
  │
  ├── ExecStop=
  └── ExecStopPost=
  │
  ▼
start
  │
  └── ExecStart=
```

因此：systemctl restart foo.service 並不是直接重新執行 `ExecStart=`。
而是先執行 `ExecStop=`、`ExecStopPost=`，再執行 `ExecStart=`。

### ExecStop= 與 ExecStart= 的關係

```
                 Service lifecycle

systemd
   │
   │ start
   ▼
ExecStart=
   │
   ▼
Service process
   │
   │ running
   │
   │ stop / restart
   ▼
ExecStop=
   │
   ▼
Service terminates
   │
   ▼
ExecStopPost=
```

## 參考資料

[ arch linux 的 systemd.service manual ](https://man.archlinux.org/man/systemd.service.5.en)