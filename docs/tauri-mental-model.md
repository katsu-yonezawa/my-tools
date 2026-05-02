# Tauri とこのプロジェクトのメンタルモデル

## このドキュメントの目的

このドキュメントは、Tauri を初めて触る人が、このプロジェクトをどう捉えればよいかを整理するためのものです。

`my-tools` は、Web フロントエンドだけのアプリではありません。一方で、すべてを Rust で作るネイティブアプリでもありません。画面は Svelte、デスクトップアプリとしての外枠と OS に近い処理は Tauri + Rust、という役割分担で動いています。

## Tauri とは何か

Tauri は、Web 技術で作った画面をデスクトップアプリとして動かすためのフレームワークです。

Tauri アプリは、主に次の 2 つを組み合わせたものです。

| 部分 | 役割 |
| --- | --- |
| WebView | HTML、CSS、JavaScript で作った画面を表示する |
| Rust 側のアプリ本体 | ウィンドウ作成、OS 連携、Tauri command、ビルドや配布を受け持つ |

WebView は、OS が提供するブラウザ表示エンジンです。Tauri はアプリに Chromium を丸ごと同梱するのではなく、OS 側の WebView を使います。そのため、Electron と比べてアプリサイズを小さくしやすい一方で、OS ごとの WebView やビルド環境の影響を受けます。

## まず持っておく考え方

このプロジェクトは、開発時には次の 3 層に分けて考えると見通しがよくなります。

```text
画面の層
  Svelte / TypeScript / CSS
  src/

橋渡しの層
  Tauri IPC
  invoke(...)
  listen(...)

ネイティブ側の層
  Rust / Tauri command / Tauri plugin / アプリ設定
  src-tauri/
```

画面の層は、通常の Web フロントエンドに近い世界です。メニュー、入力欄、ボタン、結果表示、画面内の状態、見た目はここで扱います。

ネイティブ側の層は、デスクトップアプリとして OS と近い場所にあります。通知、外部 API 呼び出し、将来的なファイル操作や安全な保存処理などはここに寄せます。

橋渡しの層は、フロントエンドと Rust をつなぐ通路です。Tauri では、フロントエンドから `invoke("command_name")` で Rust 側の command を呼び、Rust 側から `emit(...)` した event をフロントエンドで `listen(...)` して受け取れます。

## このプロジェクトの実行時イメージ

`npm run tauri:dev` を実行したとき、頭の中では次の流れで捉えると分かりやすいです。

```text
npm run tauri:dev
  ↓
Tauri CLI が起動する
  ↓
beforeDevCommand として npm run dev が動く
  ↓
Vite 開発サーバーが 127.0.0.1:1420 で起動する
  ↓
Tauri がデスクトップアプリのウィンドウを開く
  ↓
そのウィンドウの WebView が Vite の画面を読み込む
  ↓
Svelte の App.svelte が表示される
  ↓
多くのツールはブラウザ API だけで処理される
  ↓
必要なときだけ invoke で Rust 側の command を呼ぶ
```

ここで大切なのは、Tauri のウィンドウの中に Web アプリが表示されている、という見方です。ブラウザで `npm run dev` を見る場合と、Tauri のウィンドウで見る場合では、画面の中身は近いですが、通知や Tauri command を含む動作確認は Tauri 側で行う必要があります。

## 開発時とビルド時の違い

開発時とビルド時では、フロントエンドの読み込み元が変わります。

| 状態 | フロントエンドの読み込み元 | 関係する設定 |
| --- | --- | --- |
| 開発時 | Vite 開発サーバー | `devUrl`, `beforeDevCommand` |
| ビルド時 | `npm run build` で作られる `dist/` | `frontendDist`, `beforeBuildCommand` |

開発時は、`src-tauri/tauri.conf.json` の `devUrl` が `http://127.0.0.1:1420` を見に行きます。そのため、`vite.config.ts` のポートと Tauri 側の `devUrl` は揃っている必要があります。

ビルド時は、先に Vite が静的ファイルを作り、Tauri がそれをアプリに組み込みます。この違いを理解しておくと、「ブラウザでは動くが Tauri では動かない」「開発では動くがビルド後に崩れる」といった問題を切り分けやすくなります。

## ファイルの見方

このプロジェクトでは、各ファイルを次のように見ると整理しやすくなります。

