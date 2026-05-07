# Figma → WordPress (Claude Code Skill)

把 Figma 設計稿透過 Claude Code 一句話切版部署到 Elementor WordPress 頁面。

設計師原本要花 2–5 個工作天的 landing page，用這個流程一個下午能跑完：抓 Figma → 翻譯 HTML → 部署 → 截圖比對 → 迭代。

---

## 安裝

把這個 repo clone 到你的 Claude Code skills 資料夾就會自動載入：

**macOS / Linux：**
```bash
git clone https://github.com/lotushj1/figma-to-wordpress.git ~/.claude/skills/figma-to-wordpress
```

**Windows：**
```powershell
git clone https://github.com/lotushj1/figma-to-wordpress.git "$env:USERPROFILE\.claude\skills\figma-to-wordpress"
```

裝完 **重新啟動 Claude Code**（完全 quit 再開，不是 reload）。

---

## 使用

打開 Claude Code，講一句包含關鍵字的話就會觸發：

> 「幫我用 Figma 切版到 WordPress」
> 「我要把這個 Figma frame 變成 WP 頁面」
> 「幫我裝 elementor-mcp」
> 「Figma → WP」

第一次會引導你完成安裝（Node + Python + Chrome + WP MCP 設定），約 5–10 分鐘。
之後就直接走 pipeline：抓 Figma → 翻譯 HTML → 部署 → 截圖驗證。

---

## 你需要準備什麼

### 必要
- Claude Code（已安裝、已登入）
- WordPress 站台（有 Elementor、開啟 REST API）
- WP 後台帳號（會引導你產 Application Password）
- 一個 Figma 設計稿

### 可選
- Figma Personal Access Token（如果想用 Figma REST API 抓設計稿；裝 figma-mcp 也可）

---

## 這個 skill 做什麼

### Phase 1：第一次安裝
- 偵測 OS（Mac / Windows）
- 補裝 Node 18+ / Python 3.10+ / Chrome
- 引導你產 WP Application Password
- 寫入 `~/.claude/settings.json`（自動備份原檔成 `.bak`）
- 預載套件、提醒重開 Claude Code、驗證連線

### Phase 2：日常工作流
- 讀 Figma frame 結構
- 翻譯成 Elementor HTML widget（含 e-con reset、full-bleed pattern）
- 透過 elementor-mcp 部署到指定 WP page
- 用 headless Chrome 截圖跟 Figma 原稿比對
- 迭代直到視覺吻合

### 內建踩雷處理
- Elementor cache 怎麼清
- Application Password 401 怎麼救
- e-con 預設 padding 怎麼 reset
- Figma fills 順序反直覺
- 設計稿用了 blend mode 導致 export 失真

---

## 用什麼 MCP

底層用 [`elementor-mcp`](https://www.npmjs.com/package/elementor-mcp)（npm package）作為 Claude Code → WP 的橋。

如果你有 Figma Personal Access Token，搭配 figma-mcp 體驗更好（會自動讀 frame 結構），沒有也能用 REST API。

---

## License

MIT

---

## 回饋／問題

開 issue：https://github.com/lotushj1/figma-to-wordpress/issues
