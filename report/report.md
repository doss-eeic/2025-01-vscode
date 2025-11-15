# レポート原稿

## はじめに
この記事は、2025年度冬学期に実施された東京大学工学部電子情報・電気電子工学科の学生実験「大規模ソフトウェアを手探る」の
成果レポートを兼ねています。

この実験では、Visual Studio Code のソースコードをいじることで
2つの新機能を追加しました。

ここでは、コードの clone から機能の完成まで、その過程を余すことなくお伝えしていきます。
同じようなことをやろうと考えている方の参考になれば幸いです！

## Visual Studio Code とは？
Visual Studio Code (以下 vscode) は、Microsoft が開発しているソースコードエディタです。

最大の特徴はその圧倒的な人気度で、
Stack Overflow で実施された 2025 Developer Survey では、
開発環境として 75.9 % の支持を集め、
次点に 2.6 倍以上の差をつけてトップの人気となりました。

また、vscode はソースコードが [github](https://github.com/microsoft/vscode) 上で公開されています。
今回はこのレポジトリを clone してきて、新機能を開発していくことになります。

※ Microsoft が配布しているソフトウェアとしての vscode は、公開されているソースコードにカスタマイズを加えたものになっているため、公開されているソースコードをビルドしたものと完全に同じではありません。

## 今回目指したこと
今回は大きく分けて次の2つの機能を実装することを目指しました。
- 機能1. Tab キーで名前の変更を連続して行う
- 機能2. 巨大ファイルをメモリ消費を抑えつつプレビューする

### 機能1 について
Windows のエクスプローラでは、ファイルの名前を変更中に Tab キーを押すと１つ下のファイルの名前の変更を行う状態へと移ることができます。
この機能は複数のファイル名を変更したい場合にとても便利なのですが、vscode ではこれができません。
ないものは作ろう、ということでこれを vscode にも実装するのが１つ目の目標になります。

![WindowsエクスプローラのTabキーの挙動](windows_explorer_tab.gif)

### 機能2 について
for alex

## コード全体の概要と構造
コード構成については、[こちらのドキュメント](https://github.com/microsoft/vscode/wiki/Source-Code-Organization)を参照しました。

### レイヤ分割
まず、コード全体はレイヤに分割されています。
- `base` レイヤ: 汎用的なユーティリティ関数やUIコンポーネントを提供します。
- `platform` レイヤ: ファイル読み書きやコマンド実行、ワークスペース管理などを司る Service を定義しています。
- `editor` レイヤ: 単純なコードエディタ部分の機能を提供します。
- `workbench` レイヤ: エディタ等をホストし、ファイルエクスプローラーやメニューバーなどの付加的な機能を追加します。Electronによりデスクトップアプリを実装します。また、ブラウザAPIを通じてweb版の実装も行います。
- `code` レイヤ: デスクトップアプリのエントリポイントで、全コンポーネントを統合します。
- `server` レイヤ: リモート開発用のサーバアプリのエントリポイントになります。

### 実行環境分割
次に、各レイヤーは実行環境ごとにコードがディレクトリに分割されています。
- `common`: JavaScript API のみで動くコード
- `browser` Web API が必要なコード
- `node` Node.JS API が必要なコード
- `electron-browser`: `browser` が提供する API と electron との通信用 API が必要なコード
- `electron-utility`: `node` が提供するAPI と、electron utility-process API が必要なコード
- `electron-main`: `node`, `electron-utility` が提供する API と、electron main-process API が必要なコード

### `contrib` ディレクトリ
`browser` レイヤと `workbench` レイヤには、`contrib` というディレクトリが存在しています。
ここは、必要最低限のコアコードの上に載せる機能単位の実装をまとめる場所になっています。
基本的には、ここのコードを弄ることになります。

### Dependency Injection
vscode のソースコードは、Dependency Injection (依存注入, DI) と呼ばれるデザインパターンで設計されています。
具体的には、`platform` が提供する Service を、各クラスのコンストラクタ引数として受け取る形で依存性を表現しています。

## 開発環境の整備
開発を開始するにあたっての環境構築では、[こちらのドキュメント](https://github.com/microsoft/vscode/wiki/How-to-Contribute)を参照しました。

必要なツールのインストールとレポジトリのクローンを終えたら、ビルドと実行を行っていきます。

まずビルドは以下のコマンドで行います。

**注意: pnpm など npm 以外のものを使うと依存関係が上手く解決できずビルドに失敗する可能性があります。**
```sh
npm install # 初回のみ
npm run watch
```
このコマンドはソースコードが更新されると自動で増分ビルドを行ってくれます。
開発中はずっと実行しっぱなしにしておきます。

先述のコマンドで、`[watch-client    ] [xx:xx:xx] Finished compilation with 0 errors after x ms` のように出力されたら、別のターミナルを開いて以下のコマンドで起動します。

**注意: 紛らわしいのですが、`[watch-extensions] [xx:xx:xx] Finished compilation extensions with 0 errors after x ms` と見間違えないように注意してください。このタイミングで起動しようとしても失敗します。**

```sh
./scripts/code.sh
```

起動した vscode OSS 内では、Ctrl + Shift + I (あるいはコマンドパレット) で Chrome Developer Tools を利用することができます。
UI まわりのデバッグでは、これを使わないとかなり厳しいので重要です。

## 機能1. Tab キーで名前の変更を連続して行う
### コード箇所の特定
始めに、名前変更機能に関連するコード箇所を特定します。
ファイルエクスプローラ部分の機能なので、`workbench` レイヤに絞ります。

まず、ファイル名の編集状態に入る処理のスタート地点を探しました。
編集状態に入る際のキーボードショートカット（F2キー）に注目し、`KeyCode.F2` というキーワードで検索を行いました。
その結果、`workbench/contrib/files/browser/fileActions.contribution.ts` に関連コードを発見しました。
そこから `workbench/contribu/files/browser/fileActions.ts` に定義された `renameHandler` 関数を特定しました。

次に、編集を確定して通常の状態に戻る処理のスタート地点を探しました。
ファイル名の編集中にも F2 キーを押すと処理が走る（ファイル名の選択部分が変わる）ことに注目し、再び `KeyCode.F2` の検索結果を見直しました。
その結果、`workbench/contrib/files/browser/views/explorerViewer.ts` に定義された `FilesRenderer` クラスを特定しました。

### 編集状態に入る際の処理の詳細を追う
1. `renameHandler` は、`explorerService.getContext` からエクスプローラ部分で現在フォーカス中のファイルの情報を取得します。そして、`explorerService.setEditable` 関数に選択中のファイル情報と、編集終了時に呼んでもらうコールバック関数 (`onFinish`) を渡します。
1. `explorerService.setEditable` 関数は、指定されたファイルを「編集状態」として、`onFinish` と共に内部に記憶しておきます。そのうえで、`ExplorerView.setEditable` 関数に編集したいファイルの情報を転送します。 ※「編集状態」にあるファイルは多くても1つのみです。
1. `ExplorerView.setEditable` 関数は、渡されたファイルの**親ディレクトリ**を指定して、エクスプローラのツリーコンポーネントについて、そのディレクトリ以下の部分の再レンダリングを走らせます。このタイミングで「どのファイルを編集したいのか」という情報は引数のバケツリレーからは失われます。
1. かなりのコールスタックを積み重ねて、ツリーの再レンダリング処理は `workbench/contrib/files/browser/views/explorerViewer.ts` の `FilesRenderer.renderElement` 関数に至ります。ここで `explorerSerivice.getEditableData` 関数により、「編集状態」にあるファイルの情報を問い合わせて取得します。そして、これに一致するファイルのツリーコンポーネントの場合のみ、`FilesRenderer.renderInputBox` 関数を呼び出します。この際に、先述の `onFinish` も渡します。

### 編集状態を終える際の処理の詳細を追う
1. 編集状態でエンターキーやエスケープキーを押すと、`FilesRenderer.renderInputBox` 内の `DOM.addStandardDisposableListener(inputBox.inputElement, DOM.EventType.KEY_DOWN, (e: IKeyboardEvent)` の箇所で定義されているリスナーがトリガされ、`done` 関数が呼ばれます。
1. `done` 関数では、入力ボックスの中身（新しいファイル名）などを引数に渡して `onFinish` を呼び出します。
1. `onFinish` は、ファイルの読み書きAPIを呼んでファイル名の変更を行った後、`explorerService.setEditable` に `null` を渡して「編集状態」をクリアします。

### コード変更
一連の処理の最後の、`explorerService.setEditable` に `null` を渡して「編集状態」をクリアする部分に注目しました。
名前の編集が完了したファイルの次のファイル情報を `null` の代わりに渡せば、ツリーの再レンダリングによって次のファイル用の入力ボックスをレンダリングさせ、次のファイルの編集状態にスムーズに遷移することができます。

「次のファイル」を取得するにあたっては、既存コードの「フォーカス中のファイル情報の取得」の機能を再利用するために、「フォーカスを１つ進める」という処理を、情報取得前に入れることで実装しました。具体的には、以下のようにして実装しました。
```ts
const viewsService = accessor.get(IViewsService);
const view = viewsService.getViewWithId(VIEW_ID);
const explorerView = view as ExplorerView;
explorerView.focuxNext();
const next_stats = explorerService.getContext(false); // 次ファイル情報
```
そして `onFinish` の引数に `have_next: boolean` を追加し、false の場合には既存コードと同様の処理を行い、true の場合には上記で取得した次ファイル情報を使って `explorerService.setEditable` を呼ぶように変更しました。

次に、`FilesRenderer.renderInputBox` 内の先述のリスナーに、Tabキーのリスナーを追加し`done` を呼ぶようにしました。
`done` でも `next: boolean` を引数に追加して内部の `onFinish` の呼び出しの際に `have_next` に転送するようにしておき、既存コードにおける `done` の呼び出しでは全て false、Tabキーから呼ぶ箇所だけ true にセットしました。

このとき、Tab キーのデフォルト動作である「フォーカスを次のコンポーネントに移動する」という動作を以下のコードによって無効化する必要があります。
```ts
e.preventDefault();
```
実は `FilesRenderer.renderInputBox` には、「この入力ボックスからフォーカスが外れた場合には編集状態をキャンセルして終了する」という処理を走らせるためのリスナーが存在しています。
このリスナーのおかげで、編集中にエディタ部分をクリックしたりすると、自動でファイル名の編集状態を終了してくれたりするのですが、これが上記の Tab キーのデフォルト動作と致命的なミスマッチとなってしまいます。
そのため、無効化を入れる必要がありました。

### 完成品
以上のコード変更によって、このように目標の機能を実装することができました。
![vscodeエクスプローラのTabキーの挙動](vscode_explorer_tab.gif)

## 機能2. 巨大ファイルをメモリ消費を抑えつつプレビューする
for alex

## おわりに
for alex

## 参考文献やサイトなど
- https://github.com/microsoft/vscode
- https://ja.wikipedia.org/wiki/Visual_Studio_Code
- https://survey.stackoverflow.co/2025/technology#1-dev-id-es
- https://github.com/microsoft/vscode/wiki/How-to-Contribute
- https://github.com/microsoft/vscode/wiki/Source-Code-Organization
