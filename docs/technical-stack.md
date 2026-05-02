# 技術要素ドキュメント

## 目的

このドキュメントは、`my-tools` の現在の技術構成と、今後拡張するときの判断材料を整理するものです。

このプロジェクトは、macOS と Windows の両方で動かすことを想定した Tauri v2 アプリです。画面は Svelte + TypeScript で作り、OS に近い機能や外部 API 連携は Rust 側に寄せます。現在は開発者向け・作業補助向けの小さなツールを、メニューから選んで使う構成になっています。

## 採用スタック

| 領域 | 採用技術 | 現在の役割 |
| --- | --- | --- |
| デスクトップアプリ基盤 | Tauri v2 | macOS / Windows 向けのネイティブアプリ化、ウィンドウ管理、通知プラグイン、IPC |
| フロントエンドビルド | Vite 6 | 開発サーバー、フロントエンドのビルド |
| 言語 | TypeScript | UI 側の型安全性とツール処理 |
| UI フレームワーク | Svelte 5 | メニュー、各ツール画面、画面内状態の管理 |
| スタイル | CSS | 全体レイアウト、ツール別パネル、レスポンシブ表示 |
| QR 生成 | `qrcode` | 入力テキストから QR コード画像を生成 |
| ネイティブ処理 | Rust | Tauri command、Bedrock/Tavily への HTTP 通信、Tauri event 発行 |
| HTTP クライアント | `reqwest` | Rust 側から Bedrock Converse Stream API と Tavily Search API を呼ぶ |
| データ保存 | `localStorage` | タイマー色、日付別メモ、AI 設定、会話履歴の簡易保存 |
| 配布 | Tauri Bundler | macOS の `.app` / `.dmg`、Windows の `.msi` / `setup.exe` 作成 |

## 現在の依存関係

### npm 側

`package.json` では、次の主要依存を使っています。

| パッケージ | 用途 |
| --- | --- |
| `svelte` | UI 実装 |
| `@sveltejs/vite-plugin-svelte` | Vite で Svelte を扱う |
| `@tauri-apps/api` | フロントエンドから Tauri command や event を扱う |
| `@tauri-apps/plugin-notification` | ポモドーロ終了通知 |
| `qrcode` | QR コード生成 |
| `typescript`, `vite` | 型チェックを含むフロントエンドビルド |

### Rust 側

`src-tauri/Cargo.toml` では、次の主要 crate を使っています。

| crate | 用途 |
| --- | --- |
| `tauri` | Tauri アプリ本体 |
| `tauri-plugin-notification` | OS 通知 |
| `tauri-plugin-opener` | Tauri 標準構成由来の opener プラグイン |
| `reqwest` | 外部 API への HTTP 通信 |
| `serde`, `serde_json` | Bedrock/Tavily のリクエスト・レスポンス処理 |

## 全体アーキテクチャ

```text
ユーザー操作
  ↓
Svelte UI
  - メニュー
  - 各ツール画面
  - 入力値、結果、エラー表示
  ↓
ブラウザ API または Tauri IPC
  - crypto
  - Web Crypto
  - File API
  - Clipboard API
  - localStorage
  - invoke(...)
  ↓
Rust バックエンド
  - current_timestamp
  - bedrock_converse
  - Bedrock Converse Stream
  - Tavily Search
  - Tauri event
  ↓
ローカル環境 / 外部 API
```

多くのツールはフロントエンドだけで完結します。たとえば UUID、JSON、日時変換、Base64、URL、JWT、正規表現、文字数、ハッシュ、Diff、QR、色変換は、Svelte 側とブラウザ API で処理しています。

一方で、AI チャットは Rust 側の `bedrock_converse` command を通して外部 API を呼びます。Bedrock のストリーミング応答は Rust 側で読み取り、`ai-chat-stream-delta` event として UI に渡します。Web 検索を使う場合は、Bedrock の tool use を受けて Tavily Search API を呼び、検索状態を `ai-chat-search-status` event で通知します。

## Tauri v2

