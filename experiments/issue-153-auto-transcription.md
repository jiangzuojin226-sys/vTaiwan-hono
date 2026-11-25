# 實驗日誌：#153 即時會議逐字稿自動分軌錄製

對應 upstream issue：https://github.com/g0v/vTaiwan-hono/issues/153
實作位置：fork `jiangzuojin226-sys/vTaiwan-hono`

這份日誌有兩個目的：

1. 記錄做了什麼、為什麼這樣做、卡在哪裡。做不完也能讓下一個人接手。
2. 寫給第一次接觸後端開發的人看。每個段落分三層：**做了什麼** → **為什麼** → **這是什麼意思**（名詞解釋用 💡 標示）。

---

## 先弄懂這個系統在做什麼（背景知識）

vTaiwan 開線上會議時，網頁的 `/jitsi` 頁面會做三件事：

1. 用 iframe 嵌入 8x8 公司的 Jitsi 視訊會議。
2. 有人按下「紫色麥克風」，瀏覽器就開始錄**他自己**的聲音，每 30 秒切一段，上傳到我們自己的伺服器。
3. 伺服器把音檔丟給 AI 模型（Whisper）轉成文字，存進資料庫，再透過 WebSocket 即時推給所有與會者看。

💡 **伺服器端在哪裡？** 這個專案沒有自己的機器，程式跑在 Cloudflare 的「Workers」上。你可以想成：把程式碼交給 Cloudflare，它在全世界的機房幫你執行，每次有人連進來就跑一次。

💡 **資料庫 D1**：Cloudflare 提供的資料庫，底層是 SQLite。資料存成一張張「表」，每張表像 Excel 工作表，有欄（column）和列（row）。

💡 **Durable Object（DO）**：Cloudflare 的一種特殊程式。一般 Worker 每次執行都是獨立的、不記得上次的事；DO 則是「一個名字對應一個長期存在的實例」。這個專案用「一個會議日期一個 DO」，所有人的 WebSocket 連線都接到同一個 DO 上，它負責收訊息、寫資料庫、再廣播給所有人。可以想成「這場會議的櫃台人員」。

💡 **WebSocket**：一般網頁是「你問一句、伺服器答一句」（HTTP）。WebSocket 是打開一條一直連著的線，伺服器隨時可以主動推訊息給你。即時逐字稿要讓所有人同時看到新句子，就靠這個。

💡 **主鍵（primary key）**：資料表裡用來「唯一識別一列」的欄位。逐字稿表的主鍵是 `(meeting_date, ts)`，意思是「同一天不能有兩筆時間戳一樣的條目」。

💡 **時間戳（timestamp）**：用「從 1970 年 1 月 1 日到現在經過幾毫秒」這個數字表示時間，例如 `1763539200000`。電腦比字串好處理、好排序。

---

## 2026-11-19 步驟 1：時間戳改成「開始說話的時間」，並避免互相覆蓋

分支：`feat/transcript-timestamp-fix`（從 `main` d40dfba 開出，與 upstream/main 同步）

💡 **分支（branch）**：Git 裡的「平行宇宙」。在分支上改東西不會影響 `main`，改好、測好再合併回去。命名 `feat/...` 是慣例，代表「新功能」。

### 1. 讀程式碼，核對 issue 裡的假設

**做了什麼**：用 `grep` 搜關鍵字、`sed -n` 印出特定行數，逐一對照 issue 寫的是不是真的。

**為什麼**：提案是「推論」出來的，動手前要確認每一句話都對得上程式碼，否則改錯地方。

| issue 的說法         | 程式碼位置                                                                       | 結果 |
| -------------------- | -------------------------------------------------------------------------------- | ---- |
| 固定 30 秒切段       | `JitsiView.vue` `maxRecordingTime: 30 * 1000`                                    | ✅   |
| 時間戳是轉錄完成時間 | `sendAudioToTranscription()` 在 fetch 回來後才 `timestamp: Date.now()`           | ✅   |
| 兩筆可能互相覆蓋     | `meeting-room.ts` `save_entry` 用 `INSERT OR REPLACE`，主鍵 `(meeting_date, ts)` | ✅   |
| 只記一位錄音者       | `meeting_sessions.recording_speaker` 單一欄位                                    | ✅   |
| 轉錄模型             | `src/lib/transcribe.ts` `@cf/openai/whisper-large-v3-turbo`                      | ✅   |

💡 **`INSERT OR REPLACE`**：SQL 指令。「插入一列，如果主鍵已經有人用了，就把那列整個換掉」。方便，但危險：它不會告訴你蓋掉了誰。

💡 **為什麼 `Date.now()` 放錯位置是 bug？** `Date.now()` 回傳「現在」。它寫在「等 AI 轉錄完成之後」，所以記到的是轉錄完成的時間，不是開口說話的時間。音檔排隊越久、AI 越慢，誤差就越大。

### 2. 讀的時候發現、原本沒想到的事

這一段最值得學：**計畫寫得再好，真正讀程式碼時一定會冒出新問題。**

- **編輯和新增走同一條訊息。** `TranscriptPanel` 編輯條目後 emit `update-data`，`JitsiView.updateTranscriptData()` 也是送 `save_entry`，靠 `INSERT OR REPLACE` 覆蓋同一個 ts。所以「撞到 ts 就順延」不能一視同仁，否則編輯會變成新增一筆重複的。
  → **決定**：客戶端新增時帶 `create: true`，編輯不帶。DO 看旗標決定「找空位」或「原地更新」。舊版客戶端不帶旗標 → 走原地更新、找不到就新增，行為跟現在一樣。

  💡 **向下相容（backward compatible）**：部署新版伺服器時，有些人瀏覽器還開著舊版網頁。新伺服器要能看懂舊網頁送來的訊息，不能讓他們壞掉。

- **不能用 entry_id 判斷是不是編輯。** Jitsi 字幕條目的 `id` 可能是 null，歷史資料也可能 entry_id 為 null，用 id 比對會誤判。所以才改用上面的旗標。

- **D1 呼叫不受 DO input gate 保護。** 兩個 `save_entry` 可能在 SELECT 與 INSERT 之間交錯。所以不做「先 SELECT 找空位再 INSERT」，直接 `INSERT OR IGNORE`，`changes === 0` 就代表撞到主鍵，ts + 1 再試。

  💡 **競態條件（race condition）**：兩件事幾乎同時發生、順序不確定，導致結果出錯。例：A 查「1000 這個位置空著」，B 也查「1000 空著」，兩人都寫進 1000，其中一人被蓋掉。解法是用「一步到位」的原子操作：`INSERT OR IGNORE` 是「有空位就寫、沒空位就什麼都不做並回報」，中間不會被插隊。

  💡 **`changes`**：D1 執行完 SQL 後回報「這次改了幾列」。`INSERT OR IGNORE` 回 0 就代表沒寫進去（位置被占了）。

- **歷史日期走 REST 不走 DO。** `/api/meeting/:date/transcript` 也是 `INSERT OR REPLACE`，要一起改，否則同一個 bug 留一半。兩邊共用 `src/server/lib/meeting-entries.ts`。

  💡 **REST**：一般的 HTTP API，「送一個請求、拿一個回應」。今天的會議用 WebSocket 即時同步；校對過去日期的逐字稿就用 REST 一次讀寫，不用一直連著。

  💡 **DRY（Don't Repeat Yourself）**：同一段邏輯只寫一次、兩邊共用。寫兩份的話將來修 bug 一定會忘記改其中一份。

- **樂觀更新會留下舊 key。** 客戶端新增時先把條目放進 `transcriptData[requestedTs]`，若伺服器順延成 ts+1 再廣播回來，本機會同時有 `requestedTs` 和 `ts+1` 兩筆。
  → `entry_saved` 訊息多帶 `replaces: requestedTs`，客戶端收到就把舊 key 刪掉。其他客戶端本來就沒有那個 key，刪了無副作用。

  💡 **樂觀更新（optimistic update）**：先在畫面上顯示「已完成」，再等伺服器確認。好處是感覺很快；壞處是伺服器如果改了什麼，要有辦法修正畫面。

- **TranscriptPanel 用 `Object.values()` 顯示。** JavaScript 對「看起來像整數的字串 key」會自動按數值排序，所以時間戳一改成開始說話時間，順序就自動對了，不用另外排序。這是 JS 的一個冷知識，剛好幫了我們。

- **`speaker_uid` 由伺服器填。** WebSocket 連線在 upgrade 時已把 `userId` 綁在 attachment 上，REST 也有 `authCtx.user.id`。不信任客戶端送來的 uid。

  💡 **為什麼不信任客戶端？** 網頁的程式碼在使用者電腦上跑，任何人都能打開開發者工具改掉送出的資料。所以「我是誰」這種事一律由伺服器根據登入狀態判斷，不看客戶端自己宣稱什麼。這是後端安全的基本原則。

- **測試環境有 `cloudflare:workers` 的 shim**（`src/shims/cloudflare-workers.ts`），所以 DO 和 server lib 可以在 vitest 裡用假的 D1 直接測。

  💡 **shim**：一個「假的替身」。真的 `cloudflare:workers` 只有在 Cloudflare 上才有，本機跑測試時沒有，就放一個長得一樣但什麼都不做的替身讓程式能載入。

### 2.5 測試抓到一個我自己設計上的 bug（這段最值得看）

**發生什麼事**：第一版我讓伺服器把 `entry_saved` 連同 `replaces: 1000` **廣播給所有人**。寫測試時模擬這個情境：

1. Alice 的條目已經正當占用 ts=1000。
2. Bob 同一毫秒新增，被伺服器順延到 1001，廣播 `{ entry: Bob@1001, replaces: 1000 }`。
3. Alice 的瀏覽器收到後，照 `applyEntrySaved` 的規則把 key 1000 刪掉。**Alice 自己的條目從畫面上消失了。**

測試在 `expect` 那行紅燈，我才發現 `replaces` 的語意是「**你**樂觀放在 1000 的那筆要換掉」，這句話只對 Bob 成立，對 Alice 不成立。

**怎麼修**：伺服器改成兩種訊息。

- 給其他所有人：`{ type: 'entry_saved', entry: Bob@1001 }`（不帶 replaces）。
- 只給 Bob 這條連線：`{ type: 'entry_saved', entry: Bob@1001, replaces: 1000 }`。

`broadcast()` 多一個 `except` 參數跳過送出者，再用 `send()` 單獨回給他。

**學到什麼**：

- 「廣播」和「回覆」是兩種不同的訊息。只有送出請求的人才知道自己剛才樂觀地放了什麼，所以修正指令只能給他。
- 寫測試不是「驗證我寫對了」，而是「逼自己把每個角色的視角都走一遍」。這個 bug 在單人測試永遠不會出現，只有「第三個人在旁邊看」的情境才會。
- 這就是為什麼 issue 裡說要「單機、雙機、三人以上」三輪實測。

💡 **為了不讓這個錯誤回來**，加了 `src/tests/meeting-room.test.ts`：直接建構 Durable Object（靠 shim），用假的 WebSocket 和假的 D1 模擬三條連線，斷言「只有送出者收到 replaces」。以後有人改回廣播，CI 會紅燈。

### 2.7 考古：原先的程式為什麼不記開口時間（2026-11-19 晚上補）

