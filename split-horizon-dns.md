# Background 
- 網路一片平坦，要如何在 100.0.0.0/8 切分不同 zone, 並可以透過 zone 方式控制 network policy.
- 透過 gateway 方式卡控 zone policy.
- 使用 spiffe 身份驗證取代 CIDR 來當作 zone policy 卡控。
- 透過 split-horizon dns 方式，避免 client 繞過 gateway 直接跨 zone 溝通。

# 題目
- Split-horizon dns 如何實作

# 必要條件
- 減少 client 異動
- 架構簡單
- 達到目的

# 實作方式

## 透過 SPIFFE 身份來決定 IP
困難點
- DNS 不管 TCP 或 UDP 都無法傳遞身份資訊
- 標準 DNS(UDP/53) 做不到 —— DNS query 本身沒有任何 client 身分憑證,只有來源 IP。要讓 DNS 端「知道」client 的 SPIFFE ID,唯一可靠途徑是 DNS over TLS/HTTPS + mTLS,由 client 出示 X.509-SVID,server 從憑證 SAN 的 spiffe://... URI 取出身分。
- client 需要安裝 spire agent 增加導入難度
