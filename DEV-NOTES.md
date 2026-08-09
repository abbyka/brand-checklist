# brand-checklist — 開發筆記

> 呢個 repo 有兩個獨立頁面：品牌 checklist（`index.html`）同帳號密碼庫（`accounts.html`）。
> 部署喺 GitHub Pages，push 上 `main` 之後大約 1–2 分鐘自動上線。

## 網址

| 頁面 | 網址 |
|---|---|
| Checklist | https://abbyka.github.io/brand-checklist/ |
| 帳號密碼庫 | https://abbyka.github.io/brand-checklist/accounts.html |

帳號庫要用**帶密鑰嘅專屬連結**（`accounts.html#k=<密鑰>`）先開到。條密鑰只存喺用家自己部機 —— repo、伺服器、任何 log 都冇。

## 本機開發

```bash
cd /Users/abby/Downloads/brand-checklist
python3 -m http.server 8899
```

行完開 http://localhost:8899 。純靜態檔案，冇 build step、冇 npm。

⚠️ **WebCrypto 淨係喺「安全內容」（secure context）先有得用。** 呢個係最易踩嘅雷：

| 網址 | `crypto.subtle` | 開唔開到個庫 |
|---|---|---|
| `https://…` | ✅ | ✅ |
| `http://localhost:8899` | ✅（瀏覽器特別豁免） | ✅ |
| `http://127.0.0.1:8899` | ✅（同上） | ✅ |
| `http://192.168.x.x:8899` | ❌ `undefined` | ❌ 粒掣灰晒 |
| `file://…/accounts.html` | ❌ | ❌ |

所以**用內網 IP 喺手機預覽係唔得嘅**，`localhost` 嗰個豁免唔覆蓋 LAN IP。要喺手機試就要真 HTTPS：
用 `openssl req -x509 … -addext "subjectAltName=IP:<你嘅內網IP>"` 開張自簽證書，
再用 `ssl.SSLContext` 包住個 `ThreadingHTTPServer`。手機第一次會彈證書警告，撳「繼續前往」
就得 —— 用家自己接受咗嘅自簽證書一樣算安全內容。

順便：假後端最好**serve 喺同一個 origin**（`SUPABASE_URL = ''`，行相對路徑 `/rest/v1/vault`），
咁就唔會有 mixed-content，亦都唔使叫用家接受第二張證書。

## 後端：Supabase

同一個 project 俾兩頁共用（anon key 係 publishable，公開喺 HTML 入面係預期之內）。

```
URL : https://umbpnhbnyjgqmoilxbfu.supabase.co
Key : sb_publishable_l2VM6-NBoj3q6KbakOsB9A_iIyoUu4-
```

| 表 | 用途 |
|---|---|
| `checklist_state` / `custom_todos` / `todo_people` / `life_logs` | checklist（明文，anon 可讀寫） |
| `vault` | 帳號密碼庫（**只存密文**） |

`vault` 建表 SQL：

```sql
create table if not exists vault (
  id text primary key,
  payload jsonb not null,
  updated_at timestamptz default now()
);
alter table vault enable row level security;
create policy "vault_read"   on vault for select to anon using (true);
create policy "vault_insert" on vault for insert to anon with check (true);
create policy "vault_update" on vault for update to anon using (true) with check (true);
```

冇開 delete policy —— 防止有人一次過清走。

## 帳號庫加密設計（信封式）

啲資料**唔會**直接用主密碼加密。改用一條隨機 DEK，DEK 再俾兩把唔同嘅鎖各鎖一次。
`accounts.html` 入面有**兩個獨立嘅庫**，各有自己嘅 DEK、密碼同救援碼：

```
payload = {
  v: 4,
  ── 品牌庫 ──────────────────────────────
  hint     : "密碼提示"         ← 明文（用家自己寫，故意寫到得佢自己明）
  data     : {iv, ct}           ← 11 個品牌，用 DEK 加密
  wrapPw   : {salt, iv, ct}     ← DEK 被「主密碼 + 密鑰」鎖住
  wrapRec  : {salt, iv, ct}     ← DEK 被「救援碼 + 密鑰」鎖住
  ── Abby 私人庫（第二重鎖）──────────────
  pHint    : "私人密碼提示"      ← 明文
  pdata    : {iv, ct}           ← 私人資料，用另一條 PDEK 加密
  pWrapPw  : {salt, iv, ct}     ← PDEK 被「'personal|' + 私人密碼 + 密鑰」鎖住
  pWrapRec : {salt, iv, ct}     ← PDEK 被「'personal|' + 私人救援碼 + 密鑰」鎖住
}
```

- 演算法：PBKDF2-SHA256（600,000 次）→ AES-256-GCM
- **金鑰材料 = 密碼（或救援碼）＋ URL 入面條密鑰**，兩樣缺一不可
- 行編號 `rowId = SHA256(密鑰 + '|vault-row').slice(0,40)` —— 冇密鑰連邊行都搵唔到
- 改密碼／換救援碼＝淨係換 wrap，唔使重新加密啲資料
- URL fragment（`#`）唔會傳去伺服器，讀完即刻 `history.replaceState` 抹走

**主密碼開唔到私人庫**，反之亦然 —— 兩條 DEK 完全獨立。`pPass()` 加咗 `'personal|'`
前綴做域分隔，就算兩個密碼打到一模一樣，兩個 wrap 都唔會互通。

