# shelfie 設計書 (v1)

- Repository: `github.com/Saber5656/shelfie`
- Tagline: "A personal library manager for barcodes, lending, and reading-pile analytics."
- License: MIT（前提）/ 蔵書データは端末外に出さない / 個人 OSS・最小実装で早期リリース
- 作成日: 2026-07-05

---

## 1. コンセプトと既存サービスとの位置づけ

shelfie は、スマホのカメラで本の裏表紙のバーコードを**連続スキャン**して蔵書を登録し、
「積読 / 読中 / 読了」で管理する、完全ローカル動作の蔵書管理 PWA である。
書棚の前に立ち、1 冊 2〜3 秒のテンポでスキャンしていくと書影付きの蔵書リストが育っていく——この体験だけを v1 の核にする。
アカウント不要、インストール不要（URL を開くだけ）、蔵書データは端末の外に出ない。

蔵書リストは「何を読んできたか・何に関心があるか」を示す、思想・信条を推測しうる個人情報である。
既存サービスの多くはこれをクラウドに預ける前提で作られており、そこが shelfie の存在理由になる。

**既存の蔵書管理サービス・OSS との位置づけ**

| 既存 | 形態 | shelfie との違い |
|---|---|---|
| ブクログ / 読書メーター | 国内クラウドサービス + アカウント | 蔵書・読書記録が事業者サーバに保存される。shelfie は端末内のみ・アカウント不要 |
| Libib | クラウド蔵書管理 SaaS | 同上。加えて英語圏中心で、日本の書誌（openBD）には接続しない |
| calibre | デスクトップ OSS | 電子書籍ファイルの管理が主目的。紙の本をスマホでスキャン登録する体験はない |
| Jelu / Koillection 等の self-hosted OSS | 自宅サーバ + Docker | データ主権は守れるがサーバ運用スキルが前提。shelfie は URL を開くだけ |
| 蔵書管理系スマホアプリ各種 | ストア配布ネイティブアプリ | 多くがソース非公開で通信内容を検証できない。shelfie は OSS + ブラウザなので DevTools で通信を検証できる |

つまり「① ブラウザで開くだけで使える ② 蔵書データが端末から出ない ③ 日本の本のバーコード登録に強い」を
同時に満たすものが空白であり、この交点だけを v1 で取りに行く。
機能の多さでは既存サービスに勝てないし、勝ちに行かない。**導入の軽さとデータ主権**で差別化する。

---

## 2. v1 スコープ

リポジトリ説明には lending（貸し借り）と reading-pile analytics（積読統計）を含めているが、
v1 は「スキャン登録 → 一覧 → ステータス管理」のループの完成度に全振りし、両者は明示的に v2 へ送る（理由は表内）。

| 区分 | 項目 | 備考 |
|---|---|---|
| 入れる | バーコード連続スキャン登録 | コア体験。1 冊確定ごとに止まらず次を読める |
| 入れる | EAN-13 検証パイプライン | 978/979 プレフィックス + チェックディジット + 複数フレーム一致（§4） |
| 入れる | 書誌自動取得（openBD → Google Books） | ISBN 送信のみ。§7 のプライバシー設計で厳密に扱う |
| 入れる | 書影表示 + IndexedDB blob キャッシュ | 一度取得した書影はオフラインで表示。再リクエストしない |
| 入れる | オフラインスキャン（仮登録 → 解決キュー） | 電波のない書庫・実家でもスキャンは止まらない（§5） |
| 入れる | 蔵書一覧（書影グリッド / リスト）+ 検索 + ステータスフィルタ | 検索はタイトル・著者の部分一致 |
| 入れる | 読書ステータス（積読 / 読中 / 読了） | 3 値のみ。カスタムステータスは作らない |
| 入れる | 一覧ヘッダの軽い数字（全冊数・積読数） | ダッシュボードではなく 1 行のテキスト |
| 入れる | 重複登録チェック（同一 ISBN） | スキャン時に「登録済み」を即時表示 |
| 入れる | 手入力登録・編集 | ISBN のない古い本・同人誌、API ミス時の補完用 |
| 入れる | JSON / CSV エクスポート・インポート | クラウド同期を持たない代わりのデータ可搬性。CSV は UTF-8 BOM 付き |
| 入れる | PWA オフライン動作 / UI 文言 日英 2 言語 | 文言量が少ないため軽量辞書で対応 |
| 入れない | 貸し借り管理（lending） | **v2**。「誰に」という人物エンティティ、貸出中/返却/催促の状態遷移が加わり、データモデルと UI が一段複雑になる。v1 はスキーマバージョニングで拡張口だけ確保（§5） |
| 入れない | 統計ダッシュボード（analytics） | **v2**。統計は蔵書データが溜まって初めて価値が出る。v1 はヘッダの冊数のみ。エクスポートがあるため分析したい人は手元で可能 |
| 入れない | 複数端末同期・クラウドバックアップ | 「端末外に出ない」原則と正面衝突する。入れるとしても E2E 暗号化等の設計が必要で、v1 はエクスポート/インポートで代替 |
| 入れない | タグ・複数本棚（コレクション） | ステータス + 検索で v1 の用途は足りる |
| 入れない | 評価・レビュー・読書メモ | 記録系機能はスコープ肥大の入口。v2 で需要を見て判断 |
| 入れない | 表紙画像検索・OCR 等の非バーコード認識 | バーコードのない本は手入力で代替 |
| 入れない | 電子書籍・PDF 管理 | calibre の領分。紙の本に限定 |
| 入れない | 雑誌コード（491 始まり）・ISBN なしコードの解釈 | 対象外である旨をトースト表示し手入力へ誘導 |

