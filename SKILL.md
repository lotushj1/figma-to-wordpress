---
name: figma-to-wordpress
description: 把 Figma 設計稿透過 Claude Code + elementor-mcp 直接切版部署到 Elementor WordPress 頁面。當用戶提到「Figma 切版」「Figma 變成 WordPress」「Figma → WP」「Figma → Elementor」「elementor-mcp」「裝 elementor MCP」「WP application password」或想把 Figma frame 轉成已上線的 WP 頁面時觸發此技能。第一次使用會主動引導完整安裝；之後每次進來會偵測狀態，主動問「要不要繼續切下一個 frame」。
---

# Figma → WordPress Builder

設計師用 Claude Code 把 Figma 設計稿一句話切成 Elementor WP 頁面。
這個 skill **主動帶著用戶走完整流程**：偵測狀態 → 補洞 → 抓 Figma → 翻譯 → 部署 → 驗證 → 問下一步。

---

## 🎯 主動引導的核心原則（優先級最高）

執行這個 skill 時，**每一個動作前都先講出來**，**每一個動作後都明確告訴用戶下一步要做什麼**。不要悶頭跑、不要假設用戶知道現在到哪了。

### 規則
1. **動作前先預告**：「接下來我要 X，預計 Y 秒，需要你做 Z」
2. **動作後先回報**：用 Markdown checklist 列結果，不要假設用戶看得到 terminal
3. **轉場一定問**：每個 phase / step 結束都明確問「要繼續 X，還是 Y」
4. **失敗就走 debug 分支**：不要丟給「cheatsheet」要用戶自己查；直接帶他跑診斷指令
5. **不要硬猜**：缺資訊就問用戶，特別是 URL、帳號、密碼，**絕對不要編造**

---

## 🚪 進場：先判斷使用者狀態

每次 skill 觸發，**第一件事**是這個分流：

### Step 0.1 偵測 elementor-mcp 設定狀態

跑：
```bash
test -f ~/.claude/settings.json && grep -q '"elementor-mcp"' ~/.claude/settings.json && echo "configured" || echo "not configured"
```

### Step 0.2 依結果分流

**回 `not configured`：** 第一次使用 → 講開場白 → 走 **Phase 1**

開場白範本（直接對用戶講）：
> 嗨！我看到這台還沒設定過 elementor-mcp。我會帶你跑完整安裝，預計 5–10 分鐘，分成五步：(1) 確認你有什麼 WP 站台、(2) 補裝 Node / Python / Chrome、(3) 引導你產 WordPress Application Password、(4) 寫進 Claude Code 設定檔、(5) 驗證能連線。
>
> 在開始前，先請你按 **Shift + Tab 兩次**切到 Bypass permissions 模式（畫面下緣會出現橘色 `bypass permissions`），不然安裝過程每個指令都會跳對話框打斷十幾次。
>
> 切好了告訴我，我們開始。

**回 `configured`：** 之前裝過 → 走 **Phase 0.5**

### Step 0.5 對裝過的人問「想幹嘛」

直接問：
> 看到你之前已經設定過 elementor-mcp。你現在想：
> 
> **A.** 切一個新的 Figma frame 到 WP 頁面（走 Phase 2 工作流）
> **B.** 換一個 WP 站台連線（重設 settings.json）
> **C.** 修舊頁面（之前切過、要再調整）
> **D.** 安裝有問題、要 debug（跑診斷）
> 
> 選哪個？

依用戶回答路由：
- A → Phase 2
- B → 跳到 Phase 1.4（重新寫 settings.json）
- C → Phase 2 但 step A 改問「要修哪個 page？」
- D → Phase 3 診斷

---

## Phase 1：第一次安裝（5–10 分鐘）

### 1.1 確認前提（先問清楚再動手）

直接問用戶：
> 開始安裝前先確認三件事：
> 1. 你的 WordPress 站台 URL 是什麼？（例如 `https://example.com`）
> 2. 站台有 Elementor 已啟用嗎？
> 3. 你有後台帳號（不是 email、是 username）嗎？
>
> 三個都有就告訴我，沒有的話我們先解決。

