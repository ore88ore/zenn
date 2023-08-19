---
title: "S3 イベント通知を使って ECS タスクを実行する"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "awscdk", "ecs", "fargate", "stepfunctions", "eventbridge", "s3"]
published: false
---

## はじめに
S3 バケットに特定のファイルがアップロードされた際に、ECSタスク（バッチ処理）を自動で実行する方法について紹介します。具体的には、次のような要件を満たす構成にします。

- S3 のイベント通知を使用して ECS タスクを実行する
- イベント通知のトリガーは特定のフォルダ配下にファイルが作成された際に実行する
- イベントの送信先は EventBridge を利用する
- ECS タスクは Step Functions から呼び出す
- ECS タスク実行時の引数で作成されたオブジェクトのキーを受け取る
- CDKでリソースを構築する
  
作成するアーキテクチャは以下のようなイメージとなります。

![](/images/cdk-ecs-run-task-from-s3/architecture.png)

## 実行環境

- Go: `1.20`
- Node.js: `18.16.1`
- TypeScript: `5.1.6`
- CDK: `2.86.0`

## CDK で Step Functions と ECS タスクを定義

まずはファイルがアップロードされた際に実行する Step Functions と Step Functions から呼び出す ECS タスクを作成します。
ECS タスクで実行するコンテナは、実行時の引数と環境変数を出力するだけのプログラム(Go)を ECR リポジトリにプッシュしたコンテナを利用します。

```typescript:cdk-stack.ts
export class CdkStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    // 省略...

    // ECS
    const cluster = new ecs.Cluster(this, "cluster", { vpc });
    const taskDefinition = new ecs.FargateTaskDefinition(
      this,
      "taskDefinition",
      {
        cpu: 512,
        memoryLimitMiB: 1024,
      },
    );
    const goBatchContainer = taskDefinition.addContainer("goBatchContainer", {
      image: ecs.ContainerImage.fromEcrRepository(
        ecr.Repository.fromRepositoryName(this, "ecrRepository", ecrRepository),
      ),
      environment: {
        ENV: "Original env value.",
      },
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: "batch-log-",
      }),
    });

    const containerOverrides: tasks.ContainerOverride[] = [
      {
        containerDefinition: goBatchContainer,
        command: sfn.JsonPath.array(
          sfn.JsonPath.stringAt("$.detail.object.key"),
        ) as any,
      },
    ];

    const ecsRunTask = new tasks.EcsRunTask(this, "ecsRunTask", {
      integrationPattern: sfn.IntegrationPattern.RUN_JOB,
      cluster,
      taskDefinition,
      containerOverrides: containerOverrides,
      launchTarget: new tasks.EcsFargateLaunchTarget(),
      securityGroups: [runTaskSecurityGroup],
    });

    const execEcsRunStateMachine = new sfn.StateMachine(
      this,
      "execEcsRunStateMachine",
      {
        stateMachineName: "execEcsRunStateMachine",
        definitionBody: sfn.DefinitionBody.fromChainable(ecsRunTask),
      },
    );

    // 省略...
  }
}
```

ECS タスク実行時に S3 にアップロードされたオブジェクトのキーを実行時の引数に連携したいので、S3 のイベントメッセージの構造をもとに、オブジェクトのキーを `command` に設定するようにしています。

https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/ev-events.html

ここの実装でうまく行かなかった部分としては以下です。

```typescript
    const containerOverrides: tasks.ContainerOverride[] = [
      {
        containerDefinition: goBatchContainer,
        command: sfn.JsonPath.array(
          sfn.JsonPath.stringAt("$.detail.object.key"),
        ) as any,
      },
    ];
```

Step Funcitions の定義としては、`"Command.$": "States.Array($.detail.object.key)"` このように出力したかったのですが、`ContainerOverride.command` の型は `string[]` なのに対して、組み込み関数を利用するための `sfn.JsonPath.array` は `string` です。
ここの型のギャップを埋めることができず、`any` で回避しました。良い解決案がないものでしょうか。

https://docs.aws.amazon.com/ja_jp/step-functions/latest/dg/amazon-states-language-intrinsic-functions.html#asl-intrsc-func-arrays

Step Functions の定義を確認すると意図した設定になっていることは確認できました。

