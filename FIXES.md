# comfyui-photoshop 修正メモ

作業日: 2026-06-17

## 概要

cdmusic2019 フォーク（v1.9.5）に対して行った調査・修正の記録。

---

## 修正済み

### 1. プロンプトが反映されない問題 ✅

**症状:** PS プラグインで入力したポジティブ/ネガティブプロンプトが、ComfyUI の生成に反映されない。

**原因:** プラグイン JS 内の `w()` 関数（キュー送信）に `l&&` ガードがあり、`Un` ストアが `true` のときのみ `configdata`（プロンプト等）を送信していた。接続後の最初の自動送信でフラグがリセットされ、以降の手動 Render 時は `configdata` が含まれなかった。

**修正箇所:**
- ファイル: `Install_Plugin/3e6d64e0_pro/assets/index-B_-tWO9a.js`（ソース）
- インストール先: `C:\Users\shingo\AppData\Roaming\Adobe\UXP\Plugins\External\3e6d64e0_pro_1.9.5\assets\index-B_-tWO9a.js`

**変更内容:**
```
// 変更前
l&&e.push(b().then((e=>{t.configdata=e})))

// 変更後（l&& を削除）
e.push(b().then((e=>{t.configdata=e})))
```

---

### 2. ComfyUI Web パネルの CSS 修正 ✅（部分的）

**症状:** "ComfyUI Web" パネルが黒一色で何も表示されない。

**原因（CSS）:** 元の CSS で `webview` に `z-index:-1 !important` が設定されており、`uxp-panel` の黒背景の後ろに隠れていた。また `uxp-panel` に高さが指定されておらず `height:100%` が機能しなかった。

**修正箇所:**
- ファイル: `Install_Plugin/3e6d64e0_pro/assets/index-C3nL6Ca2.css`（ソース）
- インストール先: `C:\Users\shingo\AppData\Roaming\Adobe\UXP\Plugins\External\3e6d64e0_pro_1.9.5\assets\index-C3nL6Ca2.css`

**変更内容:**
```css
/* 変更前 */
webview.svelte-3elpbo.svelte-3elpbo { z-index:-1 !important; height:fit-content; width:100%; }
uxp-panel.svelte-3elpbo.svelte-3elpbo { background-color:#000; display:flex; flex-direction:column; align-items:center; }

/* 変更後 */
webview.svelte-3elpbo.svelte-3elpbo { position:absolute; top:0; left:0; width:100%; height:100%; z-index:999; }
.bg.svelte-3elpbo.svelte-3elpbo { position:relative; z-index:1; width:100%; display:flex; align-items:center; justify-content:space-between; padding:8px 2vw; }
uxp-panel.svelte-3elpbo.svelte-3elpbo { background-color:#000; position:relative; display:block; width:100%; height:100vh; overflow:hidden; }
```

---

## 未解決

### ComfyUI Web パネルの webview が表示されない ❌

**症状:** CSS 修正後もパネルは黒のまま。ComfyUI の画面が埋め込まれない。

**調査結果:**

デバッグオーバーレイで確認した webview 要素の状態：
- `WV found: true` → DOM に存在する
- `src: http://127.0.0.1:8188` → src は正しく設定されている
- `loadURL: undefined` → UXP の `loadURL()` メソッドは存在しない
- `offsetW: 595, offsetH: 400` → サイズは正常

**根本原因:**  
Svelte が `document.createElement("webview")` で動的に生成した webview 要素を、UXP が「本物の webview」として初期化しない。要素は DOM に存在してサイズも正しいが、コンテンツが一切描画されない。

**試したアプローチ（すべて失敗）:**

| 試したこと | 結果 |
|---|---|
| `wv.src = url` プロパティ代入 | 変化なし |
| `wv.loadURL(url)` | メソッドが存在しない |
| `insertAdjacentHTML` で HTML として挿入 | 変化なし |
| `index.html` に静的 `<webview>` を追加（body 直下） | UXP パネルに表示されない |
| `<iframe>` に変更 | 変化なし |
| `index.html` に静的 `<uxp-panel>` + `<webview>` を追加 | 他パネルが破損 |

**現在の対応:**  
"ComfyUI Web" パネルに「Open in Browser」ボタンを設置。クリックで `http://127.0.0.1:8188` をデフォルトブラウザで開く。