Tauri は、OS 標準の WebView を利用してデスクトップアプリを構築するフレームワークです。macOS と Windows の両方に対応しており、必要に応じて Linux、Android、iOS も対象にできます。

### 採用理由

- Electron よりアプリサイズを小さくしやすい
- OS 標準の WebView を使うため、Chromium を同梱しない
- フロントエンドは通常の Web 技術で実装できる
- OS 連携や外部 API 呼び出しを Rust 側に分離できる
- macOS / Windows の配布物を Tauri CLI で作成できる

### 現在の Tauri 設定

`src-tauri/tauri.conf.json` では、開発時とビルド時のフロントエンド連携を次のように設定しています。

| 項目 | 値 |
| --- | --- |
| `beforeDevCommand` | `npm run dev` |
| `devUrl` | `http://127.0.0.1:1420` |
| `beforeBuildCommand` | `npm run build` |
| `frontendDist` | `../dist` |
| ウィンドウサイズ | `800 x 600` |
| 最小ウィンドウサイズ | `360 x 420` |
| `bundle.targets` | `all` |

Vite 側も `vite.config.ts` で `host: "127.0.0.1"`、`port: 1420`、`strictPort: true` に設定しています。ポートを変える場合は、Vite と Tauri の両方を合わせる必要があります。

## フロントエンド

フロントエンドは Svelte 5 + TypeScript です。現時点では `src/App.svelte` に UI とツール処理が集約されています。

この構成は、小さなツールを素早く追加する段階では扱いやすい一方、機能が増えるにつれてファイルが大きくなります。次のようなタイミングで分割を検討するとよいでしょう。

| 状態 | 対応 |
| --- | --- |
| 1 つのツールの UI が大きくなった | `src/components/<tool>/` へ分割する |
| UI と独立してテストしたい変換処理が増えた | `src/lib/` に関数を切り出す |
| 複数ツールで同じ UI パターンを使う | 共通コンポーネント化する |
| AI チャットの状態管理が複雑になった | 会話・設定・送信処理を専用 module に分ける |

外部状態管理ライブラリはまだ入れていません。現状では Svelte のローカル state で十分ですが、画面分割が進んだ段階で store の導入を検討できます。

## Rust バックエンド

Rust 側は、Tauri アプリのネイティブ処理と外部 API 連携を担当します。

現在の command は次の 2 つです。

| command | 用途 |
| --- | --- |
| `current_timestamp` | Rust command 呼び出し確認用の Unix 秒取得 |
| `bedrock_converse` | Bedrock Converse Stream API を呼び、必要に応じて Tavily Search API と連携する |

`bedrock_converse` は、Bedrock のストリーム形式を読み取り、テキスト差分を UI へ通知します。Bedrock が `web_search` tool を要求した場合は、Tavily Search API を呼んで検索結果を tool result として Bedrock に戻します。

Rust 側に新しい処理を追加する場合は、次の基準で考えます。

| Rust 側に寄せる処理 | 理由 |
| --- | --- |
| OS の権限に近い処理 | フロントエンドから直接扱うより責務を分けやすい |
| 外部 API 呼び出し | CORS、認証、エラー整形をまとめやすい |
| 秘密情報に近い処理 | 将来的に安全な保存方法へ移しやすい |
| 長時間処理や重い処理 | UI の反応を保ちやすい |

## データ保存

現時点では、軽い永続化に `localStorage` を使っています。

| 保存対象 | 現在の保存先 | 注意点 |
| --- | --- | --- |
| ポモドーロの色テーマ | `localStorage` | 失敗してもタイマー自体は動く |
| 日付別メモ | `localStorage` | 端末内の簡易保存であり、同期やバックアップはない |
| AI チャット設定 | `localStorage` | API キーを含むため、配布用途では保存方法の見直し候補 |
| AI チャット履歴 | `localStorage` | 最大 24 件まで保存 |

今後、設定や履歴をより安全・堅牢に扱う場合は、Tauri Store、SQLite、OS のキーチェーン相当の仕組みを検討します。特に API キーは、一般配布を見据えるなら `localStorage` のままにしない方がよいでしょう。