修完之後回頭問：當初為什麼會這樣寫？看了 hono 第一版（Firebase 時代）、原站 `vue.vTaiwan-neo` 的 git 歷史和對應 issue，整理如下。

**演化時間線（neo）**

| 日期       | commit / issue | 發生什麼                                                                                                                 |
| ---------- | -------------- | ------------------------------------------------------------------------------------------------------------------------ |
| 2025-06-16 | ee7e5b6，#28   | 先建 Jitsi 會議元件，逐字稿還沒做                                                                                        |
| 2025-06-21 | e75f7da        | 接 JaaS 的**英文字幕**。字幕事件來一句存一句，`timestamp: new Date().getTime()` 第一次出現                               |
| 2025-06-30 | 00b6972，#42   | 中文字幕等不到客服，改成**自己錄音上傳 Whisper**。直接複製字幕那段的寫法：轉錄回來後 `timestamp: Date.now()`。每段 10 秒 |
| 2025-07-09 | #44            | 使用者反映太短想調 2 分鐘，最後折衷 30 秒                                                                                |
| 2025-08-17 | a074e00，#85   | 改成「按一下開始、按一下停止」                                                                                           |
| 2025-08-17 | e7f7424，#87   | 轉錄中也能按停，音檔**排隊**延後上傳                                                                                     |
| 2026-07-15 | hono 3732066   | 整段原樣搬進 vTaiwan-hono                                                                                                |
| 2026-08    | hono #81       | Durable Object + D1 取代 Firebase，timestamp 變成資料表主鍵                                                              |

**四個推論**

1. **timestamp 一開始的用途是「當 key」，不是「記時間」。** 最早的字幕版本把每句存成 `transcriptData[timestamp]`，timestamp 的功能是給這句一個不重複的編號。字幕幾乎即時到達，「收到」和「說話」差不到一秒，沒有人需要區分兩者。
2. **錄音功能是從字幕程式碼複製來的，沿用了同一行。** 6/30 的 commit 是為了繞過 JaaS 中文字幕問題的實驗，`sendAudioToTranscription` 結尾和字幕的 `commitTranscriptCache` 幾乎一樣。那時每段只有 10 秒、轉錄完立刻顯示，晚幾秒在畫面上看不出來。
3. **當時沒有排隊，延遲不會累積。** 6/30 版本是錄完一段、送出、等結果、再錄下一段。到 8/17 #87 加佇列後誤差才會隨排隊變長。那次 commit 其實在佇列項目存了 `timestamp: new Date().toISOString()`，代表作者意識到「這段音檔是什麼時候的」，但只拿來印 log，**沒有接到條目上**。資料有了、線沒牽。
4. **Firebase 時代不存在「覆蓋」問題，timestamp 不需要精確。** hono 第一版的 `updateMeetingData()` 是把**整個** `transcriptData` 一次 `set` 到 `/meetings/{日期}/transcripts`，每次有新句子就重寫整本，誰最後寫誰贏。timestamp 只要在同一個人的瀏覽器裡不重複就好。#81 換成 D1 後，timestamp 升格成主鍵 `(meeting_date, ts)` 並用 `INSERT OR REPLACE` 逐筆寫，但產生它的那行沒跟著升級。這次修的就是這個落差。

💡 **這是很常見的 bug 成因**：一個欄位原本只有 A 用途（編號），系統演化後多了 B 用途（排序、唯一鍵），但產生它的程式還停在 A 的假設上。每一步單獨看都合理，合起來就出問題。沒有人做錯，只是換底層時沒人回頭看這一行。

💡 **一人錄音永遠看不出來**：單人時條目依序產生，就算時間晚幾秒，順序仍對。要兩人同時錄、其中一人的音檔在排隊，順序才會錯。

💡 **怎麼考古**：`git log -S '某段字串' -- 檔案` 可以找出「這段字串是哪次 commit 加進來的」；`git show <commit>:<路徑>` 可以看當時整個檔案長什麼樣。配合 issue 編號就能還原當初的決策脈絡。

### 3. 改動清單

- `migrations/0004_add_transcript_entry_provenance.sql`：加 `speaker_uid`、`source` 兩欄。

  💡 **migration**：資料庫結構的「變更紀錄」。每個檔案是一次改動（加欄位、建表），依編號順序執行。這樣所有環境（本機、正式）的資料庫結構才會一致，也能追溯誰在什麼時候改了什麼。

- `src/lib/transcript-entries.ts`（新，前後端共用純邏輯）：`TranscriptEntry` 型別、`normalizeTranscriptSource()`、`applyEntrySaved()`。

  💡 **純函式（pure function）**：給同樣輸入一定得到同樣輸出、不碰外界（不讀資料庫、不碰網頁）的函式。最好測試，因為不需要準備環境。這個專案的規矩是：能抽成純函式的邏輯就抽出來放 `src/lib/`。

- `src/server/lib/meeting-entries.ts`（新，只在 Worker）：`insertTranscriptEntry()` 找空位寫入、`updateTranscriptEntry()` 原地更新、找不到退回新增。
- `src/durable-objects/meeting-room.ts`：`save_entry` 改用上面兩個函式；`entry_saved` 多帶 `replaces`；讀取時回傳 `speakerUid`、`source`。
- `src/server/api/meeting.ts`：REST 新增／編輯比照 DO；回應帶實際寫入的 `timestamp`。
- `src/views/JitsiView.vue`：
  - 每輪錄音開始時記 `startedAt`，跟著音檔進 `audioQueue`，轉錄完成後用它當 `timestamp`。
  - 新增條目送 `create: true`，編輯不送。
  - 錄音條目 `source: 'manual'`，Jitsi 字幕條目 `source: 'jitsi'`。
  - `entry_saved` 用 `applyEntrySaved()` 處理 `replaces`。
- 測試：新增 `transcript-entries.test.ts`、`meeting-entries.test.ts`；調整 `audio-recording.test.ts`。

### 4. 部署時要注意

- **先套 migration 再部署程式。** 新程式的 INSERT 會寫 `speaker_uid`／`source`，欄位不存在會 500。
  ```
  vp exec wrangler d1 migrations apply vtaiwan-transcriptions --remote
  ```
  本機開發同理（`--local`）。
- 舊版前端（部署前已開著的分頁）送的 `save_entry` 不帶 `create`，會走原地更新→找不到就新增，行為與現在一致。

### 5. 驗收結果（2026-11-19 18:55）

依 AGENTS.md 的四道關卡：

| 關卡               | 指令                 | 結果                                                       |
| ------------------ | -------------------- | ---------------------------------------------------------- |
| 型別 + lint + 格式 | `vp check`           | ✅ 160 檔格式正確，143 檔無錯                              |
| 自動測試           | `vp test`            | ✅ 32 個測試檔、322 個測試全過（新增 3 檔 + 2 個案例）     |
| 建置               | `vp run build`       | ✅ server + client 都建成                                  |
| 形式驗證基底       | `vp run lemma:gen`   | ✅ 3/3 模組生成成功                                        |
| Dafny 驗證         | `vp run lemma:check` | ⏭ 本機沒裝 Dafny，交給 CI（本次沒動到帶 `//@` 標注的檔案） |

💡 **為什麼要四道關卡？** 型別檢查抓「寫錯型別」、lint 抓「寫法不好」、測試抓「行為錯」、建置抓「組不起來」。每一道抓的錯不一樣，少一道就有一類 bug 溜過去。

💡 **`vp check --fix`**：格式工具自動把縮排、換行排整齊。團隊共用同一套格式，diff 才不會被「我習慣兩格你習慣四格」淹沒。

### 5.5 本機實測前的環境問題（2026-11-19 19:00–19:15）

**做了什麼**：套本機 migration、開 `vp run dev`，請使用者在瀏覽器登入後按紫色麥克風。

**卡住的地方與解法**，按發生順序：

1. **「無法取得會議權杖」**：本機 `.dev.vars` 的 `JAAS_*` 三個值是範本佔位字，簽不出 8x8 的 JWT。真金鑰只有 g0v 維護者有。
   → 不影響測試：紫色麥克風與逐字稿同步只看登入與 `meeting.join` 權限，不用進 Jitsi 會議。
2. **轉錄請求 23 ms 就回應、資料庫沒新條目**：Whisper 不可能這麼快，代表還沒呼叫 AI 就失敗了。
   用不需登入的 `POST /api/transcription/test-ai` 探測，伺服器 log 寫 `Binding AI needs to be run remotely`。
   → 一般 `vp run dev` 不開 AI 綁定（Workers AI 只能在雲端跑）。專案的 `dev:remote` 會開，**但它同時把 D1 / R2 也切到正式站**，
   而且這些資料庫在 g0v 的 Cloudflare 帳號下，使用者的帳號看不到，更不該拿正式資料庫做實驗。
3. **解法：一份只給實驗用的設定，放在 scratchpad 不進 repo**：
   - `wrangler.local.jsonc`：D1 / R2 拿掉 `remote: true`（留本機），`ai` 加 `remote: true`。
   - `vite.local.config.mts`：指定上面的 configPath、`remoteBindings: true`、`persistState` 指回專案的 `.wrangler/state`（本機 D1 已套好 0004）。
   - scratchpad 裡放 `node_modules` 與 `.dev.vars` 的符號連結，讓設定檔找得到套件與本機密鑰。
   - 啟動：`npx vp dev -c <scratchpad>/vite.local.config.mts`，跑在 **http://localhost:5176**。
   - 驗證：`test-ai` 端點回傳了 gpt-oss-120b 產生的大綱 → AI 綁定可用；`/api/meeting/20261119` 讀本機 D1 正常。
4. 中途還撞到一次 `This package is ESM only but it was tried to load by require`：設定檔放在沒有 `package.json` 的目錄會被當 CommonJS，副檔名改 `.mts` 即可。

💡 **綁定（binding）**：Worker 程式碼裡用 `env.DB`、`env.AI` 這些名字取用資源，名字和實際資源的對應寫在 wrangler 設定裡，這個對應就叫綁定。本機開發時每個綁定可以各自決定「用本機模擬」或「連雲端真的」。

💡 **為什麼 Google 登入的 cookie 在 5176 也能用？** 瀏覽器的 cookie 只認主機名（localhost），不分埠號。所以在 5173 登入過，5176 也帶著同一個 session。

💡 **教訓**：環境問題花的時間常常比寫程式多。先用最便宜的探測（這裡是一個不需登入的端點加 curl）把問題縮小到一句錯誤訊息，再決定要改什麼。

### 5.6 實機測試：單機（2026-11-19 19:14）✅

使用者在 http://localhost:5176/jitsi 登入後（未進 Jitsi 會議），按紫色麥克風講了兩段話，各按一次停止。

**伺服器側證據**（本機 D1 `meeting_transcript_entries`，時間為台北時間）：

| 條目 ts（開口時間） | 上傳請求抵達 | 差距          | speaker_uid             | source   | 文字              |
| ------------------- | ------------ | ------------- | ----------------------- | -------- | ----------------- |
| 19:14:11.640        | 19:14:17.524 | 開口早 5.9 秒 | `ag21…HjxT`（伺服器填） | `manual` | 測試測試測試測試  |
| 19:14:20.899        | 19:14:25.137 | 開口早 4.2 秒 | 同上                    | `manual` | 再次測試 再次測試 |

