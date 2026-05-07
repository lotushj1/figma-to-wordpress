---
name: figma-to-wordpress
description: 把 Figma 設計稿透過 Claude Code + elementor-mcp 直接切版部署到 Elementor WordPress 頁面。當用戶提到「Figma 切版」「Figma 變成 WordPress」「Figma → WP」「Figma → Elementor」「elementor-mcp」「裝 elementor MCP」「WP application password」或想把 Figma frame 轉成已上線的 WP 頁面時觸發此技能。第一次使用會引導完整安裝；之後直接走「抓 Figma → 翻譯 HTML → 部署 → 截圖比對」pipeline。
---

# Figma → WordPress Builder

設計師用 Claude Code 把 Figma 設計稿一句話切成 Elementor WP 頁面的 skill。
省掉手刻 HTML、來回對 Figma 的時間，把精力放在設計判斷。

---

## 使用流程

進來時先判斷「沒裝過 → 走 Phase 1」或「裝過 → 走 Phase 2」。

### 判斷裝過沒
跑這段檢查：
```bash
grep -q '"elementor-mcp"' ~/.claude/settings.json 2>/dev/null && echo "configured" || echo "not configured"
```

- 回 `not configured` → 走 **Phase 1：第一次安裝**
- 回 `configured` → 直接走 **Phase 2：日常工作流**

---

## Phase 1：第一次安裝（5 ~ 10 分鐘）

### 0. 確認前提

問用戶確認以下都有：
- 有 Claude Code 已登入（沒有就請他先去 `claude.com/claude-code` 裝完登入再回來）
- 有一個 WordPress 站台、有後台帳號、Elementor 已啟用
- WordPress 版本 5.6+（Application Password 是 5.6 以後才支援）
- WP REST API 是開的（沒被 security plugin 擋）

提醒用戶：**現在按 `Shift + Tab` 兩次切到 Bypass permissions 模式**（畫面下緣會出現橘色 `bypass permissions`），不然安裝過程會被權限對話框打斷十幾次。

### 1. 偵測 OS 並裝套件管理員

- macOS：用 Homebrew。沒裝就先裝 Xcode CLT（`xcode-select --install`）→ 再裝 brew。
- Windows：用 winget（Win 10+ 內建）。

### 2. 補裝必要工具

依序檢查、缺什麼補什麼：

| 工具 | 驗證指令 | 安裝（Mac） | 安裝（Win） |
|---|---|---|---|
| Node.js 18+ | `node -v` | `brew install node` | `winget install OpenJS.NodeJS` |
| Python 3.10+ | `python3 --version` | `brew install python@3.12` | `winget install Python.Python.3.12` |
| Google Chrome | 看應用程式 | `brew install --cask google-chrome` | `winget install Google.Chrome` |

裝完每項都跑驗證指令確認 PATH 抓得到。

### 3. 蒐集 WP 連線資訊

問用戶（不要硬猜）：
1. **WP 站台 URL**（例如 `https://example.com`，不要尾巴的斜線）
2. **後台用戶名**（不是 email，是 username）
3. **這個用戶的 Application Password** — 如果還沒產，引導他產：

> 引導產 Application Password：
> 1. 開 `<WP_URL>/wp-admin` 用後台登入
> 2. 左側 Users → Profile（或 Your Profile）
> 3. 滾到底找 **Application Passwords** 區塊
> 4. **New Application Password Name** 填 `claude-code`
> 5. 按 **Add New Application Password**
> 6. 複製跑出來的整組（含空格，類似 `xxxx xxxx xxxx xxxx xxxx xxxx`）
>
> 注意：複製時整組含空格別漏，Application Password ≠ 登入密碼。

### 4. 寫入 `~/.claude/settings.json`

- 路徑：macOS `~/.claude/settings.json`、Windows `%USERPROFILE%\.claude\settings.json`
- 檔案不存在 → 建立新檔
- 檔案已存在 → 先複製備份成 `settings.json.bak`，再把下面這段「合併」進 `mcpServers` 區塊（**絕對不要覆蓋掉其他既有的 MCP**）

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

把 `<USER_WP_URL>`、`<USER_USERNAME>`、`<USER_APP_PASSWORD>` 換成上一步收到的資訊（password 整組含空格）。

### 5. 預載套件 + 重開 Claude Code

```bash
npx -y elementor-mcp --help
```
讓 npm 把套件 cache 下來，之後啟動才不會等下載。

然後請用戶**完全關掉 Claude Code 再重開**（Cmd+Q quit，不是 reload）。

### 6. 驗證

重開後跑：
```bash
claude mcp list
```

應該看到 `elementor-mcp ✓ Connected`。

回報安裝結果給用戶（Markdown checklist 格式）：
- [ ] Node 版本：?
- [ ] Python 版本：?
- [ ] Chrome：已裝/已補裝
- [ ] settings.json 路徑：?
- [ ] settings.json.bak 備份：有/無原檔
- [ ] elementor-mcp ✓ Connected

如果 `claude mcp list` 沒看到、或顯示 `✗ Failed`：
- 檢查 settings.json JSON 語法
- 確認 application password 整組含空格（最常見問題）
- 確認 WP_URL 沒有尾巴斜線
- 確認 WP REST API 沒被擋（curl `<WP_URL>/wp-json/wp/v2/users/me` 用 basic auth 測）

安裝結束。告訴用戶可以接著跑 Phase 2。

---