**任一項缺**：停下來引導用戶補齊：
- 沒 WP → 告訴用戶這個 skill 需要 WP 站，可以用 Local by Flywheel 本機跑、Zeabur 雲端裝、或客戶站
- Elementor 沒裝 → 給「WP Admin → Plugins → Add New → Elementor → Install Activate」步驟
- 沒帳號 → 跟站長要

**全有了**：講「OK 那開始」進 1.2。

### 1.2 偵測 OS 並準備套件管理員

對用戶講：
> 接下來我先檢查你的 OS 跟套件管理員（brew / winget），缺什麼會幫你補。

跑 `uname` 或 `wmic os get caption`：
- **macOS**：檢查 brew → `brew --version`
  - 沒裝：先 `xcode-select --install`（彈窗會問用戶按同意，告訴用戶這要 3–5 分鐘）→ 再 `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
- **Windows**：檢查 winget → `winget --version`
  - 沒裝：引導用戶從 Microsoft Store 安裝「App Installer」

每步完成跟用戶講「✓ brew 已就緒」或「⏳ 正在裝 Xcode CLT，請等系統彈窗」。

### 1.3 補裝 Node / Python / Chrome

跟用戶講：
> 我要檢查並安裝三個工具，缺什麼補什麼：Node.js 18+、Python 3.10+、Chrome。

依序檢查 + 必要時安裝：

| 工具 | 驗證指令 | Mac 指令 | Win 指令 |
|---|---|---|---|
| Node 18+ | `node -v`（不夠 18 也算缺） | `brew install node` | `winget install OpenJS.NodeJS` |
| Python 3.10+ | `python3 --version` | `brew install python@3.12` | `winget install Python.Python.3.12` |
| Chrome | `ls /Applications/Google\ Chrome.app` 或登入 path | `brew install --cask google-chrome` | `winget install Google.Chrome` |

每裝一個跑驗證確認，回報用 checklist：
> - [x] Node v20.10.0
> - [x] Python 3.12.1
> - [x] Chrome 已裝
> 全部就緒，下一步：產 WP Application Password。

### 1.4 引導產 WP Application Password

**這步不是 Claude 跑、是用戶手動跑**。Claude 要清楚講出每個點擊位置，等用戶回報「產好了」才繼續。

對用戶講：
> 接下來請你打開瀏覽器手動操作一次，我這邊等你產好回來告訴我。Application Password 是 WP 給 API 用的專用密碼，跟你登入用的密碼不同。
>
> 步驟：
> 1. 開 `<USER_WP_URL>/wp-admin` → 用後台帳密登入
> 2. 左側選單最下方 **Users** → **Profile**（中文版叫「個人資料」）
> 3. 滾到頁面最底，找 **Application Passwords** 區塊（中文「應用程式密碼」）
> 4. **New Application Password Name** 欄位填 `claude-code`
> 5. 按 **Add New Application Password** 按鈕
> 6. 跑出來的整組密碼**整段含空格**複製起來（類似 `xxxx xxxx xxxx xxxx xxxx xxxx`）
>
> ⚠️ 那串密碼只會顯示一次，沒複製到要重產一組。
> ⚠️ 整組含空格別漏，最常見錯誤是空格被吃掉導致 401。
>
> 產好了把這串密碼貼給我，順便確認你的 WP username 是什麼。我接下來會把你的資訊寫進 Claude Code 設定檔。

**用戶卡住的常見狀況**：
- 找不到 Application Passwords 區塊 → WP < 5.6 或被 plugin 隱藏。引導 `Tools → Site Health → Info` 確認版本，或建議啟用 Application Passwords plugin
- 卡在登入 → 跟站長要正確密碼

### 1.5 寫入 settings.json

收到用戶的 WP_URL / username / app password 後對用戶講：
> 收到。接下來我做三件事：
> 1. 備份你現有的 `~/.claude/settings.json` 成 `.bak`
> 2. 把 elementor-mcp 設定合併進去（不會動到你其他既有的 MCP）
> 3. 跑 `npx -y elementor-mcp --help` 把套件 cache 下來
> 
> 開始：

實際操作：
- `cp ~/.claude/settings.json ~/.claude/settings.json.bak`（原檔不存在就跳過）
- 用 `python3 -c` 讀現有 JSON，merge 進新的 mcpServers entry，寫回去
- 跑 `npx -y elementor-mcp --help`，確認套件下載成功

合併進去的內容：
```json
{
  "mcpServers": {
    "elementor-mcp": {
      "command": "npx",
      "args": ["-y", "elementor-mcp"],
      "env": {
        "WP_URL": "<USER_WP_URL>",
        "WP_APP_USER": "<USER_USERNAME>",
        "WP_APP_PASSWORD": "<USER_APP_PASSWORD>"
      }
    }
  }
}
```

完成後回報：
> - [x] 備份：`~/.claude/settings.json.bak` 已建立
> - [x] settings.json 已合併（保留你既有的 N 個 MCP）
> - [x] elementor-mcp 套件已 cache
> 
> 下一步：請你**完全關掉 Claude Code 再重開**（Mac 是 Cmd+Q quit，不是 reload；Windows 是右上角 X 整個關掉）。重開後告訴我，我會跑驗證。

### 1.6 驗證連線

用戶回來後跑：
```bash
claude mcp list
```

**預期看到** `elementor-mcp ✓ Connected`

### 1.7 ✅ 失敗診斷分支（不通的話走這）

如果 `claude mcp list` 沒顯示 elementor-mcp 或顯示 `✗ Failed`，**主動跑下面這串診斷**，不要丟給用戶：

**Diag 1：JSON 語法**
```bash
python3 -c "import json; json.load(open(__import__('os').path.expanduser('~/.claude/settings.json')))"
```
失敗 → 告訴用戶「settings.json JSON 語法壞了」→ 從 .bak 還原 + 重做 1.5

**Diag 2：直接打 WP REST**
```bash
curl -u "<USER>:<APP_PWD>" "<WP_URL>/wp-json/wp/v2/users/me"
```
- 401 → app password 空格被吃掉 → 引導用戶重產一組（回 1.4）
- 403 → security plugin（Wordfence／iThemes）擋 REST API → 引導加白名單
- 連不到 → WP_URL 寫錯（多斜線、http/https 錯）→ 改 1.5
- 200 → REST 通的，問題在 MCP 層，繼續 Diag 3

**Diag 3：套件下載問題**
```bash
npx -y elementor-mcp --help
```
- 失敗 → npm registry 連不到？網路問題？引導用戶看 `~/.npm/_logs/` 最後一個 log

**Diag 4：Claude Code 沒重開**
最常見：用戶以為 reload = 重開。提醒「真的要 Cmd+Q quit 整個 app 再開」。

每跑一個 diag 都回報結果給用戶，找到問題後回到對應步驟修。

### 1.8 ✅ Phase 1 結束，主動帶到 Phase 2

驗證通過後對用戶講：
> 🎉 安裝完成！elementor-mcp ✓ Connected
> 
> 接下來通常會做以下其中之一：
> 
> **A.** 馬上切第一個 Figma frame 試試看（推薦，趁熱跑一輪驗證流程）
> **B.** 之後再做（先收工）
> 
> 選 A 的話，請告訴我：
> - 你想切的 Figma frame URL（在 Figma 點 frame → 右鍵 Copy Link）
> - 你想部署到哪個 WP page ID（在 wp-admin Pages 列表點頁面，URL 列會看到 `post=<id>`）

選 A → Phase 2。選 B → 收工（給 graduation 訊息）。

---

## Phase 2：日常工作流（抓 → 翻譯 → 部署 → 驗證）

每次切一個 frame 走這流程。**每一步動作前都預告**。

### 2.A 收齊兩個 input

如果用戶只給一個就明確要另一個。**不要假設、不要猜**：
- 沒給 Figma URL → 問「請貼 Figma frame 的 Copy Link 連結（含 `?node-id=X-Y`）」
- 沒給 page ID → 問「要部署到哪個 page？告訴我 page ID 或 page URL，我從那邊抓 ID」

兩個都拿到後預告：
> 收到。接下來我會：(1) 抓 Figma 結構、(2) 翻譯成 HTML、(3) 部署、(4) 截圖比對。每步驟做完我會告訴你看到什麼，遇到問題會停下來問你。

### 2.B 抓 Figma 結構

對用戶講：
> 接下來我去抓 Figma frame 的結構，包括尺寸、文字、字型、圖片、效果。

**判斷用什麼工具：**
1. 跑 `claude mcp list | grep -i figma` 看有沒有 figma-mcp
2. 有 → 用 figma-mcp
3. 沒有 → 問用戶「你有 Figma Personal Access Token 嗎？沒有的話我用『下載 PNG 然後肉眼分析』fallback，會慢一點但能用」

抓完後**口頭跟用戶確認三件最容易踩的雷**：
> 抓到了。在開始翻譯前，我發現幾個會影響切版結果的地方，先跟你確認：
> 
> 1. **fills 陣列順序反直覺**：Figma 的 fills[0] 是最上層（不是最底）。我看到這個 frame 有 N 個圖層用了多重 fill，會反過來疊。
> 2. **圖片有 transform / filter**：（如果有）這幾張圖（列出來）有後製，我會用 `/v1/images?ids=NODE` export 才會跟你看到的一樣。
> 3. **疊加模式（multiply / screen）**：（如果有）這個元素 X 用了 blend mode 讓白底變透明。export 成 PNG 後白底會回來、視覺會崩盤。⚠️ 強烈建議請設計師重出素材（透明效果做進素材本身），否則我切完會跟原稿不一樣。
> 
> 這些都確認過了我就開始翻譯。如果有第 3 點，要先解決素材還是先讓我用近似結果先跑？

等用戶回應再進 2.C。

### 2.C 翻譯成 HTML widget

對用戶講：
> 開始翻譯。我會用 HTML widget 包整個 section，套用 Elementor full-bleed pattern（100vw stage + 1440 inner），先 reset e-con 預設 padding 跟 max-width 1140 避免全寬背景縮水。預計 30 秒。

產 HTML 內容。骨架：
```html
<section class="my-section">
  <div class="my-stage">
    <div class="my-inner">
      <!-- ...content from Figma... -->
    </div>
  </div>