**今後の改修案:**  
Svelte ソースコードを取得してリビルドし、`<webview>` を `index.html` に静的宣言する形に変更する（工数: 3〜6時間）。

---

### 3. ワークフロー切り替え時に Photoshop 表示が更新されない問題 ✅

**症状:** ComfyUI で別のワークフローを開いても、Photoshop パネルの TOP/BOTTOM ドロップダウンが前のワークフローの内容を表示し続ける。

**原因:** `manager.js` の `workflowSwitcher` / `rndrModeSwitcher` がモジュールレベルの変数として保持されており、新しいワークフローを読み込んでも**リセットされない**。`identifyNode` 内の `if (!workflowSwitcher)` ガードにより上書きもされない。

**修正箇所:**
- ファイル: `js/manager.js`

**変更内容:**
```js
// 追加: リセット関数
function resetSwitchers() {
    workflowSwitcher = "";
    rndrModeSwitcher = "";
    if (workflowInterval) { clearInterval(workflowInterval); workflowInterval = null; }
    if (rndrInterval) { clearInterval(rndrInterval); rndrInterval = null; }
}

// 追加: loadGraphData をフックしてリセット＋再スキャン
const originalLoadGraphData = app.loadGraphData.bind(app);
app.loadGraphData = function(...args) {
    resetSwitchers();
    const result = originalLoadGraphData(...args);
    setTimeout(() => scanForSwitchers(), 500);
    return result;
};
```

---

### 4. Fast Groups Bypasser が rndrModeSwitcher として認識されない問題 ✅

**症状:** `⚙️` プレフィックスのノードが `Fast Groups Bypasser (rgthree)` の場合、BOTTOM ドロップダウンが機能しない。

**原因:** `identifyNode` の冒頭で `comfyClass !== "Fast Groups Muter (rgthree)"` の場合に即 `return` していたため、Bypasser ノードが完全に無視されていた。

**修正箇所:**
- ファイル: `js/manager.js`

**変更内容:**
```js
// 変更前
if (node.comfyClass !== "Fast Groups Muter (rgthree)") return;

// 変更後
const isMuter = node.comfyClass === "Fast Groups Muter (rgthree)";
const isBypasser = node.comfyClass === "Fast Groups Bypasser (rgthree)";
if (!isMuter && !isBypasser) return;
```

---

### 5. タイトル（📁/⚙️）より色が優先されてノードが誤判定される問題 ✅

**症状:** `⚙️` プレフィックスのノードでも色が `#000`（未設定時のデフォルト）の場合、`isBlue` 判定に引っかかり workflowSwitcher として誤認識される。

**原因:** 色ベースの判定がタイトルより先に評価されており、`#000` が `isBlue` リストに含まれていた。

**修正箇所:**
- ファイル: `js/manager.js`

**変更内容:**
```js
// タイトルを色より優先して判定
const isWorkflow = nodeTitle.startsWith("📁") || (!nodeTitle.startsWith("⚙️") && isBlue);
const isRndrMode = nodeTitle.startsWith("⚙️") || (!nodeTitle.startsWith("📁") && isGreen);
```

---

## 変更ファイル一覧

### インストール済みプラグイン
```
C:\Users\shingo\AppData\Roaming\Adobe\UXP\Plugins\External\3e6d64e0_pro_1.9.5\
├── index.html           （変更なし・現在）
├── manifest.json        （launchProcess.schemes に "http" を追加）
└── assets\
    ├── index-B_-tWO9a.js   （プロンプト修正・Open in Browser ボタン追加）
    └── index-C3nL6Ca2.css  （webview/uxp-panel CSS 修正）
```

### ソースファイル（ComfyUI カスタムノード）
```
custom_nodes\comfyui-photoshop\Install_Plugin\3e6d64e0_pro\assets\
├── index-B_-tWO9a.js   （インストール済みと同じ変更を反映）
└── index-C3nL6Ca2.css  （インストール済みと同じ変更を反映）
```

---

## 動作状況

| 機能 | 状態 |
|---|---|
| PS → ComfyUI へのキャンバス送信 | ✅ 動作 |
| プロンプト送信 | ✅ 動作（修正済み） |
| レンダリング結果を PS に返す | ✅ 動作 |
| ComfyUI Web パネル（埋め込み） | ❌ 未解決 |
| ComfyUI Web パネル（Open in Browser） | ✅ 動作 |
