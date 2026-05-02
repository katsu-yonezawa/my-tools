# 技術要素ドキュメント

## 目的

このドキュメントは、macOS と Windows の両方で動作するシンプルなクライアントアプリを作成するための技術要素を整理するものです。

初期方針としては、軽量なデスクトップアプリ基盤である Tauri v2 を中心に、Web フロントエンド技術と必要最小限の Rust 実装を組み合わせます。まずは小さく作り、必要になった機能だけを段階的に追加できる構成を採用します。

## 採用スタック

| 領域 | 採用技術 | 役割 |
| --- | --- | --- |
| デスクトップアプリ基盤 | Tauri v2 | macOS / Windows 向けのネイティブアプリ化、ウィンドウ管理、OS 機能との連携 |
| フロントエンドビルド | Vite | 高速な開発サーバー、フロントエンドのビルド |
| 言語 | TypeScript | UI 側の型安全性と保守性の確保 |
| UI フレームワーク | Svelte または React | 画面・状態・イベント処理の実装 |
| ネイティブ処理 | Rust | Tauri command、ファイル操作、OS 連携、重めの処理 |
| データ保存 | Tauri Store / SQLite | 設定や小規模データの保存。必要に応じて SQLite を採用 |
| 配布 | Tauri Bundler | macOS の `.app` / `.dmg`、Windows の `.msi` / `setup.exe` 作成 |
| CI | GitHub Actions | macOS / Windows それぞれのビルド、配布物作成 |

## 全体アーキテクチャ

```text
ユーザー操作
  ↓
フロントエンド UI
  - Vite
  - TypeScript
  - Svelte または React
  ↓
Tauri IPC
  ↓
Rust バックエンド
  - Tauri command
  - ファイル操作
  - OS 連携
  - 必要に応じた外部プロセス実行
  ↓
ローカル環境
  - 設定ファイル
  - SQLite
  - OS 標準機能
```

Tauri アプリは、UI を Web 技術で実装し、OS に近い処理を Rust 側で実装します。フロントエンドと Rust は Tauri の IPC を通してやり取りします。

この構成では、画面やフォームなどの変更は TypeScript 側で素早く進められます。一方、ファイルシステム、プロセス起動、ネイティブ通知、ローカルデータ処理などは Rust 側に閉じ込められるため、責務が分かりやすくなります。

## Tauri v2

Tauri は、OS 標準の WebView を利用してデスクトップアプリを構築するフレームワークです。macOS と Windows の両方に対応しており、必要に応じて Linux、Android、iOS も対象にできます。

### 採用理由

- Electron よりアプリサイズを小さくしやすい
- OS 標準の WebView を使うため、Chromium を同梱しない
- フロントエンドは通常の Web 技術で実装できる
- OS 連携が必要な箇所だけ Rust で実装できる
- macOS / Windows の配布物を Tauri CLI で作成できる

### 注意点

- 開発環境には Rust が必要
- Windows ビルドには Microsoft C++ Build Tools と WebView2 が必要
- macOS ビルドには Xcode または Xcode Command Line Tools が必要
- 一般配布では、macOS のコード署名・公証、Windows のコード署名を考慮する必要がある
- OS 固有の挙動は完全にはなくならないため、macOS と Windows の実機確認が必要

## フロントエンド

フロントエンドは Vite + TypeScript を基本にします。UI フレームワークは、プロジェクトの規模と開発者の慣れに応じて選択します。

### Svelte を選ぶ場合

Svelte は小規模なツールアプリと相性がよい選択肢です。記述量が少なく、状態管理も比較的単純に保ちやすいため、シンプルなクライアントアプリでは扱いやすい構成になります。

向いているケース:

- 画面数が少ない
- 状態管理が複雑ではない
- 軽量な UI を素早く作りたい
- React 前提の資産が多くない

### React を選ぶ場合

React はライブラリや知見が豊富で、チーム開発や将来的な拡張に向いています。既存の React 経験やコンポーネント資産がある場合は、React を選ぶ価値があります。

向いているケース:

- React に慣れている
- 既存の React コンポーネントや設計資産がある
- 将来的に画面数が増える見込みがある
- 外部 UI ライブラリを使いたい

### 初期推奨

特に既存資産がなければ、初期構成は Svelte を推奨します。最小構成のデスクトップツールでは、実装量を抑えやすく、アプリ全体も見通しよく保てます。

ただし、開発者が React に十分慣れている場合は React で問題ありません。この選択はアプリの成否を大きく左右するものではなく、保守する人が読みやすい方を選ぶのが実務上は重要です。

## Rust バックエンド

Rust 側は、Tauri アプリのネイティブ処理を担当します。初期段階では、過度に Rust 側へ寄せず、OS 連携やセキュリティ上 UI から直接扱いたくない処理だけを実装します。