</section>
<style>
.my-section, .my-section > .e-con-inner {
  --padding-top:0 !important; --padding-right:0 !important;
  --padding-bottom:0 !important; --padding-left:0 !important;
  --content-width:100% !important; --gap:0 !important;
  padding:0 !important; max-width:100% !important;
}
.my-stage { width:100vw; position:relative; left:50%; transform:translateX(-50%); }
.my-inner { max-width:1440px; margin:0 auto; padding:0 60px; }
</style>
```

注意事項（內建在翻譯邏輯裡，不用問用戶）：
- 字型用 Noto Sans TC / Noto Serif TC（中文站），行距比 Figma PingFang 原值縮 10–20%
- 顏色含 alpha 也要照搬
- 絕對定位的子元素，父容器一定要 `position:relative`
- 不要硬抗 Elementor 預設樣式，超過 3 個 `!important` 改 HTML widget 自己寫

### 2.D 部署

對用戶講：
> 翻譯完成。接下來部署到 page <ID>。⚠️ 部署前我先用 elementor-mcp 把現有 page 內容備份成 `/tmp/page-<ID>-backup-<timestamp>.json`，萬一要回滾。

執行：
1. 備份現有頁面（`get_page` 或 `download_page_to_file`）
2. 部署新內容（`update_page_from_file` 或直接傳 HTML）
3. 部署完跟用戶講：
> ✓ 部署完成。請開 `<PAGE_URL>?fresh=now` 看（query string 會強制清 Elementor 快取）。
> 
> 我接下來自動截圖跟 Figma 原稿做視覺比對，要不要等？

### 2.E 視覺驗證

對用戶講：
> 開始截圖比對。用 headless Chrome 拍 1440px 寬截圖，跟 Figma 原稿做 side-by-side。預計 15 秒。

跑：
```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --window-size=1440,6000 --hide-scrollbars \
  --screenshot=/tmp/wp-current.png \
  "<PAGE_URL>?fresh=now"
