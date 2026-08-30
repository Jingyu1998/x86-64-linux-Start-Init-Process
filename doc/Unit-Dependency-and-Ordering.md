---
tags: init process
---

# Unit Dependency and Ordering

systemd 有兩種類型的 dependency ：requirement dependency 和 order dependency。 

requirement dependency 指定在 start 某個 unit 時，**需要 start 或 stop 其他哪些單元。** 
order dependency 指定必須遵循的 unit 啟動順序

> requirement dependency 又稱 dependency
> order dependency 又稱 ordering


## requirement dependency

Unit file 具有 requirement dependency 特性。某個 Unit 可能 want or require 一個或多個其他單元運作之後才能運作。 這些 requirement dependency 是在 Unit file 中使用指令 `Wants` 和 `Requires` 設定的。

`Wants`
* `Wants=` 用來建立對其他 Units 的 **weak** requirement dependencies。

* `Wants=` 只建立 requirement dependency，不建立 start 或 stop 的 ordering dependency。若需要建立 ordering dependency，必須另外使用 `After=` 或 `Before=`。

eg: my.service
```ini
[Unit]
Wants=foo.service
```

表示 `my.service` 需要 `foo.service` 也被 start。
當 `my.service` 被 start 時，systemd 也會嘗試 start `foo.service`

但是 `Wants=` 的 requirement dependency 是 weak 的。
代表 `foo.service` 的 start 失敗不會使 `my.service` start 失敗。
因為沒有設定 ordering，所以 `my.service` 、 `foo.service` 可以同時開始啟動。

----

`Requires`
* 如果 Unit A 設定了 `Requires=unit B`，則這兩個 Unit 都會運行；但如果 Unit B 未成功 run，則會停用 unit A。 Unit A 的 process是否成功運作並不重要。

`Requires`
* `Requires=` 用來建立對其他 Units 的 stronger requirement dependency。
* `Requires=` 的 Stop 傳播。

eg: my.service
```ini
[Unit]
Requires=foo.service
```

表示 `my.service` 需要 `foo.service` 也被 start。
當 `my.service` 被 start 時，systemd 也會嘗試 start `foo.service`
當使用者主動 stop `foo.service` 時，systemd 會去 stop `my.service`

`Requires` + `After`

eg: my.service
```ini
[Unit]
Requires=foo.service
After=foo.service
```

`Requires=` 本身是 requirement dependency；`After=` 則決定啟動順序，而且這兩者一起使用時，`foo.service` activation failure 會阻止 `my.service` 啟動。

## ordering dependency 

如果沒有其他的指令，systemd 可以同時執行一組 Unit。 以正確的順序啟動服務對於 Linux 系統的正常運作非常重要。 可以使用 Unit file 指令 `Before` 和 `After` 來排列順序。

`Before=` / `After=` 只負責「ordering」，不負責「要不要啟動」。

`Before`
* 例如，如果 Unit A 設定了 `Before=unit B`，當這兩個 Unit 同時 start 時，會先完全執行 Unit A，然後再執行 Unit B。

`After` 
* 如果 Unit A 設定了 `After=unit B`，當這兩個 Unit 同時運作時，會先完全執行 Unit B，然後再執行 Unit A。

---

`Before=` / `After=` 不會啟動另一個 Unit

eg: my.service
```ini
[Unit]
After=foo.service
```

不代表 systemd 會因為 `my.service` 這個設定而啟動 `bar.service`。

它只表示: **如果 `my.service` 和 `foo.service` 都需要被啟動**，**`my.service` 必須後於 foo.service。**

如果這次 transaction 裡根本沒有 `bar.service`，那麼 `After=` 不會把它拉進來。

---

`Before=` / `After=` 不只影響 startup，也影響 shutdown。

eg: my.service
```ini
[Unit]
After=foo.service
```
**如果 `my.service` 和 `foo.service` 都需要被關閉**，**`my.service` 必須先於 foo.service。**

## reference

[ SUSE linux 的 systemd manual 英文版](https://documentation.suse.com/sle-micro/6.0/html/Micro-systemd-basics/index.html#concept-unit-dependencies)
[ SUSE linux 的 systemd manual 簡中版](https://documentation.suse.com/zh-cn/sle-micro/6.0/html/Micro-systemd-basics/index.html#concept-unit-dependencies)
[ arch linux 的 systemd.unit manual ](https://man.archlinux.org/man/systemd.unit.5.en)