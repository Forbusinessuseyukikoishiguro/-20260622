# 【新人エンジニア向け・丁寧解説】単一HTMLで作る「現場あんしん工房」のコードを読み解く🐰

> 対象：JS初級〜中級。1ファイルのHTMLツール（v2.8）を題材に、**保存・AI連携・音声・描画**の実装パターンを、つまずきポイントつきで丁寧に解説します。

## この記事のゴール
- 「データ→描画」という素直なUIの作り方が分かる
- ブラウザだけで動く保存層の作り方が分かる
- ブラウザからAI APIを呼ぶ流れが分かる
- 音声認識を“安全に”組み込むコツが分かる

---

## 1. まずは全体像：単一HTML＋素のJS

ビルドもフレームワークもなし。`<style>`＋`<body>`＋`<script>` の3部構成です。
メリットは「環境構築でつまずかない」「配って即動く」こと。新人が最初に触るには最高の題材です。

ポイントは、**画面を直接いじらない**こと。代わりに「データ」を持ち、それを描く関数（`render〜`）を呼びます。

```js
let tasks = [];           // データ
function renderBoard(){   // データ→画面
  board.innerHTML = COLS.map(c => /* tasksから列を組む */).join("");
  wireBoard();            // 描いた後にイベントを付け直す
}
```

> 💡 つまずき：`innerHTML` で描き直すと、前に付けたクリックイベントは消えます。だから描いた直後に必ず付け直します（`wireBoard()`）。

---

## 2. XSS対策の基本：`esc()`

ユーザーが入力した文字をそのまま `innerHTML` に入れると、タグが混入して事故ります。必ず**エスケープ**します。

```js
const esc = s => String(s ?? "").replace(/[&<>"]/g,
  c => ({ "&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;" }[c]));
```

`${esc(userText)}` のように、画面へ出す値は必ず通す——これを癖にしましょう。

---

## 3. 保存層：環境ごとの違いを1か所に隠す

このツールは2つの場所で動きます。
- claude.ai上（アーティファクト）→ `window.storage`
- ダウンロードしてブラウザで開く → `localStorage`

毎回 `if` で分岐すると地獄なので、**抽象化**します。

```js
const Store = (() => {
  let mode = "memory"; const mem = {};
  function detect(){ /* window.storage > localStorage > memory を判定 */ }
  async function get(key){ /* modeに応じて読む。JSON.parse */ }
  async function set(key,val){ /* modeに応じて書く。JSON.stringify */ }
  return { detect, get, set, del, listKeys, mode:()=>mode };
})();
```

> 💡 ねらい：呼ぶ側は `await Store.get("x")` と書くだけ。保存先がどこかを気にしなくていい。これが「抽象化」の効果です。
> `window.storage` は非同期、`localStorage` は同期ですが、**全部 async で包む**と呼び口がそろってラクになります。

---

## 4. 状態の保存と復元：スナップショット方式

画面の入力を1つのオブジェクトにまとめ、丸ごと保存・復元します。

```js
function snapshotState(){
  return { v:"2.8", log:logEntries, tasks, kick:kVal(),
           ume:{history:umeHistory, view:umeView, ...},
           learnNotes, koto:kotoSettings, /* ... */ };
}
function applyState(s){
  if (Array.isArray(s.tasks)) { tasks = s.tasks; renderBoard(); }
  // 各データを戻して、それぞれ render し直す
}
```

入力のたびに少し待ってから保存する「デバウンス」も入れます（保存しすぎ防止）。

```js
let t=null;
function persistAuto(){ clearTimeout(t); t=setTimeout(()=>Store.set("usa:autosave", snapshotState()), 600); }
```

---

## 5. AIを呼ぶ：ブラウザから `/v1/messages`

claude.ai上ではブラウザから直接APIを呼べます（キーは環境が処理）。

```js
async function callClaude({ system, history, tools, maxTokens }){
  const body = {
    model: "claude-sonnet-4-6",
    max_tokens: maxTokens || kotoSettings.len,
    temperature: kotoSettings.temp,
    system,
    messages: history,        // 会話履歴を毎回まとめて送る
  };
  if (tools) body.tools = tools;   // 論文検索など
  const r = await fetch("https://api.anthropic.com/v1/messages", {
    method:"POST", headers:{ "Content-Type":"application/json" },
    body: JSON.stringify(body),
  });
  if (!r.ok) throw new Error("api " + r.status);
  return await r.json();
}
```

返事は「textブロック」だけ取り出します（検索結果などの他ブロックは無視）。

```js
function collectText(data){
  return (data.content || []).filter(b => b.type === "text")
                             .map(b => b.text).join("\n").trim();
}
```

> 💡 大事：AIには「嘘の出典を作らない」「査読論文＋参照先を書く」と**システムプロンプトで指示**します。プロンプトは仕様の一部です。

---

## 6. 音声を“安全に”組み込む

ブラウザの音声認識は便利ですが、**使えない環境**や**マイク拒否**があります。だから「使えない前提」で守りを固めます。

```js
function attachSpeech(btnId, targetId, heardId, statusId){
  const btn = document.querySelector("#"+btnId);
  btn.innerHTML = UME_SVG;                 // 梅の花アイコン
  if (!speechSupported()){ btn.disabled = true; return; } // 非対応は無効化
  btn.addEventListener("click", () => {
    const rec = new (window.SpeechRecognition||window.webkitSpeechRecognition)();
    rec.lang = "ja-JP"; rec.interimResults = true;
    rec.onresult = ev => {
      let fin="", itm="";
      for (const r of ev.results) (r.isFinal ? fin += r[0].transcript : itm += r[0].transcript);
      // 確定＝濃い墨、途中＝薄字、で色分け表示
      heard.innerHTML = `<span class="final">${esc(fin)}</span><span class="interim">${esc(itm)}</span>`;
    };
    rec.start();
    setTimeout(()=>rec.stop(), kotoSettings.voiceWait*1000); // ハング対策の自動停止
  });
}
```

> 💡 つまずき：音声認識は「終わらない」ことがあります。だから**タイマーで自動停止**を必ず入れます。

---

## 7. テストの考え方（jsdom）

ブラウザが無くても、`jsdom` でHTMLを実行して操作を再現できます。AIは通信せず**モック**にして、ロジックだけ検証します。

```js
window.fetch = async () => ({ ok:true, json: async () => ({ content:[{type:"text",text:"ok"}] }) });
// ボタンをclick → DOMの結果をassert
```

v2.8 は `node --check`（構文）＋ jsdom（動作）で **30/30 pass**。
音声の実発火だけは自動テストできないので、「音声色分けデモ」で見え方を確認する——**できないことは正直に書く**のが大事です。

---

## まとめ（新人が持ち帰るべき4つ）
1. **データ→描画**を徹底すると壊れにくい
2. 画面に出す文字は必ず`esc()`
3. 環境差は**抽象化**で隠す（Store）
4. 外部依存（AI・音声）は**失敗前提**で守る

この4つは、フレームワークを使う現場でもそのまま効く考え方です。まずは小さな単一HTMLで体に入れましょう🐰
