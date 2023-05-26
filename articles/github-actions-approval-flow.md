---
title: "GitHub Actions の environments を使ってデプロイ時に承認プロセスを導入する"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "cicd"]
published: true
---

継続的デリバリーを実現するために、GitHub Actions を使用しているプロジェクトが多くありますよね。特に、開発環境、検証環境、本番環境など、目的に応じて複数の環境を持つプロジェクトもあります。そのような場合には、本番環境に関しては特定の人（例えば○○さんや○○チーム）の承認がなければデプロイができないという要件が生じることもあると思います。そこで、GitHub Actions の [environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) 機能を利用して、このような承認プロセスを導入する方法を紹介します。

## environments 機能について

environments は、GitHub Actions 内で特定の環境を定義し、ワークフローを制御するための機能です。environments でデプロイの保護ルール、環境毎のシークレットや変数を設定することができます。承認プロセスを導入する場合は、デプロイの保護ルールにレビュー担当者を設定することで導入することができます。詳細は、[公式ドキュメント](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)をご確認ください。

> 環境、環境のシークレット、環境保護ルールは、すべての製品のパブリック リポジトリで利用できます。 プライベートまたは内部リポジトリ内の環境、環境のシークレット、デプロイ ブランチにアクセスするには、GitHub Pro、GitHub Team または GitHub Enterprise を使う必要があります。 プライベートまたは内部リポジトリ内の他の環境保護ルールにアクセスする場合は、GitHub Enterprise を使う必要があります。 詳しくは、「GitHub の製品」をご覧ください。
> 引用元: https://docs.github.com/ja/actions/deployment/targeting-different-environments/using-environments-for-deployment

公式ドキュメントにも書かれている通り、environments 機能を利用するには、

- public リポジトリ
- private, internal リポジトリの場合は、GitHub Pro、GitHub Team または GitHub Enterprise が必要

という制約がありますので、確認してみてください。

## 承認プロセスの設定手順

今回作成する承認プロセスは、`production` の場合、ワークフロー実行時に承認を必須となるように設定します。

### environment を作成する

リポジトリから [Settings] → [Environments] と進み、[New environment] から、environment を作成します。
今回は、`productions` と `staging` を作成します。

![](/images/github-actions-approval-flow/env-name.png)
*environment 名を入力*

environment の設定で、`Required reviewers` のチェックを ON にして、レビュアーとなるユーザーまたはチームを選択します。レビュアーは最大で６ユーザーまたはチームを選択することができ、承認は１ユーザーのみとなります。
※ `staging` は `Required reviewers` を OFF にしておきます。

![](/images/github-actions-approval-flow/protection-rules.png)
*environment の設定*

以下のように２つの environment を作成しました。

![](/images/github-actions-approval-flow/create-environments.png)
*作成した environment*

### environment を利用する

ワークフロー中の各ジョブは、１つの環境を使用することができます。（今回の例だと `production` または `staging`）ワークフロー内の `jobs.<job_id>.environment` キーに environment 名を指定することで利用することができます。

environment を `staging` にすると `Required reviwers` が OFF になっているので、通常のワークフローの実行と同様にトリガーされると実行されました。一方で以下のように `production` に設定した場合は `Required reviwers` を ON に設定してあるので、どうのような動作になるかを確認してみます。

```yml
name: Deployment

on:
  workflow_dispatch:
jobs:
  deployment:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: deploy
        run: echo 'deploy...'
```

作成したワークフローを実行すると、レビューを必須にしているので、レビューで承認するまで実行されません。承認前のステータスは `Waiting` となります。

![](/images/github-actions-approval-flow/review-waiting.png)
*レビュー待ち*

[Review deployments] からレビューを実施し、[Approve and deploy] をクリックすることで、デプロイのワークフローが進みました。ステータスは `Queued` → `Success` となり、ワークフローの実行が成功しました。

![](/images/github-actions-approval-flow/exec-success.png)
*レビューOKの場合*

一方でレビューでデプロイを拒否すると、ステータスが `Failure` となり、ワークフローが実行されませんでした。意図した動きとなっているようでしたー 👏

![](/images/github-actions-approval-flow/exec-failure.png)
*レビューNGの場合*

レビューを放置したらどうでしょうか？（基本的にはないですよね？）
[公式ドキュメント](https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments#about-required-reviews-in-workflows)に記載されていますが、30日以内に承認されないジョブは自動的に失敗（ステータスが `Failure`）するようです。

## さいごに
比較的簡単に承認プロセスを導入することができて、使える機能だなぁと感じました。本番環境にデプロイする際は、複数人で実行するとか、承認を記録するとか working agreement などで定めているチームなんかもあるかと思います。そういった用途に使えそうな機能だと思います。既存のワークフローにも簡単に組み込めるのもいいですね。用途に合致するのであれば、ぜひ使ってみてください。

## 参考

https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment