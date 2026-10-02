# 自架即時通訊軟體 — 專案 Roadmap

## 專案背景與需求

- **場景**：公司內部／小團體使用
- **資料自主權**：自架（self-hosted），資料不落第三方
- **身分驗證**：支援 OIDC，可對接公司既有 IdP（如 Keycloak、Authentik、Entra ID）
- **平台**：電腦（Web/桌面）＋ 手機（iOS/Android）
- **功能需求**：語音／視訊通話、檔案傳輸與共享、端到端加密（E2EE）、機器人／Webhook 整合
- **現況**：概念驗證／興趣研究階段，尚未決定是否正式投入

## 方案比較結論

比較了 Matrix、Rocket.Chat、Mattermost、Zulip 四個開源方案在 E2EE、語音視訊、OIDC、Bot/Webhook 的支援程度：

| 方案 | E2EE | 語音/視訊 | OIDC | Bot/Webhook |
|---|---|---|---|---|
| Matrix | ✅ 原生成熟 | ✅ Element Call (LiveKit) | ✅ (MAS) | ✅ |
| Rocket.Chat | ⚠️ 較陽春 | ✅ | ✅ | ✅ |
| Mattermost | ❌ | ✅ | ✅ | ✅✅ |
| Zulip | ❌ | ⚠️ 僅靠 Jitsi 連結 | ✅ | ✅ |

E2EE 是關鍵篩選條件，Mattermost 與 Zulip 因核心不支援而出局。**最終選定 Matrix（Synapse + Element）** 為主要方向；Rocket.Chat 是「E2EE 較弱但更好上手」的備選方案。

## 目標架構（完整版）

- **Homeserver**：Synapse（功能最完整、生態最成熟，優於 Dendrite/Conduwuit）
- **身分驗證**：Matrix Authentication Service (MAS) 作為 OIDC provider，對接公司既有 IdP
- **資料庫**：PostgreSQL
- **媒體儲存**：S3 相容物件儲存（MinIO），需訂定保留政策
- **語音視訊**：Element Call，基於 MatrixRTC，需自架 LiveKit SFU + coturn（TURN server）
- **聯邦（Federation）**：建議關閉，封閉為內部系統，降低攻擊面
- **客戶端**：Element X（手機）、Element Desktop（電腦）
- **推播**：Sygnal → APNs/FCM（僅傳遞「有新訊息」的中繼資料，內容仍為 E2EE 加密）
- **Bot/Webhook**：Application Service API + Hookshot（GitHub/GitLab/Jira 等）／matrix-bot-sdk

## 成本評估

- **軟體**：全部開源免費，無授權費
- **基礎設施**：主機、頻寬（語音視訊最吃流量）、儲存空間、備份
- **維運人力**：Matrix 生態更新頻繁，目前正處於 MAS / Sliding Sync 遷移過渡期，需持續 follow
- **可選付費**：Element Enterprise（企業支援，非必要）、Apple Developer Program $99/年（僅客製品牌上架 App Store 才需要）
- **結論**：PoC 階段中等規格 VPS 即可負擔；語音視訊多人測試時頻寬成本才會明顯浮現

## 推進計畫（Phased Rollout）

### Phase 1（已完成）：最小可行 PoC

- Docker Compose：Synapse + PostgreSQL + Element Web
- 不含 OIDC、不含語音視訊，先驗證基本聊天體驗與管理介面
- `server_name: ourchat.local`，封閉測試環境，不開聯邦

### Phase 2（已完成）：加入 OIDC

- 部署 Matrix Authentication Service (MAS)，Synapse 委派登入給 MAS（`matrix_authentication_service` 設定區塊 + `password_config.enabled: false`）
- 部署測試用 Keycloak（realm `ourchat`），MAS 以 `upstream_oauth2` 對接，`claims_imports` 把 Keycloak 的 `preferred_username` 對應成 Matrix localpart，首次登入自動建立帳號
- 已用瀏覽器（chrome-devtools）實際走過完整流程驗證成功：Element Web → MAS → Keycloak 登入 → 自動建立 `@alice:ourchat.local` → 回到 Element Web 完成登入