---

## 3. 対応プラットフォームと優先順位

「書棚の前に立ってスキャンするデバイス」= スマホが主戦場。デスクトップは閲覧・整理・データ移行用と割り切る。

| 優先度 | 環境 | 対応範囲 | 理由 |
|---|---|---|---|
| 1 (v1 主戦場) | iOS Safari 16+ / Android Chrome | スキャン含む全機能 | 蔵書スキャンはスマホでしか起きない。カメラ・WASM・IndexedDB・PWA が揃う。iOS はネイティブ BarcodeDetector 非対応のため WASM で担保（§4、P0 検証） |
| 2 | デスクトップ Chrome / Edge / Firefox / Safari（各最新 2 メジャー） | 全機能（カメラ非搭載機は閲覧・手入力・エクスポート/インポート） | 一覧の整理、CSV でのバックアップ用途 |
| 非対応と明記 | IE・旧 Android WebView・LINE 等のアプリ内ブラウザ | 案内のみ | アプリ内ブラウザはカメラ制限が多い。検出時に「Safari / Chrome で開いてください」バナーを表示（§6） |

必要 API: `getUserMedia` / WebAssembly / IndexedDB / Service Worker。いずれも対象環境で安定。
ネイティブ `BarcodeDetector` は「あれば使う」progressive enhancement 扱い（§4）。
カメラは HTTPS 必須 → GitHub Pages 配信で満たされる（§8）。

---

## 4. 技術選定

### 前提知識: 書籍 JAN コードは二段バーコード

日本の書籍の裏表紙には 2 段のバーコード（書籍 JAN コード）が印刷されている。

| 段 | 内容 | shelfie での扱い |
|---|---|---|
| 上段 | EAN-13。**978 / 979 始まり = ISBN-13 そのもの** | **これのみ使用** |
| 下段 | 192 始まり（旧刊は 191）。図書分類（C コード）+ 本体価格 | 使用しない。978/979 フィルタで構造的に排除される |

- 二段が近接しているため、スキャナは高頻度で下段を先に拾う。**「EAN-13 として読めた」だけでは不十分**で、978/979 プレフィックス検証が必須（下段は 192/191 始まりなので必ず弾ける）。
- ISBN-10 しか印字のない古い本（2007 年以前の奥付など）は手入力で受け付け、**ISBN-13 へ正規化**（`978` 付与 + チェックディジット再計算）して内部表現を一本化する。

### バーコードスキャン方式（WebSearch 検証結果）

**検証した事実:**

1. ネイティブ `BarcodeDetector` API は **iOS Safari / Firefox で未実装**（caniuse / MDN）。iOS 17 系には Safari の実験フラグ（Shape Detection API）が存在したが、**iOS 18.0 以降で動作しなくなった**報告がある。→ ネイティブ API 単独では主戦場の iPhone で全滅。ブリーフの仮説「ライブラリ方式を本線」は事実と一致し、そのまま採用。
2. 主要 JS ライブラリの現況は下表のとおり。老舗 2 つが停滞し、ZXing-C++ の WASM ビルド系が現行の本命。

| 候補 | 実装 | EAN-13 | メンテ状況 | 判定 |
|---|---|---|---|---|
| **barcode-detector（採用）** | ZXing-C++ の WebAssembly ビルド（zxing-wasm）。W3C BarcodeDetector API 互換の ponyfill / polyfill | ZXing-C++ 由来で実績十分 | 活発（zxing-wasm と同一作者が継続開発） | ✅ |
| @zxing/browser（zxing-js） | Java 版 ZXing の TS 移植 | 実績あり | **メンテナンスモード宣言済み** | ✗ |
| html5-qrcode | zxing-js ベース + UI 同梱 | 可 | **2023-04 以降リリースなし**。作者が保守停止を明言 | ✗ |
| quagga2 | 純 JS・1D 専用 | **EAN の誤読（false positive）報告あり** | 継続中だが issue 滞留 | ✗ |
| ネイティブ BarcodeDetector | ブラウザ実装 | Android Chrome で高速 | 標準 API だが iOS Safari 非対応 | 併用のみ |

**採用方式: `barcode-detector` パッケージを polyfill として読み込み、コードは W3C BarcodeDetector API の形で書く。**

- ネイティブ実装がある環境（Android Chrome 等）→ polyfill は登録をスキップし**ネイティブが使われる** = progressive enhancement
- ない環境（iOS Safari / Firefox / デスクトップ Safari）→ **ZXing-C++ WASM 実装が同一 API で動く**
- ネイティブと WASM で精度・挙動差が問題になった場合は ponyfill 固定（全環境 WASM）へ 1 行で切替できる構成にし、既定をどちらにするかは **#1 spike の実機比較で確定**する
- **WASM バイナリの既定配信元は jsDelivr CDN のため、必ず self-host に上書きする**（オフライン動作・CSP・プライバシーの 3 点で必須。§7）