**判讀**：

- 條目時間比上傳時間早好幾秒，證明 timestamp 已經是「開始錄」的時間，不是轉錄完成的時間。改之前這兩個數字會幾乎一樣（甚至條目時間更晚，因為還要等 Whisper）。
- `speaker_uid` 是 Better Auth 的 user.id，由伺服器從 session 填入；`source = manual` 對應紫色麥克風。
- `entry_id = audio_<開口時間>`，與 ts 一致。

💡 **怎麼讀這張表？** 「上傳請求抵達」是伺服器 log 裡 `POST /api/transcription/zh-TW` 的時間，「條目 ts」是資料庫裡存的時間。兩者相減就是「錄音長度 + 上傳時間」。如果 bug 沒修，條目 ts 會比上傳請求還晚（要再加 Whisper 處理時間）。

**編輯測試（19:20）✅**：使用者在面板上把兩條各加了幾個字後儲存。D1 查詢結果：仍是 2 筆、ts 不變、`speaker_uid` 與 `source` 保留、只有 `text` 變成「測試測試測試測試＋編輯測試」「再次測試 再次測試＋編輯」。伺服器無錯誤。證明「編輯走原地更新、不順延」與「編輯不會蓋掉伺服器填的 speaker_uid」兩條規則在實機成立。

**還沒驗的**：

- 雙機同時錄音的 ts 衝突順延：單元測試 `meeting-room.test.ts` 已覆蓋協定；實機要兩個帳號在同一毫秒開口才會觸發，機率極低，屬「理論正確、實機難重現」的那種。

### 6. 進度

- [x] 開分支
- [x] migration 0004
- [x] 共用純邏輯模組
- [x] server 寫入模組
- [x] DO 改動（含 replaces 只回送出者的修正）
- [x] REST 改動
- [x] JitsiView 改動
- [x] 測試（transcript-entries / meeting-entries / meeting-room / audio-recording）
- [x] `vp check` / `vp test` / `vp run build` / `vp run lemma:gen` 全綠
- [x] commit 到 `feat/transcript-timestamp-fix`、push 到 fork，開 draft PR #154
- [x] 本機目視：套 migration（`--local`）後按紫色麥克風，確認條目時間是開口時間（見 5.6）
- [x] 實機測試（單機：錄音 + 編輯）
- [ ] ~~實機測試（雙機同時錄，驗證不互相覆蓋）~~ 已隨碰撞處理一併撤回（見下一節）
- [x] 回 issue #153 更新進度

---

## 2026-11-20 步驟 1 縮減：只留時間戳修正，撤回 migration 與碰撞處理

維護者 2026-11-19 晚上在 issue #153 回覆四點意見，其中兩點直接影響這個 PR：

- **同意**「時間戳改為每輪錄音開始的時間」。
- **暫不新增 `migrations/0004`**：依目前使用情境，同毫秒碰撞的機率極低，資料庫調整與處理成本超過效益，請以不新增 migration 的方式調整 PR。

另外兩點（保留主動轉錄的 UX、顯示所有轉錄中的參與者）屬於後續步驟，與本 PR 無關；issue 方向的討論見 issue 留言。

💡 **為什麼聽維護者的，而不是堅持「我已經寫好了」？** 開源專案的維護者要對整個系統的長期複雜度負責。一個功能寫好了不等於應該合進去——每多一張欄位、一條特殊路徑，之後每個人讀程式碼都要多理解一件事。這叫**維護成本**，而且是由維護者承擔，不是由貢獻者承擔。碰撞處理解的是「幾乎不會發生」的問題，撤回是合理的取捨。寫過的程式碼與測試留在 git 歷史（commit 149f2b1），將來真的需要時可以撿回來。

### 1. 撤回的東西

| 項目                                                                                                    | 處理                                                                                                             |
| ------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `migrations/0004_add_transcript_entry_provenance.sql`（`speaker_uid`／`source` 欄位）                   | 刪除                                                                                                             |
| `src/lib/transcript-entries.ts`（`TranscriptEntry`／`normalizeTranscriptSource`／`applyEntrySaved`）    | 刪除                                                                                                             |
| `src/server/lib/meeting-entries.ts`（`insertTranscriptEntry` 找空位／`updateTranscriptEntry` 原地更新） | 刪除                                                                                                             |
| `src/durable-objects/meeting-room.ts`                                                                   | 還原成 main：`save_entry` 回到 `INSERT OR REPLACE`，`entry_saved` 不帶 `replaces`，`broadcast()` 不再有 `except` |
| `src/server/api/meeting.ts`                                                                             | 還原成 main：不收 `create`／`source`，回應不帶 `timestamp`                                                       |
| `JitsiView.vue` 的 `create: true/false` 旗標、`source: 'manual'/'jitsi'`、`applyEntrySaved`             | 移除                                                                                                             |
| 測試 `transcript-entries` / `meeting-entries` / `meeting-room` 與 `helpers/fake-d1.ts`                  | 刪除（測的對象已不存在）                                                                                         |

### 2. 保留的東西（PR 的全部內容）

- `JitsiView.vue`：每輪錄音開始時記 `roundStartedAt`，跟著音檔進 `audioQueue.startedAt`，轉錄完成後拿它當條目的 `timestamp` 與 `id`。`meetingData.recordingStartTime` 也改用同一個值。
- `audio-recording.test.ts` 新增的兩個案例：音檔帶著開始時間進佇列、條目 timestamp 是開始時間而非轉錄完成時間。只拿掉對 `source: 'manual'` 的斷言。
- 這份實驗日誌。

### 3. 縮減後的驗收（2026-11-20 14:00）

| 關卡                 | 結果                                                                        |
| -------------------- | --------------------------------------------------------------------------- |
| `vp check`           | ✅ 154 檔格式正確，137 檔無錯                                               |
| `vp test`            | ✅ 29 個測試檔、291 個測試全過（比昨天少 3 檔 31 個，是刪掉的那三個測試檔） |
| `vp run build`       | ✅                                                                          |
| `vp run lemma:gen`   | ✅ 3/3                                                                      |
| `vp run lemma:check` | ⏭ 本機沒裝 Dafny，交給 CI；本次未動帶 `//@` 標注的檔案                      |

💡 **為什麼刪了程式碼還要全部重跑？** 刪東西比加東西更容易漏：某個檔案 import 了被刪的模組、某個測試還在斷言被拿掉的欄位，都只有跑過才知道。這次 `vp check` 的型別檢查就是抓「有沒有人還在 import 已刪除模組」的那道閘。

### 4. 部署時要注意（更新）

**不再需要套 migration。** 昨天寫的「先套 migration 再部署」已失效；縮減後的 PR 只改前端 `JitsiView.vue`，部署流程與平常一樣。本機 D1 若已套過 0004，多出來的兩欄不影響程式運作（`INSERT OR REPLACE` 沒列出的欄位會是 NULL），不必回滾。

### 5. 方向討論：issue 寫錯了什麼、原意是什麼（2026-11-20 上午）

**做了什麼**：重讀 issue 內文，對照維護者的回覆，發現 issue 把兩個不同的想法混在一起寫，維護者讀到的是錯的那一個。

**issue 裡互相矛盾的兩組敘述**

- 原意（只出現在提案方向第 2 點與步驟 3）：「Jitsi 裡靜音就停、取消靜音就錄。Jitsi 的麥克風開關就是轉錄開關。」也就是跟 Zoom／Teams 一樣，每個人自己管麥克風，開麥才會被轉錄。
- 蓋過原意的敘述（標題、背景第 1 點、提案方向第 1 點、步驟 4）：「自動分軌錄製」「要有人記得按」是缺點、「入會即自動開始」「本會議全程轉錄」。這些讀起來是「一進會議就錄所有人」。

**關鍵事實**：`JitsiView.vue` 已設定 `startWithAudioMuted: true`，所有人入會時麥克風預設是關的。所以在原意的模式下，「入會即自動開始錄」根本不會發生——這兩組敘述在現有設定下是互斥的，寫的時候沒抓到。

💡 **寫提案的教訓**：把「機制」（麥克風開關觸發）和「效果」（不用再按）分開寫。這次把效果寫成「自動」，讀者就把它理解成「不受控」。真正要表達的是「開關換了一個位置」，不是「沒有開關」。

**原意與維護者要求的實際差距**

| 面向             | 維護者要求         | 原意               |
| ---------------- | ------------------ | ------------------ |
| 誰決定自己被轉錄 | 使用者自己         | 使用者自己         |
| 開關在哪         | 紫色麥克風         | Jitsi 麥克風按鈕   |
| 預設狀態         | 不轉錄             | 不轉錄（預設靜音） |
| 多人同時轉錄     | 顯示所有轉錄中的人 | 同樣需要           |

差距只有一個：**保留紫色麥克風當第二個開關（方案 B），還是讓 Jitsi 麥克風直接兼任（方案 A）**。

### 6. 考古：為什麼原先會有紫色麥克風（2026-11-20 上午）

**做了什麼**：讀 neo 的 issue 時間線，找「為什麼要加按鈕」的理由。沒有任何文件寫明，但可以拼出來。

| 日期        | 事件                                                                                                                                            |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| 2025-06-15  | neo #28：想用 JaaS 付費字幕（每分鐘 0.06 美元），英文可用、中文等不到客服。原本就是「開麥就有字幕」，沒有按鈕                                   |
| 2025-06-30  | neo #42 ＋ transcription-worker #4：「優先嘗試自架轉錄分軌音訊」。第一版按鈕文案「開始錄音轉錄（最多 10 秒）」。按鈕誕生                        |
| 2025-07-09  | neo #44：10 秒太短，調到 30 秒                                                                                                                  |
| 2025-07-30  | neo #77：分享螢幕時換頁會中斷，改成不在焦時自動續錄                                                                                             |
| 2025-08-01  | **neo #82（維護者自提）：「自動記錄模式：只要有開麥克風就自動開啟語音記錄，解決忘記開的問題」**。唯一留言自問「跟 #77 重疊？」，8/13 關閉未實作 |
| 2025-08-03  | neo #81：社群要求入會預設關麥、關鏡頭，當天改好                                                                                                 |
| 2025-08-11  | neo #83：顯示「誰按了紫色麥克風正在發言」                                                                                                       |
| 2025-08-13  | neo #84：研究用 JaaS「主要發言者藍框」觸發轉錄。結論：藍框是出聲之後才偵測到，用它觸發會漏掉開頭                                                |
| 2025-08-16  | neo #85、#87：按一下開始、按一下停止，加上傳佇列。按鈕定型                                                                                      |
| 2026-09、10 | neo #157 顏色改回紫色加 tooltip、#165 修「某某人轉錄中」不顯示給別人                                                                            |

**第一版是按鈕的三個技術理由（推論）**

1. Jitsi 在 `8x8.vc` 的 iframe 裡，同源政策下拿不到它的音訊。第一版程式碼註解寫「獨立的音訊流，不影響 Jitsi Meet」，網頁要自己再開一次 `getUserMedia`，按鈕是最直接的觸發方式。
2. 它是從字幕流程複製出來的實驗，只想驗證 Whisper 轉中文行不行，10 秒一段、按了才錄是最小驗證方式。
3. 每段音檔都要花 Whisper 算力，按了才錄可以避免轉錄靜音。