```
    "Overrides": {
        "ContainerOverrides": [
          {
            "Name": "goBatchContainer",
            "Command.$": "States.Array($.detail.object.key)"  ← ココ
          }
        ]
    },
```

aws_stepfunctions_tasks の Evaluate Expression を利用して、`resultPath` を `$.command` に詰め直すというタスクを作成すれば、やろうとしていることができそうでした。
このためだけに State が増えたり、リソースが増えたりすることを考えると、利用の有無を検討する必要があるかなぁと思いました。

https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_stepfunctions_tasks-readme.html#evaluate-expression

## CDK で S3 イベント通知を定義

S3 バケットに対して、特定のフォルダにファイルがアップロードされた際に、ECS タスクがトリガーされるように定義します。

```typescript:cdk-stack.ts
    const bucket = new s3.Bucket(this, "eventBucket", {
      bucketName: `${this.account}-${bucketName}`,
      eventBridgeEnabled: true,
    });

    new events.Rule(this, "S3EventRule", {
      eventPattern: {
        source: ["aws.s3"],
        account: [this.account],
        region: [this.region],
        detailType: events.Match.equalsIgnoreCase("object created"),
        detail: {
          bucket: {
            name: [bucket.bucketName],
          },
          object: {
            key: [{ prefix: "target/" }],
          },
        },
      },
      targets: [new events_targets.SfnStateMachine(execEcsRunStateMachine)],
    });
```

S3 バケットは、EventBridge の通知が送信されるように、`eventBridgeEnabled` オプションを `true` にしています。
また、該当のイベントに合致するようにイベントパターンを設定します。イベントパターンは、イベントの種類によってイベントデータの構造が決まっていますので、補足したいイベントに合致するように設定します。

https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html

今回は、特定のフォルダ(実装例だと`target`フォルダ)にファイルが作成された場合にイベントをトリガーしたいので、オブジェクトのキーにプレフィックスを指定して設定しています。プレフィックス以外にも様々なパターンでフィルタリングすることができるので、補足したいイベントに合わせて設定することができます。

https://docs.aws.amazon.com/ja_jp/eventbridge/latest/userguide/eb-event-patterns-content-based-filtering.html

イベントパターンを定義する際に考慮すべきプラクティスが公式サイトにありますので、設定する際は見ておくと良いと思います。

https://docs.aws.amazon.com/ja_jp/eventbridge/latest/userguide/eb-patterns-best-practices.html

## 動作確認

これまで定義したリソースをデプロイして、S3 バケットへのファイルアップロードとそれに応じた ECS タスクの実行を確認します。

### バケットに target/test.csv をアップロード

意図したリソースが作成されていることを確認できましたので、実際にファイルをアップロードしてみます。
`test.csv` というファイルを `target` フォルダにアップロードしてみます。

![](/images/cdk-ecs-run-task-from-s3/result-s3-upload-success.png)

配置したフォルダがプレフィックスに合致しているので、イベント通知されて処理が実行されていることを確認できました。

![](/images/cdk-ecs-run-task-from-s3/result-sfn.png)

続いて実行された ECS タスクの実行時の引数を確認してみます。

![](/images/cdk-ecs-run-task-from-s3/result-ecs-task.png)

実行時の引数にオブジェクトのキー（今回の例だと `target/test.csv`）が指定されていることを確認できました。意図した動きになってそうですね。

### バケットに hoge/test.csv をアップロード

意図したフォルダにファイルがアップロードされた場合、イベント通知されることを確認できたので、イベント通知対象外のパスにファイルが作成された際の動きも確認してみます。

![](/images/cdk-ecs-run-task-from-s3/result-s3-upload-fail.png)

実行履歴を確認して、処理が実行されていないことを確認できました。動作としては、イベント通知は行われる(Event Bridge)が、フィルタリングの条件（イベントパターン）に合致しないので、ターゲット（Step Functions + ECS タスク）が**実行されない**という動きになってそうですね。
意図した動きになってそうでした！

## さいごに
S3 イベント通知を使用して、特定のフォルダにファイルがアップロードされた時に ECS タスクを自動で実行することができました。配置されたファイルを読み込んでバッチ処理をする際に利用できそうな構成かと思います。処理の内容に応じて Lambda を選択するのもいいかなぁと思います。
CDK を使用することで、このようなシステムを簡単に構築できるので、是非試してみてください。