| ファイル | 頭の中での役割 |
| --- | --- |
| `src/main.ts` | Svelte アプリをページに載せる入口 |
| `src/App.svelte` | メニュー、各ツール画面、フロントエンド側の処理 |
| `src/app.css` | 画面全体の見た目 |
| `src/assets/menu/` | メニューで表示する PNG アイコン |
| `vite.config.ts` | フロントエンド開発サーバーの設定 |
| `package.json` | フロントエンド側の依存関係と npm scripts |
| `src-tauri/src/main.rs` | ネイティブアプリとしての入口 |
| `src-tauri/src/lib.rs` | Tauri command、外部 API 連携、event 発行 |
| `src-tauri/tauri.conf.json` | アプリ名、ウィンドウ、開発時・ビルド時の接続設定 |
| `src-tauri/Cargo.toml` | Rust 側の依存関係 |

`src/` は Web アプリ側、`src-tauri/` はデスクトップアプリ側、と見るのが基本です。ただし、AI チャットのように両方をまたぐ機能では、Svelte の UI と Rust の command をセットで読む必要があります。

## 現在の実装を例にした流れ

### フロントエンドだけで完結するツール

UUID、JSON、日時変換、Base64、URL、JWT、Regex、文字数、ハッシュ、Diff、QR、色変換、メモの多くは、`src/App.svelte` とブラウザ API だけで完結しています。

たとえば UUID は `crypto.randomUUID()`、ハッシュは `crypto.subtle.digest(...)`、文字数は `Intl.Segmenter` と `TextEncoder` を使います。これらは OS に近い処理ではないため、まず Svelte 側に置く判断で問題ありません。

### 通知を使うツール

ポモドーロタイマーは、終了時に通知を出します。

```ts
import {
  isPermissionGranted,
  requestPermission,
  sendNotification,
} from "@tauri-apps/plugin-notification";
```

通知は Tauri plugin を通します。ブラウザ表示だけでは実機と同じ挙動にならない場合があるため、`npm run tauri:dev` で確認してください。

### Rust command を呼ぶツール

`current_timestamp` は、フロントエンドから Rust command を呼べることを確認するための軽い command です。

```ts
void invoke<number>("current_timestamp").catch(() => undefined);
```

Rust 側では、`src-tauri/src/lib.rs` に command が定義されています。

```rust
#[tauri::command]
fn current_timestamp() -> u64 {
    ...
}
```

そして `run()` の中で登録されています。

```rust
.invoke_handler(tauri::generate_handler![
    current_timestamp,
    bedrock_converse
])
```

この 3 点がそろって、フロントエンドから Rust 側の関数を呼べるようになります。

1. Rust 側に `#[tauri::command]` 付きの関数を作る
2. `generate_handler![...]` に登録する
3. フロントエンドから `invoke("command_name")` で呼ぶ

## AI チャットの流れ

AI チャットは、このプロジェクトの中で Tauri らしい役割分担がもっとも分かりやすい機能です。

```text
ユーザーがメッセージを送信
  ↓
Svelte が会話履歴と設定を組み立てる
  ↓
invoke("bedrock_converse") で Rust 側へ渡す
  ↓
Rust が Bedrock Converse Stream API を呼ぶ
  ↓
Bedrock の応答差分を ai-chat-stream-delta event で Svelte へ返す
  ↓
必要に応じて Bedrock が web_search tool を要求する
  ↓
Rust が Tavily Search API を呼び、検索結果を Bedrock へ戻す
  ↓
検索状態を ai-chat-search-status event で Svelte へ返す
  ↓
最終応答、トークン数、Web 検索有無を UI に表示する
```

この機能では、UI の入力・履歴表示・設定フォームは Svelte 側、外部 API への HTTP 通信とストリーム処理は Rust 側に置いています。外部 API のエラー整形も Rust 側で行うため、UI 側はユーザーに表示するメッセージへ集中できます。

## どこに何を書くべきか

新しい処理を追加するときは、まず「これは画面の都合か、OS や外部 API に近い処理か」を考えます。

| やりたいこと | 置き場所 |
| --- | --- |
| ボタン、入力欄、一覧表示を作る | `src/` |
| 画面内だけで完結する状態を持つ | `src/` |
| 表示用にデータを軽く整形する | まずは `src/` |
| ブラウザ API で安全に完結する変換処理 | `src/` または `src/lib/` |
| ファイルを読む・書く | `src-tauri/` を基本に検討 |
| OS の標準ディレクトリを扱う | `src-tauri/` |
| 外部 API を呼び出す | `src-tauri/` を基本に検討 |
| 秘密情報や権限が関わる処理を扱う | `src-tauri/`、または安全な保存方法を検討 |
| Rust 側の処理結果を画面に表示する | `src-tauri/` に command、`src/` に呼び出しと表示 |