**過程中踩到的坑（供 Phase 3/4 參考）：**

1. **MAS 容器的 hostname 解析壞掉**：MAS（Rust 實作）沒辦法透過 Docker Compose 的服務名稱（如 `db`、`synapse`、`keycloak`）連線，會報 `Connection refused`，但用 IP 直接連就正常。解法是幫這幾個服務在自訂 network 上釘死固定 IP（`docker-compose.yml` 裡的 `networks.ourchat.ipam`），MAS 的設定一律用 IP 而非 hostname。Phase 3 接 LiveKit 時要留意同樣的狀況。
2. **MAS 預設拒絕 HTTP 的 client 註冊**：Element Web 走 OAuth Dynamic Client Registration 時，MAS 的政策引擎預設只接受 HTTPS（或原生 App 的 loopback 例外，Web 類型不算在內），本機測試需要在 `mas-data/config.yaml` 加上 `policy.data.client_registration.allow_insecure_uris: true` 才能過。**正式環境不該開這個，應該直接上真的 HTTPS。**
3. **Keycloak 的 issuer 需要釘死**：沒設定 `KC_HOSTNAME` 時，Keycloak 會依請求的 Host header 動態回傳不同的 issuer 網址，導致瀏覽器端（`localhost:8082`）跟 MAS 後端（走內部 IP）看到的 issuer 不一致，驗證失敗（`could not validate claim "iss"`）。解法是明確設定 `KC_HOSTNAME`/`KC_HOSTNAME_PORT`/`KC_HOSTNAME_STRICT_HTTPS`，讓 issuer 固定為單一網址；`token_endpoint`/`jwks_uri` 等實際連線端點可以照樣用內部 IP（`discovery_mode: disabled` 手動指定）。
4. MAS 的 upstream OAuth2 callback 路徑格式是 `/upstream/callback/{provider_id}`（不是 `/upstream/oauth2/callback/{id}`），註冊 IdP client 時 redirect URI 要對齊這個路徑。
5. 啟用 MAS 委派後，Synapse 原生密碼登入會整個關閉，Phase 1 用 `register_new_matrix_user` 建立的舊帳號（`admin`）就登不進去了。正式遷移真實使用者時需要用 MAS 內建的 `syn2mas` 工具做帳號搬遷，這次 PoC 因為只是測試帳號所以直接跳過。

### Phase 3：加入語音視訊

- 部署 LiveKit SFU + coturn
- 整合 Element Call

### Phase 4：正式化準備

- 媒體儲存遷移至 S3/MinIO
- 備份策略（PostgreSQL + 媒體 + E2EE key backup）
- 手機推播（Sygnal + APNs/FCM）
- Bot/Webhook 整合（依實際需求，如 CI/CD 通知）
- 評估是否需要 Worker 模式做水平擴展／K8s 部署

## 待決事項

- 手機 App 發布方式：官方 Element X（免費，快）vs 客製品牌（需 Apple Developer 帳號）
- 是否需要 Worker 模式／K8s 部署（視實際使用規模）
- 內部 IdP 選型：確認會是公司共用的正式 IdP（跟信箱、VPN 等系統一起用），非聊天系統專用
- **IdP 是登入唯一入口的風險**：啟用 MAS 委派後 Synapse 原生密碼登入整個關閉，代表 IdP 掛了 = 沒人能新登入。因為是公司共用 IdP，HA/SLA 是組織既有責任、不是聊天系統獨有的新風險，但正式部署前仍需跟 IdP 維運團隊確認可用性等級。聊天系統自己可控的緩解手段：把 MAS 的 access/refresh token 效期調寬，讓 IdP 短暫中斷不會把已登入的使用者一起踢出（已登入者的請求驗證是問 MAS 本地資料庫，不會每次都重新連 IdP，只有新登入／refresh token 過期後才需要 IdP 活著）
