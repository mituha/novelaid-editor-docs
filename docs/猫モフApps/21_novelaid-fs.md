# 猫モフ Apps - 小説執筆アプリを創ろう - 11. novel-fs

![](./images/20250622_note_1280_670.png)

猫モフ Apps は、猫をモフモフしながら思いついたアイデアを、バイブコーディングでゆるっと創っていく企画です。  

前回はTauri版のプラグインの作成方法について確認しました。
今回は前回言及した、`novelaid-fs`部分について深堀りしていきます。

※この記事は途中経過を含んでおり、修正状況に合わせて更新していく予定です。

## IPCチャネル

### 元々のIPCチャネル

作成中の`novelaid-editor-clone`のファイル処理関連

| チャネル名 | 機能概要 | 引数 | 戻り値 |
| :--- | :--- | :--- | :--- |
| `get-app-path` | 特殊なディレクトリのパスを取得 | `name: string` (userData, home) | `string \| null` |
| `ensure-dir` | ディレクトリの存在確認と作成 | `directoryPath: string` | `boolean` |
| `read-json` | JSON ファイルを読み込む | `filePath: string` | `any` (存在しない場合は null) |
| `write-json` | JSON ファイルを書き込む | `filePath: string`, `data: any` | `boolean` |
| `list-files` | 指定パス内のファイル一覧を取得 | `directoryPath: string` | `{ name: string, isDirectory: boolean }[]` |
| `read-file` | テキストファイルを読み込む | `filePath: string` | `string` (UTF-8) |
| `write-file` | テキストファイルを書き込む | `filePath: string`, `content: string` | `boolean` |

オリジナルの`novelaid-editor`のファイル処理関連

| チャネル名 | 引数 | 戻り値 | 説明 |
| :--- | :--- | :--- | :--- |
| `fs:readDirectory` | `dirPath: string` | `Promise<Dirent[]>` | ディレクトリ内容の読み取り。 |
| `fs:getDirectoryType` | `dirPath: string` | `Promise<DocumentType>` | ディレクトリの優先ドキュメントタイプを取得。 |
| `fs:getDocumentType` | `filePath: string` | `Promise<DocumentType>` | ファイルのドキュメントタイプを取得。 |
| `fs:readFile` | `filePath: string` | `Promise<string>` | テキストファイルの読み取り。 |
| `fs:writeFile` | `filePath: string, content: string` | `Promise<void>` | テキストファイルの保存。 |
| `fs:readDocument` | `filePath: string` | `Promise<any>` | メタデータを含むドキュメントの読み取り。 |
| `fs:saveDocument` | `filePath: string, data: any` | `Promise<void>` | メタデータを含むドキュメントの保存。 |
| `fs:createFile` | `filePath: string` | `Promise<boolean>` | 空ファイルの作成。 |
| `fs:createUntitledDocument` | `dirPath: string` | `Promise<string>` | 未命名ドキュメントの自動生成・保存。 |
| `fs:createDirectory` | `dirPath: string` | `Promise<boolean>` | ディレクトリの作成。 |
| `fs:rename` | `oldPath: string, newPath: string` | `Promise<boolean>` | ファイル・ディレクトリのリネーム。 |
| `fs:move` | `oldPath: string, newPath: string` | `Promise<boolean>` | 移動（競合時はダイアログ表示）。 |
| `fs:copy` | `srcPath: string, destPath: string` | `Promise<boolean>` | コピー（競合時はダイアログ表示）。 |
| `fs:delete` | `targetPath: string` | `Promise<boolean>` | 削除。 |

なお、Tauri版は元々、標準のプラグインのファイル関連機能を利用していたため、IPCチャネルは存在しませんでした。

AIによって都度作成されていたIPCチャネルを再検討しつつ調整していきます。  

## `novelaid-fs`

Tauri版ではプラグインとして`novelaid-fs`を作成しました。  
Electron版では処理が`main`および`renderer`に分かれています。
そこで、Electron版でも`novelaid-fs`フォルダーを作成して、処理をまとめたいと思います。
なお、このようなフォルダー構成が良いかと言われると微妙かもしれませんが、まずは試してみたいと思います。  

### IPCチャネル

新たに調整した、または、追加予定のIPCチャネルは以下の通りです。  
なお、prefixとして、`novelaid-fs:`を付与しています。
また、Tauri版においては、snake_caseで定義し、Electron版ではcamelCaseで定義するので、適宜読み替えて下さい。


| チャネル名 | 説明 |
| :--- | :--- |
| setProjectDirectory | プロジェクトディレクトリを設定する。 |
| getProjectDirectory | プロジェクトディレクトリを取得する。 |
| TODO | 以下は検討中 |
| `readDirectory` | ディレクトリ内容の読み取り。 |
| `getDirectoryType` | ディレクトリの優先ドキュメントタイプを取得。 |
| `getDocumentType` | ファイルのドキュメントタイプを取得。 |
| `readFile` | テキストファイルの読み取り。 |
| `writeFile` | テキストファイルの保存。 |
| `readDocument` | メタデータを含むドキュメントの読み取り。 |
| `saveDocument` | メタデータを含むドキュメントの保存。 |
| `createFile` | 空ファイルの作成。 |
| `createUntitledDocument` | 未命名ドキュメントの自動生成・保存。 |
| `createDirectory` | ディレクトリの作成。 |
| `rename` | ファイル・ディレクトリのリネーム。 |
| `move` | 移動（競合時はダイアログ表示）。 |
| `copy` | コピー（競合時はダイアログ表示）。 |
| `delete` | 削除。 |

### setProjectDirectory / getProjectDirectory

プロジェクトディレクトリを設定します。  

### getDocumentType / getDirectoryType

ファイルおよびディレクトリのドキュメントタイプを取得します。  
このメソッド自体は最終的には不要になるかもしれません。  








## MORE

## 名前付けの規則について

snake_case(スネークケース)は、アンダースコアで区切る命名規則です。
camelCase(キャメルケース)は、先頭の単語以外は先頭を大文字にする命名規則です。

スネークケースの`get_document_type`をキャメルケースにすると`getDocumentType`となります。








