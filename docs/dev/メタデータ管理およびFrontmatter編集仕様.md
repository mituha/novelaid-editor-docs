# メタデータ管理およびFrontmatter編集実装仕様書

## 1. 概要と背景

小説の執筆において、キャラクター設定、地名・施設、プロット、伏線、年表などの「設定資料」は作品のクオリティを左右する重要な情報です。
本仕様は、**`novelaide`** において Markdown ファイル先頭の **YAML Frontmatter** を活用し、執筆者が設定資料や原稿の属性を直感的に閲覧・入力できる **メタデータ管理機能（`MetadataPanel` / `MetadataListPanel`）** の実装仕様を定義するものです。

先行プロジェクトである `novelaid-editor`（Electron版）および `novelaid-editor-next`（Tauri版）の知見を継承し、`novelaide` の統合UIプラグインアーキテクチャ（`PanelRegistry`）に完全に適合した疎結合な形で実装します。

---

## 2. アーキテクチャ設計

```mermaid
flowchart TD
    subgraph Document["Markdown ドキュメント (DocumentContext)"]
        FileContent["ファイル全体テキスト\n(--- YAML Frontmatter ---\n本文...)"]
        Parser["Frontmatter パーサー & シリアライザー\n(js-yaml / gray-matter / 独自パーサー)"]
        State["DocumentState\n- content (本文)\n- metadata (Record<string, any>)\n- rawFrontmatter"]
        FileContent <--> Parser <--> State
    end

    subgraph RightPane["右サイドペイン (ドキュメント個別)"]
        MetaPlugin["メタデータパネルプラグイン\n(novelaid.panel.metadata)"]
        MetaPanel["MetadataPanel\n- 基本情報 (title, tags, summary)\n- クイックタグトグル (登場人物/地名/プロット)\n- 詳細属性 (id, name, reading, role等)\n- カスタムフィールド動的追加"]
        MetaPlugin --> MetaPanel
    end

    subgraph LeftPane["左サイドペイン (プロジェクト全体)"]
        ListPlugin["設定資料一覧プラグイン\n(novelaid.panel.characters / metadataList)"]
        ListPanel["MetadataListPanel\n- 登場人物一覧 (character タグ)\n- 地名・舞台一覧 (location タグ)\n- プロット一覧 (plot タグ)"]
        ListPlugin --> ListPanel
    end

    State <-->|"双方向リアクティブ同期"| MetaPanel
    State -.->|"プロジェクト内タグ収集"| ListPanel
```

---

## 3. 機能要件

### 3.1 Frontmatter の双方向同期基盤
1. **パース（読み込み）**:
   - ファイルオープン時、先頭の `---` で囲まれた YAML ブロックをパースし、オブジェクト（`Record<string, any>`）として `metadata` ステートに保持。
   - `---` が存在しないファイルでも、メタデータパネルで編集が開始された際に自動で先頭に `---` を付与。
2. **シリアライズ（保存）**:
   - メタデータパネルで値（タイトル、タグ、あらすじ等）が変更された場合、本文（Body）と結合してファイル内容をリアルタイム更新。
   - エディターで直接 Frontmatter の YAML テキストを編集した場合も、メタデータパネルのフォームに即時反映。

### 3.2 ドキュメントメタデータ編集パネル (`MetadataPanel`)
右ペイン用プラグイン（`novelaid.panel.metadata` / アイコン: `Database` または `Tag` / `defaultPlacement: 'right'`）として提供。

1. **基本プロパティ**:
   - **タイトル (`title`)**: ドキュメントの正式名称や話数タイトル
   - **クイック分類タグ (`tagGroups`)**:
     - 👤 **登場人物** (`character` / `chara`)
     - 📍 **地名・施設** (`location` / `place`)
     - 📜 **プロット・年表** (`plot` / `timeline`)
     - ワンクリックでトグル追加/削除可能
   - **タグ入力欄 (`tags`)**: カンマ区切りの自由なキーワードタグ
   - **あらすじ / 概要メモ (`summary` / `description`)**: 複数行テキストエリア
