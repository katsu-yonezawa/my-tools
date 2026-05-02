# Tauri とこのプロジェクトのメンタルモデル

## このドキュメントの目的

このドキュメントは、Tauri を初めて触る人が、このプロジェクトをどう捉えればよいかを整理するためのものです。

コマンドの手順だけを覚えると、画面変更、Rust 側の処理追加、ビルドエラーの切り分けで迷いやすくなります。先に全体像を持っておくと、「今どの層の問題を見ているのか」を判断しやすくなります。

## Tauri とは何か

Tauri は、Web 技術で作った画面をデスクトップアプリとして動かすためのフレームワークです。

もう少し分けて言うと、Tauri アプリは次の 2 つを組み合わせたものです。

| 部分 | 役割 |
| --- | --- |
| WebView | HTML、CSS、JavaScript で作った画面を表示する |
| Rust 側のアプリ本体 | ウィンドウ作成、OS 連携、ファイル操作、ビルドや配布を受け持つ |

WebView は、OS が提供するブラウザ表示エンジンです。Tauri はアプリに Chromium を丸ごと同梱するのではなく、OS 側の WebView を使います。そのため、Electron と比べてアプリサイズを小さくしやすい一方で、OS ごとの WebView やビルド環境の影響を受けます。

このプロジェクトでは、画面を Svelte + TypeScript で作り、デスクトップアプリとしての外枠と OS に近い処理を Tauri + Rust が担当します。

## まず持っておく考え方

このプロジェクトは、ひとつのアプリに見えますが、開発時には次の 3 層に分けて考えると見通しがよくなります。

```text
画面の層
  Svelte / TypeScript / CSS
  src/

橋渡しの層
  Tauri IPC
  invoke(...)

ネイティブ側の層
  Rust / Tauri command / アプリ設定
  src-tauri/
```

画面の層は、通常の Web フロントエンドに近い世界です。表示、入力、画面内の状態、見た目はここで扱います。

ネイティブ側の層は、デスクトップアプリとして OS と近い場所にあります。ファイル読み書き、外部コマンド実行、OS のパス取得、配布物作成などはここに寄せます。

橋渡しの層は、フロントエンドから Rust の処理を呼ぶための通路です。Tauri では、この通路を通じて `invoke("command_name")` のように Rust 側の command を呼び出します。

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
必要に応じて invoke で Rust 側の command を呼ぶ
```

ここで大切なのは、Tauri のウィンドウの中に Web アプリが表示されている、という見方です。ブラウザで `npm run dev` を見る場合と、Tauri のウィンドウで見る場合では、画面の中身は近いですが、Tauri API や OS 連携を含む動作確認は Tauri 側で行う必要があります。

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
| `src/App.svelte` | 現在の画面そのもの |
| `src/app.css` | 画面全体の見た目 |
| `vite.config.ts` | フロントエンド開発サーバーの設定 |
| `package.json` | フロントエンド側の依存関係と npm scripts |
| `src-tauri/src/main.rs` | ネイティブアプリとしての入口 |
| `src-tauri/src/lib.rs` | Tauri の本体設定と Rust 側 command |
| `src-tauri/tauri.conf.json` | アプリ名、ウィンドウ、開発時・ビルド時の接続設定 |
| `src-tauri/Cargo.toml` | Rust 側の依存関係 |

`src/` は Web アプリ側、`src-tauri/` はデスクトップアプリ側、と見るのが基本です。ただし、実際のアプリではこの 2 つが IPC でつながっているため、完全に無関係ではありません。

## 現在の実装を例にした流れ

現在の画面は、時刻を表示するだけの最小実装です。

`src/main.ts` が `App.svelte` をマウントし、`App.svelte` が 1 秒ごとに現在時刻を更新します。表示に関する処理は Svelte 側で完結しています。

一方で、`App.svelte` には次の呼び出しがあります。

```ts
void invoke<number>("current_timestamp").catch(() => undefined);
```

これは、Rust 側の `current_timestamp` command を呼び出す処理です。画面表示には戻り値を使っていませんが、フロントエンドから Rust 側を呼べることを確認するための細い接続になっています。

Rust 側では、`src-tauri/src/lib.rs` に command が定義されています。

```rust
#[tauri::command]
fn current_timestamp() -> u64 {
    ...
}
```

そして同じファイルの `run()` の中で、次のように登録されています。

```rust
.invoke_handler(tauri::generate_handler![current_timestamp])
```

この 3 点がそろって、フロントエンドから Rust 側の関数を呼べるようになります。

1. Rust 側に `#[tauri::command]` 付きの関数を作る
2. `generate_handler![...]` に登録する
3. フロントエンドから `invoke("command_name")` で呼ぶ