判断に迷う場合は、最初から複雑な設計にせず、小さな UI と小さな command でつなぐのが扱いやすいです。機能が増えてから、Svelte の component や Rust の module に分けていく方が、読みやすさを保ちやすくなります。

## 依存関係の見方

このプロジェクトには、JavaScript/TypeScript 側と Rust 側の 2 種類の依存関係があります。

| 種類 | ファイル | 使う場面 |
| --- | --- | --- |
| npm パッケージ | `package.json`, `package-lock.json` | Svelte、Vite、Tauri API、QR 生成、通知プラグイン |
| Rust crate | `src-tauri/Cargo.toml`, `src-tauri/Cargo.lock` | Tauri 本体、Tauri plugin、HTTP 通信、JSON 処理 |

UI 部品やフロントエンドのユーティリティを追加するなら npm 側を見ます。OS 連携や Rust 側の処理を増やすなら Cargo 側を見ます。

## エラーの切り分け方

問題が起きたときは、まずどの層のエラーかを見ます。

| 症状 | まず見る場所 |
| --- | --- |
| メニューやツール画面の表示が崩れる | `src/App.svelte`, `src/app.css` |
| TypeScript の型エラーが出る | `src/`, `tsconfig.json` |
| Vite が起動しない | `package.json`, `vite.config.ts` |
| Tauri のウィンドウが起動しない | `src-tauri/tauri.conf.json`, Rust 側のログ |
| 通知が出ない | OS 権限、`@tauri-apps/plugin-notification`、Tauri 実行環境 |
| `invoke` が失敗する | command 名、`generate_handler` 登録、Rust 側の戻り値 |
| AI チャットが失敗する | Bedrock API キー、リージョン、モデル ID、モデル利用権限、Rust 側のエラー |
| Web 検索だけ失敗する | Tavily API キー、検索深度、検索件数、Rust 側の Tavily エラー |
| `npm run build` が失敗する | フロントエンドのビルド設定や型 |
| `npm run tauri:build` が失敗する | Rust、Tauri、OS 側のビルド要件 |

`npm run dev` で問題がなく、`npm run tauri:dev` だけ失敗する場合は、Tauri 側の設定、Rust 側、Tauri plugin、または Tauri API の使い方を疑います。反対に、ブラウザでも Tauri でも同じ表示崩れが起きる場合は、まず Svelte と CSS を見ます。

## Tauri を使うときの注意点

Tauri は Web 技術でデスクトップアプリを作れる便利な仕組みですが、通常の Web アプリとまったく同じではありません。

- OS ごとの WebView 差分があるため、macOS と Windows の実機確認が必要です。
- 通知やファイル操作などの OS 連携は、ブラウザ単体では確認しきれません。
- 外部 API 呼び出しや秘密情報に近い処理は、Rust 側へ寄せるか、保存方法を慎重に検討します。
- 配布段階では、macOS の署名・公証、Windows のコード署名が問題になります。
- Rust 側に処理を増やしすぎると、フロントエンド中心の開発者には読みづらくなる場合があります。
- 反対に、OS や権限に近い処理をフロントエンド側へ寄せすぎると、責務の境界が曖昧になります。

## このプロジェクトでの基本姿勢

現在の `my-tools` は、ローカルで使う小さなデスクトップツール集です。最初から大きな設計を作るより、次の姿勢で進めると保守しやすくなります。

- 画面の変更は、まず `src/App.svelte` と `src/app.css` で小さく進める
- ブラウザ API で完結する変換処理は、まず Svelte 側で実装する
- 処理が大きくなったら、UI と独立する関数を `src/lib/` に切り出す
- OS 連携や外部 API が必要になったら、`src-tauri/src/lib.rs` に小さな command を追加する
- command を増やしたら、フロントエンドの `invoke` と Rust の `generate_handler` をセットで確認する
- Tauri event を使う場合は、`listen` と `unlisten` の扱いも確認する
- 開発時は `npm run build`、Tauri 連携時は `npm run tauri:dev`、配布前は `npm run tauri:build` で確認する

この見方を持っておくと、新しいファイルや設定が増えても、「これは画面の話か、橋渡しの話か、ネイティブ側の話か」と整理しながら読み進められます。