**誤読対策は 3 層**: (1) 検出フォーマットを `ean_13` のみに限定 (2) 978/979 プレフィックス + ISBN-13 チェックディジット検証 (3) 同一値が連続 2〜3 フレームで一致したときのみ確定。ライブラリ選定と多層検証の両方で誤登録を潰す。

### 書誌 API（WebSearch 検証結果）

**ブリーフ前提の重要な修正**: openBD は 2023-07-25 に「API v1 の提供終了」を発表済み。ただし**同一仕様の代替 API を少なくとも 60 ヶ月（≒2028 年以降まで）提供中**で、2026 年 7 月現在も `https://api.openbd.jp/v1/get?isbn={ISBN}` は利用できる。一方、データソースが国立国会図書館の CC-BY 書誌ベースへ移行したことに伴い**書影の収録範囲は大幅に低下**した。→「openBD だけで書影が揃う」前提は成立しない。**書影の Google Books フォールバックを v1 必須**とする。なお NDL サーチの書影 API も 2025-12-17 での終了が発表されており、書影ソース候補から除外する。

| 候補 | キー | 日本語書誌 | 書影 | 制限・懸念 | 判定 |
|---|---|---|---|---|---|
| openBD（代替 API） | 不要 | 強い（国内出版流通由来） | **収録が大幅減** | 利用は「本の販促・紹介目的」に限る条件 / 提供期限あり（60 ヶ月〜） | ✅ 主 |
| Google Books API | キーレス可 | 中程度 | thumbnail あり | IP 単位のレート制限。キー有りでも既定 ~1,000 件/日で、429 前提の設計が必要 | ✅ 従（書誌ミス時 + 書影補完） |
| NDL サーチ | 不要 | 網羅的 | **書影 API が 2025-12 終了** | 応答が SRU/XML 中心で重い | ✗ v1 見送り |
| 楽天ブックス API | アプリ ID 必須 | 強い | あり | 静的 PWA に埋めたキーは公開キーになる。アフィリエイト前提の規約 | ✗ |
| Open Library | 不要 | 弱い | covers API あり | 日本語書誌の欠落が多い | △ v2 の洋書補完候補 |

静的 PWA には「キーを隠せるサーバ」が存在しないため、キー必須 API は選外。
キー不要で CORS 越しに直接叩ける openBD + キーレス運用の Google Books という構成は、
**中継サーバを立てない = 作者にすら蔵書情報が送られない**という §7 のプライバシー設計と一貫する。

**解決フロー**: openBD で hit → 書誌採用、書影 URL があれば取得 → 書影なし or miss → Google Books（`volumes?q=isbn:`）で書誌 / thumbnail を補完 → それでも無ければプレースホルダ表示 + 手入力編集可。取得した書影は blob で IndexedDB にキャッシュし、以後の表示はオフライン完結（通信の最小化）。openBD の CORS 応答と実カバレッジ（手持ち実本でのヒット率・書影率）は #1 spike で計測する。

### 採用スタック

| 層 | 技術 | 理由 |
|---|---|---|
| 言語 | TypeScript | ISBN 正規化・検証・シリアライズを型で固定 |
| UI | Preact（+ 素の CSS） | React 互換 API で ~4KB。スマホ回線での初回ロードを最小化。状態管理ライブラリ不要の規模 |
| ビルド | Vite + vite-plugin-pwa | 静的出力・Service Worker 生成の定番 |
| スキャン | barcode-detector（ZXing-C++ WASM、self-host） | 上記調査結果 |
| ローカル DB | IndexedDB + idb（~1KB wrapper） | 素の IndexedDB API は冗長でバグの温床。Dexie（~30KB）はこの規模には過剰 |
| 書誌 | openBD → Google Books フォールバック | 上記調査結果 |
| テスト | Vitest | ISBN 検証・ISBN-10→13 変換・CSV/JSON シリアライズは純関数化して単体テスト |
| CI/CD | GitHub Actions → GitHub Pages | push で build + deploy + privacy-guard（§7） |
| 依存方針 | ランタイム依存は preact / idb / barcode-detector の 3 つに固定 | 依存が少ないほど §7 の監査可能性が上がる |

---

## 5. アーキテクチャ

### 全体構成

設計の核は**「スキャン確定」と「書誌解決」の非同期分離**。スキャンはローカル書き込みだけで完結するため、
電波ゼロの書庫や実家でも止まらない。ネットワークは Resolver だけが触る（= §7 の唯一の外部通信点）。

