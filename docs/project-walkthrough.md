# 新人向けプロジェクト歩き方

## このプロジェクトの全体像

`my-tools` は、Tauri v2 と Svelte で作られたデスクトップ向けの小さなツール集です。画面は Svelte + TypeScript + CSS で実装し、デスクトップアプリとしての起動、通知、AI チャットの外部 API 呼び出しは Tauri + Rust 側で扱います。

現在のアプリは、メニュー画面から各ツールへ移動する構成です。単なる時計画面ではなく、開発や日々の作業で使う変換・検証・生成系の機能がまとまっています。

```text
ユーザー
  ↓
メニュー画面
  - src/App.svelte
  - src/app.css
  - src/assets/menu/*.png
  ↓
各ツール画面
  - ブラウザ API で完結する処理
  - localStorage への軽い保存
  - 必要に応じた Tauri command 呼び出し
  ↓
Rust 側
  - current_timestamp
  - bedrock_converse
  - Tauri event による AI 応答・検索状態の通知
```

## 現在入っているツール

| ツール | 主な機能 | 主な実装場所 |
| --- | --- | --- |
| 時計 | ポモドーロタイマー、現在時刻表示、終了通知、色テーマ保存 | `src/App.svelte`, `@tauri-apps/plugin-notification` |
| UUID | UUID v4 の生成、履歴表示、コピー | `crypto.randomUUID()` |
| JSON | 整形、圧縮、エラー表示、コピー | `JSON.parse`, `JSON.stringify` |
| 日時変換 | Unix 秒、Unix ミリ秒、ISO、UTC、JST の相互確認 | `Date`, `Intl.DateTimeFormat` |
| Base64 | テキスト encode/decode、画像の Base64/Data URL 化 | `TextEncoder`, `TextDecoder`, File API |
| URL | URL encode/decode、`+` を空白として扱うオプション | `encodeURIComponent`, `decodeURIComponent` |
| JWT | header/payload/signature のデコード、`exp`/`iat`/`nbf` の日時表示 | Base64URL decode |
| Regex | 正規表現テスト、フラグ指定、キャプチャ表示、コピー | `RegExp` |
| 文字数 | 表示上の文字数、UTF-8 バイト数、行数、単語数、trim 後集計 | `Intl.Segmenter`, `TextEncoder` |
| ハッシュ | SHA-1/SHA-256/SHA-384/SHA-512 の生成 | Web Crypto API |
| Diff | 行単位の差分表示、追加・削除・変更数、コピー | LCS ベースの簡易 diff |
| QR | QR コード生成、PNG 保存、画像コピー | `qrcode` |
| 色変換 | HEX/RGB/HSL の相互変換、カラーピッカー、コピー | TypeScript の変換関数 |
| メモ | 日付単位の簡易メモ、保存、コピー | `localStorage` |
| AIチャット | Amazon Bedrock Converse Stream、会話履歴、Tavily Web 検索連携 | `src/App.svelte`, `src-tauri/src/lib.rs` |

## まず読むファイル

初めてこのリポジトリを見るときは、次の順番で読むと流れをつかみやすくなります。

| 順番 | ファイル | 見ること |
| --- | --- | --- |
| 1 | `README.md` | 起動方法、ビルド方法、関連ドキュメント |
| 2 | `package.json` | npm scripts、Svelte、Tauri、QR、通知プラグインの依存関係 |
| 3 | `vite.config.ts` | Vite の開発サーバー設定。Tauri と同じ `127.0.0.1:1420` を使う |
| 4 | `src/main.ts` | Svelte アプリを DOM にマウントする入口 |
| 5 | `src/App.svelte` | 現在の UI とツールロジックの中心 |
| 6 | `src/app.css` | メニュー、各ツール、レスポンシブ表示のスタイル |
| 7 | `src/assets/menu/` | メニューカードで使う PNG アイコン |
| 8 | `src-tauri/src/lib.rs` | Tauri command、Bedrock/Tavily 連携、Tauri event 発行 |
| 9 | `src-tauri/tauri.conf.json` | アプリ名、ウィンドウサイズ、Vite 連携、バンドル設定 |
| 10 | `docs/technical-stack.md` | 採用技術と今後の拡張判断 |

