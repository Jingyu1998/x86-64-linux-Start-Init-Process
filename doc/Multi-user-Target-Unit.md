---
tags: init process
---

# Multi-user Target Unit

## bootup picture

```
                         basic.target
                              |      
                              |
                              |
                              |
                              v
                    (various system  services)
                              |
                              v
                       multi-user.target
```

## 此 target 代表什麼 synchronization point

`multi-user.target` 是 systemd 建立 non-graphical multi-user system 的 synchronization point。

抵達 `multi-user.target` 時，systemd 已經不只是完成「基本 userspace infrastructure」，而是進一步完成了正常多使用者 system 所需要的 services。

## 到達此 target 時，哪些重要 Units 已經完成？

被配置為 multi-user system 所需的 Units，已經按照 dependency + ordering 完成。

一個 service 如果「需要在 multi-user system 中啟動」，它可以在 `[Install]` 中設定：

eg: foo.service
```ini
[Install]
WantedBy=multi-user.target
```

執行 `systemctl enable foo.service` 後，systemd 會建立相應的 `.wants/` symlink。

## 哪些後續 Units 會依賴此 target

`graphical.target` 依賴於 `multi-user.target`