```
[カメラ] ──getUserMedia──▶ <video>
    │ フレーム（100ms 間隔）
    ▼
┌─ Scanner ─────────────────────────────┐
│ barcode-detector（native or WASM）      │
│   │ raw EAN-13                         │
│   ▼                                    │
│ Validator                              │
│  978/979 + チェックディジット + N フレーム一致 │
└───┬────────────────────────────────────┘
    │ 確定 ISBN-13
    ▼
Registrar ── 重複チェック ──▶ 既登録: トースト表示のみ
    │ 新規（resolved:false で即保存）
    ▼
IndexedDB ◀──────────────────────────────┐
  ├ books    … 蔵書レコード                │ 書誌 + 書影で更新
  ├ covers   … 書影 blob キャッシュ         │
  └ settings … 書誌ソース選択・言語等        │
    │ resolved:false を監視                │
    ▼                                     │
Resolver（オンライン時のみ動く / 唯一の外部通信）
  └ openBD ──(miss / 書影なし)──▶ Google Books

UI（一覧 / スキャン / 詳細 / 設定）は books ストアを購読して描画
```

### データモデル

```ts
// object store: books（keyPath: id, index: isbn13(unique), status, updatedAt）
interface Book {
  id: string;                // ULID。ISBN なし本too含めた主キー
  isbn13: string | null;     // 正規化済み ISBN-13。手入力本は null
  title: string;             // 未解決時は ""（一覧では ISBN を表示）
  authors: string[];
  publisher?: string;
  pubdate?: string;          // "YYYY-MM" 等、API の返す粒度のまま
  coverId?: string;          // covers ストアへの参照
  status: 'tsundoku' | 'reading' | 'finished';
  resolved: boolean;         // 書誌解決済みか（false = 解決キュー対象）
  resolveAttempts: number;   // 失敗回数（バックオフ・諦め判定用）
  source: 'openbd' | 'googlebooks' | 'manual' | null;
  addedAt: number;
  updatedAt: number;
}

// object store: covers（keyPath: id）
interface Cover { id: string; blob: Blob; source: string; fetchedAt: number; }

// object store: settings（key-value）
// metadataSource: 'openbd' | 'googlebooks' | 'off'   ← §7 のユーザー制御
// lang: 'ja' | 'en', viewMode: 'grid' | 'list'
```

- **解決キューは独立ストアにしない**: `resolved === false` の books がキューそのもの。ストアを増やさず、二重管理のバグを構造的に避ける。試行メタデータ（attempts）はレコードに持つ。
- **スキーマバージョニング**: idb の upgrade コールバックで DB version 1 として規律を作る。v2 の貸し借り（`Loan` ストア or `Book.loan`）・統計をマイグレーションで足せる拡張口をここで確保する。
- JSON エクスポートは books + settings の全量（書影 blob は含めない。再取得可能なため）。インポート時は isbn13 で突合して重複を除き、書影未保持レコードは解決キューに載せ直す。CSV は表計算向けの平坦形式（isbn13, title, authors, publisher, pubdate, status, addedAt）。

### スキャン → 登録フロー

1. スキャン画面で `getUserMedia({ video: { facingMode: 'environment' } })`
2. `<video>` フレームを **100ms 間隔**で `detector.detect()` に投入（rAF 毎は発熱・電池に過剰。間隔は #1 spike で調整）
3. Validator: `ean_13` のみ / 978・979 プレフィックス / チェックディジット / 直近フレーム一致 → 確定
4. Registrar: `isbn13` インデックスで重複チェック → 既登録なら「登録済み:『タイトル』」トースト（スキャンは継続）
5. 新規: `resolved: false` の仮レコードを**即座に IndexedDB へ書き込み**、バイブ + 確定カードをセッションリストに積む。スキャンは止まらない（連続スキャン）
6. Resolver: オンラインなら即時解決。オフラインなら `online` イベント / アプリ起動時 / 一覧の手動更新をトリガに、`resolved: false` を **1 件ずつ順次**（§7 のレート制御）openBD → Google Books の順で解決し、書影 blob を covers に保存
7. 両 API miss: `resolveAttempts` を加算し、一覧では ISBN + プレースホルダ表示。詳細画面から手入力補完を促す

---

## 6. UI/UX

### 画面構成(4 画面)

**蔵書一覧（ホーム）**

```
┌──────────────────────────┐
│ shelfie      全128冊 積読42 │ ← 軽い数字はここまで（v1 の統計）
│ [🔍 タイトル・著者で検索      ] │
│ (すべて)(積読)(読中)(読了) ▦/≡ │ ← フィルタチップ + グリッド/リスト切替
│ ┌──┐ ┌──┐ ┌──┐ ┌──┐     │
│ │書│ │書│ │書│ │書│     │ ← 書影グリッド（未解決は ISBN 表示）
│ └──┘ └──┘ └──┘ └──┘     │
│   …                       │
│                    (📷)    │ ← スキャン開始 FAB
└──────────────────────────┘
```

**スキャン画面（全画面カメラ）**

```
┌──────────────────────────┐
│ ✕ 終了            🔦  ⌨ 手入力│ ← 🔦 はトーチ対応環境のみ表示
│  ┌────────────────────┐  │
│  │     カメラ映像         │  │
│  │  ┌‥‥‥‥‥‥‥‥‥‥┐  │  │ ← ガイド枠:
│  │  ┆  ▐║▌║║▌║▐║▌   ┆  │  │   「裏表紙の "上の" バーコードを
│  │  └‥‥‥‥‥‥‥‥‥‥┘  │  │     枠に合わせてください」
│  └────────────────────┘  │
│ ✓『◯◯◯◯』を追加しました      │ ← 確定ごとに下からカード
│ ▤ このセッションで 3 冊         │ ← タップで獲得リスト展開
└──────────────────────────┘
```