## ディレクトリの役割

```text
.
├── docs/
│   ├── menu-icon-image-generation.md  メニューアイコン生成ルール
│   ├── project-walkthrough.md         このファイル
│   ├── tauri-mental-model.md          Tauri の見方
│   └── technical-stack.md             技術要素と方針
├── src/
│   ├── assets/menu/                   メニュー用 PNG アイコン
│   ├── App.svelte                     UI とフロントエンド側のツール処理
│   ├── app.css                        グローバルな見た目
│   ├── main.ts                        Svelte の起動処理
│   └── vite-env.d.ts                  Vite 用の型定義
├── src-tauri/
│   ├── src/
│   │   ├── lib.rs                     Tauri command と Builder 設定
│   │   └── main.rs                    ネイティブアプリ側の入口
│   ├── Cargo.toml                     Rust/Tauri 側の依存関係
│   └── tauri.conf.json                Tauri アプリ設定
├── package.json                       npm scripts とフロントエンド依存関係
├── package-lock.json                  npm の lockfile
├── tsconfig.json                      TypeScript 設定
├── tsconfig.node.json                 Node/Vite 用 TypeScript 設定
└── vite.config.ts                     Vite 設定
```

## 開発環境の準備

このプロジェクトでは Node.js/npm と Rust が必要です。Tauri は OS ごとに前提条件が異なるため、環境構築で失敗する場合は `docs/technical-stack.md` の「開発環境」も確認してください。

依存関係のインストールは、プロジェクトルートで行います。

```sh
npm install
```

## 起動方法

デスクトップアプリとして動かす場合は、Tauri の開発コマンドを使います。

```sh
npm run tauri:dev
```

このコマンドは内部で Vite の開発サーバーも起動します。`src-tauri/tauri.conf.json` の `beforeDevCommand` が `npm run dev` を呼び、`devUrl` が `http://127.0.0.1:1420` を参照します。

フロントエンドだけをブラウザで確認したい場合は、次のコマンドを使います。

```sh
npm run dev
```

Vite は `vite.config.ts` の設定により、`127.0.0.1:1420` で起動します。通知や Tauri command を含む動作は、ブラウザだけでは確認できない場合があります。

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

### UI と状態管理

現在の画面とツール処理は、ほぼ `src/App.svelte` に集約されています。`activeView` でメニューと各ツール画面を切り替え、各ツールの入力値、結果、エラーメッセージを Svelte のローカル state として持っています。

画面が増えた段階では、次の単位で分割を検討するとよいでしょう。

| 分割候補 | 目安 |
| --- | --- |
| `src/components/` | ツールごとの表示が大きくなったとき |
| `src/lib/` | Base64、色変換、diff など、UI と独立してテストしやすい処理が増えたとき |
| Rust module | Tauri command や外部 API 連携が増えたとき |

現時点では単一ファイルの見通しがやや重くなってきていますが、まだ外部状態管理ライブラリは使っていません。まずは機能単位で小さく切り出す方針が扱いやすいと考えられます。

### 保存しているデータ

一部の設定や履歴は `localStorage` に保存しています。

| 保存内容 | キー |
| --- | --- |
| ポモドーロの色テーマ | `my-tools:pomodoro-timer-palette` |
| 日付別メモ | `my-tools:daily-memos` |
| AI チャット設定 | `my-tools:ai-chat-settings` |
| AI チャット履歴 | `my-tools:ai-chat-conversations` |

AI チャット設定には Bedrock API キーや Tavily API キーが含まれます。現状はローカル保存の簡易実装です。より安全に扱う必要が出てきた場合は、OS のキーチェーン相当の仕組みや Tauri 側の保存方法を検討してください。

### Rust 側との接続

`src/App.svelte` から Tauri の `invoke` を使って Rust 側を呼び出しています。

