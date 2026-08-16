# 用 iPhone 分享選單一鍵把 X 推文丟回 Mac 自動翻譯

> 在 X app 看到一篇推文／長文 → 點「分享」→ 點一個捷徑 → 人就可以走了。
> Mac 那頭會自動抓取全文與圖片、翻成繁體中文、存檔並 push 到 GitHub。
> 全程不用回桌前、不用打字、不用碰指令。

本文記錄 2026-06-29 實作並端到端驗證通過的完整流程，供日後換機或調整時重建。

---

## 一、它是怎麼運作的（架構）

```
iPhone X app
   │ 分享 → 捷徑「存這篇推文」
   ▼
從分享表單取得 URL
   │
   ▼  Tailscale 內網（綁機器名，不綁浮動 IP）
SSH（ed25519 金鑰，免密碼）
   │
   ▼  iMac
wrapper 腳本  ~/.local/bin/tw-intake <url>
   │  ① 驗證只收 http(s) URL（擋非網址的亂打文字）
   │  ② 依來源正規化（X 砍 ?#；YouTube 保留影片 ID；一般網頁只去 #、保留 query）
   │  ③ 確保專用 tmux session 存在
   ▼
tmux send-keys 灌指令進常駐 session
   │
   ▼
claude --dangerously-skip-permissions
   執行  /twitter-article-zh <url>              ← 一般連結：抓取 → 翻譯 → 寫檔 → push
   執行  /learning-note <article-zh.md 網址>    ← note 模式：寫 digest → 文末加「寫成讀書筆記」預填連結
   │
   ▼
GitHub repo  twitter_article
```

**核心設計哲學**：不改動原本的 `twitter-article-zh` skill 引擎，只給它接一個乾淨的「觸發頭」。抓取、翻譯、產出、推回 repo 的引擎原封不動。

> **2026-06-30 起支援 YouTube 與任何一般文章網址。** 分享任何連結時，同一個捷徑、同一條鏈路照走；wrapper 放行所有 http(s) URL 並依來源正規化，skill 的 `fetch.sh` 會自動分流：X→翻譯文進 `twitter_article`、YouTube→影片整理（封面＋逐字稿＋文章＋社群短文）進 `youtubeKnow`、一般網頁→翻譯文進 `twitter_article`。**捷徑本身不用改**（它只是把分享到的 URL 原樣傳給 wrapper）。

> **2026-08-16 起新增 note 模式（自動偵測，捷徑一樣不用改）。** 當你分享的是某篇「已經翻譯好的譯文」`article-zh.md` 的 GitHub 網址（`github.com/chrisincite/twitter_article/…/article-zh.md`，`blob`／`tree`／`raw` 或 `raw.githubusercontent.com` 都認），wrapper 會認出這不是要翻譯的新文章，而是要為它做 [[learning-note]]——於是改灌 `/learning-note`：先讀該篇全文、寫好 digest（摘要＋逐字核對的金句），再跑 `add_note_links.py --only <網址> --apply`，在那篇譯文**文末插入一個「寫成讀書筆記」的 GitHub 預填連結**（檔名＋標題＋出處＋摘要＋金句都預先填好）。你日後在手機上點那個連結，就直接開啟一個「檔名與內容都填好」的 GitHub 編輯器，只要寫上自己的想法即可。分享推文／一般網頁／YouTube 仍照舊翻譯，互不干擾。**這個模式只「備好入口」，不會代寫你的觀點**（learning-note 鐵則一）。因為是無人值守的自動觸發，灌進去的指令已明令 session：不寫正文、不揣測你的想法、不提問、不碰網站 repo；跑完會自動 `git -C ~/Documents/twitter_article pull --rebase` 讓本機 clone 跟上（腳本改的是遠端）。

**為什麼選這條路（而非 iMessage 監看 chat.db）**：不用讀私有資料庫、不用解 `attributedBody`、沒有寄件人偽造風險。代價是每次要主動在分享選單點一下——但分享連結本來就是你看到推文當下最自然的動作。

**為什麼能「射後不理」**:SSH 進去只負責 `send-keys` 把指令灌進一個**已經跑著的**互動 Claude session，灌完秒返回、SSH 立刻斷。真正的長工作在那個常駐 session 裡跑，跟手機端脫鉤。

---

## 二、前置條件

| 項目 | 說明 |
|------|------|
| Mac（被觸發端） | 已安裝並登入過 Claude Code；已裝 tmux；twitter-article-zh skill 可用 |
| Tailscale | iPhone 與 Mac 都登入**同一個 tailnet**、MagicDNS 開啟 |
| Mac 遠端登入 | 系統設定 → 一般 → 共享 → **遠端登入（Remote Login）開啟**，允許該使用者 |
| iPhone | 裝 Tailscale app 並連線；用內建「捷徑」App |