- 確定の手応え: バイブ（`navigator.vibrate`、対応環境）+ カードのスライドイン。効果音は入れない（図書館・書店で鳴ると事故）
- 書誌解決はカード上で非同期に反映（「解決中…」→ 書影 + タイトル）。**オフライン時は「ISBN 登録済み・オンライン時に取得します」表示**でそのまま次へ
- 同一 ISBN を連続で読み続けても 1 回だけ確定（確定後 2 秒間は同一値を無視するクールダウン）

**詳細 / 編集画面**: 書影大 + 書誌、ステータス 3 ボタン、手入力項目の編集、削除、「書誌を再取得」
**設定画面**: 書誌ソース選択（openBD / Google Books / オフ = §7）、エクスポート / インポート、言語、全データ削除

### 初回起動体験（勝負は最初の 1 冊）

1. 開くと空の本棚 + 大ボタン「📷 最初の 1 冊をスキャン」+ 1 行「蔵書データはこの端末の外に出ません」
2. ボタンタップ → カメラ許可の**事前説明を 1 画面**（「バーコードを読むためにカメラを使います。映像は端末内で処理され、送信されません」+ 書誌取得の説明と送信先選択。§7）→ `getUserMedia`
3. 1 冊目の確定 → 書影が現れる小さな驚き。ここまで 30 秒以内を目標
4. カメラ拒否 / 非搭載: 手入力フォームへ自動フォールバック（ISBN 直接入力でも書誌解決は動く）

### エッジケースの扱い

| ケース | 挙動 |
|---|---|
| 下段バーコード（192/191）を検出 | Validator が無言で破棄（UI には出さない。上段に当たれば即確定するため案内不要） |
| 雑誌コード（491）・非書籍 EAN | 「書籍のバーコードではありません」トースト 1 回 + 手入力導線 |
| ISBN のない古い本・同人誌 | 手入力登録（タイトルのみで可）。isbn13 は null |
| openBD にも Google Books にもない | ISBN + プレースホルダで保持。詳細画面で手入力補完を促す |
| 暗所 | トーチ対応環境（主に Android Chrome）のみ 🔦 ボタン表示。非対応環境では「明るい場所で」ヒント |
| アプリ内ブラウザ（LINE 等）でカメラ不可 | UA + 失敗検出で「Safari / Chrome で開いてください」バナー |
| ストレージ逼迫・eviction | `navigator.storage.persist()` を要求 + 定期的にエクスポートを促すバナー（§10 P1） |

---

## 7. プライバシー設計（最重要）

### 7.1 なぜ最重要か

蔵書リストは読書傾向そのものであり、**思想・信条・関心・健康状態まで推測しうる個人情報**である
（図書館界に「利用者の読書事実を外部に漏らさない」規範が確立しているのと同じ理由）。
shelfie の設計原則は 3 つ:

1. **蔵書データ（リスト・ステータス・検索語）は端末外に出ない。** サーバも同期も持たない
2. **唯一の外部通信は「書誌 API への ISBN 送信」**（+ その書影取得）。それ以外の通信経路をコード上・CSP 上の両方で持たない
3. その唯一の通信も**ユーザーが送信先を選べる / 完全にオフにできる**

### 7.2 通信の全量（これ以外は存在しない）

| 種別 | 送信先 | 送るもの | 送らないもの |
|---|---|---|---|
| 書誌解決 | `api.openbd.jp`（既定） | ISBN-13（1 冊ずつ、URL クエリ） | Cookie・資格情報（`credentials: 'omit'`）・蔵書リスト・ステータス・検索語・端末識別子 |
| 書影取得 | `cover.openbd.jp` / `books.google.com` | 書影 URL への GET | 同上 |
| 書誌フォールバック | `www.googleapis.com` | ISBN-13 | 同上 |
| それ以外 | **なし** | — | アナリティクス・クラッシュレポート・外部フォント・CDN、すべて不使用 |

**正直に書くべきこと**: ISBN を照会する行為自体が「この IP がこの本に関心を持つ」ことを送信先（と経路上）に開示する。
これは書誌自動取得の原理的コストであり、ゼロにはできない。だから shelfie は次の制御をユーザーに渡す。

### 7.3 ユーザー制御（設定 > 書誌データの取得先）

| 選択肢 | 動作 | 想定ユーザー |
|---|---|---|
| **openBD（既定）** | 国内の非営利プロジェクト（カーリル + 版元ドットコム運営）へ ISBN のみ送信 | 日本の本が中心の標準ユーザー |
| Google Books | Google へ ISBN のみ送信（Google のプライバシーポリシー配下になることを設定画面に明記） | 洋書中心 / openBD ミスが多い場合 |
| **オフ** | **外部通信ゼロ。** スキャンは ISBN 仮登録まで、書誌は手入力 | 照会先にすら関心を知られたくないユーザー |

- 初回のカメラ許可前の説明画面（§6）でこの 3 択を提示し、**既定値のまま進んでも「何が送られるか」を読んだ状態**にする
- インポート・オフライン復帰時の一括解決も **1 件ずつ順次 + 約 1 req/秒のレート制御**。蔵書全体が 1 リクエストに載る API・実装を構造的に持たない

