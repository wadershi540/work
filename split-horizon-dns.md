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
- 不使用 CIDR 相關的 solution
- 不影響既有的查詢紀錄跟功能

# 實作方式

## 透過 SPIFFE 身份來決定 IP
# 困難點
- DNS 不管 TCP 或 UDP 都無法傳遞身份資訊
- 標準 DNS(UDP/53) 做不到 —— DNS query 本身沒有任何 client 身分憑證,只有來源 IP。要讓 DNS 端「知道」client 的 SPIFFE ID,唯一可靠途徑是 DNS over TLS/HTTPS + mTLS,由 client 出示 X.509-SVID,server 從憑證 SAN 的 spiffe://... URI 取出身分。
- client 需要安裝 spire agent 增加導入難度
- 每台 VM 要裝 agent、改 resolv.conf、處理 TCP-only DNS

# 方案
A. mTLS DoT/DoH + identity-aware DNS(最貼近你的原始想法)
Code
SPIRE server + agent 發 SVID 給每個 workload
前端用 Envoy 終結 TLS、驗 trust domain,把 SPIFFE ID 以 DoH header 或自訂 EDNS0 option 傳給後端
CoreDNS 用 metadata + view plugin 依 SPIFFE ID 選 zone;複雜規則就自己寫 plugin
B. 用 IP↔identity 對照表做 split-horizon(最快落地)
不改 client,DNS 仍走 53,但用 workload registry 把來源 IP 對回身分,再套 CoreDNS view。缺點:IP 可偽造、K8s 出去可能已 SNAT、Pod IP 會回收再配。適合當過渡。
C. 不在 DNS 做,改回 per-tenant VIP(我建議跟你的 zone gateway 結合)
DNS 回一個依 tenant/身分區分的 VIP,zone gateway 上對每個 VIP 綁定「只接受哪些來源/哪個 SPIFFE ID」。DNS 負責分流,gateway 負責執法 —— 這樣即使 DNS 答案外洩也不會破功。