**為什麼後來沒改成「開麥即轉錄」（推論）**：#82 被 #77 的解法岔開——「不在焦時自動續錄」解掉了最痛的情境，痛感降低後 #82 就關了。#84 繼續找的是「發言偵測觸發」，沒有往「麥克風開關事件」想，而 `audioMuteStatusChanged` 一直都在。之後一年按鈕累積了 #83、#157、#165 等功能與修 bug，成為大家認得的介面；到 2026-11-19 的回覆，它被賦予「知情同意的保障」這層意義——這是事後長出來的理由，不是 2025 年 6 月設計它的原因。

💡 **考古的價值**：知道「現狀是怎麼來的」才能判斷它是「刻意的設計」還是「實驗留下來的慣性」。這次的發現讓討論從「新人要推翻既有設計」變成「接續維護者自己擱置的想法（#82）」，說服力完全不同。

### 7. 「後端已能處理多人同時轉錄」是真的嗎？（2026-11-20 上午）

維護者回覆說「目前後端已能處理多人同時轉錄，但 UI 只顯示一位轉錄者」。對了一次程式碼，結論是**「不阻擋」，不是「有管理」**：

- `/api/transcription/:lang` 只檢查 `meeting.join` 權限，不看誰是錄音者；DO 的 `save_entry` 收到什麼就寫什麼。多人同時錄，各自的條目都會進表。這部分對。
- 但 `meeting_sessions` 只有一格 `recording_speaker`、一格 `recording_start_time`。`sync_recording` 是誰最後送誰贏。甲在錄、乙按下又停掉送 `null`，所有人看到「沒人在錄」，但甲還在錄。每輪 30 秒重開時還會再 sync 一次，兩人同時錄時名字每 30 秒互相蓋。
- 前端 `toggleAudioRecording` 沒有「別人在錄就不給按」的檢查，自己的 `isRecordingAudio` 存在本機不受別人影響，所以多人可以同時錄、不會被互相停掉。

**結論**：「只改前端顯示」不夠，資料模型只有一格。要讓 DO 在 WebSocket attachment 維護一份「轉錄中名單」並以新訊息廣播，前端改顯示名單。不需要 migration。

### 8. 方案 A 的技術可行性與費用（2026-11-20 上午）

兩件事都查了官方文件，不憑印象。**以方案 A（拿掉紫色麥克風、Jitsi 麥克風兼任開關）為前提**。

**iframe 架構不會擋住這個做法**：iframe API 的 `audioMuteStatusChanged` 事件（送 `{ muted: boolean }`）與 `isAudioMuted()` 函式都是官方文件列出的功能。收到 `muted: false` 就呼叫現有的 `startAudioRecording()`、`muted: true` 就 `stopAudioRecording()`，30 秒切段、佇列、倒數全部沿用。

**要實測的三件事**（都是「頁面自己開第二條麥克風」造成的，A／B 都有，A 會從偶爾變成必然）：

1. **麥克風對不上**：`deviceId` 依網域不同，Jitsi 用的跟我們錄的可能不是同一支。用 `getCurrentDevices()` 取裝置名稱比對本站 `enumerateDevices()` 清單。名稱要在權限同意後才看得到。
2. **同一支麥克風同時兩條音訊流**：桌機 Chrome 沒問題；neo #120 回報某些 iOS 版本紫色麥克風不能用，iPhone 要重點測。
3. **權限視窗跳兩次**：iframe 一次、本站一次。時機會從「按紫色時」變成「第一次開麥時」。

回音問題照舊：錄的是自己麥克風，別人的聲音從喇叭出來也會被收到，戴耳機才解。這是物理限制。

**費用**：JaaS、D1、R2、DO 都不變，只有 Workers AI 的 Whisper 會變。`whisper-large-v3-turbo` 每分鐘音檔 0.0005 美元（46.63 neurons），免費額度每天 10,000 neurons 約 214 分鐘。**算式對 A／B 一樣：所有人轉錄開關開著的分鐘數加總，不管有沒有在說話。** 2 小時、10 人會議試算：

| 情境                                   | 音檔分鐘 | 費用（美元） |
| -------------------------------------- | -------- | ------------ |
| 方案 B（現況）：一人按紫色麥克風錄全場 | 120      | 0.06         |
| 方案 A：大家守規矩，平均 2 人開麥      | 240      | 0.12         |
| 方案 A：10 人全程不關麥                | 1,200    | 0.60         |

金額小，但 A 的靜音分鐘變多會帶來兩個副作用：Whisper 對靜音的幻覺輸出變多（現有過濾「字幕志願者」等假字的邏輯會更吃重）、免費額度可能單場會議就用完。這就是原提案要有 VAD 那一步的原因；先擱置，用「大家自己管麥克風」當成本控制。

💡 **neuron**：Cloudflare 自訂的 AI 用量單位，不同模型換算不同，可以想成「AI 算力的點數」。

來源：Cloudflare Workers AI Pricing、whisper-large-v3-turbo 模型頁、Jitsi IFrame API Events／Functions 文件。

### 9. 回 issue 與更新 PR（2026-11-20 下午）

- issue #153 第一則留言：承認寫法誤導、說明原意、提出方案 A／B 請裁定、附上三個地雷與費用試算、其餘三點照辦。
- **踩到一個 GitHub 的坑**：留言寫 `neo #82`、`#84`，貼在 vTaiwan-hono 會被自動連到 **hono 自己的** #82、#84（後者是「修訂貢獻者」，所以句子裡冒出這串字）。改成 `g0v/vue.vTaiwan-neo#82` 這種跨 repo 完整寫法才會連對。

  💡 **跨 repo 參照**：GitHub 的 `#數字` 永遠指「這個 repo」的 issue。要指別的 repo 要寫 `owner/repo#數字`。

- PR #154：commit 03b2114 推到 fork，PR 標題改為「fix: 逐字稿時間戳改為開口時間（#153 步驟 1）」，說明改寫為縮減後範圍並加「歷史」段落註明第一版留在 149f2b1。從 draft 轉為 ready for review。
- issue #153 第二則留言：回報 PR 已縮減、等方案 A／B 裁定、裁定後才修 issue 標題與內文。不 tag 任何人，參與者會自動收到通知。

### 10. 進度（2026-11-20）

- [x] 重讀 issue，找出與原意矛盾的敘述
- [x] 考古 neo 的紫色麥克風由來（#28 → #42 → #82 → #84）
- [x] 核對「後端已能處理多人同時轉錄」的實際程度
- [x] 查 Jitsi iframe API 與 Whisper 計費，整理三個地雷與費用試算
- [x] 回 issue #153 澄清原意、提方案 A／B
- [x] PR #154 縮減、驗收全綠、push、改標題說明、轉 ready for review
- [x] 回 issue #153 回報縮減進度
- [x] 等維護者裁定方案 A／B → 方案 B
- [x] 裁定後修訂 issue #153 標題與內文（拿掉「自動」「全程」「入會即」）
- [ ] 依 PR #154 審閱意見修改
- [ ] 步驟 2（轉錄中名單）等 PR #154 合併後開新分支

### 11. 維護者裁定方案 B，修訂 issue（2026-11-20 下午）

維護者 15:03 回覆：**採方案 B**，保留紫色麥克風作為獨立開關，再加轉錄名單顯示。理由：vTaiwan 小松有社群聚會性質，不是全程轉錄的工作會議；Jitsi 麥克風開著時與會者可能在閒聊或私人溝通，不是每句都想被轉錄，知情同意要保留。

💡 **為什麼接受**：原本的 Zoom 模式假設是「開麥等於對全場發言」。在小松的場域裡開麥聊天是常態，這個假設不成立。這是場域特性，不是技術問題。

**修訂 issue 前先釐清「提案跟現況差在哪」**：對照程式碼後確認，現行程式已經可以多人同時按紫色麥克風，各自的條目都會寫進同一份逐字稿。方案 B 沒有新增任何能力，剩下的只有兩件事：讓已有的能力**被看見**（名單取代單格顯示）、讓多人的條目**排對順序**（PR #154）。issue 內文照這個角度改寫，新增「現況」與「不會改變的事」兩節，避免讀者以為是新功能。

**修訂結果**：標題改為「改善多人同時轉錄的體驗：顯示所有轉錄中的參與者，並修正逐字稿時間戳」，內文全面改寫，原文備份在 [`issue-153-original-proposal.md`](./issue-153-original-proposal.md)。

**過程中的一個小插曲**：改 issue 內文的指令被 Claude Code 自動模式的安全分類器擋下（覆寫公開 issue 屬不可逆的外部修改），改由使用者在提示列以 `!` 前綴親自執行三行指令：備份原文、改標題與內文、留言。這個設計是對的——AI 代理不該自行覆寫別人也在看的公開內容。

---

## 2026-11-20 晚上：步驟 2 改動計畫（待確認，尚未動工）

計畫全文貼在 fork 的 issue（jiangzuojin226-sys/vTaiwan-hono，供隔天閱讀與批註），這裡記摘要與理由。

**目標**：把「{某人} 錄音中」的單格顯示換成「正在轉錄：甲、乙」名單。不加 migration、不碰逐字稿寫入、不碰授權。

**核心設計**

| 層                                  | 改動                                                                                                                    | 為什麼                                                            |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Worker（`meeting.ts`）              | WebSocket 升級時多轉送 `X-User-Name`（`encodeURIComponent`）                                                            | 名字由伺服器從 session 填，不信任客戶端；標頭不能放中文           |
| DO（`meeting-room.ts`）             | 連線便條加 `name`、`capturing`；新訊息 `capture_status`（客→服）與 `capture_list`（服→客）；`init` 帶現有名單；斷線重算 | 所有連線都在 DO 手上，只有它看得到全貌；名單是暫態，放便條不放 D1 |
| DO 相容橋                           | 舊客戶端的 `sync_recording` 也會更新便條並廣播名單                                                                      | 部署後還開著舊網頁的人不會從名單消失                              |
| 純函式（`src/lib/capture-list.ts`） | `capturersFromMetas()`：過濾、以 userId 去重、排序                                                                      | 同一人兩個分頁只算一人；可單元測試                                |
| 前端（`JitsiView.vue`）             | 開始錄送一次 `capture_status: true`，停止送 false，**每 30 秒的新一輪不重送**；顯示改讀名單，不顯示秒數                 | 每輪重送是現在名字互蓋的根源；秒數只對自己有意義                  |
| i18n                                | 新增 `jitsi.capturingList`，移除 `jitsi.recordingStatus`                                                                | 三檔同步                                                          |

**要小心的三件事**：標頭中文編碼、斷線時排除正在斷的那條連線、錄音中斷線重連後要重送 `capture_status`。

**測試**：純函式測試、回收 149f2b1 的 DO 測試骨架、`audio-recording.test.ts` 補送出時機。

**待決定**：名單是否包含自己（建議包含）；`recordingStatus` key 現在移除還是之後（建議現在）。

💡 **為什麼計畫跟實作分開、隔天再決定？** 累的時候做決定容易漏掉邊角，而計畫寫下來後隔天讀，常常能一眼看出前一晚沒注意的問題。開源協作裡「先貼計畫、再動手」也讓維護者有機會在寫程式之前就說「這裡不對」，省掉改完再推翻的成本。

