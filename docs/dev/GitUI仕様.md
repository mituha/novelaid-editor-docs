# Git UI ライブラリ仕様書

本ドキュメントは、`novelaid-editor` の Git UI 部分を独立した React コンポーネントライブラリとして分離・ライブラリ化するための設計およびインターフェース仕様をまとめたものです。

---

## 1. 背景と目的

現在の Git UI 実装（`GitPanel`、`GitGraph`、`DiffViewer`）は、Electron 環境（IPC 通信）およびエディタ固有の React コンテキスト（`ProjectContext`、`DocumentContext`）に強く依存しています。
これらを疎結合な React ライブラリとして再設計し、以下の目的を達成します。

- **再利用性の向上**: Electron 以外の環境（Webブラウザ + WASM Git、一般的な Node.js / Web アプリなど）でも同じ UI コンポーネント群を利用可能にする。
- **テスタビリティの向上**: モック実装を注入しやすくすることで、UI コンポーネント単体でのビジュアルテストやユニットテストを容易にする。
- **ホストアプリとの分離**: UI デザインの変更や機能追加が、本体のテキストエディタ機能へ悪影響を及ぼさないように隔離する。

---

## 2. 現在の実装の分析

現在の `novelaid-editor` 内の Git 関連ファイルの依存関係および役割は以下の通りです。

```mermaid
graph TD
    subgraph Renderer Process (Host)
        useProject[ProjectContext / projectPath]
        useDocument[DocumentContext / openDiff]
    end

    subgraph Git UI Component (Target)
        GP[GitPanel.tsx] --> GG[GitGraph.tsx]
        GP --> DV[DiffViewer.tsx]
        GP --> GC[GitContext.tsx]
        DV --> GC
    end

    subgraph Electron IPC Bridge
        Preload[preload.ts / window.electron.git]
    end

    subgraph Main Process (Backend)
        GS[GitService.ts] --> SG[simple-git]
    end

    GC --> Preload
    DV --> Preload
    Preload --> GS
    GP -.-> useDocument
    GC -.-> useProject
```

### 抽出された主な課題
1. **プラットフォーム（Electron API）への直接依存**
   `GitContext` や `DiffViewer` が、グローバルな `window.electron.git` や `window.electron.ipcRenderer` を直接参照しています。
2. **ホスト側の React コンテキストへの依存**
   - リポジトリパスの取得に `useProject()` (`ProjectContext`) を使用している。
   - 差分表示のハンドリングに `useDocument()` (`DocumentContext` の `openDiff`) を使用している。
3. **ファイル監視のホスト依存**
   - ファイル変更時に Git ステータスを自動更新する処理が、`window.electron.fs.onFileChange` に依存している。
4. **スタイルのハードコーディング**
   - アプリ全体の CSS 変数（例: `var(--border-color)`、`var(--text-color)` 等）を前提にスタイルが定義されている。
5. **言語/ロケール**
   - "ステージ済みの変更" や "コミット" などの文言が日本語で直接ハードコードされている。

---

## 3. ライブラリ化後のアーキテクチャ

ライブラリ化にあたり、依存性を排除するために **Adapter パターン** を導入し、実行環境やホストアプリの機能はすべて外部から注入する設計に変更します。

```mermaid
graph TD
    subgraph Host Application (e.g. novelaid-editor)
        HostProject[Project Path]
        HostTab[Tab Manager / Diff View Opener]
        ElectronGitClient[ElectronGitClient]
        FileWatcher[File Watcher]
    end

    subgraph Git UI Library (React Package)
        GP[GitPanel] --> GG[GitGraph]
        GP --> DV[DiffViewer]
        GP --> GL_Context[GitProvider]
        GL_Context --> GitClient_Interface[GitClient Interface]
    end

    ElectronGitClient -- Implements --> GitClient_Interface
    HostProject -- Pass as Prop --> GL_Context
    HostTab -- Pass callback Prop --> GP
    FileWatcher -- Bind to --> GL_Context
```

---

## 4. インターフェース定義

### 4.1 GitClient (Adapter インターフェース)
Git 操作の実体（Electron IPC、Node-Git、WASM-Git等）を抽象化するインターフェースです。ホストアプリ側はこのインターフェースを実装したクラス（あるいはオブジェクト）を作成し、ライブラリに注入します。

```typescript
export interface GitFileStatus {
  path: string;
  index: string;      // Gitステータスコード (index)
  working_dir: string; // Gitステータスコード (working tree)
}

export interface GitLogEntry {
  hash: string;
  date: string;
  message: string;
  author_name: string;
  author_email: string;
  refs: string;
  parents: string[];
}

export interface GitClient {
  init(dir: string): Promise<void>;
  status(dir: string): Promise<GitFileStatus[]>;
  log(dir: string): Promise<GitLogEntry[]>;
  add(dir: string, files: string[]): Promise<void>;
  reset(dir: string, files: string[]): Promise<void>;
  commit(dir: string, message: string): Promise<void>;
  diff(dir: string, path: string, staged: boolean): Promise<string>;
  getRemotes(dir: string): Promise<string[]>;
  currentBranch(dir: string): Promise<string>;
  push(dir: string, remote: string, branch: string): Promise<void>;
}
```

