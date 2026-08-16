# tw-intake

從 iPhone 分享選單一鍵把連結丟回 Mac，經 Tailscale SSH 灌進一個常駐 tmux session，
由 `claude --dangerously-skip-permissions` 自動處理——**射後不理**。

這個 repo 是 wrapper 腳本與教學文的**備份真相源**。實際部署在 iMac
（`Chrisde-iMac-3.local`）的 `~/.local/bin/`，此處保留一份可還原的副本。

## 檔案

| 檔案 | 部署位置 | 說明 |
|------|----------|------|
| `tw-intake` | `~/.local/bin/tw-intake` | 主 wrapper：驗證 URL → 正規化 → 灌指令進 `tw-intake` session |
| `tw-intake-batch` | `~/.local/bin/tw-intake-batch` | 清單檔批次模式（由 `tw-intake --list <檔>` 或直接傳檔案路徑委派） |
| `tw-intake-教學.md` | `~/Documents/tw-intake-教學.md` | 完整架構、Mac 端設定、iPhone 捷徑設定、疑難排解 |

## 三種模式（同一個 iPhone 捷徑，wrapper 自動分流）

分享的連結原樣傳給 wrapper，由它依 URL 判斷：

- **翻譯（預設）** — x.com／一般網頁 → `/twitter-article-zh <url>`（翻成繁中存 `twitter_article`）；
  YouTube → 影片整理進 `youtubeKnow`。
- **易讀** — `tw-intake --easy <url>`（另一個捷徑）：翻譯後再多產一篇 `article-zh-easy.md`。
- **note（2026-08-16 新增，自動偵測）** — 分享的是某篇已翻好譯文
  `.../twitter_article/.../article-zh.md` 的 GitHub 網址時，改灌 `/learning-note`：
  讀全文 → 寫 digest → 用 `add_note_links.py` 在該篇**文末插入「寫成讀書筆記」的預填連結**。
  只備好入口，不代寫想法。

## 從本機同步這份備份

wrapper 真身在 `~/.local/bin/`，此 repo 是副本。改完 wrapper 後：

```bash
cp -p ~/.local/bin/tw-intake ~/.local/bin/tw-intake-batch ~/Documents/tw-intake-repo/
cp -p ~/Documents/tw-intake-教學.md ~/Documents/tw-intake-repo/
cd ~/Documents/tw-intake-repo && git add -A && git commit -m "sync from ~/.local/bin" && git push
```

還原到新機：把三個檔複製回上表的部署位置、`chmod +x` 兩支腳本，其餘照教學文設定
Tailscale／SSH 金鑰／tmux session。

## 相關

- 內容 repo：`chrisincite/twitter_article`（譯文歸檔）
- skill 傘 repo：`chrisincite/housearch_skill`（`twitter-article-zh`、`learning-note`）