## 2026-11-23：PR 審查回覆、步驟 2 計畫貼上 issue、計畫逐列教學

### 1. 維護者 Link1515 的審查：experiments/ 不進 repo

PR #154 收到 CHANGES_REQUESTED，程式碼沒有意見，只要求移除 `experiments/` 目錄：實驗紀錄會增加維護負擔、讓 PR 範圍不聚焦；要保留的決策整理到 issue 或 PR 描述即可。

處理方式（1 加 3）：

- 從 PR 分支加一筆 `chore: 依審查意見移除 experiments/ 目錄`（4d6ca5a），不重寫歷史。PR 現只剩 `JitsiView.vue` 與對應測試。
- 日誌搬到 fork 的 `docs/issue-153-experiments` 分支保存。本機工作副本留在 `experiments/`，用 `.git/info/exclude` 排除——這個檔案不進 git，所以 PR 不會多一筆 `.gitignore` 改動。
- PR 說明刪掉提到日誌的那行，驗證一節改指向 issue 的 11/19 留言；回覆一句告知已移除並請求核准 CI。issue #153 內文「歷史」段的日誌連結也改指向 fork 分支。

💡 **為什麼不選重寫歷史？** 把五筆日誌 commit 從分支上拿掉再 force push，PR 會乾淨很多，但新手操作 force push 容易出錯，而且既有審查留言會對不到舊 commit。「加一筆刪除 commit」歷史裡還看得到日誌，可是安全、可回溯，審查者要的只是 diff 乾淨。

### 2. 步驟 2 計畫貼到 upstream issue

兩個待決定點定案：名單**包含自己**；`jitsi.recordingStatus` key **現在移除**（程式碼只剩被替換掉的那一行用到它）。計畫全文貼在 g0v/vTaiwan-hono#153（issuecomment-7100414346），等維護者回饋後再從 `main` 開 `feat/153-capture-list`。

### 3. 計畫逐列教學（給自己的筆記）

貼完計畫後，請 AI 把「改動」表格一列一列解釋。以下是整理後的筆記，日後讀程式碼對不上時回來看。

**先建立三個觀念**

1. **HTTP 像寄信，WebSocket 像通電話。** 平常瀏覽器與伺服器是「寄一封信、收一封回信」，收完就斷。逐字稿要「別人新增一句我馬上看到」，所以頁面一開就撥一條電話線（`JitsiView.vue` 的 `connectMeetingWs()`），兩邊隨時能講話。
2. **Durable Object 是「一個日期一間會議室」。** DO 是全世界只有一份、有自己記憶的小程式。本專案用日期當會議室名字（`idFromName('meeting-' + date)`），同一天所有人的電話線都接進同一間。**名單只有 DO 算得出來，瀏覽器各自看不到彼此**——這是整個計畫的核心。
3. **每條電話線上可以貼一張便條。** DO 用 `serializeAttachment` / `deserializeAttachment` 在每條連線上存一小段資料。現在便條只有兩格：登入了沒、是誰（`ConnectionMeta`）。計畫就是多貼兩格。

**現在為什麼會壞**：`meeting_sessions` 表只有**一格**「誰在錄音」。前端每輪錄音開始都呼叫 `syncRecordingStatus()` 把自己的名字寫進去，停止寫 null。兩人同時錄，像兩個人搶一塊白板輪流擦掉對方的名字，每 30 秒換一輪。不是「顯示」壞了，是「只有一格」這個資料模型裝不下兩個人。

**逐列**

| 列                                                                                       | 重點                                                                                          | 為什麼                                                                                                                                                                                                                                                                                                 |
| ---------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Worker 多轉送 `X-User-Name`                                                              | 資料流是瀏覽器 → Worker → DO。Worker 讀 cookie 查出是誰，寫進自訂標頭轉給 DO；DO 只信這些標頭 | 瀏覽器送的任何東西都能在 F12 改，**身分相關資料一律伺服器填**。HTTP 標頭只允許 ASCII，中文名字要先 `encodeURIComponent`（跟網址中文變 `%E4%B8%AD` 同一套規則），DO 端再 decode                                                                                                                         |
| DO 便條加 `name`、`capturing`；新訊息 `capture_status`（客→服）、`capture_list`（服→客） | 訊息型別是 TypeScript union type，加兩種訊息就是各加一行、`switch` 多一個 `case`              | `init` 要帶名單：新連線第一包就是「目前全部狀態」，不帶的話中途加入的人要等下一次有人開關才看得到。斷線要重算：關分頁不會先送「我停了」，靠 `webSocketClose` / `webSocketError` 回呼把人移出，否則永遠掛著                                                                                             |
| 相容橋                                                                                   | DO 收到舊 `sync_recording` 時照舊做，另外多更新便條、多廣播名單                               | 部署那一刻一定有人還開著舊網頁，只會送舊訊息。**向下相容**＝新程式要能跟舊程式共處一段時間，等確定沒人用舊版再拆                                                                                                                                                                                       |
| 純函式 `capturersFromMetas()`                                                            | 輸入便條陣列、輸出名單；不依賴 DO                                                             | DO 要 Cloudflare 環境才跑得起來，測試麻煩。**純函式**＝同樣輸入永遠同樣輸出、不碰外部狀態，可以單獨測。以 `userId` 去重，否則同一人開兩分頁會出現兩次                                                                                                                                                  |
| 前端送出時機與顯示                                                                       | 只在**狀態改變**時送：開始送一次 `true`，停止／出錯送一次 `false`，30 秒輪替不送              | **同步「狀態」，不要同步「事件」**。狀態是「我在錄／沒在錄」，事件是「我又開始新一輪了」；同步事件會把接收端淹沒，這正是現在每 30 秒互蓋的根源。名單含自己（畫面一致、確認有被算進去）、不顯示秒數（只有錄的人在意，按鈕上已有倒數）。`syncRecordingStatus()` 先不刪，刪了舊版使用者就看不到任何人在錄 |
| i18n 三檔同步                                                                            | 新增 `jitsi.capturingList`，刪 `jitsi.recordingStatus`                                        | 介面文字用 key 查三個語言檔，`vp test` 會檢查三檔 key 集合完全一致，少一個多一個都紅燈                                                                                                                                                                                                                 |

**三件要小心的事**

1. **編碼要雙向**：Worker encode、DO decode，少一邊就顯示 `%E7%8E%8B`。測試故意用中文名字當測資就能抓到。
2. **斷線時排除正在斷的那條連線**：`webSocketClose` 被呼叫那一瞬間，該連線可能還在 `getWebSockets()` 裡、也可能不在，文件沒保證順序。賭「它已經不在」會時好時壞，這種 bug 最難抓。解法是不賭：重算時明確排除參數裡那條 `ws`，在不在清單裡結果都對。
3. **重連後要重送**——⚠️ **計畫寫得比現況樂觀**。對程式碼後發現 `ws.onclose` 只把 `this.ws` 設成 null，**沒有自動重連**；`sendMeetingMessage` 在沒連線時把訊息排進 `wsPendingMessages`，等下次 `onopen` 一次沖出。重連只在切換日期時發生。所以：
   - 錄音中斷線又沒重連：訊息卡在佇列，別人的名單停在斷線前狀態；DO 端因為收到 close 會把人移出，這部分是對的。
   - 日後若加自動重連：新連線在 DO 眼中是全新便條，`capturing` 預設 false，自己會從名單消失。
   - **動工時第一步**：把這條改成「在 `onopen` 補一行『若 `audioRecordingActive` 為真就重送 `capture_status: true`』」，不管重連怎麼發生都安全。

💡 **計畫寫完要回頭對程式碼。** 貼 issue 前我對了標頭與便條的部分，重連這一段只對了一半。教學時逐列對程式碼才發現，這就是「解釋給別人聽」會逼你驗證假設的價值。

💡 **暫態放記憶體，長期資料放資料庫。** 這份計畫沒加任何資料庫欄位，因為「誰正在錄」人一離線就不成立，存進資料庫反而要處理殘留的髒資料。判斷一筆資料該放哪，先問「程式重啟後它還應該存在嗎」——逐字稿內容要（D1），轉錄中名單不要（DO 便條）。

### 4. 教學：issue 留言第 3 點「DO 測試 vs 純函式測試」在問什麼（2026-11-23 晚）

> 📚 **明天（11/24）要學的段落。** 讀完試著用自己的話回答：為什麼 `capturersFromMetas()` 要抽出來？DO 測試多驗到哪三件事？代價是什麼？

留言第 3 點問維護者「新增 `meeting-room.test.ts` 直接建構 DO 加假 WebSocket 來測，可以接受嗎？還是維持純函式層級就好？」白話是：**我打算用一種成本較高、但能驗到更多東西的測法，你們願意為它養一套替身嗎？**

**兩種測法差在「測的對象有多大」**

_純函式層級_：測的對象是一個獨立函式。看 `src/tests/transcription-versions.test.ts`：

```ts
expect(formatVersionId(epoch)).toBe('20260803T091500123Z')
expect(isValidVersionId('../../secret')).toBe(false)
```

結構是「餵一個輸入，斷言一個輸出」。`formatVersionId` 不讀資料庫、不碰網路、不看時間，同樣輸入永遠同樣輸出，所以叫**純函式**。測它不需要任何準備。步驟 2 的 `capturersFromMetas()` 刻意設計成這種函式，測試三行就釘住去重與過濾：

```ts
const metas = [
  { userId: 'a', name: '小明', capturing: true },
  { userId: 'a', name: '小明', capturing: true }, // 同一人第二個分頁
  { userId: 'b', name: '阿華', capturing: false },
]
expect(capturersFromMetas(metas)).toEqual([{ userId: 'a', name: '小明' }])
```

_直接建構 DO_：測的對象是整個 `MeetingRoom` 類別。DO 設計上要活在 Cloudflare 環境，依賴測試機器上沒有的東西，每一樣都要做替身：

| DO 依賴什麼               | 正式環境誰提供      | 測試時怎麼辦                                                                                          |
| ------------------------- | ------------------- | ----------------------------------------------------------------------------------------------------- |
| `cloudflare:workers` 模組 | Cloudflare 執行環境 | `vite.config.ts` 在 VITEST 模式把它換成 `src/shims/cloudflare-workers.ts`                             |
| `ctx.getWebSockets()`     | DurableObjectState  | 自己做一個假 `ctx`，回傳一個陣列                                                                      |
| 真的 WebSocket            | 瀏覽器連進來        | `FakeWebSocket` 類別：`send()` 不真的送，只把訊息存進陣列；`deserializeAttachment()` 回我們塞的假便條 |
| D1                        | Cloudflare D1       | `FakeD1`，用 `Map` 假裝資料表                                                                         |

這套在 11/19 的 commit `149f2b1` 寫過一次（`meeting-room.test.ts` 148 行 + `helpers/fake-d1.ts`），11/20 縮減 PR 時整個刪掉。骨架簡化後：