```ts
void invoke<number>("current_timestamp").catch(() => undefined);
```

`current_timestamp` は、フロントエンドから Rust 側の command を呼べることを確認するための軽い command です。画面表示には戻り値を使っていません。

AI チャットでは、次の command を呼び出します。

```ts
const response = await invoke<BedrockConverseResponse>("bedrock_converse", {
  request: {
    api_key,
    region,
    model_id,
    messages,
    tavily_api_key,
    use_web_search,
  },
});
```

`bedrock_converse` は Rust 側で Amazon Bedrock Converse Stream API を呼び出し、必要に応じて Tavily Search API も利用します。ストリーミング中のテキストや検索状態は、Tauri event として Svelte 側へ通知されます。

| event 名 | 用途 |
| --- | --- |
| `ai-chat-stream-delta` | Bedrock の応答テキストを逐次 UI に反映する |
| `ai-chat-search-status` | Tavily 検索の開始、完了、エラー状態を表示する |

Tauri command を追加するときは、Rust 側で関数を定義するだけでなく、`tauri::generate_handler![...]` への登録も必要です。

## よく触る場所

| やりたいこと | 主に触る場所 | 補足 |
| --- | --- | --- |
| メニューのツールを増やす | `src/App.svelte`, `src/assets/menu/` | `ToolId`、画像 import、`tools` 配列、表示分岐を揃えます |
| 既存ツールの UI を変える | `src/App.svelte`, `src/app.css` | 画面ごとの class 名を追うと見つけやすいです |
| ブラウザ API で完結する変換処理を追加する | `src/App.svelte` または `src/lib/` | UI と独立する処理は `src/lib/` 化を検討します |
| OS 機能や外部 API を使う | `src-tauri/src/lib.rs` | Tauri command として公開します |
| npm script を追加する | `package.json` | チームで使う確認コマンドはここへ寄せます |
| Vite のポートを変える | `vite.config.ts`, `src-tauri/tauri.conf.json` | 両方の整合が必要です |
| ウィンドウサイズやアプリ名を変える | `src-tauri/tauri.conf.json` | 配布物の設定にも関係します |
| Rust の依存関係を追加する | `src-tauri/Cargo.toml` | 追加後は Tauri ビルドで確認します |

## 変更作業の進め方

小さな変更でも、次の流れで確認すると問題を切り分けやすくなります。

1. 変更対象の層を決める
2. フロントエンドだけで済む場合は `src/` 配下を変更する
3. OS 機能、通知、外部 API、秘密情報に近い処理が必要な場合は `src-tauri/` 側を確認する
4. `npm run build` でフロントエンドの型とビルドを確認する
5. Tauri command や通知を含む変更なら `npm run tauri:dev` で実機動作を確認する
6. 配布物に影響する変更なら `npm run tauri:build` まで確認する

## つまずきやすい点

- `npm run dev` と `npm run tauri:dev` は目的が違います。前者はブラウザ確認用、後者はデスクトップアプリ確認用です。
- Vite のポートを変える場合は、`vite.config.ts` と `src-tauri/tauri.conf.json` の両方を確認してください。
- Tauri command を追加しただけではフロントエンドから呼べません。`generate_handler` への登録が必要です。
- 通知機能は OS の権限や Tauri 実行環境に依存します。
- AI チャットは Bedrock API キー、モデル利用権限、リージョン、必要に応じて Tavily API キーが必要です。
- 配布段階では、macOS の署名・公証や Windows の署名を別途検討する必要があります。

## 参考ドキュメント

- `README.md`: 最短の起動・ビルド手順
- `docs/technical-stack.md`: 技術選定、配布、環境構築の考え方
- `docs/tauri-mental-model.md`: Tauri とこのプロジェクトの見方
- `docs/menu-icon-image-generation.md`: メニュー画像を追加・差し替えるときのルール
- `package.json`: 実際に使える npm scripts
- `src-tauri/tauri.conf.json`: Tauri と Vite の接続設定