### 4.2 GitProvider (コンテキスト・プロバイダー)
Git UI の状態管理をカプセル化するプロバイダーです。

```typescript
interface GitProviderProps {
  children: React.ReactNode;
  client: GitClient;
  currentDir: string | null;
  /**
   * 外部のファイル監視機能とリフレッシュ処理をバインドするためのコールバック。
   * ホスト側でファイル変更を検知した際、本ライブラリの status などを自動リフレッシュさせるために使用します。
   */
  subscribeFileChange?: (onChanged: () => void) => () => void;
}
```

### 4.3 GitPanel (メインパネル・コンポーネント)
サイドバー等に配置される Git 管理のメイン UI です。

```typescript
interface GitPanelProps {
  /**
   * ユーザーがファイルの差分表示（ダブルクリックまたはクリック）を選択した際のイベントハンドラ。
   * ホスト側でタブを開く、専用ビューアを起動する等の処理を行います。
   */
  onOpenFileDiff?: (path: string, staged: boolean) => void;

  /**
   * 言語リソース。指定しない場合はデフォルト（日本語）が適用されます。
   */
  locale?: GitUILocale;
}
```

### 4.4 DiffViewer (差分表示コンポーネント)
ファイルの変更差分を色分け表示する UI コンポーネントです。

```typescript
interface DiffViewerProps {
  path: string;
  staged: boolean;
  /**
   * 言語リソース
   */
  locale?: GitUILocale;
}
```

---

## 5. スタイルおよびテーマ設定の抽象化

ホストアプリケーションのデザインシステムとシームレスに統合できるよう、ライブラリの CSS は専用の **CSS カスタムプロパティ (CSS Variables)** に基づいてスタイリングを行います。

### 5.1 CSS カスタムプロパティ設計
ライブラリのルート要素（`.git-ui-root`）に以下の CSS 変数を定義し、ホスト側の変数をフォールバック付きで参照します。

```css
.git-ui-root {
  /* テーマ変数 (ホストから注入可能、未定義時はデフォルト値) */
  --git-ui-bg: var(--git-theme-bg, #1e1e1e);
  --git-ui-text-primary: var(--git-theme-text-primary, #f5f5f5);
  --git-ui-text-secondary: var(--git-theme-text-secondary, #aaaaaa);
  --git-ui-border: var(--git-theme-border, #333333);
  --git-ui-input-bg: var(--git-theme-input-bg, #252526);
  --git-ui-button-bg: var(--git-theme-button-bg, #3c3c3c);
  --git-ui-button-hover-bg: var(--git-theme-button-hover-bg, #4c4c4c);
  --git-ui-accent: var(--git-theme-accent, #007acc);

  /* Gitステータスカラー */
  --git-ui-color-added: var(--git-theme-color-added, #4caf50);
  --git-ui-color-modified: var(--git-theme-color-modified, #2196f3);
  --git-ui-color-deleted: var(--git-theme-color-deleted, #f44336);
  --git-ui-color-untracked: var(--git-theme-color-untracked, #ff9800);
}
```

---

## 6. 国際化 (i18n) 仕様

UI 上のテキストラベルやプレースホルダーはすべて `locale` オブジェクトを通じてカスタマイズ可能とします。

### 6.1 ロケールインターフェースの定義
```typescript
export interface GitUILocale {
  sections: {
    stagedChanges: string;
    changes: string;
    history: string;
  };
  actions: {
    commit: string;
    committing: string;
    push: string;
    pushing: string;
    initRepo: string;
    refresh: string;
    stageAll: string;
    unstageAll: string;
  };
  placeholders: {
    commitMessage: string;
    insertDateTime: string;
    noStagedFiles: string;
    noChanges: string;
    noRepository: string;
  };
  status: {
    untracked: string;
    added: string;
    deleted: string;
    modified: string;
  };
}
```

---

## 7. 提供パッケージ構成

ライブラリパッケージ（例: `@novelaid/git-ui`）からエクスポートされる API は以下の通りです。

```typescript
// Components
export { GitPanel } from './components/GitPanel';
export { GitGraph } from './components/GitGraph';
export { DiffViewer } from './components/DiffViewer';

// Context & Providers
export { GitContextProvider, useGit } from './contexts/GitContext';

// Interfaces & Types
export type {
  GitClient,
  GitFileStatus,
  GitLogEntry,
  GitUILocale,
} from './types';
```
