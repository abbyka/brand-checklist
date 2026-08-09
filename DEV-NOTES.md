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
⚠️ 要用 http server，唔好直接 `file://` 開 —— WebCrypto 同 Supabase CORS 會有問題。

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

啲資料**唔會**直接用主密碼加密。改用一條隨機 DEK，DEK 再俾兩把唔同嘅鎖各鎖一次：

```
payload = {
  v: 3,
  hint    : "密碼提示"          ← 明文（用家自己寫，故意寫到得佢自己明）
  data    : {iv, ct}            ← 真資料，用 DEK 加密
  wrapPw  : {salt, iv, ct}      ← DEK 被「主密碼 + 密鑰」鎖住
  wrapRec : {salt, iv, ct}      ← DEK 被「救援碼 + 密鑰」鎖住
}
```

- 演算法：PBKDF2-SHA256（600,000 次）→ AES-256-GCM
- **金鑰材料 = 主密碼（或救援碼）＋ URL 入面條密鑰**，兩樣缺一不可
- 行編號 `rowId = SHA256(密鑰 + '|vault-row').slice(0,40)` —— 冇密鑰連邊行都搵唔到
- 改主密碼／換救援碼＝淨係換 wrap，唔使重新加密啲資料
- URL fragment（`#`）唔會傳去伺服器，讀完即刻 `history.replaceState` 抹走

**冇後備鎖匙存喺任何地方。** 冇密碼又冇救援碼＝真係開唔返，呢個係刻意設計。

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
accounts.html   帳號密碼庫（獨立頁，自成一體）
DEV-NOTES.md    呢份
favicon-32.png / icon-192.png / apple-touch-icon.png
```

兩個 HTML 都係單檔（CSS + JS 全部 inline），冇任何外部依賴。改嘢直接改嗰個檔就得。

## 改嘢注意

- 用**繁體中文廣東話**寫 UI 文案，同現有語氣一致
- 手機優先 —— 老闆九成時間用手機
- 配色跟 `:root` 啲 CSS variable，唔好硬寫色碼
- 改完加密相關嘅嘢，**一定要實測**：建立 → 填資料 → 鎖 → 解鎖 → 救援碼重設 → 確認 Supabase 入面搵唔到明文
- 任何時候都唔好將帳號密碼、密鑰、主密碼寫入 repo
