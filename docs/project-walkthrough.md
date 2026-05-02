# 新人向けプロジェクト歩き方

## このプロジェクトの全体像

`my-tools` は、Tauri v2 と Svelte で作られたデスクトップアプリです。画面は Web 技術で実装し、デスクトップアプリとしての起動や OS に近い処理は Tauri と Rust が受け持ちます。

現時点では、現在時刻を表示する最小構成の画面が実装されています。機能はまだ小さいため、最初は「どの層が何を担当しているか」を把握すると、今後の拡張を追いやすくなります。

```text
ユーザー
  ↓
Svelte UI
  - src/App.svelte
  - src/app.css
  ↓
Tauri IPC
  - invoke("current_timestamp")
  ↓
Rust 側の Tauri command
  - src-tauri/src/lib.rs
  ↓
デスクトップアプリとして起動
  - src-tauri/src/main.rs
  - src-tauri/tauri.conf.json
```

## まず読むファイル

初めてこのリポジトリを見るときは、次の順番で読むと流れをつかみやすくなります。

| 順番 | ファイル | 見ること |
| --- | --- | --- |
| 1 | `README.md` | 起動方法、ビルド方法、関連ドキュメント |
| 2 | `package.json` | npm scripts、フロントエンドと Tauri CLI の依存関係 |
| 3 | `vite.config.ts` | Vite の開発サーバー設定。Tauri と同じ `127.0.0.1:1420` を使う |
| 4 | `src/main.ts` | Svelte アプリを DOM にマウントする入口 |
| 5 | `src/App.svelte` | 現在の画面本体。時刻表示と Tauri command 呼び出しがある |
| 6 | `src/app.css` | 画面全体の見た目 |
| 7 | `src-tauri/tauri.conf.json` | アプリ名、ウィンドウサイズ、ビルド時の連携設定 |
| 8 | `src-tauri/src/lib.rs` | Tauri command とアプリ起動処理 |
| 9 | `docs/technical-stack.md` | 採用技術、将来の拡張方針、配布時の注意点 |

## ディレクトリの役割

```text
.
├── docs/
│   ├── project-walkthrough.md  新人向けの歩き方
│   └── technical-stack.md      技術選定と方針
├── src/
│   ├── App.svelte              UI の中心
│   ├── app.css                 グローバルな見た目
│   ├── main.ts                 Svelte の起動処理
│   └── vite-env.d.ts           Vite 用の型定義
├── src-tauri/
│   ├── src/
│   │   ├── lib.rs              Tauri command と Builder 設定
│   │   └── main.rs             ネイティブアプリ側の入口
│   ├── Cargo.toml              Rust/Tauri 側の依存関係
│   └── tauri.conf.json         Tauri アプリ設定
├── package.json                npm scripts とフロントエンド依存関係
├── package-lock.json           npm の lockfile
├── tsconfig.json               TypeScript 設定
├── tsconfig.node.json          Node/Vite 用 TypeScript 設定
└── vite.config.ts              Vite 設定
```

## 開発環境の準備

このプロジェクトでは Node.js/npm と Rust が必要です。Tauri は OS ごとに前提条件が異なるため、環境構築で失敗する場合は `docs/technical-stack.md` の「開発環境」も確認してください。

依存関係のインストールは次のコマンドで行います。

```sh
npm install
```

## 起動方法

デスクトップアプリとして動かす場合は、Tauri の開発コマンドを使います。

```sh
npm run tauri:dev
```

このコマンドは内部で Vite の開発サーバーも起動します。`src-tauri/tauri.conf.json` の `beforeDevCommand` が `npm run dev` を呼び、`devUrl` が `http://127.0.0.1:1420` を参照する構成です。

フロントエンドだけをブラウザで確認したい場合は、次のコマンドを使います。

```sh
npm run dev
```

Vite は `vite.config.ts` の設定により、`127.0.0.1:1420` で起動します。

## ビルドと確認

フロントエンドだけをビルドする場合は、次のコマンドです。

```sh
npm run build
```

デスクトップアプリとして配布物まで作る場合は、Tauri のビルドを使います。

```sh
npm run tauri:build
```

Tauri のビルドは Rust 側のコンパイルも含むため、フロントエンドだけのビルドより時間がかかります。OS 固有のビルド要件や署名の話は、`docs/technical-stack.md` を参照してください。

## 現在の実装の読み方

### 画面表示

