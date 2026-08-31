---
tags: init process
---

# Target Unit

Target Unit 用於將多個 Unit 分組，並作為與其他 Unit file 建立 ordering 和 dependency 時的同步點。

## Description 
名稱以「.target」結尾的 unit configuration file 編碼了 systemd Target Unit 的資訊。

有關所有 unit configuration file 的**通用選項**，請參閱 systemd.unit(5)。
通用設定項在通用 `[Unit]` 和 `[Install]` 部分中進行設定。

此 unit type 沒有特定 option，因此不存在單獨的 `[Target]` 部分。

Target Unit 除了 Unit 提供的通用功能外，不提供任何額外的功能。
Target Unit 只是將單元分組，
* 允許在 `Wants=` 和 `Requires=` 設定中**使用單一 target name** 來建立**一組單元的 dependency**，
* 允許在 `Before=` 和 `After=` 設定中**使用單一 target name** 來建立**一組單元的 ordering**，。

Target Unit 為啟動和關閉期間的 synchronization point 建立標準化名稱。
重要的 Target Unit，請參閱 systemd.special(7)，以了解 standard systemd target 的範例和說明。

Target Unit 可以用來**描述系統目前位於某個運作狀態。**

Target Unit 為經典 SysV 初始化系統中的 SysV runlevel 提供了更靈活的替代方案。

請注意，Target Unit File 不能為空，否則將被視為已屏蔽的 unit。建議提供一個 `[Unit]` 部分，其中包含描述性描述 (Description=) 和文件 (Documentation=) 選項。

## Implicit Dependencies

Target Unit 沒有自己的 Implicit Dependencies

## Default Dependencies

第一個 Default Dependency

除非: `DefaultDependencies=no` 
否則 Target 會自動取得：
* `Conflicts=shutdown.target`
* `Before=shutdown.target`

也就是：
```
target
   │
   │ Before=
   ▼
shutdown.target
```
因此正常 Target 會在 shutdown 時被處理

----

第二個 Default Dependency

如果在 **Target Unit** 定義

```ini
[Unit]
Wants=foo.service
Requires=bar.service
```

Target Unit 會自動補上

```ini
After=foo.service
After=bar.service
```

也就是 Target Unit 會被排在 foo.service 和 bar.service 之後抵達。

----

第二個 Default Dependency 的 **Reverse direction** 不成立

如果 **Service Unit** 定義如下, eg: `Some.service`

```ini
[Unit]
Wants=that.target
```

不會自動補上：`After=that.target`
如果 `Some.service` 真的需要在 `that.target` 之後啟動，`Some.service` 就必須明確寫：

```ini
[Unit]
Wants=that.target
After=that.target
```

## Example: Simple standalone target 

`emergency-net.target` 是 systemd.target(5) 提供的 **standalone target**，用來建立一個包含 networking 的 emergency mode。

```ini
[Unit]
Description=Emergency Mode with Networking
Requires=emergency.target systemd-networkd.service
After=emergency.target systemd-networkd.service
AllowIsolate=yes
```

`Requires=` 將 `emergency.target` 和 `systemd-networkd.service` 加入 Target 的 dependency

```
emergency-net.target
       │
       ├── Requires → emergency.target
       │
       └── Requires → systemd-networkd.service
```

`After=` 則建立 ordering，使 `emergency-net.target` 排在這兩個 Units 之後。

```
emergency.target ──────►
                        \
                         emergency-net.target
                        /
systemd-networkd.service ►
```

這個 Example 特別將 `DefaultDependencies=` 納入考量。
由於 `emergency.target` 和 `systemd-networkd.service` 都設定 `DefaultDependencies=no`。

因此它們不會因為 Service 的 default dependencies 而拉入 `sysinit.target`，適合用於這個特殊的 emergency mode。

`AllowIsolate=yes` 允許使用：`systemctl isolate emergency-net.target`
將系統切換到這個 Target。
也可以在 kernel command line 使用：`systemd.unit=emergency-net.target`
讓系統開機時，抵達這個 Target。

其他的 Unit, eg: `foo.service` 則可以透過

```ini
[Install]
WantedBy=emergency-net.target  
```
加入這個 Target 的 wanted dependencies。

啟用後，`foo.service` 會在 `emergency-net.target` 之前啟動。也可以使用 `systemctl add-wants`，在不修改其他 Unit 的情況下，將它加入 Target 的 dependencies。

## 參考來源
[ arch linux 的 systemd.target manual ](https://man.archlinux.org/man/systemd.target.5.en)