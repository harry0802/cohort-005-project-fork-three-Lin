# 站內通知方案研究：Hosted 服務 vs. SQLite + Drizzle + sonner

> 查核日期：2026-11-22。所有價格與 API 細節以當日官方頁面為準；標示「未驗證」者代表官方文件未明確說明。

## 問題

在 React 19 + React Router 8.3（framework mode、`react-router-serve` SSR、單一 Node process）+ SQLite/better-sqlite3 + Drizzle + vitest 的 web app 中，站內通知要選哪條路：

1. Hosted 通知服務：Knock、Novu、MagicBell，外加 SuprSend、Courier。
2. 自己用 Drizzle 存在 SQLite，UI 用已安裝的 sonner（v2.x）。

需求：
- 通知要持久化，並以 toast 顯示；收件者下次進站時 toast **不會自動消失**。
- 同類通知合併為一則（例如「12 new refund requests」）。
- 有 Dismissed 狀態，一旦 dismiss 就不再顯示。
- 由 server 端觸發（React Router action）。
- 透過 navigation 或 polling 更新即可，不需要 WebSocket。
- UI 只有 toast，沒有 bell、inbox 或 feed 列表。
- 部分通知（例如「新的 open refund request」通知 admin）必須在顯示時重新檢查；若該 request 已處理完畢就隱藏。

## 簡短結論

**建議走自建路線：SQLite + Drizzle 存通知，sonner 負責顯示。** 理由如下：