## どこに何を書くべきか

新しい処理を追加するときは、まず「これは画面の都合か、OS に近い処理か」を考えます。

| やりたいこと | 置き場所 |
| --- | --- |
| ボタン、入力欄、一覧表示を作る | `src/` |
| 画面内だけで完結する状態を持つ | `src/` |
| 表示用にデータを軽く整形する | まずは `src/` |
| ファイルを読む・書く | `src-tauri/` |
| OS の標準ディレクトリを扱う | `src-tauri/` |
| 外部プロセスを起動する | `src-tauri/` |
| 秘密情報や権限が関わる処理を扱う | `src-tauri/` を基本に検討 |
| Rust 側の処理結果を画面に表示する | `src-tauri/` に command、`src/` に呼び出しと表示 |

判断に迷う場合は、最初から複雑な設計にせず、小さな command と小さな UI でつなぐのが扱いやすいです。機能が増えてから、`src/components/` や Rust の module に分けていく方が、読みやすさを保ちやすくなります。

## 依存関係の見方

このプロジェクトには、JavaScript/TypeScript 側と Rust 側の 2 種類の依存関係があります。

| 種類 | ファイル | 使う場面 |
| --- | --- | --- |
| npm パッケージ | `package.json`, `package-lock.json` | Svelte、Vite、Tauri CLI、フロントエンドライブラリ |
| Rust crate | `src-tauri/Cargo.toml`, `src-tauri/Cargo.lock` | Tauri 本体、Tauri plugin、Rust 側の処理 |

UI 部品やフロントエンドのユーティリティを追加するなら npm 側を見ます。OS 連携や Rust 側の処理を増やすなら Cargo 側を見ます。

## エラーの切り分け方

問題が起きたときは、まずどの層のエラーかを見ます。

| 症状 | まず見る場所 |
| --- | --- |
| 画面の表示が崩れる | `src/App.svelte`, `src/app.css` |
| TypeScript の型エラーが出る | `src/`, `tsconfig.json` |
| Vite が起動しない | `package.json`, `vite.config.ts` |
| Tauri のウィンドウが起動しない | `src-tauri/tauri.conf.json`, Rust 側のログ |
| `invoke` が失敗する | command 名、`generate_handler` 登録、Rust 側の戻り値 |
| `npm run build` が失敗する | フロントエンドのビルド設定や型 |
| `npm run tauri:build` が失敗する | Rust、Tauri、OS 側のビルド要件 |

`npm run dev` で問題がなく、`npm run tauri:dev` だけ失敗する場合は、Tauri 側の設定、Rust 側、または Tauri API の使い方を疑います。反対に、ブラウザでも Tauri でも同じ表示崩れが起きる場合は、まず Svelte と CSS を見ます。

## Tauri を使うときの注意点

Tauri は Web 技術でデスクトップアプリを作れる便利な仕組みですが、通常の Web アプリとまったく同じではありません。

- OS ごとの WebView 差分があるため、macOS と Windows の実機確認が必要です。
- ファイル操作や外部プロセス実行は、セキュリティを意識して Rust 側に寄せます。
- 配布段階では、macOS の署名・公証、Windows のコード署名が問題になります。
- Rust 側に処理を増やしすぎると、フロントエンド中心の開発者には読みづらくなる場合があります。
- 反対に、OS に近い処理をフロントエンド側へ寄せすぎると、責務や権限の境界が曖昧になります。

このプロジェクトでは、まず UI は Svelte、OS に近い処理は Rust、両者の接続は Tauri command という分け方を基本にするとよいでしょう。

## このプロジェクトでの基本姿勢

現時点の `my-tools` は、まだ小さなデスクトップツールの土台です。最初から大きな設計を作るより、次の姿勢で進めると保守しやすくなります。

- 画面の変更は、まず `src/App.svelte` と `src/app.css` で小さく進める
- OS 連携が必要になったら、`src-tauri/src/lib.rs` に小さな command を追加する
- command を増やしたら、フロントエンドの `invoke` と Rust の `generate_handler` をセットで確認する
- 機能が増えてから、Svelte の component 分割や Rust の module 分割を検討する
- 開発時は `npm run build`、Tauri 連携時は `npm run tauri:dev`、配布前は `npm run tauri:build` で確認する

この見方を持っておくと、新しいファイルや設定が増えても、「これは画面の話か、橋渡しの話か、ネイティブ側の話か」と整理しながら読み進められます。
