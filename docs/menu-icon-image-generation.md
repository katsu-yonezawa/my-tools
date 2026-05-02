# メニューアイコン画像生成ルール

## 目的

このドキュメントは、`src/assets/menu/` に配置するメニューアイコン画像を、同じ品質と方向性で生成するためのルールをまとめたものです。

このアプリのメニュー画像は、機能をすばやく識別できることを最優先にします。装飾的な一枚絵ではなく、少し高級感のあるアプリアイコンとして見える画像を目指します。

## 現在の配置

現在は、次の PNG を `src/assets/menu/` に置いています。いずれも `720 x 720` の正方形です。

| ファイル | メニュー表示 | ツール ID |
| --- | --- | --- |
| `clock.png` | 時計 | `clock` |
| `uuid.png` | UUID | `uuid` |
| `json.png` | JSON | `json` |
| `datetime.png` | 日時変換 | `datetime` |
| `base64.png` | Base64 | `base64` |
| `url.png` | URL | `url` |
| `jwt.png` | JWT | `jwt` |
| `regex.png` | Regex | `regex` |
| `text-counter.png` | 文字数 | `text-counter` |
| `hash.png` | ハッシュ | `hash` |
| `diff.png` | Diff | `diff` |
| `qr.png` | QR | `qr` |
| `color.png` | 色変換 | `color` |
| `memo.png` | メモ | `memo` |
| `ai-chat.png` | AIチャット | `ai-chat` |

## 基本方針

メニューアイコンは、グローバルに見ても意味が伝わりやすい、単純で視認性の高いモチーフにします。ピクトグラムほど平面的にしすぎず、控えめな立体感と質感でアプリらしい印象を持たせます。

| 項目 | 方針 |
| --- | --- |
| 画像形式 | PNG |
| サイズ | 正方形。アプリ組み込み時は `720 x 720` に揃える |
| 構図 | 中央に単一の主モチーフを置く |
| 背景 | 明るい白系またはニュートラルな角丸タイル |
| スタイル | モダンな 3D アイコン、ソフトマットな質感、控えめな影 |
| 情報量 | 小さく表示しても分かる程度に抑える |
| 文字 | 原則として使わない |
| 地域性 | 特定国の色、旗、記号、文化的装飾に寄せない |

## 視認性の基準

生成後は、次の条件を満たしているか確認します。

- 一目で主機能が推測できる
- 128px 程度に縮小しても主要な形が読める
- 背景より主モチーフの輪郭がはっきりしている
- 小さな部品や細かい文字に意味を依存していない
- 他のメニュー画像と並べたとき、質感と余白が大きくずれていない

## 避ける表現

次の要素は、メニュー画面では情報量が増えやすいため避けます。

- 風景、室内、街並みなどのシーン表現
- 写実的な小物を多数置いた構図
- 文字列、コード断片、長い URL、数字列
- 国旗、星条旗、愛国色など特定地域に見える要素
- 過度に強いグラデーション、派手な光、反射
- サイバー風の複雑な回路、ネットワーク線、粒子表現
- 小さな装飾に意味を持たせる構図

## 共通プロンプトの型

imagegen で生成するときは、次の型を基本にします。`Primary request` と `Subject` だけを機能ごとに差し替えます。

```text
Use case: logo-brand
Asset type: square menu icon for a desktop utility app
Primary request: Create a premium, globally recognizable app icon for a <tool name> tool. The icon should be simple, polished, and easy to understand at a glance.
Style: refined modern 3D icon, soft matte materials, subtle depth, clean geometry, high-end productivity/developer utility aesthetic.
Subject: <single centered motif that directly represents the tool>
Backdrop: simple rounded-square tile background with a calm light neutral base and one restrained accent color. No country-specific styling, no flags, no patriotic colors.
Composition: single centered object, generous padding, strong silhouette, readable at small size.
Avoid: complex scenes, photorealistic clutter, tiny decorative objects, text, numbers, watermark, busy background, dominant gradients.
```