## 配布

Tauri では、OS ごとに配布形式を作成します。

| OS | 主な配布形式 | 備考 |
| --- | --- | --- |
| macOS | `.app`, `.dmg` | App Store 外配布では `.dmg` が一般的 |
| Windows | `.msi`, `setup.exe` | `setup.exe` は NSIS、`.msi` は WiX 系の扱いになる |

現在の設定では `bundle.targets` が `all` です。対象 OS 上で `npm run tauri:build` を実行すると、Tauri が利用可能な形式の配布物を作成します。

## 署名と公証

開発中や社内検証では、未署名のビルドでも進められます。ただし、一般ユーザーに配布する場合は署名を前提に考える必要があります。

### macOS

macOS で App Store 外配布をする場合、コード署名と公証を行わないと、起動時に警告が出たり、開けない扱いになる可能性があります。一般配布するなら Apple Developer Program の利用を見込んでおきます。

### Windows

Windows でもコード署名がない場合、SmartScreen による警告が表示されやすくなります。最初の検証段階では必須ではありませんが、顧客や一般ユーザーに配布する段階では署名証明書を用意する方針が安全です。

## 開発環境

### 共通

- Node.js LTS
- npm
- Rust
- Tauri CLI

### macOS

- Xcode または Xcode Command Line Tools
- Apple Silicon / Intel Mac の両方を考慮する場合はビルドターゲットを確認

### Windows

- Microsoft C++ Build Tools
- Microsoft Edge WebView2
- Rust の MSVC toolchain

Windows 用アプリのビルドは Windows 上で行うのがもっとも安定します。macOS から Windows 向けにクロスビルドする方法もありますが、初期運用では GitHub Actions の Windows runner または実機 Windows 環境でのビルドを推奨します。

## 今後の整理候補

現在は機能が `src/App.svelte` に集まっています。大きな問題ではありませんが、次の変更を行う場合は、先に構造を少し整えると作業しやすくなります。

| 変更内容 | 整理候補 |
| --- | --- |
| ツールをさらに増やす | `tools` 定義と画面コンポーネントを分離する |
| 各ツールのテストを増やす | 変換処理を `src/lib/` へ切り出す |
| AI チャットを強化する | 設定、会話履歴、送信処理、表示を分ける |
| API キーを安全に扱う | Rust 側保存や OS キーチェーン相当の仕組みを検討する |
| 履歴やメモを検索したい | SQLite などのローカル DB を検討する |

## 判断基準

この構成を維持するか見直すかは、次の観点で判断します。

| 判断軸 | 現在のままでよい状態 | 見直しを検討する状態 |
| --- | --- | --- |
| アプリサイズ | 小さく保ちたい | Node.js 互換性を優先したい |
| UI | 単一画面内のツール集で足りる | 画面遷移や複雑な状態管理が増える |
| OS 連携 | Tauri command で十分扱える | Node.js ライブラリを大量に使いたい |
| データ保存 | 設定や軽い履歴で足りる | 検索、同期、暗号化、複数端末利用が必要 |
| AI 連携 | Bedrock/Tavily の単純な呼び出しで足りる | 複数プロバイダ、認証管理、監査ログが必要 |
| 配布 | macOS / Windows の通常配布で足りる | 自動更新、企業配布、厳格な署名運用が主課題になる |

現時点では、ローカルで使う小さなデスクトップツール集という目的に対して、Tauri v2 + Svelte は妥当な構成です。次に大きく効いてくる判断は、`App.svelte` の分割、API キー保存方法、配布時の署名・公証です。

## 参考リンク

- [Tauri](https://tauri.app/)
- [Tauri Prerequisites](https://v2.tauri.app/start/prerequisites/)
- [Tauri Windows Installer](https://v2.tauri.app/distribute/windows-installer/)
- [Tauri DMG](https://v2.tauri.app/distribute/dmg/)
- [Tauri macOS Code Signing](https://v2.tauri.app/distribute/sign/macos/)
- [Amazon Bedrock Converse API](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html)
- [Tavily API](https://docs.tavily.com/)
