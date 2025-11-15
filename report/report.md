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