```

跟 Figma 原稿（再 export 一張）做比對，列出差異：
> 比對結果：
> 1. **位置差異**：（具體說哪個元素往哪偏 X px）
> 2. **顏色差異**：（具體說哪個地方）
> 3. **缺漏**：（具體說什麼）
> 
> 你要我：
> A. 直接修這三個再部署一次（推薦）
> B. 你自己看截圖再決定要改什麼
> C. 已經夠像了、收工

### 2.F Loop 直到吻合

選 A → 改 HTML → 回 2.D 部署 → 回 2.E 比對。

**典型情況**：3 輪內視覺會吻合。如果第 4 輪還沒吻合，主動跟用戶講：
> 改了 4 輪還沒完全吻合，可能是底層問題（不是文字／顏色微調）。
> 
> 常見原因：
> 1. Elementor cache 沒重建（試 `?fresh=now2` 或 WP Admin 手動 Regenerate CSS）
> 2. CSS 被 Elementor 預設樣式覆蓋（要改用 HTML widget 整段重寫）
> 3. Figma 跟 WP 的字型 metric 不同（Noto Sans TC vs PingFang），這個本來就會差幾 px，不要硬追
> 
> 你想先試哪個？

### 2.G ✅ 完成、問下一步

選 C 或視覺吻合後對用戶講：
> 🎉 這個 frame 切完了。Page URL：`<PAGE_URL>`
> 
> 接下來：
> A. 切下一個 frame（請貼 Figma URL + page ID）
> B. 修剛剛這個 page 某個地方
> C. 收工，下次再來
> 
> 選哪個？

---

## Phase 3：診斷模式（Phase 0.5 選 D 或安裝後出問題）

主動帶用戶跑這串診斷，按順序：

```
1. claude mcp list
   → 看 elementor-mcp 狀態