### 7.4 実装上の担保（設定ではなく構造で守る）

| 層 | 施策 |
|---|---|
| CSP | `connect-src 'self' https://api.openbd.jp https://cover.openbd.jp https://www.googleapis.com https://books.google.com`（書影の実ホストは #1 spike で確定して列挙）。**列挙外への fetch はブラウザが拒否する** |
| コード隔離 | ネットワーク呼び出しは `src/resolver/` の単一モジュールに限定。ESLint（`no-restricted-globals` 等）で他所での `fetch` / `XMLHttpRequest` / `sendBeacon` / `WebSocket` を禁止 |
| CI ガード | GitHub Actions で禁止シンボルの grep + ESLint を実行し、違反で fail（「守っていること」をバッジで常時証明） |
| WASM self-host | barcode-detector の WASM 既定配信元は jsDelivr CDN のため、**必ず同一オリジン配信に上書き**（CSP でも jsDelivr はブロックされる構成にする） |
| アセット同梱 | フォント・アイコンも同梱。外部リソース参照ゼロ |
| 書影キャッシュ | 取得済み書影は IndexedDB の blob を表示。閲覧で再通信しない |
| データ主権 | エクスポートでいつでも全量持ち出し可 / 設定から全データ削除（books・covers・settings・SW キャッシュを wipe） |

### 7.5 検証可能性（README に手順を載せる）

1. **機内モード試験**: 書誌ソース「オフ」なら全機能が機内モードで動く。「openBD」でも閲覧・スキャン仮登録は動く
2. **DevTools > Network**: スキャン・解決を実行しても、選択した書誌 API 以外へのリクエストが 1 本もないことを確認できる
3. **コード監査**: 外部通信は `src/resolver/` だけ。README から該当モジュールへパーマリンク
4. **CI バッジ**: privacy-guard ワークフローの通過を常時表示

---

## 8. 配布方法

| 項目 | v1 の方針 | 理由 |
|---|---|---|
| ホスティング | GitHub Pages（`https://saber5656.github.io/shelfie/`） | 無料・HTTPS 標準（getUserMedia の必須条件）・個人 OSS で維持コストゼロ |
| デプロイ | GitHub Actions: main への push → Vite build → Pages deploy | 手作業ゼロ。privacy-guard も同一ワークフローで実行 |
| PWA | vite-plugin-pwa。「ホーム画面に追加」で疑似アプリ化、オフライン動作 | 書棚の前でワンタップ起動。ストア申請はしない |
| 更新 | Service Worker の autoUpdate + 「新しいバージョンがあります」トースト | 静的アプリのため「更新 = 再訪で最新」 |
| バージョニング | git タグ + GitHub Releases（CHANGELOG） | データスキーマ変更時はマイグレーション番号と対応付ける |
| 独自ドメイン | 持たない（任意・後回し） | 維持費と DNS 管理を増やさない |
| ライセンス | MIT。書誌・書影データの出典（openBD / Google Books）と利用条件を NOTICE 節に記載 | §10 P2 |

---

## 9. README 構成案（英語）

```markdown
<banner: 本棚 + スマホスキャンのイラスト横長 PNG>

# shelfie 📚
> Scan. Shelve. Actually read them someday.
> A personal library manager that runs 100% in your browser —
> your library never leaves your device.

<demo GIF: 書棚の前で 3 冊連続スキャン → 書影付きリストが育つ 15 秒>

**👉 Open the app: https://saber5656.github.io/shelfie/**
No install. No account. No upload.

## Features
- 📷 Continuous barcode scanning — point at the upper barcode on the back cover
- 🏷 Reading status: to-read (tsundoku) / reading / finished
- 🔎 Instant search and status filters, with cover-art grid
- 📶 Offline-first: scan now (even in a basement full of books), resolve later
- 📦 Your data, portable: JSON / CSV export & import
- 🔒 Privacy by design: the only network request is an ISBN lookup — you choose the source, or turn it off

## Privacy — "What does it send?"
- Your library lives in your browser (IndexedDB). No server. No analytics. No cookies.
- The only outgoing data is the ISBN being looked up, sent one book at a time to the
  metadata source you picked: openBD (default), Google Books, or **Off** (fully offline).
- Verify it yourself: turn on Airplane Mode (still works), or watch DevTools → Network.
- All network code lives in one module: [`src/resolver/`](permalink). CI fails if fetch
  appears anywhere else.

## How scanning works
Japanese books carry two stacked barcodes. shelfie reads only the upper one
(EAN-13 starting 978/979 = the ISBN) and validates the check digit across
multiple frames, so misreads don't reach your shelf.

## Export / Import（3 行）
## Development（clone / npm i / npm run dev の 3 行）
## Roadmap
v2: lending tracker & reading-pile analytics — see issues.
## License
MIT. Book metadata & covers from openBD and Google Books (see NOTICE).
```

ポイント: デモ GIF と「Open the app」1 行導線がファーストビューに収まること。
バッジは license / deploy(Pages) / privacy-guard の 3 つまで。

---

## 10. リスクと実装前検証項目