**關鍵觀念：登入是「整台機器一次性」的事。** Claude Code 的憑證存在機器層級（Keychain／`~/.claude`），所有 `claude` 程序共用。只要這台 Mac 登入過一次，新開的 tmux session 跑 `claude` 會直接讀那份憑證、跳過 login。**不需要每次重登。**

---

## 三、Mac 端設定

### 3-1　建立 wrapper 腳本

存到 `~/.local/bin/tw-intake`（這目錄需在 PATH；不在就自己加，或改用絕對路徑呼叫）：

> ⚠️ **下方這份是 2026-06-29 的初版骨架，只保留當作教學說明。** 實際線上版已多出 `--easy`（易讀版）、`--list`／清單檔（批次委派 `tw-intake-batch`）、以及 note 模式三項功能，內容較長。**真身以備份 repo `chrisincite/tw-intake` 的 `tw-intake` 檔為準**（連同 `tw-intake-batch`），別直接抄下方骨架覆蓋線上版。

```bash
#!/bin/bash
# tw-intake — 從 iPhone Shortcut 經 SSH 觸發：把一則 X 連結灌進專用 tmux session 跑 twitter-article-zh
# 用法： tw-intake <x-url>
# 設計：idempotent（session 沒有就建、有就用）、切掉追蹤參數、只收 x/twitter 連結、射後不理。

set -u

# SSH 非登入 shell 的 PATH 可能很空，硬指死關鍵工具位置（依你的機器調整）
export PATH="$HOME/.local/bin:/usr/local/bin:/usr/bin:/bin"

SESSION="tw-intake"
REPO_DIR="$HOME/Documents/twitter_article"
LOG="$HOME/.local/share/tw-intake.log"
mkdir -p "$(dirname "$LOG")"

log() { printf '%s  %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*" >> "$LOG"; }

URL="${1:-}"

# --- 1. 驗證輸入：只收 http(s) URL，擋掉亂灌任意文字 ---
if [[ -z "$URL" ]]; then
  log "REJECT 空輸入"
  echo "錯誤：沒有提供 URL" >&2
  exit 1
fi

# --- 2. 依來源正規化 ---
#   X：切掉 ?# 後的追蹤尾巴。
#   YouTube：保留影片 ID，丟掉 &si/&list/&t（直接砍 ? 會連 v= 一起砍掉，URL 會壞）。
#   一般網頁：只去 # 錨點、保留查詢字串（文章站的 ?id/?page 常是必要參數，不能盲砍）。
if [[ "$URL" =~ ^https?://(www\.)?(x|twitter)\.com/ ]]; then
  CLEAN="${URL%%\?*}"; CLEAN="${CLEAN%%#*}"
elif [[ "$URL" =~ ^https?://(www\.|m\.)?youtube\.com/watch ]]; then
  VID="$(printf '%s' "$URL" | sed -n 's/.*[?&]v=\([A-Za-z0-9_-][A-Za-z0-9_-]*\).*/\1/p')"
  [[ -z "$VID" ]] && { log "REJECT youtube watch 無 v 參數：$URL"; echo "錯誤：YouTube 連結缺 v 參數" >&2; exit 1; }
  CLEAN="https://www.youtube.com/watch?v=$VID"
elif [[ "$URL" =~ ^https?://(www\.)?youtu\.be/([A-Za-z0-9_-]+) ]]; then
  CLEAN="https://youtu.be/${BASH_REMATCH[2]}"
elif [[ "$URL" =~ ^https?://(www\.|m\.)?youtube\.com/(shorts|live)/([A-Za-z0-9_-]+) ]]; then
  CLEAN="https://www.youtube.com/${BASH_REMATCH[2]}/${BASH_REMATCH[3]}"
elif [[ "$URL" =~ ^https?://[^[:space:]]+ ]]; then
  CLEAN="${URL%%#*}"
else
  log "REJECT 非 URL：$URL"
  echo "錯誤：只接受 http(s) 開頭的網址" >&2
  exit 1
fi

# --- 3. 確保專用 session 存在且 claude 跑著 ---
if ! tmux has-session -t "$SESSION" 2>/dev/null; then
  log "建立 session $SESSION 並冷啟動 claude"
  tmux new-session -d -s "$SESSION" -c "$REPO_DIR"
  tmux send-keys -t "$SESSION" 'claude --dangerously-skip-permissions' Enter
  sleep 8   # 等 claude 開到提示符（冷啟動才需要）
fi

# --- 4. 先清掉上一篇的上下文，避免常駐 session 累積到爆 ---
#    前提：上一篇已跑完、session 閒置（偶爾丟一篇的常態用法都成立）。
#    若上一篇仍在跑，這個 /clear 可能打斷它——要連續批次丟多篇請改用「每篇獨立 session」架構。
tmux send-keys -t "$SESSION" -l "/clear"
tmux send-keys -t "$SESSION" Enter
sleep 1   # 給 /clear 一點時間落地，避免下一行緊接著被吃掉

# --- 5. 灌指令：-l 字面送出避免按鍵名被解讀，Enter 另送一發 ---
tmux send-keys -t "$SESSION" -l "/twitter-article-zh $CLEAN"
tmux send-keys -t "$SESSION" Enter

log "SENT $CLEAN"
echo "已送出：$CLEAN"
```