**冇後備鎖匙存喺任何地方。** 冇密碼又冇救援碼＝真係開唔返，呢個係刻意設計。
私人庫要**另外一串**救援碼，同主庫嗰串唔通用。

### 私人庫（`👤 Abby's`）

右上角粒掣切換。資料結構同品牌庫唔同 —— 銀行嘢淨係三格（帳號／密碼／備註）唔夠用，
所以改用自由欄位：

```js
item = { id, cat, icon, title,
         fields:[{id, label, value, secret}],   // 隨時加減、改 label
         note }
```

`CATS` 有四類（銀行／社交／會員／其他），揀咗分類就自動出嗰類慣常要填嘅欄位。
`secret:true` 嘅欄位預設遮蔽，亦**唔會攞嚟做搜尋比對**。

### 批量貼上（`parseBulk`）

一次過貼一大段落，喺瀏覽器度自己拆。段落之間**空一行**分隔，每嚿第一行當標題。

最重要嗰條規矩：**寧願分錯類，都唔可以漏咗一行。** 認得出嘅（網址／電郵／一串號碼／
`X : Y`／`名 12345678`）就變欄位，認唔出嘅中文句子或者有空格嘅一律掉入備註。
剩低嘅純字串：得一個當密碼，兩個或以上就頭嗰個做帳號、尾嗰個做密碼。

⚠️ 淨得個標題、冇帳號冇密碼嗰啲（例如「Facebook meta」）**一定要留低**，
唔可以因為 `fields` 同 `notes` 都係空就 `return`。呢個 bug 我踩過一次。

預覽只列**欄位名**、唔顯示任何值 —— 唔好喺屏幕度攤開一版密碼。
`closeBulkModal()` 喺 `lock()` 同 `lockPersonal()` 都要叫，唔可以留住成個 textarea 嘅明文；
但佢只會清批量貼上嗰個彈窗，唔會掃走救援碼彈窗（嗰串碼淨係顯示一次，清咗就真係冇）。

⚠️ 改 `save()` 嗰陣要記住：**私人庫鎖住嗰陣冇 `pdek`，就一定唔好郁 `envelope.pdata`。**
讀唔到嘅嘢唔可以重寫，否則一 save 就會蓋爛啲私人資料。`btnExport` 同一道理。

⚠️ 由備份檔還原私人庫嗰陣，備份入面兩把 `pWrap*` 係跟住**備份嗰條密鑰**整嘅。
還原完要用**自己條密鑰**重新 wrap 一次，順便發張新私人救援碼（舊嗰張必然作廢）。

## 開庫流程（`boot()`）

```
URL 有 #k=        → 存落 localStorage → 抹走網址 → initWithKey()
localStorage 有   → initWithKey()
兩樣都冇          → showChooser()：叫用家貼返連結，或者明確撳「建立新庫」
```

⚠️ **千祈唔好改返做「靜靜雞自動生成密鑰」。** 早期版本咁做，結果用家喺第二部機／
iOS 主畫面捷徑（獨立 storage）撳入去，會靜靜開多一個空庫，以為資料唔見咗。
而家「設定主密碼」畫面永遠留住「🔗 我之前已經開過庫 → 貼返我條連結」呢條後路。

## 檔案結構

```
index.html      品牌 checklist（右上角有「🔒 Account」入口）
accounts.html   帳號密碼庫 + Abby 私人庫（獨立頁，自成一體）
DEV-NOTES.md    呢份
favicon-32.png / icon-192.png / apple-touch-icon.png
```

兩個 HTML 都係單檔（CSS + JS 全部 inline），冇任何外部依賴。改嘢直接改嗰個檔就得。

## 改嘢注意

- 用**繁體中文廣東話**寫 UI 文案，同現有語氣一致
- 手機優先 —— 老闆九成時間用手機
- 配色跟 `:root` 啲 CSS variable，唔好硬寫色碼
- 改完加密相關嘅嘢，**一定要實測**（見下面）
- 任何時候都唔好將帳號密碼、密鑰、主密碼寫入 repo

### CSS 有個陷阱

`input[type=text]`（0-1-1）嘅特異性**高過**單一個 class（0-1-0）。所以想整一個
「睇落唔似輸入框」嘅 input（例如 `.plat-title`、`.plabel`），一定要寫 `input.plabel`，
唔係嗰句 `border:none;background:transparent` 會被蓋過，變返個白色框。

### 點測（唔好掂真嘅 Supabase）

`vault` 表**冇開 delete policy**，寫咗垃圾行係刪唔返嘅。所以測試一定要隔離：

1. 複製 `accounts.html` 去暫存資料夾，將 `SUPABASE_URL` 改指去本機 stub server
2. stub 實現兩個端點就夠：`GET /rest/v1/vault?id=eq.X&select=payload` 同 `POST /rest/v1/vault`
3. 用 Chrome DevTools Protocol（`--headless=new --remote-debugging-port`）驅動，唔使裝 playwright
4. 行足一次：建立 → 填 → 鎖 → 解鎖 → 私人救援碼重設 → 抓 stub 收到嘅 payload grep 明文

要驗到嘅重點：主密碼開唔到私人庫、主庫救援碼開唔到私人庫、鎖咗之後 DOM 冇明文殘留、
上雲嗰嚿嘢 grep 唔到任何密碼／密鑰／救援碼（淨係兩個 hint 係明文，刻意嘅）。

注意：試錯密碼會喺 console 出 `OperationError`，嗰個係 `unwrapDEK` 正常行為，唔係 bug。