2. cat ~/.claude/settings.json | python3 -m json.tool
   → 看 JSON 是否合法

3. curl -u "<USER>:<APP_PWD>" "<WP_URL>/wp-json/wp/v2/users/me"
   → 看 REST API 是否能用（401 / 403 / 200）

4. npx -y elementor-mcp --help
   → 看套件是否能跑

5. node -v && python3 --version
   → 看基礎工具版本
```

每步跟用戶回報結果。找到問題就跳對應修復步驟（1.4 / 1.5 / 1.6 / 1.7）。

---

## ⚙️ 設定回報通用格式

每個重要動作完成都用 Markdown checklist 回報：
```
- [x] 完成的事
- [x] 確認的事
- [ ] 還沒做的事（標出來）
```

不要假設用戶看得到 terminal 輸出，重要結果在訊息裡複述一次。

## ⚙️ 互動原則

- 任何敏感資訊（帳號、密碼、token）絕對不在 prompt 裡寫死，要問用戶現場給
- 寫入 settings.json 前一定先備份成 `.bak`
- 部署前一定先備份目前 page 內容（線上站特別重要）
- 卡住先跑 Phase 3 診斷，不要叫用戶自己查文件

## ⚙️ 觸發此 skill 後第一件事的 mental model

1. 偵測狀態（裝過沒）
2. 講開場白／問用戶想幹嘛
3. 一步一步推進，**每步講 (a) 接下來要做什麼 (b) 做完是什麼 (c) 下一步要不要做**
4. 失敗就跳診斷，不要自己卡住、不要丟回給用戶
5. 完成就問下一步，不要靜默結束
