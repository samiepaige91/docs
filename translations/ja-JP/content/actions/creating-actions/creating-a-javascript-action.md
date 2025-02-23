---
title: JavaScript アクションを作成する
intro: このガイドでは、アクションツールキットを使って JavaScript アクションをビルドする方法について学びます。
redirect_from:
  - /articles/creating-a-javascript-action
  - /github/automating-your-workflow-with-github-actions/creating-a-javascript-action
  - /actions/automating-your-workflow-with-github-actions/creating-a-javascript-action
  - /actions/building-actions/creating-a-javascript-action
versions:
  fpt: '*'
  ghes: '*'
  ghae: '*'
  ghec: '*'
type: tutorial
topics:
  - Action development
  - JavaScript
shortTitle: JavaScript action
---

{% data reusables.actions.enterprise-beta %}
{% data reusables.actions.enterprise-github-hosted-runners %}

## はじめに

このガイドでは、パッケージ化されたJavaScriptのアクションを作成して使うために必要な、基本的コンポーネントについて学びます。 アクションのパッケージ化に必要なコンポーネントのガイドに焦点を当てるため、アクションのコードの機能は最小限に留めます。 このアクションは、ログに "Hello World" を出力するものです。また、カスタム名を指定した場合は、"Hello [who-to-greet]" を出力します。

