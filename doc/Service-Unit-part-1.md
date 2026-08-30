---
tags: init process
---

# Service Unit- part 1

Service Unit 用於啟動並控制 daemon，以及 daemon 所包含的 processes。

## Description

名稱以 `.service` 結尾的 **unit configuration file** 編碼了由 systemd 控制和監督的 process 的資訊。

**本手冊列出了 Service Unit 特有的設定選項。**

有關所有 unit configuration file 的**通用選項**，請參閱 systemd.unit(5)。
通用設定項在通用 `[Unit]` 和 `[Install]` 部分中進行設定。

**特定於服務的 configuration option 在 `[Service]` 部分中進行配置。**

## Service Template

systemd 服務可以透過 `service@argument.service` 語法，此語法**接受單一 argument**。

此類服務稱為「 instantiated 」服務，而**沒有參數**的 unit definition 稱為「 Template 」。

Service Template 的範例
例如，`dhcpcd@.service` Service Template 接受網路介面作為參數來形成 instantiated 的服務

在 Service file 中，可以使用 `%-specifiers` 存取此 argument 或「 instance name 」。有關詳細信息，請參閱 systemd.unit(5)。

## Implicit Dependency 

**Implicit Dependency** 代表你**沒有在 unit configuration file 裡明確寫出的 dependency**，但 systemd 根據 Service Unit 的設定，自動替你加入的 dependency。

Implicit Dependency 範例一
* Services with `Type=dbus` set automatically acquire dependencies of type `Requires=` and `After=` on `dbus.socket`.

在 Unit Config File 的 `[Service]` section 中寫入 `Type=dbus`

```ini
[Service]
Type=dbus
```

你雖然沒有寫：

```ini
[Unit]
Requires=dbus.socket
After=dbus.socket
```

但 systemd 的 dependency model 裡仍然會有這兩個關係。

----

Implicit Dependency 範例二
* Socket activated **services** are automatically ordered **after** their activating **`.socket`** units **via an automatic `After=` dependency**.

Socket activation
```
foo.socket
    │
    │ activates
    ▼
foo.service
```

systemd 知道：`foo.service` 是由 `foo.socket` 啟動的。
因此 systemd 的 dependency model 裡會自動加入以下 dependency。

```ini
After=foo.socket
```

----

Implicit Dependency 範例三
* Services also pull in all `.socket` units listed in Sockets= via automatic Wants= and After= dependencies.

假設：
```ini
[Service]
Sockets=foo.socket bar.socket
```

那 systemd 的 dependency model 裡會自動加入以下 dependency：

```
foo.service
    │
    ├── Wants=foo.socket
    ├── After=foo.socket
    │
    ├── Wants=bar.socket
    └── After=bar.socket
```

以上範例並不是 implicit dependency 的全部種類，Service Unit 的其他設定，也可能讓 systemd 在 dependency model 自動增加 dependency。

## Default Dependency

`Default Dependencies` 是 systemd 預設為 Service Unit 加入的一組 dependencies。
除非設定：
```ini
[Unit]
DefaultDependencies=no
```

否則 systemd 會自動加入這些 default dependencies。

### 一般 Service Unit
一般 Service Unit 預設會取得：

```ini
Requires=sysinit.target
After=sysinit.target
After=basic.target
Conflicts=shutdown.target
Before=shutdown.target
```

* 一般 Service 會在 system initialization 完成的情況下才能啟動。
* 一般 Service 會在 `basic.target` 之後啟動。
* system shutdown 時，一般 Service 會在 `shutdown.target` 之前被終止。

只有涉及 early boot 或 late system shutdown 的 Service，才通常需要設定：

```ini
DefaultDependencies=no
```

設定 `DefaultDependencies=no` **並不代表 Unit 沒有 dependencies**，而是表示 systemd 不再自動加入上述這組 default dependencies；Unit 仍然可以具有其他 explicit 或 implicit dependencies。

### Instanced service units

Instanced Service Unit 是名稱中包含 `@` 的 Service，例如：
```
foo@1.service
foo@2.service
```

它們預設會被放入一個以 template unit 命名的 Slice unit：

```
foo.slice
├── foo@1.service
├── foo@2.service
```

這個 slice unit 用來將同一個 template 的所有 instances 組織在一起，並可用於 resource management。

這個 Slice unit 預設也會在 system shutdown 時停止。

如果不希望使用這個預設行為，可以：
* 在 `template unit` 中設定 `DefaultDependencies=no`，並自行定義相應的 Slice。
* 在 `template unit` 中使用 `Slice=system.slice` 或其他適合的 Slice。

## 參考資料

[ arch linux 的 systemd.service manual ](https://man.archlinux.org/man/systemd.service.5.en)