設定可執行：

```bash
chmod +x ~/.local/bin/tw-intake
```

> **腳本防呆的用意**
> - 只收 `http(s)://` 開頭：擋掉誤觸或亂灌的非網址字串（安全主要靠 Tailscale 內網＋SSH 金鑰雙重收斂，網域白名單已放寬成任意 URL）。
> - 依來源正規化：X 切 `?`/`#` 追蹤尾巴；YouTube 保留 `v=` 影片 ID 只丟追蹤參數；一般網頁保留 query（只去 `#`）。
> - session idempotent：開機後 session 不存在也沒關係，第一次觸發會自動建立並冷啟動 claude。

### 3-2　第一次手動啟動專用 session（過一次性關卡）

```bash
tmux new-session -d -s tw-intake -c ~/Documents/twitter_article
tmux send-keys -t tw-intake 'claude --dangerously-skip-permissions' Enter
tmux attach -t tw-intake
```

attach 進去後會遇到 **Bypass Permissions 警告**，用方向鍵選 `2. Yes, I accept` 按 Enter。**這個每台機器只問一次**，之後 wrapper 自動跑就不再跳。

- `--dangerously-skip-permissions`：讓這個無人看管的 session 跑到 `git push`／寫檔時**不會卡在等核可**。觸發源已被 Tailscale ＋ SSH 金鑰雙重收斂，此範圍內可控。
- `-c <repo 目錄>`：把工作目錄設在一個**已信任過的目錄**，避開首次啟動的「Do you trust the files in this folder？」提示。

確認 attach 進去**沒有叫你 login**（直接到輸入提示符）就代表機器憑證 OK。離開用 `Ctrl-b d`（detach，不要 `Ctrl-c` 或 `exit`，那會關掉 session 裡的 claude）。

### 3-3　裝 iPhone 的公鑰（免密碼）

先在 iPhone 捷徑裡產生 SSH 金鑰（見第四節），複製公鑰，再到 Mac 執行（把 `<貼上公鑰>` 換成那一整行）：

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo '<貼上公鑰，例如 ssh-ed25519 AAAA... 註解>' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

> 權限很重要：`~/.ssh` 要 `700`、`authorized_keys` 要 `600`，否則 sshd 會拒用金鑰。

### 3-4　查你的 Tailscale 連線位址（給捷徑用）

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale status --self --json \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["Self"]["DNSName"])'
```

會得到類似 `chris-imac-1.tail53a80f.ts.net.` 的**完整 MagicDNS 名稱**。捷徑主機欄就填這個——**用名稱不用 IP**，這樣浮動 IP 也不影響。

---

## 四、iPhone 捷徑設定

開「捷徑」App → 新增捷徑，命名例如「存這篇推文」。

### 動作順序

1. **（自動）接收分享輸入** — 右上設定（ⓘ）→ 開「**在分享工作表中顯示**」→ 接受類型勾 **URL**（可順手加文字）。
2. **「從輸入取得 URL」**（Get URLs from Input） — X app 分享有時給的是「含網址的一段文字」，這步把乾淨網址抽出來。
3. **「透過 SSH 執行工序指令」** — 填：

   | 欄位 | 值 |
   |------|-----|
   | 指令稿 | `/Users/<你>/. local/bin/tw-intake "「URL變數」"`（見下方說明） |
   | 主機 | `chris-imac-1.tail53a80f.ts.net`（你的完整 MagicDNS 名） |
   | 連接埠 | `22` |
   | 使用者 | 你的 Mac 帳號 |
   | 認證 | **SSH 金鑰**（選 ed25519，讓它產生，複製公鑰回 3-3 用） |

### 指令稿那行的兩個地雷（最容易失敗）

```
/Users/<你>/.local/bin/tw-intake "「URL變數」"
```

- ❶ 引號中間必須插入「**真的變數**」——也就是上一步「取得 URL」的輸出（會顯示成**藍色膠囊標籤**）。**不要照字面打「URL變數」這幾個字**，否則送過去的是字串而非網址，被 wrapper 擋掉。
- ❷ **結尾不要留逗號**或多餘字元；逗號會黏到網址尾巴。
- 用**絕對路徑**呼叫 wrapper:SSH 非登入 shell 不一定有 `~/.local/bin` 在 PATH。
- 變數用**雙引號**包住：保護網址裡的 `&`。

---

## 五、測試

> ⚠️ **不能用捷徑編輯畫面的 ▶️ 試跑來測**——那沒有「分享進來的 URL」，「取得 URL」會拿到空值，wrapper 正確地擋下並回「錯誤：沒有提供 URL」。這是設計，不是 bug。
> （補充：能在 Mac log 看到 `REJECT 空輸入`，反而證明 SSH 連線與 wrapper 執行都通了。）

**正確測法：去 X app，打開一則推文 → 分享 → 點「存這篇推文」。**

預期：

1. 手機跳通知「**已送出：https://x.com/...**」。
2. Mac 的 `tw-intake` session 自動接住，開始跑 `/twitter-article-zh`。
3. log 多一行 `SENT https://x.com/...`（不是 REJECT）。
4. 跑完 push 進 `twitter_article` repo。

