# DocumentContext 仕様

## 1. 概要と設計理念

`DocumentContext` は、`novelaid` シリーズ（`novelaid-editor`, `novelaid-editor-next`, `novelaide`）におけるドキュメント管理の中心的な仕組みであり、開かれている全ドキュメントの状態（ファイルパス、タイトル、本文バッファ、メタデータ、未保存フラグなど）や表示ペインを一元管理する React Context です。

### 1.1. 小説執筆における状態管理の理念
小説執筆におけるドキュメント管理は、単一ファイルの編集にとどまりません。

1. **章・話の並行推敲**:
   前後の章や別エピソードを参照・比較しながら執筆するため、マルチタブや画面分割（2ペイン）が必須となります。
2. **メモリバッファと保存（Dirty）の厳格な分離**:
   執筆中の思考を中断させないため、キー入力ごとのI/O負荷を避けつつ、未保存状態（`isDirty`）を常に作家へ明示し、`Ctrl + S` やオートセーブで確実に永続化します。
3. **メタデータと本文の不可分な連動**:
   本文編集と同時に、登場人物のタグや伏線ステータスなどのYAMLフロントマターを双方向で整合性を保ちます。

### 1.2. 実装の変遷と novelaide での改善意図
- **現行（novelaid-editor）**: 1200行を超える巨大な Context であり、Git差分やWebブラウザタブ、パネル状態まで同居するモノリス状態でした。
- **新標準（novelaide）での改善**:
  - **関心事の分離**: ドキュメント本文とタブ管理（`DocumentContext`）に責任を絞り込み、Git履歴やAIパネル等は独立したコンテキスト/プラグインへ移譲。
  - **`tauri-plugin-novelaid-fs` との連携**: `ts-rs` で生成された `NovelaidDocument` 型に完全準拠し、Rustバックエンドとの安全なデータ同期を実現。

---

## 2. DocumentState の仕様
`DocumentState` は、現在メモリ上に開かれている各ドキュメントの最新状態を表すインターフェースです。

### 共通の基本フィールド

| フィールド名 | 型 | 説明 |
| :--- | :--- | :--- |
| `path` | `string` | ドキュメントの絶対パス（または仮想URIスキーム）。 |
| `baseName` | `string` | 拡張子を含むファイル名（例: `01_プロローグ.md`）。 |
| `fileTitle` | `string` | 拡張子を除いたファイルタイトル、またはメタデータに指定された表示名。 |
| `content` | `string` | ドキュメントのテキスト本文（メモリ上の最新バッファ）。 |
| `metadata` | `Record<string, any>` | YAMLフロントマター等から抽出されたメタデータのオブジェクト。 |
| `documentType` | `NovelaidDocumentType` | 小説種別（`novel`, `markdown`, `image` 等）。 |
| `isDirty` | `boolean` | 未保存の変更があることを示すフラグ。 |
| `leftMainView` | `DocumentViewMode` | 左側ペインのメインビュー表示モード（`'none' \| 'editor' \| 'preview'` 等）。 |
| `rightMainView` | `DocumentViewMode` | 右側ペインのメインビュー表示モード。 |

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

### 共通で提供される基本機能
* **`openDocument(path)`**: 指定されたパスのドキュメントを開き、アクティブタブに設定する。
* **`closeDocument(path)`**: 指定されたドキュメントを閉じ、タブ一覧から除去する。
* **`setActivePath(path)`**: タブの切り替え。
* **`updateContent(path, content)`**: エディター入力による本文バッファの更新と `isDirty: true` 設定。
* **`saveDocument(path?)`**: 対象ドキュメントの内容をファイルシステムに安全保存し、`isDirty: false` にリセットする。
* **`toggleSplit`**: エディターの2ペイン（左右分割）表示を切り替える。
