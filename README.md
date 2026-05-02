# my-tools

Tauri v2 + Vite + TypeScript + Svelte で構成した、シンプルなデスクトップアプリです。

現在は、現在時刻を表示する最小画面だけを実装しています。

## Development

実行ディレクトリ: プロジェクトルート（この `README.md` があるディレクトリ）

```sh
npm install
npm run tauri:dev
```

ブラウザでフロントエンドだけを確認する場合は、次のコマンドを使います。

実行ディレクトリ: プロジェクトルート（この `README.md` があるディレクトリ）

```sh
npm run dev
```

## Build

Tauri のビルドは、基本的に配布したい OS 上で実行します。特に Windows 向けの `.msi` は Windows 上での作成が前提です。

事前に依存関係をインストールしておきます。

実行ディレクトリ: プロジェクトルート（この `README.md` があるディレクトリ）

```sh
npm install
```

### macOS 向け

macOS で次のコマンドを実行します。

実行ディレクトリ: プロジェクトルート（この `README.md` があるディレクトリ）

```sh
npm run tauri:build
```

このプロジェクトでは `src-tauri/tauri.conf.json` の `bundle.targets` が `all` になっているため、macOS では主に次の成果物が作成されます。

- アプリ本体: `src-tauri/target/release/bundle/macos/*.app`
- 配布用 DMG: `src-tauri/target/release/bundle/dmg/*.dmg`

通常の配布では、`.dmg` を利用者へ渡します。App Store 外で広く配布する場合は、Apple Developer ID によるコード署名と notarization（公証）が必要です。署名・公証をしていない場合、利用者側で Gatekeeper の警告が表示されることがあります。

### Windows 向け

Windows で次のコマンドを実行します。

実行ディレクトリ: プロジェクトルート（この `README.md` があるディレクトリ）

```sh
npm run tauri:build
```

Windows では、設定に応じて次のようなインストーラーが作成されます。

- MSI インストーラー: `src-tauri/target/release/bundle/msi/*.msi`
- NSIS インストーラー: `src-tauri/target/release/bundle/nsis/*-setup.exe`

現在の設定では `bundle.targets` が `all` のため、Tauri が利用可能な Windows 向けバンドルを作成します。`.msi` の作成には WiX Toolset が使われるため、Windows 環境でビルドしてください。環境によっては Windows の VBSCRIPT オプション機能が必要になる場合があります。

Windows で一般配布する場合は、コード署名証明書でインストーラーへ署名することを推奨します。署名していないインストーラーは、Microsoft Defender SmartScreen などで警告が表示されることがあります。

### 配布前の確認

配布前に、次の点を確認してください。

- `package.json`、`src-tauri/Cargo.toml`、`src-tauri/tauri.conf.json` のバージョンが意図した値になっていること
- プロジェクトルート（この `README.md` があるディレクトリ）で `npm run tauri:build` が対象 OS 上で正常終了すること
- 生成された `.dmg`、`.msi`、`-setup.exe` を実機または検証用環境でインストールできること
- 初回起動時に OS の警告や権限確認が想定どおりであること
- 外部配布する場合は、macOS の署名・公証、Windows のコード署名を済ませていること

## Docs

- [技術要素ドキュメント](docs/technical-stack.md)
- [新人向けプロジェクト歩き方](docs/project-walkthrough.md)
- [Tauri とこのプロジェクトのメンタルモデル](docs/tauri-mental-model.md)