在 Mac 上查證：

```bash
tail -3 ~/.local/share/tw-intake.log          # 看有沒有 SENT
tmux capture-pane -t tw-intake -p | tail -20  # 看 session 是否在跑 skill
```

---

## 六、長期維運

- **Mac 別系統睡眠**：睡眠時任務會等喚醒才補跑。設成「螢幕睡、系統不睡」最穩。
- **context 自動清**：wrapper 每次送 skill 指令前會先送 `/clear`，所以常駐 session 只裝「當前這一篇」，不會累積到爆。前提是上一篇已跑完、session 閒置（偶爾丟一篇的常態用法都成立）。若 session 行為怪了，直接 `tmux kill-session -t tw-intake`，下次 wrapper 會自動重建。
- **連續批次丟多篇的限制**：常態用法（一次一篇、跑完再丟下一篇）沒問題；但若在前一篇還沒跑完時就丟下一篇，後者的 `/clear` 可能打斷前者。真要常態連丟多篇，改成「每篇開獨立 session」架構（命名帶序號、跑完回收）才完全免干擾。
- **note 模式會改動遠端 `twitter_article`**：`add_note_links.py` 用 GitHub API 直接在遠端建 commit，本機 clone 不會自動同步。wrapper 的 note 指令已在尾端自動 `git -C ~/Documents/twitter_article pull --rebase` 讓本機跟上，避免下次翻譯 push 撞 non-fast-forward；若哪次 note 沒跑完就中斷，手動補跑一次 pull 即可。

---

## 七、疑難排解

| 症狀 | 可能原因 | 處理 |
|------|----------|------|
| 捷徑連不上、逾時 | Tailscale 沒連／Mac 睡了／遠端登入沒開 | 確認兩端 Tailscale 連線、Mac 醒著、共享→遠端登入開啟 |
| log 一直 `REJECT 空輸入` | 用編輯畫面 ▶️ 試跑，或變數沒插對 | 改從 X 分享測；確認指令稿是藍色變數膠囊不是文字 |
| log `REJECT 非 X/YouTube 連結` | 「取得 URL」抽到非 x.com/twitter.com/YouTube 網址 | 確認分享的是推文或 YouTube 連結；必要時在捷徑加篩選 |
| 連線要求輸入密碼 | 公鑰沒裝好或權限不對 | 檢查 `authorized_keys` 內容、`~/.ssh` 700 / 檔案 600 |
| session 收到指令但 claude 卡住等核可 | 不是用 `--dangerously-skip-permissions` 啟動 | 用該旗標重啟 session，首次選 Yes,I accept |
| 首次啟動跳資料夾信任 | session 工作目錄沒信任過 | `tmux new-session -c <已信任目錄>`，或按一次同意 |

---

## 八、踩過的坑（都已解）

1. **以為開新 session 要重新 login** → 不用，登入是機器層級一次性，新 session 繼承憑證。
2. **headless 權限卡死** → `--dangerously-skip-permissions` 跳過互動核可，首次手選接受一次。
3. **資料夾信任提示** → `-c` 指到已信任目錄躲開。
4. **捷徑指令稿把「URL變數」打成文字** → 要插真的藍色變數膠囊；結尾別留逗號。
5. **編輯畫面 ▶️ 試跑測不出來** → 必得從 X 分享表單真測。
6. **追蹤參數的 `&` 炸 shell** → wrapper 自己切掉 `?`/`#` 後綴。
7. **浮動 IP** → 用 Tailscale 完整 MagicDNS 名稱，不綁 IP。