現在の画面は `src/App.svelte` にあります。`onMount` で 1 秒ごとに `Date` を更新し、`Intl.DateTimeFormat` で日本語向けの時刻表示に整えています。

見た目は `src/app.css` で定義されています。現時点では画面中央に大きく時刻を表示するだけなので、UI を変更したい場合はまず `App.svelte` と `app.css` を確認します。

### Rust 側との接続

`src/App.svelte` では、Tauri の `invoke` を使って `current_timestamp` を呼び出しています。

```ts
void invoke<number>("current_timestamp").catch(() => undefined);
```

現時点では戻り値を画面表示には使っておらず、フロントエンドから Rust 側の command を呼べることを確認するための最小実装です。

呼び出し先は `src-tauri/src/lib.rs` の `#[tauri::command]` 付き関数です。

```rust
#[tauri::command]
fn current_timestamp() -> u64 {
    ...
}
```

Tauri command を追加するときは、関数を定義するだけでなく、`tauri::generate_handler![...]` に登録する必要があります。登録を忘れると、フロントエンドから呼んだときに command が見つからないエラーになります。

## よく触る場所

| やりたいこと | 主に触る場所 | 補足 |
| --- | --- | --- |
| 画面を変える | `src/App.svelte`, `src/app.css` | 画面が増えてきたら `src/components/` の追加を検討します |
| フロントエンドの入口を変える | `src/main.ts` | 通常はあまり触りません |
| npm script を追加する | `package.json` | チームで使う確認コマンドはここへ寄せます |
| Vite のポートや開発サーバー設定を変える | `vite.config.ts`, `src-tauri/tauri.conf.json` | 両方の整合が必要です |
| Rust 側の処理を追加する | `src-tauri/src/lib.rs` | Tauri command として公開する処理を置きます |
| ウィンドウサイズやアプリ名を変える | `src-tauri/tauri.conf.json` | 配布物の設定にも関係します |
| Rust の依存関係を追加する | `src-tauri/Cargo.toml` | 追加後は Tauri ビルドで確認します |

## 変更作業の進め方

小さな変更でも、次の流れで確認すると問題を切り分けやすくなります。

1. 変更対象の層を決める
2. フロントエンドだけで済む場合は `src/` 配下を変更する
3. OS 機能やローカルファイル操作が必要な場合は `src-tauri/` 側に command を追加する
4. `npm run build` でフロントエンドの型とビルドを確認する
5. Tauri 連携を含む変更なら `npm run tauri:dev` で実機動作を確認する
6. 配布物に影響する変更なら `npm run tauri:build` まで確認する

現時点では機能が少ないため、最初から細かい抽象化を増やす必要はありません。画面や command が増えてきた段階で、`src/components/` や Rust の module 分割を考える方が扱いやすいでしょう。

## 新しい機能を追加するときの判断軸

フロントエンドと Rust 側のどちらに実装するか迷った場合は、次の基準で考えます。

| 処理内容 | 置き場所の目安 |
| --- | --- |
| 表示、入力、画面内の状態管理 | `src/` |
| API 呼び出しの前後に行う軽い整形 | まずは `src/` |
| ファイルの読み書き | `src-tauri/` |
| OS のパス、通知、外部プロセス起動 | `src-tauri/` |
| 秘密情報や権限に関わる処理 | `src-tauri/` を基本に検討 |
| 大きなデータ処理や長時間処理 | Rust 側に寄せることを検討 |

UI に近い判断は Svelte 側に置き、OS に近い処理や権限を伴う処理は Rust 側に寄せると、責務が分かりやすくなります。

## つまずきやすい点

- `npm run dev` と `npm run tauri:dev` は目的が違います。前者はブラウザ確認用、後者はデスクトップアプリ確認用です。
- Vite のポートを変える場合は、`vite.config.ts` と `src-tauri/tauri.conf.json` の両方を確認してください。
- Tauri command を追加しただけではフロントエンドから呼べません。`generate_handler` への登録が必要です。
- Tauri のビルド失敗は、フロントエンドではなく Rust や OS 側の前提条件が原因の場合があります。
- 配布段階では、macOS の署名・公証や Windows の署名を別途検討する必要があります。

## 参考ドキュメント

- `README.md`: 最短の起動・ビルド手順
- `docs/technical-stack.md`: 技術選定、配布、環境構築の考え方
- `package.json`: 実際に使える npm scripts
- `src-tauri/tauri.conf.json`: Tauri と Vite の接続設定