## ツール別モチーフ

| ツール | 主モチーフ | アクセント色の目安 | 補足 |
| --- | --- | --- | --- |
| 時計 | シンプルなアナログ時計、タイマーの目盛り | グレー、ブルーグレー、トマト色 | ポモドーロを連想できるが、食品表現へ寄せすぎない |
| UUID | 指紋風の抽象マーク、または一つだけ強調されたグリッド | ティール、ブルー | 文字や ID 表記に頼らず、唯一性を表現する |
| JSON | 中括弧と階層化されたブロック | グリーン | コード画面ではなく、構造化データの抽象表現にする |
| 日時変換 | カレンダー、時計、循環矢印 | バイオレット、インディゴ | 矢印は大きく単純にし、日付の数字は入れない |
| Base64 | 文書ブロック、ドットまたはタイルのパターン、双方向矢印 | アンバー | 文字列ではなく、データ変換として見せる |
| URL | リンクチェーン、ブラウザ枠、必要なら小さなパーセント記号 | シアン、ティール | 長い URL や細かいアドレスバー表現は避ける |
| JWT | 3 分割されたトークン、虫眼鏡、控えめなセキュリティ記号 | ネイビー、スレート、ゴールド | 署名検証を連想させすぎないよう、鍵や盾は補助程度にする |
| Regex | ドット、アスタリスク、検索ルーペ、パターンの抽象表現 | バイオレット、ブルー | `.*` の文字を主役にしすぎず、パターン検出として見せる |
| 文字数 | 文書、カウンター、整列した行 | ティール、グリーン | 小さな文字を読ませず、文字量の測定として表現する |
| ハッシュ | ダイジェスト、六角形、指紋風の抽象マーク | ブルー、インディゴ | 暗号資産やブロックチェーン風に寄せすぎない |
| Diff | 左右の文書、差分ハイライト、追加削除の抽象表現 | グレー、グリーン、レッド | 細かいコード行ではなく比較の概念を見せる |
| QR | QR コード風のブロック、スキャン枠 | シアン、ダークグレー | 実際に読ませる QR ではなく、視認用モチーフにする |
| 色変換 | カラーピッカー、スウォッチ、色相リング | コーラル、ティール、イエロー | 色数は増やしすぎず、中心モチーフを保つ |
| メモ | ノート、付箋、ペン | アンバー、落ち着いたイエロー | 文字は使わず、書き残す用途として表現する |
| AIチャット | チャット吹き出し、抽象的な AI マーク、簡素なノード | ティール、バイオレット | 吹き出しを主役にし、人物表現は避ける |

## 生成後の処理

生成した画像は、プロジェクト内で参照できるように `src/assets/menu/` へコピーします。元画像は imagegen の生成ディレクトリに残しておきます。

配置後は、次のように `720 x 720` に揃えます。

```sh
sips -Z 720 src/assets/menu/*.png
```

最後に、画像サイズとビルドを確認します。

```sh
sips -g pixelWidth -g pixelHeight src/assets/menu/*.png
npm run build
```

## 新しいメニュー画像を追加するとき

新しいツールを追加するときは、画像ファイルだけでなく `src/App.svelte` 側も合わせて更新します。

1. `src/assets/menu/<tool-id>.png` を追加する
2. `src/App.svelte` で画像を import する
3. `ToolId` に新しい ID を追加する
4. `tools` 配列に表示名、アクセント色、画像を追加する
5. `activeView` の表示分岐にツール画面を追加する
6. `npm run build` で import と型を確認する

## 差し替え時の注意

既存のメニュー画像を差し替える場合は、ファイル名を変えずに同じ場所へ配置します。`src/App.svelte` 側では画像を import して参照しているため、ファイル名が変わる場合は import と `tools` 定義の両方を更新してください。

アイコンの質感や明るさを変更するときは、1 枚だけ大きく雰囲気を変えず、全体の並びで統一感が出るように調整します。