- 這個 app 的需求很小，只有 toast，沒有 inbox。hosted 服務最有價值的部分，例如多通道、inbox UI、realtime 和偏好設定，在這裡用不到。
- 「顯示時重新檢查 refund request 是否已處理」必須查本地 DB。用 hosted 服務時仍要自己寫這段邏輯，等於同一份狀態放在兩個地方。
- 分組可以用一句 `GROUP BY kind` 完成。持久化和 Dismissed 狀態各需一個 table、一個欄位。polling 或 navigation 更新可直接沿用 React Router 的 loader revalidation（[reactrouter.com route-module](https://reactrouter.com/start/framework/route-module)）。
- sonner 原生支援 `duration: Infinity`、用 `id` 更新與去重，以及 `onDismiss`、`onAutoClose`、`action`。這是從已安裝的 v2.0.8 原始碼確認的（見下文）。
- hosted 服務的代價：需要把 user id 和 payload 送到第三方；Knock 的 subprocessor 全部在美國（[knock.app/legal/subprocessors](https://knock.app/legal/subprocessors)）；它們的 React SDK 多以 client 端和 WebSocket 為中心；vitest 也需要 mock 外部 API。

## 比較表

| 方案 | 免費額度／價格 | React 19 / SSR | 分組 | Dismissed 狀態 | Server 觸發 | 送給第三方的資料 | 只用 polling | 本機開發／測試 |
|---|---|---|---|---|---|---|---|---|
| **Knock** | Free：10,000 messages/月；Starter：$250/月含 50k，超出每則 $0.005 ([pricing](https://knock.app/pricing)) | `@knocklabs/react` peer：`react ^17\|\|^18\|\|^19` ([package.json](https://raw.githubusercontent.com/knocklabs/javascript/main/packages/react/package.json))；SSR 未說明 | batch function，可用 `total_activities` ([docs](https://docs.knock.app/designing-workflows/batch-function))；Free 是否可用：未驗證 | `archived_at`、`seen_at`、`read_at` ([docs](https://docs.knock.app/send-notifications/message-statuses)) | `@knocklabs/node` ([npm](https://unpkg.com/@knocklabs/node/package.json))；REST trigger ([docs](https://docs.knock.app/send-notifications/triggering-workflows/api)) | user id，可 inline 帶 name/email，加上 `data` payload；資料在美國 | 可以：`feedClient.fetch()`，不呼叫 `listenForUpdates()` ([README](https://raw.githubusercontent.com/knocklabs/javascript/main/packages/client/README.md)) | 有 development environment ([docs](https://docs.knock.app/concepts/environments))；無 self-host；需自行 mock |
| **Novu** | Free：10,000 workflow runs/月、5 workflows；Pro 起價 $30/月 ([pricing](https://novu.co/pricing)) | `@novu/react` 3.20.0 peer：`react ^18\|\|^19` ([package.json](https://raw.githubusercontent.com/novuhq/novu/next/packages/react/package.json))；SSR 未說明 | digest，可用 `steps.<id>.eventCount`；Free 的 window 最長 1 天 ([docs](https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/digest)) | `archive`、`read`、`seen` ([@novu/js](https://docs.novu.co/platform/sdks/javascript.md)) | `@novu/api` 的 `novu.trigger` | subscriberId，可帶 email/firstName，加上 payload；Free 可選 US/EU ([pricing](https://novu.co/pricing)) | hooks 會自動監聽 realtime ([headless](https://docs.novu.co/platform/inbox/headless-mode.md))；文件未提供關閉 socket 的選項 | Community Edition 可用 Docker self-host ([docs](https://docs.novu.co/community/self-hosting-novu/deploy-with-docker))；有 `npx novu dev` Local Studio |
| **MagicBell** | Free（Builder）：1,000 deliveries/月；Startup：$249/月 ([pricing](https://www.magicbell.com/pricing)) | `@magicbell/react` 1.14.0 未宣告 peerDeps，依賴 `reconnecting-websocket` ([package.json](https://unpkg.com/@magicbell/react/package.json)) | 官方文件找不到 digest/batch（未驗證） | read、seen、archived ([docs](https://www.magicbell.com/docs/primitive/notification)) | Broadcast API，`magicbell-js` ([docs](https://www.magicbell.com/docs/primitive/broadcast)) | external_id、email、first/last name、custom_attributes | 未驗證 | 未找到 test mode 或 self-host |
| **SuprSend** | Free：10,000 notifications/月；batching/digest 從 Business（$250/月）才有 ([pricing](https://www.suprsend.com/pricing)) | `@suprsend/react-core` 2.5.0 peer：`react >=16.8` ([package.json](https://unpkg.com/@suprsend/react-core/package.json)) | 需付費的 Business 方案 | 有 archived 過濾 ([docs](https://docs.suprsend.com/docs/react-sdk-headless-feed)) | Node SDK/REST（細節未驗證） | user + payload；Enterprise 才有 self-host/dedicated | 文件未提供關閉 socket 的選項 | 未驗證 |
| **Courier** | Free（Developer）：10,000 sends/月；Business 每則 $0.005 ([pricing](https://www.courier.com/pricing)) | `@trycourier/courier-react` 9.3.5 peer：`react >=18` ([package.json](https://unpkg.com/@trycourier/courier-react/package.json)) | batch step，可用 user 或 dynamic key ([docs](https://www.courier.com/docs/platform/automations/batching.md)) | archive/unarchive/read ([JS SDK](https://www.courier.com/docs/platform/inbox/web/javascript-sdk)) | REST/Node SDK，JWT 由後端簽發 | userId + payload；EU datacenter 只在付費方案 ([docs](https://www.courier.com/docs/platform/workspaces/eu-datacenter.md)) | 可以：`getMessages()` 不需要 socket ([JS SDK](https://www.courier.com/docs/platform/inbox/web/javascript-sdk)) | 未驗證 |
| **SQLite + Drizzle + sonner** | $0（已安裝） | sonner 2.0.8 peer：`react ^18\|\|^19` ([package.json](https://raw.githubusercontent.com/emilkowalski/sonner/main/package.json))；dist 以 `'use client'` 開頭，並有 `typeof document` guard | 自己寫 `GROUP BY` | 自己加 `dismissed_at` 欄位 | 在 action 內直接 `db.insert` | 不送出任何資料 | root loader 加 `useRevalidator` 即可 ([docs](https://reactrouter.com/api/hooks/useRevalidator)) | 可用 vitest 搭配 in-memory SQLite 直接測 |

## 各方案細節

### Knock

- **價格**：Free 為 $0，每月最多 10,000 messages，不硬性封頂，超量時 Knock 會聯絡你加付款方式。Starter 為 $250/月，含 50,000 messages，超出每則 $0.005（[knock.app/pricing](https://knock.app/pricing)）。
- **React SDK**：`@knocklabs/react` 0.15.2 的 peer 是 `react ^17.0.0 || ^18.0.0 || ^19.0.0`，`next` 與 `@tanstack/react-router` 為 optional peer。它依賴多個 `@telegraph/*` 和 Radix UI 套件，對只要 toast 的需求來說偏重（[package.json](https://raw.githubusercontent.com/knocklabs/javascript/main/packages/react/package.json)）。官方文件沒有寫 SSR 或 React Router 的使用方式（未驗證）。
- **Toast 範例**：官方 toast 範例訂閱 `feedClient.on("items.received.realtime", ...)`，再用 `react-hot-toast` 以 `{ id: notification.id }` 顯示（[docs](https://docs.knock.app/in-app-ui/react/toasts)）。也就是說，官方 toast 模式靠 realtime 推送；「下次進站時仍顯示」需要自己 fetch 未 archived 的項目。
- **Polling**：`@knocklabs/client` 的 `feedClient.fetch({ status, page_size, ... })` 可以單獨使用。`listenForUpdates()` 才會連 realtime socket，因此只用 polling 是可行的。另有 `markAsSeen` 和 `markAsArchived`（[README](https://raw.githubusercontent.com/knocklabs/javascript/main/packages/client/README.md)）。
- **REST feed**：`GET /v1/users/{user_id}/feeds/{id}` 支援 `status`、`archived`、`page_size`，回應的 item 帶有 `total_activities`。文件只寫 client 端用 publishable key 加 user token；能否在 server loader 用 secret key 呼叫，文件未說明（[docs](https://docs.knock.app/api-reference/users/feeds/list_items)，未驗證）。
- **分組**：batch function 在 window 期間收集 activities，可用 batch key 細分，模板可用 `total_activities` 做單複數。window 最長 90 天（[docs](https://docs.knock.app/designing-workflows/batch-function)）。要注意 batch 會在 window 結束後才送出，不是即時累加的計數。
- **狀態**：`seen_at`、`read_at`、`interacted_at`、`archived_at`。archived 對應這裡的 Dismissed（[docs](https://docs.knock.app/send-notifications/message-statuses)）。
- **Server 觸發**：`@knocklabs/node` 1.41.1（[npm](https://unpkg.com/@knocklabs/node/package.json)）。trigger 可帶 `recipients`（可 inline 帶 `name` 和 `email`）、`data`（上限 10MB）、`actor` 和 `cancellation_key`（[docs](https://docs.knock.app/send-notifications/triggering-workflows/api)）。`cancellation_key` 可以在 refund request 被處理時取消尚未送出的 workflow，但無法撤回已送出的訊息。
- **資料駐留**：subprocessor 清單中所有項目（含 AWS）的地點都是 USA，沒有找到 EU 選項（[subprocessors](https://knock.app/legal/subprocessors)）。
- **開發與測試**：每個 environment 有獨立資料與 API key（[docs](https://docs.knock.app/concepts/environments)）。沒有官方 test mode、mock 套件或 self-host；vitest 需自行 mock `@knocklabs/node`（未找到官方指引）。

### Novu

- **價格**：Free 為 10,000 workflow runs/月、5 個 workflows、2 個 environments、activity feed 保留 24 小時，含 US/EU data residency。Pro 起價 $30/月，Team 起價 $250/月。self-host 方案屬於 Enterprise／On-Prem（[novu.co/pricing](https://novu.co/pricing)）。
- **React SDK**：`@novu/react` 3.20.0 的 peer 是 `react ^18.0.0 || ^19.0.0`（[package.json](https://raw.githubusercontent.com/novuhq/novu/next/packages/react/package.json)）。官方文件說 headless hooks 會「automatically listen for real-time events」（[headless-mode](https://docs.novu.co/platform/inbox/headless-mode.md)）。SSR 支援未說明（未驗證）。
- **JS SDK**：`notifications.list({ read, seen, archived, tags })`，以及 `archive`/`unarchive`、`read`、`seen`。EU 使用 `socketUrl: "wss://eu.socket.novu.co"`。文件未提供關閉 WebSocket 的選項（[@novu/js](https://docs.novu.co/platform/sdks/javascript.md)）。
- **分組**：digest step 可用 Regular 或 Scheduled window，以及「Group events by」。Free 方案的 digest window 最長 1 天（[digest docs](https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/digest)）。模板變數 `steps.digest-step.eventCount` 的資訊來自搜尋結果摘要（[docs.novu.co/workflow/digest](https://docs.novu.co/workflow/digest)），本研究未直接打開頁面確認。
- **Server 觸發**：`@novu/api` 的 `novu.trigger({ workflowId, to: { subscriberId, email, firstName, ... }, payload })`；舊版 `@novu/node` 的呼叫方式不同（[docs](https://docs.novu.co/.mintlify/skills/trigger-notification/references/single-trigger-examples.md)，取自搜尋摘要）。
- **Server 端讀取**：舊的 `GET /v1/subscribers/{id}/notifications/feed` 已 deprecated，官方指向 v2（[API ref](https://docs.novu.co/api-reference/subscribers/retrieve-subscriber-notifications.md)）。v2 的參數細節未驗證。
- **Self-host**：Community Edition 採 MIT 授權，用 Docker Compose 部署（[deploy-with-docker](https://docs.novu.co/community/self-hosting-novu/deploy-with-docker)）。self-host 版包含 digest 與 Inbox，但 snooze 等功能只在 Cloud；限制為 2 environments、1 team member（[比較頁](https://docs.novu.co/community/self-hosted-and-novu-cloud)）。self-host 需要哪些基礎設施（MongoDB、Redis），比較頁沒有說明；第三方 Railway 模板建議至少 2 vCPU / 4GB RAM（非官方數字）。
- **本機開發**：`npx novu@latest dev` 會啟動 Local Studio（localhost:2022），並建立 tunnel 讓 Cloud 呼叫本機（取自搜尋摘要，[studio-and-cli](https://docs.novu.co/.mintlify/skills/framework-integration/references/studio-and-cli.md)）。

### MagicBell

- **價格**：Builder 免費，1,000 deliveries/月、1 個 project。Startup 為 $249/月含 50,000，超出每則 $0.0025（[pricing](https://www.magicbell.com/pricing)）。
- **React SDK**：`@magicbell/react` 1.14.0 的 package.json 沒有 peerDependencies，依賴 `reconnecting-websocket`，可見是以 WebSocket 為中心設計（[package.json](https://unpkg.com/@magicbell/react/package.json)）。SDK 需要 User JWT（[docs](https://www.magicbell.com/docs/libraries/react-headless)）。React 19 與 SSR 支援未驗證。
- **狀態**：notification 會記錄 read、seen、archived（[docs](https://www.magicbell.com/docs/primitive/notification)）。
- **分組**：官方文件中沒有找到 digest 或 batching 功能（未驗證／可能不支援）。category 只用於偏好設定和 inbox 分頁（[community docs](https://community.magicbell.io/docs/categories)）。
- **Server 觸發**：Broadcast，recipient 可帶 `external_id`、`email`、`first_name`、`last_name`、`custom_attributes`（[docs](https://www.magicbell.com/docs/primitive/broadcast)）。
- 免費額度最小，也缺少分組功能，不適合這個需求。

### SuprSend（額外候選之一）

- **價格**：Free 為 10,000 notifications/月，in-app 會顯示 SuprSend branding。**Batching/digest 只在 Business（$250/月）以上提供**。self-host 只在 Enterprise（[pricing](https://www.suprsend.com/pricing)）。
- **React SDK**：`@suprsend/react-core` 2.5.0 的 peer 是 `react >= 16.8.0`（[package.json](https://unpkg.com/@suprsend/react-core/package.json)）。舊的 `@suprsend/react-inbox` 已停止開發，改用 JWT user token（[docs](https://docs.suprsend.com/docs/react-inbox-integration.md)）。headless hooks 有 `useFeedClient` 和 `useFeedData`，archived 預設隱藏，但沒有提供關閉 socket 的選項（[docs](https://docs.suprsend.com/docs/react-sdk-headless-feed)）。
- 分組需要付費方案，因此在免費層無法滿足需求。

### Courier（額外候選之二）

- **價格**：Developer 免費，10,000 sends/月。Business 採 pay-as-you-go，每則 $0.005，且從第一則就計費（[pricing](https://www.courier.com/pricing)）。
- **React SDK**：`@trycourier/courier-react` 9.3.5 的 peer 是 `react >=18.0.0`（[package.json](https://unpkg.com/@trycourier/courier-react/package.json)）。
- **Polling**：`inbox.getMessages()`、`getArchivedMessages()`、`getUnreadCounts()` 不需要 `socket.connect()`；WebSocket 只用於 live updates。另有 `archive`/`unarchive`/`read`。JWT 必須由後端用 API key 簽發（[JS SDK](https://www.courier.com/docs/platform/inbox/web/javascript-sdk)）。
- **分組**：batch step 有 Global、User、Dynamic（batch key）三種 scope，在達到最大事件數或 window 結束時送出（[docs](https://www.courier.com/docs/platform/automations/batching.md)）。
- **資料駐留**：EU datacenter（`api.eu.courier.com`）只在付費方案提供（[docs](https://www.courier.com/docs/platform/workspaces/eu-datacenter.md)）。

### 自建：SQLite + Drizzle + sonner

**sonner 能做的**。以下來自官方文件與已安裝的 `node_modules/sonner/dist/index.mjs`（v2.0.8）：

- **不自動消失**：`toast.duration === Infinity` 時不啟動關閉計時器。原始碼註解說明 `setTimeout(, Infinity)` 等同 0，因此特別略過。這個行為由 [PR #373](https://github.com/emilkowalski/sonner/pull/373) 修正。官方 API 表列出 `duration` 預設為 4000（[sonner.emilkowal.ski/toast](https://sonner.emilkowal.ski/toast)）。
- **`id` 更新與去重**：用相同 `id` 再呼叫 `toast()` 時，會合併更新既有 toast，不會新增。所以「3 new refund requests」可以直接用 `id: "refund-request"` 更新成「12 new…」。如果該 id 已被 dismiss，則會建立全新的 toast（見 `index.mjs` 的 create 邏輯）。
- **`onDismiss`**：關閉按鈕、滑動關閉和程式呼叫 `toast.dismiss(id)` 都會觸發 `onDismiss`。因此要把「使用者主動 dismiss」寫回 DB 時，需要區分程式化的 dismiss（例如 revalidate 後 toast 不再需要時），否則會誤記。
- **`onAutoClose`**：只在計時器到期時觸發；`duration: Infinity` 下不會觸發。
- **`action` / `cancel`**：點擊後會關閉 toast 並執行 `onClick`（[docs](https://sonner.emilkowal.ski/toast)）。原始碼顯示 action 關閉時走 `deleteToast()`，**不會**觸發 `onDismiss`；可呼叫 `event.preventDefault()` 讓 toast 保持開啟。若 action 也代表 dismiss，要在 `onClick` 內自行寫回 DB。
- **`dismissible`**：預設為 true。Toaster 的 `visibleToasts` 預設 3，另有 `closeButton`、`expand` 和 `position`（[toaster docs](https://sonner.emilkowal.ski/toaster)）。
- **SSR 與 React 19**：peer 是 `react ^18 || ^19`（[package.json](https://raw.githubusercontent.com/emilkowalski/sonner/main/package.json)）。dist 開頭為 `'use client'`，CSS 注入有 `typeof document == 'undefined'` guard。`toast()` 必須在 client 的 `useEffect` 內呼叫，不能在 render 或 loader 中呼叫。

**sonner 不做的**：持久化、跨裝置狀態、分組、權限、「顯示前重新檢查」。這些都要由 loader 和 DB 負責。

**React Router 8 的相關原語**（docs 顯示的最新版為 8.5.2，專案使用 8.3.0）：

- framework mode + SSR 下，loader 預設會在所有 navigation 和表單送出後 revalidate（[route-module](https://reactrouter.com/start/framework/route-module)）。因此在 root loader 讀取未 dismiss 的通知，navigation 時就會自動更新。
- `useRevalidator()` 官方說明的用途是「Window focus or polling on an interval」，回傳 `revalidate()` 與 `state`（[useRevalidator](https://reactrouter.com/api/hooks/useRevalidator)）。
- `useFetcher()` 可以在不觸發 navigation 的情況下提交（例如 POST `/notifications/dismiss`），也可以用 `fetcher.load` 輪詢某個 resource route（[useFetcher](https://reactrouter.com/api/hooks/useFetcher)）。

**建議資料流**（設計建議，非文件內容）：

1. table `notifications(id, recipient_user_id, kind, subject_id, created_at, dismissed_at)`。
2. 在 action 內與業務寫入放在同一個 transaction 裡 `insert`。
3. root loader 用 `SELECT kind, count(*), max(id) ... WHERE recipient=? AND dismissed_at IS NULL GROUP BY kind` 查詢。針對 `refund_request` 類別，再 join refund 表並過濾 `status = 'open'`，這樣「顯示時重新檢查」只是查詢條件的一部分。
4. client 用 `toast(..., { id: kind, duration: Infinity, onDismiss })` 顯示。
5. dismiss 時用 `fetcher.submit` 把該 kind 中 `id <= maxId` 的列標記為 dismissed，避免誤把 dismiss 後才新增的通知一起標掉。

測試方面，vitest 可直接對 in-memory better-sqlite3 跑 loader 和 action，不需要 mock 任何外部服務。

## 主要風險

**Hosted 路線**
- 狀態重複：refund request 是否已處理的資料在本地 DB，通知狀態在第三方，兩邊需要同步。`cancellation_key` 等機制只能取消尚未送出的通知。
- 分組時機不同：batch 和 digest 是在 window 結束後才送出，與「進站時看到累積計數」的語意不同。SuprSend 的 batching 還需要 Business 方案（$250/月）。
- 隱私與駐留：user id、payload（可能含 email 或姓名）都會送到第三方。Knock 全部在美國；Courier 的 EU 只在付費方案；Novu Free 有 EU。
- SDK 偏向 client 端和 WebSocket；React Router SSR 的用法官方沒有文件。Knock 的 React 套件依賴量大。
- 測試需要 mock 外部 API，或依賴 dev environment 和網路；只有 Novu 能 self-host，但部署相對重。
- 廠商定價和 SDK 變動頻繁，例如 SuprSend 已淘汰舊 inbox 套件，Novu v1 API 也已 deprecated。

**自建路線**
- 分組、dismiss、權限和清理舊資料都要自己實作和測試，不過規模很小。
- sonner 的 `onDismiss` 也會在程式化 dismiss 時觸發，action 點擊則不會，寫回 DB 時容易出錯。
- polling 會在單一 Node process 上增加 SQLite 讀取；查詢有 index 時負擔很小，但輪詢間隔要合理。
- 未來若需要 email、push 或 inbox 等多通道功能，要自己擴充，或屆時再遷移到 hosted 服務。

## 本 app 的建議

採用 **SQLite + Drizzle + sonner**：

- 在 root loader 讀取「未 dismiss、依 kind 分組、並經過業務條件過濾」的通知。
- client 端以 `id = kind`、`duration: Infinity` 顯示 toast。
- dismiss 透過 `useFetcher` 呼叫 action，寫入 `dismissed_at`。
- 更新依靠 navigation 後的自動 revalidation，必要時再用 `useRevalidator` 每 30–60 秒輪詢或在 window focus 時更新。

hosted 服務要等到出現多通道或完整 inbox 需求時再評估。屆時 Knock（API 與文件完整）或 Novu（可 self-host、Free 有 EU）是首選。MagicBell 缺少分組功能，SuprSend 的分組需要付費，兩者不建議。

## 未能驗證的項目

- Knock：batch 是否在 Free 方案可用；server 端用 secret key 讀取 user feed 的方式；React SDK 的 SSR 行為。
- Novu：`eventCount` 變數與 `npx novu dev` 只取自搜尋摘要；v2 feed API 參數；SSR 行為。
- MagicBell：React 19 peer、realtime 機制、是否有 digest。
- SuprSend 和 Courier：Node SDK 細節與測試方式。
- sonner 的 `Infinity`、`onDismiss` 和 action 行為以已安裝的 v2.0.8 原始碼與 PR #373 為準，官方網站 API 表沒有逐條說明。

## 來源

- Knock：https://knock.app/pricing ・ https://docs.knock.app/designing-workflows/batch-function ・ https://docs.knock.app/send-notifications/message-statuses ・ https://docs.knock.app/in-app-ui/react/toasts ・ https://docs.knock.app/api-reference/users/feeds/list_items ・ https://docs.knock.app/send-notifications/triggering-workflows/api ・ https://docs.knock.app/concepts/environments ・ https://knock.app/legal/subprocessors ・ https://raw.githubusercontent.com/knocklabs/javascript/main/packages/react/package.json ・ https://raw.githubusercontent.com/knocklabs/javascript/main/packages/client/README.md ・ https://unpkg.com/@knocklabs/node/package.json
- Novu：https://novu.co/pricing ・ https://raw.githubusercontent.com/novuhq/novu/next/packages/react/package.json ・ https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/digest ・ https://docs.novu.co/workflow/digest ・ https://docs.novu.co/platform/inbox/headless-mode.md ・ https://docs.novu.co/platform/sdks/javascript.md ・ https://docs.novu.co/api-reference/subscribers/retrieve-subscriber-notifications.md ・ https://docs.novu.co/community/self-hosting-novu/deploy-with-docker ・ https://docs.novu.co/community/self-hosted-and-novu-cloud ・ https://docs.novu.co/.mintlify/skills/framework-integration/references/studio-and-cli.md ・ https://docs.novu.co/.mintlify/skills/trigger-notification/references/single-trigger-examples.md
- MagicBell：https://www.magicbell.com/pricing ・ https://www.magicbell.com/docs/libraries/react-headless ・ https://www.magicbell.com/docs/primitive/notification ・ https://www.magicbell.com/docs/primitive/broadcast ・ https://unpkg.com/@magicbell/react/package.json ・ https://community.magicbell.io/docs/categories
- SuprSend：https://www.suprsend.com/pricing ・ https://unpkg.com/@suprsend/react-core/package.json ・ https://docs.suprsend.com/docs/react-sdk-headless-feed ・ https://docs.suprsend.com/docs/react-inbox-integration.md
- Courier：https://www.courier.com/pricing ・ https://unpkg.com/@trycourier/courier-react/package.json ・ https://www.courier.com/docs/platform/inbox/web/javascript-sdk ・ https://www.courier.com/docs/platform/automations/batching.md ・ https://www.courier.com/docs/platform/workspaces/eu-datacenter.md
- sonner：https://sonner.emilkowal.ski/toast ・ https://sonner.emilkowal.ski/toaster ・ https://raw.githubusercontent.com/emilkowalski/sonner/main/package.json ・ https://github.com/emilkowalski/sonner/pull/373 ・ 本機 `node_modules/sonner/dist/index.mjs`（v2.0.8）
- React Router：https://reactrouter.com/api/hooks/useRevalidator ・ https://reactrouter.com/api/hooks/useFetcher ・ https://reactrouter.com/start/framework/route-module