| 優先度 | 項目 | 内容 / 検証方法 |
|---|---|---|
| **P0** | iOS Safari 実機での WASM 連続スキャン品質 | 主戦場の iPhone で、barcode-detector（WASM）の認識速度・誤読率・発熱が「1 冊 2〜3 秒の連続スキャン」に耐えるか。実本 10 冊以上 × 照明条件 2 種で計測（**Issue #1 spike**）。Android ではネイティブ / WASM も比較し既定方式を確定 |
| **P0** | openBD 代替 API の実カバレッジと CORS | 手持ちの実本 20〜30 冊の ISBN で書誌ヒット率・書影収録率を計測し、ブラウザからの CORS 直叩きを確認（**同 spike**）。書影率が想定以上に低ければ Google Books 側を書影の主とする再判断 |
| P1 | Google Books キーレス利用の 429 頻度 | 連続解決（インポート時など）でどの程度で制限されるか。1 req/秒制御 + 指数バックオフ + 「後で再試行」で吸収できるか spike で確認 |
| P1 | iOS ホーム画面（standalone PWA）でのカメラ動作 | 過去に standalone モードで getUserMedia 不可の時期があった経緯があるため、現行 iOS 実機でホーム画面追加後のスキャン動作を確認 |
| P1 | Safari のストレージ eviction | 未使用が続くと IndexedDB が削除されうる。`navigator.storage.persist()` の効果と、ホーム画面追加時の扱いを実機確認。緩和策 = エクスポート促しバナー |
| P2 | openBD 利用条件との整合 | 「本の販促・紹介目的」条件と個人蔵書管理での書影キャッシュ表示の位置づけを README / NOTICE の文言で整理。懸念が残る場合は書影を都度取得へ切替できる設計にしておく |
| P2 | アプリ内ブラウザのカメラ制限 | LINE / X 内ブラウザでの getUserMedia 失敗を検出し、外部ブラウザ誘導バナーが機能するか |
| P2 | CSV の Excel 文字化け | UTF-8 BOM 付与で回避。実 Excel / Numbers で開いて確認 |
| P3 | "shelfie" の名前衝突 | 同名の過去サービス（電子書籍系等）や商標の簡易確認。README 冒頭の説明文で性格の違いを明確化 |

**最重要リスク**: P0 の 2 件。iOS でスキャンが実用水準に達しない場合は方式転換（検出間隔・解像度チューニング、最悪はネイティブ対応環境優先の再設計）、openBD のカバレッジが低すぎる場合は Google Books 主軸への転換になり、いずれも設計の根幹に響く。**本実装前に #1 の捨てられる spike で必ず確定させる。**

---

## 11. v1 Issue 分割案（9 個）

- **#1 `Spike: validate on-device EAN-13 scanning and book-metadata coverage`** — ラベル: `spike`, `design`
  barcode-detector（WASM を self-host）の最小スキャンページを作り、iPhone（iOS Safari）+ Android Chrome の実機で実本 10 冊以上の連続スキャンを検証する。同時に手持ち 20〜30 冊の ISBN で openBD 代替 API のヒット率・書影収録率・CORS、Google Books キーレスの 429 挙動を計測し、ネイティブ BarcodeDetector と WASM の速度・精度も比較する。
  受け入れ条件: 実機 2 台での認識時間・誤読率、両 API のヒット率/書影率の計測結果が Issue に記録され、既定スキャン方式（polyfill / ponyfill 固定）と書誌フォールバック方針が確定している。

- **#2 `Set up PWA skeleton with Vite, strict CSP, and Pages deploy`** — ラベル: `infra`
  TypeScript + Preact + Vite + vite-plugin-pwa の骨格、GitHub Actions（build → Pages deploy + privacy-guard の禁止シンボル grep / ESLint）、§7 の CSP、WASM の同一オリジン配信、日英辞書ユーティリティを整備する。
  受け入れ条件: Pages で HTTPS 配信され、2 回目以降オフラインで起動できる。CSP が有効で、privacy-guard ワークフローが green。

- **#3 `Implement IndexedDB data layer with schema versioning and duplicate detection`** — ラベル: `enhancement`
  books / covers / settings ストア、isbn13 ユニークインデックス、重複チェック、ISBN-10→13 正規化・チェックディジット検証の純関数群、マイグレーション枠組み（DB version 1）を実装する。
  受け入れ条件: Vitest で ISBN 検証・正規化・重複判定が通り、スキーマ変更手順がコードコメントで規定されている。

- **#4 `Build continuous barcode scan screen with validation pipeline`** — ラベル: `enhancement`, `ux`
  全画面カメラ + ガイド枠、100ms 間隔の検出ループ、978/979 + チェックディジット + 複数フレーム一致の Validator、重複トースト、確定カードのセッションリスト、バイブ、トーチ（対応環境のみ）、オフライン時の仮登録動作を実装する。
  受け入れ条件: 実機で連続スキャンが途切れず動き、下段バーコード（192/191）が登録されず、機内モードでも ISBN 仮登録が成功する。