```ts
const alice = new FakeWebSocket({ isAuthenticated: true, userId: 'uid-alice', date })
const meeting = new MeetingRoom({ getWebSockets: () => [alice, bob] }, { DB: fakeDb })
await meeting.webSocketMessage(alice, JSON.stringify({ type: 'capture_status', capturing: true }))
expect(bob.sent.at(-1)).toEqual({ type: 'capture_list', capturers: [...] })
```

做的事是「假裝 Alice 送了一則訊息，檢查 Bob 的假 WebSocket 有沒有收到正確的廣播」。測的不再是一個函式的輸入輸出，而是 DO 收到訊息後**一連串動作**的結果：改便條 → 重算名單 → 廣播所有人。

**為什麼要問維護者**：不是哪個對，是值不值得。

- _純函式測不到的洞_：`capturersFromMetas()` 對了，不代表 DO 在正確時機呼叫它。「斷線時排除正在斷的連線」「舊版 `sync_recording` 也要更新便條」「未登入送 `capture_status` 要被拒」三件事全在 DO 的 `switch` 與回呼裡，純函式碰不到。11/19 就是靠 DO 測試模擬第三個旁觀者，才抓到「修正指令廣播給所有人會讓別人誤刪自己條目」的 bug。
- _DO 測試的代價_：要養 `FakeWebSocket`、假 `ctx`、`FakeD1`、shim。每個替身都是「我猜 Cloudflare 會這樣行為」的模型，Cloudflare 改 API 替身不會跟著變。Link1515 才剛以維護負擔為由要求移除 experiments/，bestian 一再強調維持簡潔，多一套測試基礎設施可能被視為負擔。

所以第 3 點是在問：**願意為了多驗到 DO 層的行為，多養這一套替身嗎？** 答「只要純函式」→ 把去重排序測好，DO 層靠本機開兩個瀏覽器手動驗；答「可以」→ 把 `149f2b1` 的骨架撿回來改。

**延伸觀念：測試金字塔**

- **底層**：純函式單元測試。多、快、便宜、穩定。本 repo 絕大多數測試在這層。
- **中層**：模組測試，用替身換掉外部依賴。DO 測試在這層。能驗流程，但要養替身。
- **頂層**：端對端，真瀏覽器真服務。最像真實使用，但慢、貴、容易因環境紅燈。AGENTS.md 說「尚無帶 session 的端點層級整合測試」就是這層目前是空的。

💡 **能推到底層的邏輯盡量抽成純函式推下去，中層只留真正需要驗流程的部分。** 步驟 2 把排序去重抽成 `capturersFromMetas()` 就是在做這件事；剩下真的留在 DO 裡的，才需要問值不值得用中層測試蓋。

## 2026-11-25：PR #154 合併、bestian 放行步驟 2、狀態契約逐條教學

### 1. 進度：步驟 1 完結，步驟 2 可以動工

- **PR #154 已合併。** Link1515 於 11/24 01:12 APPROVED、01:13 merge（merge commit `2096622`），CI 兩個 job（`check / test / build / lemma:gen`、`lemma:check`）皆綠。
- **六封「Run failed: CI … No jobs were run」通知信不是程式錯。** 它們對應 11/19–11/20 推上 PR 的六個舊 commit。外部 fork 第一次貢獻，每次 push 的 CI 都要維護者按核准才會跑；那六個 run 一直停在「等核准」、job 數為 0。bestian 只核准了最新 commit `4d6ca5a` 的 run（11/24 00:06 跑、00:08 全綠）。01:13:10 合併 PR 時，GitHub 把其餘「還在等核准」的舊 run 一次標成 failure 收掉（六個 run 的 `updated_at` 全是 01:13:12），然後寄信。以後推 PR 前在本機跑完 `vp check`、`vp test`、`vp run build`，盡量一次推到位，可減少中間 commit 與這類信。
- **bestian 於 11/23 22:11 回覆步驟 2 計畫（issuecomment-7115440143）：整體方向合理，可依方案 B 進行，從 `main` 開分支即可。** 三個問題的答案：方向 ✅、範圍沒有多做但漏了四組細節、測試 ✅ 可以直接建構 DO 加假 WebSocket。他另外補了一份「狀態契約」，下面逐條整理。

### 2. 用比喻理解整件事

把線上會議想成一間**會議室**。

- **會議室本身 = Durable Object（DO）。** 一天一間，房間管理員負責記住誰在裡面、把訊息廣播給所有人。
- **每個人進門時拿到的名牌貼紙 = attachment。** 上面寫「我是誰（userId）、我叫什麼（name）、我有沒有在錄音（capturing）」。
- **門口櫃台 = Worker 的 `meeting.ts`。** 櫃台看身分證（session）幫你寫名牌，不准你自己寫，否則有人會冒充別人。
- **舊做法：白板上只有一格「現在錄音的人：＿＿」**（D1 的 `recording_speaker`）。甲寫上名字，30 秒後乙也寫，甲就被擦掉。這是「名字每 30 秒互蓋」的根源。
- **新做法：管理員看所有人的名牌，算出「正在錄音的名單」，廣播給大家。** 白板那一格先留著不擦，給還沒更新的舊客戶看。

我的計畫是「把白板單格改成名牌制」，bestian 的回覆是「方向對，但名牌制有幾個細節會出錯，先補進計畫」。

### 3. 對照表：我問的三個問題 vs bestian 的回答

| 我問的 | bestian 的答案 | 白話 |
| --- | --- | --- |
| **方向**：名單放 DO 連線狀態、不加 migration、保留舊欄位做相容橋，夠簡潔嗎？ | **可以。** 名單是「各連線現在回報的狀態」，不用存進資料庫。以 userId 去重合理。 | 名牌制通過。同一人開兩個分頁，只要還有一個在錄，名單就要有他。 |
| **範圍**：有沒有多做或漏做？ | **沒有多做，但漏了四組細節**（第 5 節）。 | 他沒砍項目，而是加了「這些情況計畫沒寫到，會出 bug」。 |
| **測試**：可以直接建構 DO 加假 WebSocket 來測嗎？ | **可以，而且要這樣測。** 但要呼叫真的 DO 方法、檢查其他人收到什麼，不能只檢查「假物件有被叫到」。 | 他選了成本高但驗得多的那條路，還多列了六個測試案例。 |

### 4. 對照表：計畫每一層改動，bestian 的態度

| 層 | 我寫的 | 回應 | 加的條件 |
| --- | --- | --- | --- |
| Worker `meeting.ts` | 多轉送 `X-User-Name`，先編碼，名字從 session 填 | ✅ | **Worker 無條件覆寫這個標頭。** 外部請求自帶的同名標頭要整個蓋掉，不然能自己填名字冒充別人。 |
| DO `meeting-room.ts` | 名牌加 `name`、`capturing`；新增 `capture_status`／`capture_list`；`init` 帶名單；斷線時重算 | ✅ | `capture_status` 只接受布林；身分只信名牌不信訊息；未登入不能送；**每次改名牌都要 `serializeAttachment()` 貼回去**。 |
| DO 相容橋 | 舊客戶端送 `sync_recording` 時照舊寫白板，也更新那條連線的名牌 | ✅ | 舊訊息**只改送它的那個人**的名牌，不能拿白板那一格反推名單。新名單只看 `init`／`capture_list`。相容橋的限制與移除條件寫進 PR 說明。 |
| 純函式 `capture-list.ts` | `capturersFromMetas()` 過濾、去重、排序 | ✅ | 無。 |
| 前端 `JitsiView.vue` | 按下麥克風送 true、停止送 false、每 30 秒不重送；顯示改讀名單 | ⚠️ | true 要**錄音器真的啟動後**才送；false 要多涵蓋一條路徑；顯示條件拿掉 `!isTranscripting`。 |
| i18n | 新增 `jitsi.capturingList`、移除 `jitsi.recordingStatus` | ✅ | 無。 |

### 5. 對照表：bestian 補的四組「沒想到的情況」（狀態契約）

| # | 情況 | 我原本寫的 | 會出什麼 bug | bestian 要求 |
| --- | --- | --- | --- | --- |
| 1 | 錄音什麼時候算開始 | 按下麥克風時送 true | `audioRecordingActive` 在**還沒拿到麥克風權限**時就變 true（`JitsiView.vue:1022`）。使用者拒絕權限，名單上卻顯示他在錄。 | 錄音器**成功啟動後**才送 true。 |
| 1 | 錄音什麼時候算結束 | 手動停止或錄音錯誤時送 false | 還有第三條路：登入失效時 `authSession` watcher 直接呼叫 `cleanupAudioRecording()`（第 566 行），不會關 WebSocket。沒送 false，別人的名單上你永遠在錄。 | false 要涵蓋這條清理路徑；30 秒換段不能讓名單閃一下。 |
| 2 | 名牌要貼回去 | 改 attachment 的 `capturing` | Cloudflare 給的名牌是**影本**，改影本原件不變。休眠醒來讀到舊原件。 | 每次改完都 `serializeAttachment()`。 |
| 2 | 正在離開的人 | 斷線時排除正在斷的連線 | 叫了 `close()` 之後 `getWebSockets()` **還是可能列出那條連線**（CLOSING）。下次重算又把他算進去。 | 把名牌標成沒在錄並貼回去，或聚合時排除非 OPEN。不要另養一份名單快取。 |
| 3 | 斷線後畫面怎麼顯示 | 重連時 onopen 重送 true | 沒寫「斷線期間畫面顯示什麼」。繼續顯示最後一份名單，使用者以為是即時的。 | 斷線或切歷史日期時**清空名單或標示「無法同步」**。名單空不代表沒人在錄。 |
| 4 | 名單和音檔處理是兩件事 | 顯示改讀名單 | 現在顯示有 `!isTranscripting` 條件（第 225 行）：自己一轉錄，**整份名單消失**。 | 拿掉這個條件。佇列提示不再用單一 `recordingSpeaker`。名單、自己的倒數、待處理音檔數各自顯示。 |
| 4 | 版面 | 不顯示秒數 | 原框 `w-48 h-6` 只放得下一個名字，多人或長名字爆版。 | 要能換行。連帶刪掉只服務共用秒數的 `recordingDuration` 與每秒計時器。 |

### 6. 細講：「每 30 秒換段」是什麼、為什麼會讓名單閃

**現在就有的功能，不是步驟 2 新加的。** `JitsiView.vue:395` 的 `maxRecordingTime: 30 * 1000`。按一次紫色麥克風，不是一路錄到按停止，而是**每 30 秒切一段、送去轉錄、馬上再開下一段**：

| 步驟 | 行號 | 發生什麼 |
| --- | --- | --- |
| 1 | 1077 | `recorder.start()` 開始錄這一段 |
| 2 | 1084 | 設 30 秒鬧鐘 `setTimeout(..., maxRecordingTime)` |
| 3 | 1086 | 鬧鐘響，`stopAudioRecordingForNextRound()` 把錄音器 stop |
| 4 | 1055 | 觸發 `recorder.onstop` |
| 5 | 1063 | 這 30 秒音檔丟去轉錄 `processRecordedAudio` |
| 6 | 1065 | `audioRecordingActive` 還是 true → `startNextRecordingRound()` |
| 7 | 1490 | 再呼叫 `startAudioRecording()`，回到步驟 1 |

