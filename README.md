# x86-64-linux-Start-Init-Process
## Overview

本專案整理
* linux kernel 如何從 kernel initialization 過渡到 userspace 的 init process 的流程。
* 主流 init process- Systemd 的概念與開機流程
* 研究如何利用 systemd 建立一個比 `multi-user.target` 更小、且能滿足應用服務需求的 system state。

## Handoff

研究系統開機時，linux kernel 如何從 kernel initialization 過渡到 userspace 的 init process 的流程。

[Kernel initialization 交接給 Init process](http://100.71.125.87:3000/43KbrfQ7ShyHqjvfa5eZ8Q)

## Init Process- Systemd

許多主流 Linux 發行版使用 systemd 作為 init process。因此本篇文章整理 systemd 的基本觀念，並建立後續研究 systemd bootup 所需的基礎。

[Systemd 簡介](http://100.71.125.87:3000/wQJTGySKSyu8MwvCFVnxMA)
- description
- unit
- unit type
- Transaction

在 systemd 中，Unit 指的是系統能夠操作和管理的任何 resource。
Unit 是 systemd 使用的主要 Object，用來封裝與系統啟動和維護相關的資訊。
Transaction 是 systemd 為一次 Unit operation 暫時建立的一組 jobs。

## Service and Target Unit

Service Unit 用於啟動並控制 daemon，以及 daemon 所包含的 processes。
Target Unit 用於將多個 Unit 分組，並作為與其他 Unit file 建立 ordering 和 dependency 時的 synchronization point。

Service Unit
* [Service Unit- part 1](http://100.71.125.87:3000/w-E4nBPcQ6iVF7JBSKKxfw)
* [Service Unit- part 2](http://100.71.125.87:3000/cvzyNUqLR46CbKc5-xGBHg)

[Target Unit](http://100.71.125.87:3000/SLVOk7uwTXKlcg0L-KiFdQ)

## Dependency and Ordering

Dependency 指定在 start 某個 unit 時，需要 start 或 stop 其他哪些單元。
Ordering 指定必須遵循的 unit 啟動順序

[Unit Dependency and Ordering](http://100.71.125.87:3000/rzuL5xPATjqoVBgcwgpI5Q)

## Systemd Bootup

研究 systemd system manager 的 bootup 流程，並研究 bootup 流程的重要的 Target Unit

[Systemd Bootup](http://100.71.125.87:3000/ipqWOTp4SqSU4TnmlJKpvg)

重要的 Target Unit
- [ Sysinit Target Unit ](http://100.71.125.87:3000/WIIcQutFQMy1U5fgRcIYyA)
- [ Basic Target Unit ](http://100.71.125.87:3000/yiA53_5xTnioWVEFDoa6Ww)
- [ Multi-user Target Unit ](http://100.71.125.87:3000/3mVAb06-Qm6biyDjMwrCrw)

## Hedgedoc 可用

本專案以部署於 Docker 中的 HedgeDoc 為實際案例，研究如何利用 systemd 建立一個比 `multi-user.target` 更小、且能滿足應用服務需求的 system state。

研究首先從定義「HedgeDoc 可用」開始，分析 HedgeDoc 所依賴的 `docker.service` 與 `tailscaled.service`，並研究兩者在 systemd 中的 dependency 與 ordering。接著，以 systemd bootup flow 中的 `basic.target` 作為較早的 boot synchronization point，分析其 dependency graph 能涵蓋哪些 HedgeDoc 所需的 units，進而找出 `basic.target` 之後仍需要補足的 dependencies。

在確認 HedgeDoc 所需的最小服務集合後，進一步建立 `hedgedoc.target`，將這些 units 組合成一個獨立的 system state。實作過程透過多個版本的 `hedgedoc.target` 進行實驗，除了讓 HedgeDoc 可以在未 reached `multi-user.target` 的情況下使用，也補足 Linux user login 與 shell 操作所需的 units。

最後，將 `hedgedoc.target` 納入正常的 systemd boot flow，並透過 `AllowIsolate=yes` 使其可以成為 `systemctl isolate` 的目標，驗證 systemd 可以在不同 system state 之間切換。

細節參考以下文件:
* [研究 hedgedoc 可用所需要的最小服務集合](http://100.71.125.87:3000/82B5A7p5S9WEQqsPnUq91A)
* [實作 HedgeDoc 可用的最小 Boot State](http://100.71.125.87:3000/fryK7sW-Ss6zY_jesQj_rw)