主な担当範囲:

- ファイルの読み書き
- OS のパス取得
- 外部コマンドの実行
- 設定ファイルの保存
- SQLite へのアクセス
- ネイティブ通知
- アプリ更新処理

UI の状態管理や表示ロジックは、原則として TypeScript 側に置きます。Rust 側は小さな command を定義し、フロントエンドから必要なタイミングで呼び出す形にします。

## データ保存

初期段階では、保存するデータの性質に応じて選択します。

| 保存対象 | 推奨 |
| --- | --- |
| アプリ設定、表示設定、小さなプリファレンス | Tauri Store |
| 一覧、履歴、検索対象になるデータ | SQLite |
| 一時的な UI 状態 | フロントエンドの state |
| 秘密情報、トークン | OS のキーチェーン相当の仕組みを検討 |

最初から SQLite を入れると構成は少し増えます。設定値だけで足りるうちは Tauri Store で始め、検索や履歴管理が必要になった段階で SQLite を追加する方針が扱いやすいです。

## 配布

Tauri では、OS ごとに配布形式を作成します。

| OS | 主な配布形式 | 備考 |
| --- | --- | --- |
| macOS | `.app`, `.dmg` | App Store 外配布では `.dmg` が一般的 |
| Windows | `.msi`, `setup.exe` | `setup.exe` は NSIS、`.msi` は WiX 系の扱いになる |

基本的には、macOS は `.dmg`、Windows は `setup.exe` を初期候補にします。Windows の `.msi` は企業配布では便利ですが、ビルド要件や運用を確認してから採用するのがよいです。

## 署名と公証

開発中や社内検証では、未署名のビルドでも進められます。ただし、一般ユーザーに配布する場合は署名を前提に考える必要があります。

### macOS

macOS でブラウザ配布する場合、コード署名と公証を行わないと、起動時に警告が出たり、開けない扱いになる可能性があります。一般配布するなら Apple Developer Program の利用を見込んでおきます。

### Windows

Windows でもコード署名がない場合、SmartScreen による警告が表示されやすくなります。最初の検証段階では必須ではありませんが、顧客や一般ユーザーに配布する段階では署名証明書を用意する方針が安全です。

## 開発環境

### 共通

- Node.js LTS
- npm または pnpm
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

## 初期ディレクトリ方針

Tauri の標準構成を大きく崩さず、次のような形で始めます。

```text
.
├── src/
│   ├── App.svelte または App.tsx
│   ├── components/
│   ├── lib/
│   └── main.ts
├── src-tauri/
│   ├── src/
│   │   └── main.rs
│   ├── Cargo.toml
│   └── tauri.conf.json
├── docs/
│   └── technical-stack.md
├── package.json
└── README.md
```

フロントエンドの共通処理は `src/lib/`、UI 部品は `src/components/` に置きます。Rust 側は最初から細かく分けすぎず、command が増えてきた段階で module 分割します。

## 初期実装の進め方

1. Tauri + Vite + TypeScript の最小プロジェクトを作成する
2. macOS で起動確認する
3. 空のメイン画面を作成する
4. Tauri command を 1 つ追加し、フロントエンドから呼び出せることを確認する
5. 設定保存が必要になった段階で Tauri Store を追加する
6. Windows 環境または CI でビルド確認する
7. 配布形式を `.dmg` と `setup.exe` に絞って検証する
8. 一般配布が見えた段階で署名・公証の準備を進める

## 判断基準

この構成を維持するか見直すかは、次の観点で判断します。

| 判断軸 | Tauri のままでよい状態 | 見直しを検討する状態 |
| --- | --- | --- |
| アプリサイズ | 小さく保ちたい | サイズより Node.js 互換性を重視したい |
| OS 連携 | 必要な処理を Rust に寄せられる | Node.js のライブラリを大量に使いたい |
| 開発速度 | Web UI と少量の Rust で進められる | Rust がチームの負担になっている |
| 配布 | macOS / Windows の通常配布で足りる | 自動更新、署名、企業配布が主課題になっている |
| UI | 通常の Web UI で十分 | 高度なネイティブ UI が必要 |

現時点では、シンプルな macOS / Windows クライアントアプリという目的に対して、Tauri v2 は妥当な選択です。最初は機能を絞り、ビルドと配布の流れを早めに確認することが重要です。

## 参考リンク

- [Tauri](https://tauri.app/)
- [Tauri Prerequisites](https://v2.tauri.app/start/prerequisites/)
- [Tauri Windows Installer](https://v2.tauri.app/distribute/windows-installer/)
- [Tauri DMG](https://v2.tauri.app/distribute/dmg/)
- [Tauri macOS Code Signing](https://v2.tauri.app/distribute/sign/macos/)
