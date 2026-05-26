# DocumentContext 仕様

このドキュメントでは、`novelaid-editor`（Electron版）および `novelaid-editor-next`（Tauri版）におけるドキュメント管理の中心的な仕組みである `DocumentContext` と、その内部状態を保持する `DocumentState` の仕様についてまとめます。

---

## 1. 概要
`DocumentContext` は、アプリケーション上で開かれているすべてのドキュメントの状態（ファイルパス、タイトル、本文、メタデータ、未保存フラグなど）や、2ペイン（左右分割画面）における表示位置・モードを集中管理するための React Context です。

---

## 2. DocumentState の仕様
`DocumentState` は、現在メモリ上に開かれている各ドキュメントの最新状態を表すインターフェースです。

### 共通の基本フィールド

| フィールド名 | 型 | 説明 |
| :--- | :--- | :--- |
| `path` | `string` | ドキュメントの絶対パス（または `browser://`, `gitDiff://` などの仮想URIスキーム）。 |
| `baseName` | `string` | 拡張子を含むファイル名（例: `chapter1.txt`）。 |
| `fileTitle` | `string` | 拡張子を除いたファイルタイトル、またはメタデータに指定された表示名。 |
| `content` | `string` | ドキュメントのテキスト本文（メモリ上の最新バッファ）。 |
| `metadata` | `Record<string, any>` | YAMLフロントマター等から抽出されたメタデータのオブジェクト。 |
| `isDirty` | `boolean` | 未保存の変更があることを示すフラグ。 |
| `leftMainView` | `DocumentViewMode` | 左側ペインのメインビュー表示モード。 `'none'` の場合は表示されません。 |
| `rightMainView` | `DocumentViewMode` | 右側ペインのメインビュー表示モード。 `'none'` の場合は表示されません。 |
| `leftPreviewView` | `DocumentViewMode` | 左側ペインのプレビュー表示モード（通常 `'none'` または `'preview'`）。 |
| `rightPreviewView` | `DocumentViewMode` | 右側ペインのプレビュー表示モード（通常 `'none'` または `'preview'`）。 |

> **DocumentViewMode**
> ビューポートごとの表示モードを指します。
> 主な値: `'none' | 'editor' | 'canvas' | 'reader' | 'preview'`

### プラットフォーム別の固有フィールド

#### Electron版 (`novelaid-editor`) 固有
* **`documentType`** (`NovelaidDocumentType`): `novelaid-fs` で判定されたドキュメントタイプ（`'novel' | 'markdown' | 'chat' | 'image' | 'browser' | 'gitDiff' | 'unknown'`）。
* **`lastSource`** (`string`): 変更の発生源（自動保存や外部変更同期時の競合回避のため、`'user-left'`, `'user-right'`, `'external'` などを格納）。
* **`initialLine` / `initialColumn`** (`number`): ファイルを開いた直後にカーソルをジャンプさせる対象の行・列番号。
* **`searchQuery`** (`string`): 開いた直後にエディタ内で検索ハイライトする文字列。
* **`deleted`** (`boolean`): ファイルが外部または内部から削除されたかどうかの状態フラグ。
* **`isRenaming`** (`boolean`): ファイル名変更（リネーム）処理の実行中フラグ。
* **`openPanelIds`** (`string[]`): サイドバーパネルなど、エディタ外の領域で開かれているパネルIDのリスト。
* **`isPanel`** (`boolean`): 互換性維持のための古いフラグ。

#### Tauri版 (`novelaid-editor-next`) 固有
* **`documentType`** (`NovelaidDocumentType`): `tauri-plugin-novelaid-fs-api` から提供されるドキュメントタイプ。

---

## 3. TabItem の仕様
`TabItem` は、各ペイン（左・右）で現在アクティブになっているタブを特定するために使用される軽量な構造体です。

```typescript
export interface TabItem {
  path: string;       // 対象ドキュメントの path
  isPreview: boolean; // プレビューモードとして開かれているかどうか
}
```

---

## 4. DocumentContextType (提供されるAPIと機能)

### 共通で提供される機能
* **`openDocument`**: 指定されたパスのドキュメントを開き、指定ペインのアクティブタブに設定する。
* **`toggleSplit`**: エディタの2ペイン（左右）分割表示を切り替える。
* **`changeViewMode`**: 特定のペインのドキュメントの表示モードを変更する。
* **`saveDocument`**: 対象ドキュメントの内容をファイルシステムに書き込み、`isDirty` を `false` にリセットする。

### Electron版 (`novelaid-editor`) の特徴的な機能
* **`leftTabs` / `rightTabs`**: `openDocuments` の状態から動的に算出される、タブバー（`TabBar`）描画用のタブ情報リスト。
* **`openPanelDocument`**: メインエディタではなく、サイドパネル領域でドキュメントを読み込むための機能。
* **`renameDocument`**: コンテキストを維持したまま、ファイル名変更および対応するタブパスの自動更新を行う。
* **`openDiff` / `openWebBrowser`**: Git差分ビューや、投稿用の仮想Webブラウザタブを開くための特化関数。
* **`closeOtherTabs`**: 指定されたタブ以外のすべてのタブを一括で閉じます。
* **`closeAllTabs`**: 指定されたサイド（左または右）のすべてのタブを一括で閉じます。

### Tauri版 (`novelaid-editor-next`) の特徴的な機能
* **`splitRatio` / `setSplitRatio`**: 左右ペインの分割比率（0〜1の数値）を調整する状態管理。
* **`activeFilePath` / `content` / `metadata` / `isDirty`**: 現在選択されているアクティブドキュメントの情報に直接アクセスするための簡易（レガシー互換）ゲッター。