このガイドでは、開発の速度を高めるために{% data variables.product.prodname_actions %} ToolkitのNode.jsモジュールを使います。 詳しい情報については、[actions/toolkit](https://github.com/actions/toolkit) リポジトリ以下を参照してください。

このプロジェクトを完了すると、あなたの JavaScript コンテナのアクションをビルドして、ワークフローでテストする方法が理解できます

{% data reusables.actions.pure-javascript %}

{% data reusables.actions.context-injection-warning %}

## 必要な環境

Before you begin, you'll need to download Node.js and create a public {% data variables.product.prodname_dotcom %} repository.

1. Download and install Node.js {% ifversion fpt or ghes > 3.3 or ghae-issue-5504 or ghec %}16.x{% else %}12.x{% endif %}, which includes npm.

  {% ifversion fpt or ghes > 3.3 or ghae-issue-5504 or ghec %}https://nodejs.org/en/download/{% else %}https://nodejs.org/en/download/releases/{% endif %}

1. Create a new public repository on {% data variables.product.product_location %} and call it "hello-world-javascript-action". 詳しい情報については、「[新しいリポジトリの作成](/articles/creating-a-new-repository)」を参照してください。

1. リポジトリをお手元のコンピューターにクローンします。 詳しい情報については[リポジトリのクローン](/articles/cloning-a-repository)を参照してください。

1. ターミナルから、ディレクトリを新しいリポジトリに変更します。

  ```shell{:copy}
  cd hello-world-javascript-action
  ```

1. From your terminal, initialize the directory with npm to generate a `package.json` file.

  ```shell{:copy}
  npm init -y
  ```

## アクションのメタデータファイルの作成

Create a new file named `action.yml` in the `hello-world-javascript-action` directory with the following example code. 詳しい情報については、「[{% data variables.product.prodname_actions %} のメタデータ構文](/actions/creating-actions/metadata-syntax-for-github-actions)」を参照してください。

```yaml{:copy}
name: 'Hello World'
description: 'Greet someone and record the time'
inputs:
  who-to-greet:  # id of input
    description: 'Who to greet'
    required: true
    default: 'World'
outputs:
  time: # id of output
    description: 'The time we greeted you'
runs:
  using: {% ifversion fpt or ghes > 3.3 or ghae-issue-5504 or ghec %}'node16'{% else %}'node12'{% endif %}
  main: 'index.js'
```

このファイルは、`who-to-greet` 入力と `time` 出力を定義しています。 また、アクションのランナーに対して、この JavaScript アクションの実行を開始する方法を伝えています。

## アクションツールキットのパッケージの追加

アクションのツールキットは、Node.js パッケージのコレクションで、より一貫性を保ちつつ、JavaScript を素早く作成するためのものです。

ツールキットの [`@actions/core`](https://github.com/actions/toolkit/tree/main/packages/core) パッケージは、ワークフローコマンド、入力変数と出力変数、終了ステータス、およびデバッグメッセージへのインターフェースを提供します。

このツールキットは、認証された Octokit REST クライアントと GitHub Actions コンテキストへのアクセスを返す [`@actions/github`](https://github.com/actions/toolkit/tree/main/packages/github) パッケージも提供します。

ツールキットは、`core` や `github` パッケージ以外のものも提供しています。 詳しい情報については、[actions/toolkit](https://github.com/actions/toolkit) リポジトリ以下を参照してください。

ターミナルで、アクションツールキットの `core` および `github` パッケージをインストールします。

```shell{:copy}
npm install @actions/core
npm install @actions/github
```

これで、`node_modules` ディレクトリと先ほどインストールしたモジュール、`package-lock.json` ファイルとインストールしたモジュールの依存関係、およびインストールした各モジュールのバージョンが表示されるはずです。

## アクションのコードの記述

このアクションは、ツールキットを使って、アクションのメタデータファイルに必要な `who-to-greet` 入力変数を取得し、ログのデバッグメッセージに "Hello [who-to-greet]" を出力します。 次に、スクリプトは現在の時刻を取得し、それをジョブ内で後に実行するアクションが利用できる出力変数に設定します。

GitHub Actions は、webhook イベント、Git ref、ワークフロー、アクション、およびワークフローをトリガーした人に関するコンテキスト情報を提供します。 コンテキスト情報にアクセスするために、`github` パッケージを利用できます。 あなたの書くアクションが、webhook イベントペイロードをログに出力します。

以下のコードで、`index.js` と名付けた新しいファイルを追加してください。

{% raw %}
```javascript{:copy}
const core = require('@actions/core');
const github = require('@actions/github');

try {
  // `who-to-greet` input defined in action metadata file
  const nameToGreet = core.getInput('who-to-greet');
  console.log(`Hello ${nameToGreet}!`);
  const time = (new Date()).toTimeString();
  core.setOutput("time", time);
  // Get the JSON webhook payload for the event that triggered the workflow
  const payload = JSON.stringify(github.context.payload, undefined, 2)
  console.log(`The event payload: ${payload}`);
} catch (error) {
  core.setFailed(error.message);
}
```
{% endraw %}

上記の `index.js` の例でエラーがスローされた場合、`core.setFailed(error.message);` はアクションツールキット [`@actions/core`](https://github.com/actions/toolkit/tree/main/packages/core) パッケージを使用してメッセージをログに記録し、失敗の終了コードを設定します。 詳しい情報については「[アクションの終了コードの設定](/actions/creating-actions/setting-exit-codes-for-actions)」を参照してください。

## READMEの作成

アクションの使用方法を説明するために、README ファイルを作成できます。 README はアクションの公開を計画している時に非常に役立ちます。また、アクションの使い方をあなたやチームが覚えておく方法としても優れています。

`hello-world-javascript-action` ディレクトリの中に、以下の情報を指定した `README.md` ファイルを作成してください。

- アクションが実行する内容の詳細
- 必須の入力引数と出力引数
- オプションの入力引数と出力引数
- アクションが使用するシークレット
- アクションが使用する環境変数
- ワークフローでアクションを使う使用方法の例

```markdown
# Hello world javascript action

This action prints "Hello World" or "Hello" + the name of a person to greet to the log.

## Inputs

## `who-to-greet`

**Required** The name of the person to greet. デフォルトは `"World"`。

## Outputs

## `time`

The time we greeted you.

## 使用例

uses: actions/hello-world-javascript-action@v1.1
with:
  who-to-greet: 'Mona the Octocat'
```

## アクションの GitHub へのコミットとタグ、プッシュ

{% data variables.product.product_name %} が、動作時にワークフロー内で実行される各アクションをダウンロードし、コードの完全なパッケージとして実行すると、ランナーマシンを操作するための`run` などのワークフローコマンドが使えるようになります。 つまり、JavaScript コードを実行するために必要なあらゆる依存関係を含める必要があります。 アクションのリポジトリに、ツールキットの `core` および `github` パッケージをチェックインする必要があります。

ターミナルから、`action.yml`、`index.js`、`node_modules`、`package.json`、`package-lock.json`、および `README.md` ファイルをコミットします。 `node_modules` を一覧表示する `.gitignore` ファイルを追加した場合、`node_modules` ディレクトリをコミットするため、その行を削除する必要があります。

アクションのリリースにはバージョンタグを加えることもベストプラクティスです。 アクションのバージョン管理の詳細については、「[アクションについて](/actions/automating-your-workflow-with-github-actions/about-actions#using-release-management-for-actions)」を参照してください。

```shell{:copy}
git add action.yml index.js node_modules/* package.json package-lock.json README.md
git commit -m "My first action is ready"
git tag -a -m "My first action release" v1.1
git push --follow-tags
```

`node_modules` ディレクトリをチェックインすると、問題が発生する可能性があります。 別の方法として、[`@vercel/ncc`](https://github.com/vercel/ncc) というツールを使用して、コードとモジュールを配布に使用する 1 つのファイルにコンパイルできます。

1. ターミナルで次のコマンドを実行し、`vercel/ncc` をインストールします: `npm i -g @vercel/ncc`

1. 次のコマンドで、`index.js` ファイルをコンパイルします: `ncc build index.js --license licenses.txt`

  コードの書かれた、新しい `dist/index.js` ファイルと、コンパイルされたモジュールが表示されます。 また、使用している `node_modules` のすべてのライセンスを含む、`dist/licenses.txt` ファイルも表示されます。

1. 新しい `dist/index.js` ファイルを利用するため、次のコマンドで `action.yml` の `main` キーワードを変更します: `main: 'dist/index.js'`

1. すでに `node_modules` ディレクトリをチェックインしていた場合、次のコマンドで削除します: `rm -rf node_modules/*`

1. ターミナルから、`action.yml`、`dist/index.js`、および `node_modules` ファイルをコミットします。
```shell
git add action.yml dist/index.js node_modules/*
git commit -m "Use vercel/ncc"
git tag -a -m "My first action release" v1.1
git push --follow-tags
```

## ワークフローでアクションをテストする

これで、ワークフローでアクションをテストできるようになりました。 プライベートリポジトリにあるアクションは、同じリポジトリのワークフローでしか使用できません。 パブリックアクションは、どのリポジトリのワークフローでも使用できます。

{% data reusables.actions.enterprise-marketplace-actions %}

### パブリックアクションを使用する例

This example demonstrates how your new public action can be run from within an external repository.

Copy the following YAML into a new file at `.github/workflows/main.yml`, and update the `uses: octocat/hello-world-javascript-action@v1.1` line with your username and the name of the public repository you created above. `who-to-greet`の入力を自分の名前に置き換えることもできます。

{% raw %}
```yaml{:copy}
on: [push]

jobs:
  hello_world_job:
    runs-on: ubuntu-latest
    name: A job to say hello
    steps:
      - name: Hello world action step
        id: hello
        uses: octocat/hello-world-javascript-action@v1.1
        with:
          who-to-greet: 'Mona the Octocat'
      # Use the output from the `hello` step
      - name: Get the output time
        run: echo "The time was ${{ steps.hello.outputs.time }}"
```
{% endraw %}

When this workflow is triggered, the runner will download the `hello-world-javascript-action` action from your public repository and then execute it.

### プライベートアクションを使用する例

ワークフローコードを、あなたのアクションのリポジトリの `.github/workflows/main.yml` ファイルにコピーします。 `who-to-greet`の入力を自分の名前に置き換えることもできます。

{% raw %}
**.github/workflows/main.yml**
```yaml{:copy}
on: [push]

jobs:
  hello_world_job:
    runs-on: ubuntu-latest
    name: A job to say hello
    steps:
      # このリポジトリのプライベートアクションを使用するには
      # リポジトリをチェックアウトする
      - name: Checkout
        uses: actions/checkout@v2
      - name: Hello world action step
        uses: ./ # Uses an action in the root directory
        id: hello
        with:
          who-to-greet: 'Mona the Octocat'
      # 「hello」ステップの出力を使用する
      - name: Get the output time
        run: echo "The time was ${{ steps.hello.outputs.time }}"
```
{% endraw %}

リポジトリから [**Actions**] タブをクリックして、最新のワークフロー実行を選択します。 {% ifversion fpt or ghes > 3.0 or ghae or ghec %}Under **Jobs** or in the visualization graph, click **A job to say hello**. {% endif %}"Hello Mona the Octocat"、または `who-to-greet` 入力に指定した名前とタイムスタンプがログに出力されます。

{% ifversion fpt or ghes > 3.0 or ghae or ghec %}
![ワークフローでアクションを使用しているスクリーンショット](/assets/images/help/repository/javascript-action-workflow-run-updated-2.png)
{% elsif ghes %}
![ワークフローでアクションを使用しているスクリーンショット](/assets/images/help/repository/javascript-action-workflow-run-updated.png)
{% else %}
![ワークフローでアクションを使用しているスクリーンショット](/assets/images/help/repository/javascript-action-workflow-run.png)
{% endif %}