為什麼要切段：音檔要送去轉錄才會變文字，等講完一小時再送就不是即時逐字稿。**每一輪都重跑 `startAudioRecording()`**，而它第 1078 行有 `syncRecordingStatus()`——這就是計畫裡「每輪 30 秒重開錄音時都會再 sync 一次」，名字互蓋的來源之一。

**為什麼會閃：** 計畫已寫「每 30 秒的新一輪不重送」，但 bestian 的「true 要成功啟動後才送」會把送出點推進 `startAudioRecording()`，而它每輪都跑；最直覺對稱寫法「`recorder.start()` 後送 true、`onstop` 送 false」兩個點每 30 秒各跑一次：

```
0 秒     啟動 → 送 true  → 名單出現「你」
30 秒    鬧鐘響、stop → 送 false → 「你」消失
30.0x 秒 下一輪啟動 → 送 true  → 「你」又出現
```

**解法：分清兩種停止。**

| 種類 | 觸發點 | 送 false？ |
| --- | --- | --- |
| 換段停止（使用者沒要停） | 1086 鬧鐘、1483 `stopAudioRecordingForNextRound` | **不送**，`audioRecordingActive` 仍 true |
| 真的停止 | `stopAudioRecording()` 1106、`handleAudioRecordingError()` 1097、`cleanupAudioRecording()` 1177 | **送**，session 層級結束 |

true 只在**第一輪**成功啟動後送一次：加旗標 `captureAnnounced`，第一輪 `recorder.start()` 成功後若旗標 false 就送 true 並設 true，之後每輪跳過；真的停止時送 false 並歸零。

相容橋補充：舊 `sync_recording` 每輪都送（1078），DO 收到時把那條連線 capturing 設 true；每輪都是 true 所以不閃，只是每 30 秒多廣播一次。真正的 false 來自 `stopAudioRecording` 1110 送的 `recording_speaker: null`。這就是為什麼 bestian 說舊訊息「只更新送出者自己的連線」就夠。

💡 **輪（round）**：一次 30 秒錄音。**session 層級 vs 輪層級**：session 是從按下到按停止整段，名單反映的是 session。**旗標（flag）**：記「某件事做過沒」的 true／false 變數。

### 7. 細講：換行、刪 `recordingDuration` 與計時器

現在的紅框（224–229 行）：`v-if="meetingData.recordingSpeaker && !isTranscripting"`、class `h-6 w-48 … rounded-full justify-center`。

| class | 意思 | 放名單會壞在哪 |
| --- | --- | --- |
| `w-48` | 寬 192px 固定 | 多人名字塞不下 |
| `h-6` | 高 24px 固定 | 只容一行，換了行第二行被切 |
| `rounded-full` + `justify-center` | 膠囊置中 | 多行形狀怪 |

改成最大寬度加自動高度（如 `max-w-64 h-auto px-3 py-1 whitespace-normal`）。

「已錄 N 秒」來自 computed `recordingDuration`（460）：現在時間減 `recordingStartTime`，後者從 DO 廣播的 `recording_synced` 來，**所有人看到同一數字**，所以叫共用秒數。Vue computed 只在響應式依賴變動時重算，`Date.now()` 不是響應式，所以 mounted 第 490 行開了每秒鬧鐘改 `recordingTimer`，computed 裡 `void this.recordingTimer` 是「故意碰一下讓 Vue 以為我依賴它」。按鈕上的倒數是 `recordingTimeLeft`（1082），本機從 30 倒數，另一套東西。共用秒數不顯示後整條鏈沒人用，要刪六處：

| 要刪 | 行號 |
| --- | --- |
| `recordingTimerInterval: null` | 415 |
| `recordingTimer: 0` | 416 |
| computed `recordingDuration` | 460–464 |
| mounted `setInterval` | 489–492 |
| unmount `clearInterval` | 529–532 |
| `this.recordingTimer = 0` | 1111 |

留著就是每秒醒一次卻什麼都不做的鬧鐘，白耗電、每秒重渲染。💡 **死程式碼**：沒人用但還在檔案裡，不報錯所以容易忽略。

### 8. 細講：「名牌重新反序列化後仍是最新狀態」

DO 現在只有兩處碰名牌：158 行 `server.serializeAttachment(meta)` 連線建立時貼一次；111 行 `ws.deserializeAttachment()` 每則訊息讀一次。名牌內容貼上後沒改過，所以現在沒事。步驟 2 要改 `capturing`，問題就來了。

Cloudflare 的設計：`deserializeAttachment()` 給**影本**；改影本原件不變；要改原件必須 `serializeAttachment()` 整張換掉。

```ts
const meta = ws.deserializeAttachment()
meta.capturing = true
ws.serializeAttachment(meta)     // 少這行，廣播時讀到的還是舊原件
this.broadcastCaptureList()
```

為什麼這樣設計：第 13 行註解說這個 DO 用 Hibernation API。房間沒動靜時 DO 睡著，記憶體清空，**只有名牌活下來**。

**測試要防的是替身藏 bug**。最偷懶的 `FakeWebSocket`：`serializeAttachment(v){this.att=v}`、`deserializeAttachment(){return this.att}`——回傳**原件本人**，DO 忘了貼回去也直接改到原件，測試假綠。正確替身兩邊都複製：`structuredClone(v)`。然後測：送 `capture_status: true` 後**再次** `deserializeAttachment()`，斷言 `capturing` 為 true。

| 替身寫法 | DO 忘了 serializeAttachment | 結果 |
| --- | --- | --- |
| 回傳同一物件 | 測試仍綠 | bug 被藏起來，上線才爆 |
| 回傳複製品 | 測試紅燈 | 開發時抓到 |

💡 **序列化／反序列化**：物件變成可存格式（拍照存檔）／讀回來變物件（沖洗出來，是新的一張）。**參照（reference）**：物件變數存的是地址，兩個變數指同一地址改一個另一個也變；`structuredClone` 做地址不同的完整複製。

### 9. 對照表：12 條測試案例（小明、阿華、小美）

| # | 情境 | 測試怎麼做 | 要看到什麼 | 沒測會漏的 bug | 層 |
| --- | --- | --- | --- | --- | --- |
| 1 | 小明按下麥克風 | 小明連線送 `capture_status: true` | 阿華收到含小明的 `capture_list` | DO 沒廣播或廣播錯人 | DO |
| 2 | 小明開兩分頁都在錄 | 兩條連線同 userId、都 capturing | 名單「小明」只一次 | 顯示「小明、小明」 | 純函式 |
| 3 | 小明關分頁 | 呼叫 `webSocketClose(小明)` | 阿華收到新名單，無小明 | 走了還掛著 | DO |
| 4 | 未登入訪客送 `capture_status` | `isAuthenticated: false` 連線送訊息 | 收到 error，名單沒變 | 任何人能塞進名單 | DO |
| 5 | 阿華用舊版只送 `sync_recording` | 假裝阿華送舊訊息 | 名單出現阿華 | 舊版使用者從名單消失 | DO |
| 6 | 小明在錄，小美才進來 | 檢查推給小美的 `init` | `init` 名單含小明 | 小美要等 30 秒才看到 | DO |
| 7 | 小明兩分頁都在錄，**關一個** | 對其中一條 `webSocketClose` | 名單**還有小明**；第二條也關才消失 | 關一個分頁就踢人 | DO |
| 8 | DO 改了小明名牌 | 送 true 後**再次** `deserializeAttachment()` | `capturing` 為 true | 忘了貼回去；替身需回傳複製品 | DO |
| 9a | 小明**拒絕麥克風權限** | 模擬 `getUserMedia` 失敗 | 沒送 true | 別人看到他在錄 | 前端 |
| 9b | 小明錄到一半**被登出** | 觸發 watcher 走 `cleanupAudioRecording()` | 有送 false | 永遠掛著 | 前端 |
| 9c | 小明正常錄，**30 秒換段** | 跑完一輪進第二輪 | 不再送 true、不送 false | 每 30 秒閃 | 前端 |
| 10a | 小明新版、阿華舊版**訊息交錯** | 兩種訊息交錯幾次 | 兩人都在，誰都不踢誰 | 舊「誰最後送誰贏」滲進新名單 | DO |
| 10b | 中途加入者 `init` 正確 | 同 6 加上 7、10a 混合 | `init` 與現場廣播一致 | `init` 另一套邏輯算出不同名單 | DO |
| 11a | 名字是中文 | session 名字 `encodeURIComponent` 進標頭，DO 解碼 | 顯示「王小明」非 `%E7…` | 標頭不能放中文；沒解碼顯示亂碼 | Worker + DO |
| 11b | 壞人自帶 `X-User-Name: bestian` | 外部請求自帶身分標頭 | DO 收到 session 真名，外部的被覆寫 | 冒充管理員 | Worker |
| 12 | 真瀏覽器再做一次 | 兩瀏覽器不同帳號手動操作 | 名單一致；一邊停另一邊即時更新；轉錄中名單仍在；關一分頁不影響另一個 | 替身模仿不到的真實行為 | 人工 |

分組記：基本動作 1、3、6；去重 2、7；安全 4、11b；相容橋 5、10a、10b；儲存正確 8；前端時機 9a–9c；編碼 11a；真實環境 12。

💡 **假綠**：測試通過但程式有 bug，第 8 條專防。**狀態類 vs 事件類訊息**：「我在錄」是狀態只有最新算數；「存這條逐字稿」是事件每筆都要到。

### 10. 細講：CLOSING 連線——離開的人會詐屍

DO 現在的 `webSocketClose`（132）只回 `ws.close(code, reason)`。計畫要在這裡重算名單並廣播、排除正在斷的連線。但名單從 `this.ctx.getWebSockets()`（219）算，而官方文件說：**叫了 `close()` 之後，這條連線有一小段時間還會出現在 `getWebSockets()` 裡**，狀態 CLOSING。

💡 **readyState**：0 CONNECTING、1 OPEN、2 CLOSING（站起來走向門口還沒跨出去）、3 CLOSED。

| 時間 | 發生什麼 | 名單 | 對不對 |
| --- | --- | --- | --- |
| 0 秒 | 小明、阿華都在錄 | 小明、阿華 | ✅ |
| 10 秒 | 小明關分頁，`webSocketClose` 用「排除這條 ws」算名單廣播 | 阿華 | ✅ |
| 10.5 秒 | 小美送 `capture_status: true`，DO 重算。`getWebSockets()` 還回傳小明的 CLOSING 連線，名牌 capturing 仍 true | 小明、阿華、小美 | ❌ 詐屍 |
| 12 秒 | Cloudflare 移除小明連線 | 阿華、小美 | ✅ |

只在 `webSocketClose` 那一次排除了他，**下一次由別人觸發的重算不知道要排除誰**。

| 解法 | 怎麼做 | 原理 |
| --- | --- | --- |
| 改名牌 | `webSocketClose` 一開始就把名牌設 `capturing: false` 並 `serializeAttachment()` | 之後誰觸發重算，讀到都是沒在錄；名牌隨連線被清掉不用善後 |
| 聚合時過濾 | 只看 `ws.readyState === 1` | CLOSING 直接跳過 |

兩個一起做最保險，`webSocketError` 比照。**不要另養記憶體 Set**：休眠清空記憶體；兩份真相會打架。💡 **單一真相來源**：一件事只存一個地方，其他都從它算。