2. **キャラクター・設定資料用 詳細プロパティ**:
   - **ID (`id`)**: 固有識別子（例: `chara_reo`）
   - **表示名 / 本名 (`name`)**: キャラクター名（例: `神代 玲央`）
   - **フリガナ (`reading`)**: 読み仮名（例: `かみしろ れお`）
   - **年齢 / 性別 (`age`, `gender`)**: 属性情報
   - **立場・役割 (`role`)**: 主人公、ヒロイン、敵役など
3. **カスタムフィールドの動的追加・削除**:
   - ユーザーが独自のキー（例: `magic_attribute`, `birthday`, `faction` など）と値を自由に追加・削除できる拡張フォーム。

### 3.3 プロジェクト設定資料・登場人物一覧パネル (`MetadataListPanel`)
プロジェクト内の全ファイルからメタデータを抽出し、カテゴリごとに一覧・検索・クイックアクセスできるパネル。

- **登場人物一覧**: `character` タグを持つファイル（キャラクター設定シート）を一覧表示。クリックでそのファイルを開く。
- **地名・世界観一覧**: `location` タグを持つファイルの一覧表示。
- **プロット・構成一覧**: `plot` タグを持つファイルの一覧表示。

---

## 4. UIデザイン方針

- **右ペインとの自然な統合**:
  - 見出しアウトライン（`novelaid.panel.outline`）と並ぶ右ペインの第2パネルとして配置。
  - 右アクティビティバーに `Database` または `Tags` アイコンを追加。
- **テーマ連動**:
  - ダーク・ライト・セピアの各テーマに最適化された入力フォーム、ボタンスタイル、バッジ配色。
- **キーボードショートカット**:
  - `Ctrl + Shift + M`（macOS: `Cmd + Shift + M`）でメタデータパネルの表示/非表示をトグル可能にする。

---

## 5. 段階的実装計画（全4ステップ）

| ステップ | 作業内容 | 対象ファイル |
| :--- | :--- | :--- |
| **Step 1** | **Frontmatter パース & 双方向同期基盤の構築**<br>- YAMLパース/シリアライズユーティリティの導入<br>- `DocumentContext` への `metadata`, `updateMetadata` 統合 | `src/utils/frontmatter.ts`<br>`src/contexts/DocumentContext.tsx`<br>`src/types/document.ts` |
| **Step 2** | **ドキュメントメタデータ編集パネルの実装**<br>- `MetadataPanel` コンポーネント作成<br>- 基本情報、クイックタグ、キャラクター属性、カスタムフィールド<br>- 右ペインプラグイン（`novelaid.panel.metadata`）登録 | `src/components/Metadata/MetadataPanel.tsx`<br>`src/components/Metadata/MetadataPanel.css`<br>`src/plugins/panels/MetadataPanelPlugin.tsx`<br>`src/core/panelRegistry/index.ts` |
| **Step 3** | **設定資料・登場人物一覧パネルの実装**<br>- プロジェクト横断でのタグ別メタデータ抽出フック<br>- `MetadataListPanel` コンポーネント作成<br>- アコーディオンツリーによる一覧・ワンクリックオープン | `src/hooks/useProjectMetadata.ts`<br>`src/components/Metadata/MetadataListPanel.tsx`<br>`src/plugins/panels/MetadataListPanelPlugin.tsx` |
| **Step 4** | **検証・自動スクリーンショット撮影 & ドキュメント更新**<br>- Playwright撮影スクリプト更新（メタデータパネル表示状態）<br>- `docs/manual/メタデータと設定資料の使い方.md` 作成<br>- メイキング連載第16回記事の執筆と目次更新<br>- バージョン更新 & Gitコミット | `scripts/capture-screenshots.js`<br>`docs/manual/メタデータと設定資料の使い方.md`<br>`making/notes/16_メタデータ管理とFrontmatter編集パネルの実装.md`<br>`making/logs/2026-10-24.md` |