## Phase 2：日常工作流（抓 Figma → 翻譯 → 部署 → 驗證）

每次要把一個 Figma frame 切成 WP 頁面就走這流程。

### A. 收 Figma URL

問用戶：
- 哪個 Figma frame？（請貼右鍵 Copy Link 出來的完整 URL，含 `?node-id=X-Y`）
- 哪個 WP page ID？（去 wp-admin Pages 找，URL 列會看到 `post=<id>`）

### B. 讀 Figma 結構

如果用戶有裝 figma-mcp，直接用：
> 請用 figma mcp 抓這個 frame 的完整結構：<FIGMA_URL>
> 列出整個 frame 的尺寸、每個元素的位置/寬高/文字、字型、顏色、圖片資產

沒裝 figma-mcp 就用 Figma REST API（要用戶提供 Figma Personal Access Token）：
```bash
curl -H "X-Figma-Token: <FIGMA_PAT>" \
  "https://api.figma.com/v1/files/<FILE_KEY>/nodes?ids=<NODE_ID>" \
  | python3 -m json.tool
```

讀完後跟用戶**口頭確認 3 件事**（這些都是切版時會踩的雷）：
1. **`fills` 陣列順序反直覺**：fills[0] 是最上層，不是最底層
2. **圖片有 transform/filter**：要走 `/v1/images?ids=<NODE>` export，不能直接抓 imageRef raw
3. **疊加模式（multiply / screen）造的視覺**：export 成 PNG 後白底會回來、視覺崩盤。如果有，要請設計師重出素材（透明效果做進素材本身）

### C. 翻譯成 HTML widget

用 Elementor html widget 包整個 section。預設模式：

```html
<section class="my-section">
  <div class="my-stage"><!-- 100vw 全寬 -->
    <div class="my-inner"><!-- max-width 1440 置中 -->
      <!-- ...content... -->
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

注意事項：
- 字型用 Noto Sans TC / Noto Serif TC（中文站），行距比 Figma PingFang 原值再縮 10–20%
- 顏色含 alpha 也要照搬
- 絕對定位的子元素，父容器一定要 `position:relative`
- 不要硬抗 Elementor 預設樣式，超過 3 個 `!important` 就改用 HTML widget 自己寫

### D. 部署到 WP

純對話路線（一段 HTML 直接丟）：
> 請用 elementor mcp 把下面這段 HTML 部署到 page <PAGE_ID>，status 設為 publish，完成後告訴我頁面 URL：
>
> <貼 HTML>

Builder 路線（多 section 大頁面用 Python 產 JSON）：
> 請用 elementor mcp 把 /tmp/my-page.json 部署到 page <PAGE_ID>，status 設為 publish

部署完開頁時 URL 加 `?fresh=now` 強制清 Elementor 快取。

### E. 視覺驗證

```
請幫我做視覺驗證：
1. 用 headless Chrome 截圖 <PAGE_URL>?fresh=now，window-size 設 1440x6000，存到 /tmp/wp-current.png
2. 對照原 Figma frame
3. 列出最明顯的 3 個差異（位置 / 顏色 / 尺寸 / 缺漏），按重要性排序
```

跟用戶確認差異 → 改 HTML → 部署 → 再驗證，每輪約 30 秒，2–3 輪就會吻合。

---

## 常見踩雷（卡住超過 5 分鐘就翻這段）

### 連線類

**401 Unauthorized**：
- Application Password 複製時空格被吃掉 → 重產一組重貼
- WP REST API 被 security plugin 擋（Wordfence、iThemes Security）→ 暫時加白名單或關 plugin

**部署成功但前台沒變**：
Elementor cache 沒重建。3 選 1：
- URL 加 `?fresh=now`
- WP Admin → Elementor → Tools → Regenerate CSS & Data
- 部署成 `status: draft` 再切 `publish`

### 切版類

**`widget` 上設的 `css_classes` 沒 render**：
只有 container 才會輸出 css_classes。CSS selector 改用 `.elementor-element-<widget_id>`。

**`e-con` 容器內建 padding + max-width 1140**：
全寬背景永遠縮成 1140。Section 加 reset：`--padding-*:0`、`--content-width:100%`、`max-width:100% !important`。

**絕對定位跑版／子元素跳到下段**：
父容器沒 `position:relative`。改用 HTML widget 自己寫 wrapper。

**對抗 Elementor 預設樣式 > 3 個 `!important`**：
停手改 HTML widget 整段自己寫。

### Figma 端

**圖片 export 後跟 Figma 看到的不一樣**：
- 設計師用了 blend mode（multiply / screen 等）讓白底變透明
- 解：請設計師把透明效果做進素材本身、上線前先 flatten

**`imageRef` 抓到的圖跟 Figma 顯示的不一樣**：
node 上有 transform 或 filter。改用 `/v1/images?ids=<NODE_ID>` export。

---

## 設定回報

每次重大動作（裝完、部署、發現差異）都用 Markdown checklist 回報用戶。
不要假設用戶看得到 terminal 輸出，重要結果在訊息裡複述一次。

## 互動原則

- 任何敏感資訊（帳號、密碼）不要在 prompt 裡寫死，要問用戶現場給
- 寫入 settings.json 前一定先備份成 `.bak`
- 部署前一定先 `download_page_to_file` 備份目前 page 內容（如果是線上站）
- 卡住先翻「常見踩雷」這段，再丟 GitHub issue