### 11. 細講：斷線時畫面怎麼顯示——別把舊照片當直播

前端 `ws.onclose`（1380）只把 `this.ws` 設 null，**畫面資料一個字都不動**，也不重連。小明網路斷了，他畫面上「正在轉錄：阿華、小美」會**永遠停在斷線前那一刻**。💡 **快照 vs 即時**：照片 vs 直播；斷線後只剩照片但畫面沒說。

| 狀況 | 畫面 | 為什麼 |
| --- | --- | --- |
| 連線正常有人錄 | 正在轉錄：阿華、小美 | 正常 |
| 連線正常沒人錄 | 不顯示 | 正常 |
| **連線斷了** | 清空或「無法同步」 | 舊名單不能假裝即時 |
| **切歷史日期** | 清空 | 歷史走 REST，沒有即時名單 |

**程式碼陷阱**：`disconnectWs()`（1390）關閉前**先把 `onclose` 拔掉**（`this.ws.onclose = null`），切歷史日期走的 `connectToMeeting()`（1349）就是呼叫它。寫在 `onclose` 的清空邏輯切日期時不會跑。

| 斷線方式 | `onclose` 會跑？ | 清空該放哪 |
| --- | --- | --- |
| 網路自己斷 | 會 | `onclose` 可以 |
| 切歷史日期／離開頁面 | **不會** | `disconnectWs()` |

結論：放 `disconnectWs()` 一處，兩條路都經過。

**「名單為空不代表沒人在錄」兩層意思：**

- 別人：斷線後名單清空，阿華可能還在錄，只是收不到更新。用 `null` 表示「不知道」、`[]` 表示「確定沒人」：

| 值 | 意思 | 畫面 |
| --- | --- | --- |
| `['阿華', '小美']` | 確定這些人在錄 | 顯示名單 |
| `[]` | 確定沒人 | 不顯示 |
| `null` | 斷線，不知道 | 「無法同步」 |

- 自己：`audioRecordingActive` 與 WebSocket 是獨立變數，WS 斷了錄音器還在跑、30 秒換段還在、音檔還在排隊。**不能**看到名單空就停掉自己的錄音。

**重連後的訊息順序**：`sendMeetingMessage`（903）在連線非 OPEN 時把訊息排進 `wsPendingMessages`，`onopen` 一口氣沖出。若 `onopen` 先送目前狀態再沖佇列，佇列尾端過期的 `false` 會蓋掉。安全做法：**先沖佇列、最後送目前狀態**；或更乾淨——沖出前把佇列裡的 `capture_status` 全丟掉只送目前狀態，因為狀態類訊息只有最後一筆有意義（事件類如 `save_entry` 不能丟）。

### 12. 下一步

1. 回答第 4 節的三個自測問題。
2. `git fetch upstream`，從 `upstream/main` 開 `feat/153-capture-list`，依第 4、5 節的契約實作，測試照第 9 節 12 條。

## 2026-11-25 晚：步驟 2 實作完成（分支 `feat/153-capture-list`，尚未開 PR）

### 1. 進度

從 `upstream/main`（f9ff5de）開分支，五筆 checkpoint commit 皆已推到 fork `origin`，upstream 未碰：

| 步 | commit | 內容 |
| --- | --- | --- |
| 1 | 0826483 | `src/lib/capture-list.ts` 純函式 + 10 測試 |
| 2 | 303e368 | Worker `buildForwardHeaders()` 無條件覆寫身分標頭 + 5 測試 |
| 3、4 | d60fd0b | DO 名單聚合 + `helpers/fake-ws.ts` + 23 個 DO 測試 |
| 5、7 | d23b8b6 | 前端名單顯示、`announceCapture`、斷線顯示、刪計時器、i18n 三檔 |
| 6 | 50b0ec2 | 前端 19 個測試（送出時機、onopen 順序、斷線顯示） |

驗收（第 8 步）：`vp check --no-fmt --no-lint`、`vp run build`、`vp run lemma:gen`、`vp check`、`vp test`（32 檔 348 個）全綠。本機沒裝 Dafny，`lemma:check` 交給 CI。

### 2. 實作時學到的三件事

- **Node 建不出 101 的 Response**，所以 DO 的 WebSocket 升級路徑無法在測試中跑。解法：把「組 init 訊息」抽成公開方法 `initMessage(date)`，升級路徑呼叫它、測試直接呼叫它。這又是「純核心抽取」的變形。
- **變異測試**（mutation testing）：故意把一行關鍵程式拿掉，看測試會不會紅。拿掉 DO 的 `serializeAttachment()` 紅 8 個、拿掉前端 `announceCapture` 的去重守衛紅 5 個。紅了才證明測試真的在守那行；沒紅就是假綠。
- **`disconnectWs()` 會先拔掉 `onclose`**，所以切歷史日期時寫在 `onclose` 的清理不會跑。清名單要放 `disconnectWs()` 本身。測試「主動斷線 onclose 被拔掉仍會清名單」釘住這點。

### 3. 分隔符也要走翻譯

名單名字之間的分隔符中日文用「、」、英文用「, 」，寫死在程式裡會違反「介面文字一律走翻譯」，所以多加了 `jitsi.capturingListSeparator`。計畫裡原本沒這個 key。

### 4. 下一步

1. 本機雙瀏覽器實測（需 `.dev.vars` 與兩個帳號），填進 PR 草稿的勾選清單。
2. 以草稿開 PR 到 upstream（先給使用者看過）。

### 5. PR 說明草稿

## 標題

feat: 顯示所有正在轉錄的參與者（#153 步驟 2）

## 說明

依 #153 的方案 B 與 bestian 在 [issue 留言](https://github.com/g0v/vTaiwan-hono/issues/153#issuecomment-7115440143) 補的狀態契約實作。把按鈕旁「{某人} 錄音中」的單格顯示，換成「正在轉錄：甲、乙」名單。不加 migration、不碰逐字稿寫入路徑、不碰授權模型。

### 改動

| 層 | 改動 |
| --- | --- |
| `src/lib/capture-list.ts` | 純函式 `capturersFromMetas()`：過濾 capturing、以 userId 去重、依名字排序；`encodeUserName` / `decodeUserName` 處理中文名稱進出 HTTP 標頭 |
| Worker `meeting.ts` | 抽出 `buildForwardHeaders()`：`X-Authenticated` / `X-User-Id` / `X-User-Name` 一律依 session **無條件覆寫**，外部請求自帶的同名標頭不會穿透到 DO |
| DO `meeting-room.ts` | 名牌（attachment）加 `name` / `capturing`；新增 `capture_status`（只收布林、走既有登入門口之後）與 `capture_list` 廣播；`init` 帶現有名單。每次改 `capturing` 都重新 `serializeAttachment()`，沒變不廣播。`webSocketClose` / `webSocketError` 先把名牌標為沒在錄再關閉；聚合只看 `readyState === OPEN`，排除 CLOSING 連線 |
| 前端 `JitsiView.vue` | `announceCapture()`：錄音器**成功啟動後**才送 true，每 30 秒換段不重送；`stopAudioRecording` 與 `cleanupAudioRecording`（登出失去權限、錄音錯誤、離頁）送 false。`onopen` 丟掉佇列裡過期的 `capture_status`，最後依當下狀態補送一次。意外斷線清空名單並顯示「名單無法同步」；切歷史日期在 `disconnectWs()` 清空。顯示移到按鈕左側可換行，與待處理音檔數各自顯示，拿掉 `!isTranscripting` 條件。移除 `recordingDuration` 與每秒計時器 |
| i18n | 三檔新增 `jitsi.capturingList` / `capturingListSeparator` / `captureUnavailable`，移除 `jitsi.recordingStatus` |

### 相容橋（過渡用，日後移除）

- 保留 `recording_speaker` 欄位與 `sync_recording` / `recording_synced` 訊息。DO 收到舊版 `sync_recording` 時照舊寫 D1 與廣播，另外**只更新送出該訊息那條連線**的 `capturing`（`recording_speaker` 非 null 即在錄），不拿全房間的 `recording_speaker` 反推名單；名字取自名牌，不採信訊息內容。
- 新版名單只以 `init` / `capture_list` 為資料來源，不受 `recording_synced` 的單格狀態影響。
- **限制**：這讓新版頁面能看到仍用舊協定的參與者，但不會修好舊版頁面自己的單人顯示；部署後尚未再送出狀態的舊版頁面，不保證立刻出現在新名單中（要等它下一次 30 秒換段送 `sync_recording`）。
- **移除條件**：確認線上已無舊版頁面後（例如部署一週後），可移除前端 `syncRecordingStatus()`、DO 的 `sync_recording` 相容橋與 `recording_speaker` 的讀寫；`meeting_sessions.recording_speaker` 欄位可留著不用。

### 測試

自動測試（`vp test` 348 passed，新增 69 個）：

- `capture-list.test.ts`（10）：去重、排序、未登入不列入、中文名稱編碼往返與損毀處理。
- `meeting-auth.test.ts`（+5）：外部自帶身分標頭被覆寫、停權視為未授權、其他標頭保留。
- `meeting-room.test.ts`（23）：直接建構 DO，搭配 `helpers/fake-ws.ts` 的假 WebSocket（attachment 兩端 `structuredClone`，close 後留在 `getWebSockets()` 且為 CLOSING）、假 ctx、假 D1。呼叫真正的 `webSocketMessage` / `webSocketClose` / `webSocketError` / `initMessage`，斷言其他連線收到的名單：同帳號兩分頁其一關閉／停止另一仍在、attachment 重新反序列化仍是最新狀態、CLOSING 連線不詐屍、未登入被拒、訊息內塞別人身分無效、新舊訊息交錯不互踢、`init` 與現場廣播一致。
- `audio-recording.test.ts`（+7）與 `jitsi-capture-ws.test.ts`（12）：等待授權／拒絕權限／啟動失敗不送 true、跨三輪換段只送一次 true、登出與錄音錯誤送 false、`onopen` 沖出順序、意外斷線與主動斷線的名單顯示。
- 變異測試（手動）：拿掉 DO 的 `serializeAttachment()` 會紅 8 個；拿掉前端 `announceCapture` 去重守衛會紅 5 個。

這些單元測試驗的是應用邏輯，驗不到真實 Worker 的 WebSocket 升級、休眠與斷線行為。以下為本機雙瀏覽器實測：

<!-- 實測後填：每項寫「✅ / ❌ + 觀察到什麼」 -->

- [ ] 兩個瀏覽器登入不同帳號，各按紫色麥克風：兩邊名單一致（含自己）
- [ ] 一邊停止：另一邊即時更新
- [ ] 一邊的音檔送去轉錄期間（轉錄中提示出現）：名單仍可見
- [ ] 同帳號開兩個分頁都錄，關掉其中一個：另一個分頁仍在名單；關第二個才消失
- [ ] 錄音中拔網路／停掉 dev server：名單變灰顯示「名單無法同步」，本機按鈕倒數仍繼續
- [ ] 切到歷史日期：名單消失、不顯示無法同步
- [ ] 中文名稱正確顯示；三語切換文案正確
- [ ] 手機寬度：多人名單換行、不蓋住按鈕
- [ ] devtools console 無 hydration mismatch 警告

🤖 Generated with [Claude Code](https://claude.com/claude-code)