- **#5 `Implement metadata resolver with openBD, Google Books fallback, and cover cache`** — ラベル: `enhancement`
  `src/resolver/` に唯一の外部通信モジュールを実装する。openBD → Google Books のフォールバック、書影 blob の covers 保存、`resolved:false` の順次解決（1 req/秒 + バックオフ）、online イベント / 起動時 / 手動更新のトリガ、429・タイムアウト処理を含む。
  受け入れ条件: オフラインで仮登録した本がオンライン復帰後に自動解決される。fetch が resolver 以外に存在しないことを CI が検証している。

- **#6 `Build library list with search, status filters, and header stats`** — ラベル: `enhancement`, `ux`
  書影グリッド / リスト切替、タイトル・著者の部分一致検索、ステータスフィルタチップ、ヘッダの全冊数・積読数、未解決本の ISBN プレースホルダ表示を実装する。
  受け入れ条件: 500 冊規模でスクロール・検索が実用速度で動き、フィルタと件数表示が整合する。

- **#7 `Add book detail, manual entry, and editing screens`** — ラベル: `enhancement`, `ux`
  詳細画面（書影・書誌・ステータス切替・削除・書誌再取得）、手入力登録フォーム（ISBN あり = 解決に回す / なし = タイトルのみで登録）、手入力項目の編集を実装する。
  受け入れ条件: ISBN のない本を登録・編集・削除でき、API 未ヒット本を手入力で補完できる。

- **#8 `Add settings with metadata-source control, export/import, and data wipe`** — ラベル: `enhancement`, `ux`
  書誌ソース選択（openBD / Google Books / オフ）と初回説明画面への組み込み、JSON / CSV（UTF-8 BOM）エクスポート・インポート、言語切替、全データ削除を実装する。
  受け入れ条件: 「オフ」設定で外部リクエストが 0 件になり、エクスポート → 全削除 → インポートで蔵書が完全復元される（書影は再解決で復元）。

- **#9 `Write English README with banner, demo GIF, and privacy section`** — ラベル: `docs`
  §9 の構成で英語 README を作成する。バナー、連続スキャンの 15 秒デモ GIF、「Open the app」導線、二段バーコードの説明、プライバシー節（検証手順 + resolver パーマリンク）、Roadmap（lending / analytics は v2）、NOTICE（openBD / Google Books の出典と利用条件）を含める。
  受け入れ条件: デモ GIF とアプリ URL がファーストビューに収まり、プライバシー検証手順が再現可能で、バッジが 3 個以内である。

推奨着手順: **#1 → #2 → #3 → (#4, #5, #6 並行可) → #7 → #8 → #9**。
#4〜#6 は #3 のデータ層に依存するが相互には独立。#9 は実機スキャンの GIF が撮れる #4 完了後が効率的。

---

## 参考資料（設計時の調査ソース）

**BarcodeDetector API の対応状況**
- MDN: Barcode Detection API — https://developer.mozilla.org/en-US/docs/Web/API/Barcode_Detection_API
- Can I use: BarcodeDetector — https://caniuse.com/mdn-api_barcodedetector
- Apple Developer Forums: Shape Detection API (Barcode Detector) Safari 18.x — https://developer.apple.com/forums/thread/767761
- iOS でのバーコードスキャンと WASM 代替: https://dev.to/ilhannegis/barcode-scanning-on-ios-the-missing-web-api-and-a-webassembly-solution-2in2

**スキャンライブラリ**
- Sec-ant/barcode-detector（ZXing-C++ WASM の ponyfill/polyfill）— https://github.com/Sec-ant/barcode-detector
- zxing-wasm — https://www.npmjs.com/package/zxing-wasm
- quagga2 — https://github.com/ericblade/quagga2
- Quagga2 vs html5-qrcode 比較（Scanbot）— https://scanbot.io/blog/quagga2-vs-html5-qrcode-scanner/
- オープンソース JS バーコードスキャナ概観（Scanbot）— https://scanbot.io/blog/popular-open-source-javascript-barcode-scanners/
- html5-qrcode の性能 issue — https://github.com/mebjas/html5-qrcode/issues/582

**書籍 JAN コード（二段バーコード）**
- 日本図書コード管理センター「ISBNと書籍JANコードとは」— https://isbn.jpo.or.jp/index.php/fix__about/fix__about_3/
- GS1 Japan 書籍JANコード — https://www.gs1jp.org/code/jan_publication/
- レファレンス協同データベース（二段バーコードの仕組み）— https://crd.ndl.go.jp/reference/entry/index.php?page=ref_view&id=1000316425

**openBD**
- openBD 公式 — https://openbd.jp/
- openBD 書誌 API データ仕様 (v1) — https://openbd.jp/spec/
- 「openBD API（バージョン1）」の提供終了について（2023-07-25）— https://openbd.jp/news/20230725.html
- カーリルブログ: openBD v1 終了の影響と代替 API — https://blog.calil.jp/2023/07/openbd-v1-end.html
- カレントアウェアネス: openBD API v1 提供終了 — https://current.ndl.go.jp/car/185804
- カレントアウェアネス: NDL サーチ書影 API 終了 — https://current.ndl.go.jp/car/267841

**Google Books API**
- Using the API — https://developers.google.com/books/docs/v1/using
- Volume リファレンス — https://developers.google.com/books/docs/v1/reference/volumes

---

## Changelog

- 2026-07-05: 初版
