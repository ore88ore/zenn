---
title: "Step Functions と EventBridge Scheduler を用いて ECS タスクを定期実行する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "awscdk", "ecs", "fargate", "stepfunctions"]
published: false
---

## はじめに

Step Functions と EventBridge Scheduler を使って ECS タスクを定期実行する方法を紹介します。具体的には、次のような要件を満たすような構成にします。

- Step Functions を使って ECS タスク（バッチ処理）を実行する
- １度の実行で複数の ECS タスクを直列に実行する
- ECS タスク実行時にコマンドライン引数を設定（上書き）する
- EventBridge Schedulerを使って、これらのタスクを定期的に実行する
- CDK でリソースを構築する

作成するアーキテクチャは以下のようなイメージとなります。

![](/images/cdk-scheduled-sfn-run-task/architecture.png)

## 実行環境

- Go: `1.20`
- Node.js: `18.16.1`
- TypeScript: `5.1.6`
- CDK: `2.86.0`

## CDK によるStep Functions の定義

まずはスケジュール実行するターゲットの Step Functions と Step Functions から呼び出す ECR タスクを作成します。
ECR タスクで実行するコンテナは、実行時の引数と環境変数を出力するだけのプログラム(Go)を ECR リポジトリにプッシュしたコンテナを利用します。
今回は一つずつ ECS タスクを実行しやすいように、ECS タスク毎に Step Functions を作成することにしました。複数の ECS タスクを実行する際は１つの ECS タスクを実行する Step Functions を Step Functions から呼ぶように実装しています。

![](/images/cdk-scheduled-sfn-run-task/step-functions-architecture.png)


Step Functions(ECS タスク) を直列に２つ実行するように Step Functions を作成します。

```typescript:cdk-stack.ts
export class CdkStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // 省略...

    // ECS
    const cluster = new ecs.Cluster(this, "cluster", { vpc });
    // タスク定義
    const taskDefinition = new ecs.FargateTaskDefinition(
      this,
      "taskDefinition",
      {
        cpu: 512,
        memoryLimitMiB: 1024,
      },
    );
    // コンテナ定義
    const goBatchContainer = taskDefinition.addContainer("goBatchContainer", {
      image: ecs.ContainerImage.fromEcrRepository(
        ecr.Repository.fromRepositoryName(
          this,
          "ecrRepository",
          "[リポジトリ名]",
        ),
      ),
      environment: {
        ENV: "Original env value.",
      },
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: "batch-log-",
      }),
    });

    // １つ目の ECS RunTask を作成
    const containerOverrides1: tasks.ContainerOverride[] = [
      {
        containerDefinition: goBatchContainer,
        command: sfn.JsonPath.listAt("$.commands"),
        environment: [{ name: "ENV", value: sfn.JsonPath.stringAt("$.env") }],
      },
    ];
    const ecsRunTask1 = new tasks.EcsRunTask(this, "ecsRunTask1", {
        integrationPattern: sfn.IntegrationPattern.RUN_JOB,
      cluster,
      taskDefinition,
      containerOverrides: containerOverrides1,
      launchTarget: new tasks.EcsFargateLaunchTarget(),
      securityGroups: [runTaskSecurityGroup],
    });
    
    // ２つ目の ECS RunTask を作成
    const containerOverrides2: tasks.ContainerOverride[] = [
      {
        containerDefinition: goBatchContainer,
        command: sfn.JsonPath.listAt("$.commands"),
        environment: [{ name: "ENV", value: "From step functions2." }],
      },
    ];
    const ecsRunTask2 = new tasks.EcsRunTask(this, "ecsRunTask2", {
      integrationPattern: sfn.IntegrationPattern.RUN_JOB,
      cluster,
      taskDefinition,
      containerOverrides: containerOverrides2,
      launchTarget: new tasks.EcsFargateLaunchTarget(),
      securityGroups: [runTaskSecurityGroup],
    });

    // １つ目の ECS RunTask を実行する Step Functions を作成
    const execEcsRunStateMachine1 = new sfn.StateMachine(
      this,
      "execEcsRunStateMachine1",
      {
        stateMachineName: "execEcsRunStateMachine1",
        definitionBody: sfn.DefinitionBody.fromChainable(ecsRunTask1),
      },
    );
    // ２つ目の ECS RunTask を実行する Step Functions を作成
    const execEcsRunStateMachine2 = new sfn.StateMachine(
      this,
      "execEcsRunStateMachine2",
      {
        stateMachineName: "execEcsRunStateMachine2",
        definitionBody: sfn.DefinitionBody.fromChainable(ecsRunTask2),
      },
    );

    // ２つの ECS RunTask(Step Functions) を直列に実行するように定義
    const stepFunctionsRunTask = new tasks.StepFunctionsStartExecution(
      this,
      "stepFunctionsRunTask1",
      {
        stateMachine: execEcsRunStateMachine1,
        integrationPattern: sfn.IntegrationPattern.RUN_JOB,
        input: TaskInput.fromObject({
          commands: ["from1", "object1"],
          env: "Start Execution input1",
        }),
      },
    ).next(
      new tasks.StepFunctionsStartExecution(this, "stepFunctionsRunTask2", {
        stateMachine: execEcsRunStateMachine2,
        integrationPattern: sfn.IntegrationPattern.RUN_JOB,
        input: TaskInput.fromObject({
          commands: ["from2", "object2"],
          env: "Start Execution input2",
        }),
      }),
    );

    // 実行する Step Functions を作成
    const execStepFunctionsStateMachine = new sfn.StateMachine(
      this,
      "execStepFunctionsStateMachine",
      {
        stateMachineName: "execStepFunctionsStateMachine",
        definitionBody: sfn.DefinitionBody.fromChainable(stepFunctionsRunTask),
      },
    );

    // 省略...
  }
}
```

ECS タスクを実行する際に、引数を指定して実行するという要件を満たすために Step Functions の実行時の入力パラメータで引数を上書きするように実装しています。

```typescript
    // １つ目の ECS RunTask を作成
    const containerOverrides1: tasks.ContainerOverride[] = [
      {
        containerDefinition: goBatchContainer,
        command: sfn.JsonPath.listAt("$.commands"), // ← ここ
        environment: [{ name: "ENV", value: sfn.JsonPath.stringAt("$.env") }],
      },
    ];
```

実行時はこのようなパラメータを指定します。

```json
{
  "commands": [
    "arg-1",
    "arg-2",
  ]
}
```

今回は引数を複数指定したかったので複数指定できるように、`.listAt('$.Field')` を利用して、文字列の配列で受け取るようにしました。文字列で受け取る場合は、`.stringAt('$.Field')` あたりが利用できそうです。詳細は公式ドキュメントを確認してください。

https://docs.aws.amazon.com/ja_jp/step-functions/latest/dg/connect-ecs.html

https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_stepfunctions-readme.html#passing-parameters-to-tasks

## CDK による EventBridge Scheduler の定義

複数の ECS タスクを実行する Step Functions を作成することができたので、この処理を定期的に実行できるように、EventBridge Scheduler を作成します。

```typescript
    // 省略...
    const eventSchedulerRole = new iam.Role(this, "eventSchedulerRole", {
      assumedBy: new iam.ServicePrincipal("scheduler.amazonaws.com"),
    });
    eventSchedulerRole.addToPolicy(
      new iam.PolicyStatement({
        resources: ["*"],
        actions: ["states:StartExecution"],
      }),
    );

    // 執筆時点で L2 コンストラクトが無かったので、L1 コンストラクトで実装
    new scheduler.CfnSchedule(this, `execStepFunctionsSchedule`, {
      scheduleExpression: "cron(0 10 * * ? *)", // 毎日 10:00 に実行
      scheduleExpressionTimezone: "Asia/Tokyo", // タイムゾーンを指定しているので cron は JST で指定
      flexibleTimeWindow: { mode: "OFF" },
      state: "DISABLED",
      target: {
        arn: execStepFunctionsStateMachine.stateMachineArn,
        roleArn: eventSchedulerRole.roleArn,
        // 必要に応じてリトライ設定(retryPolicy)
        // 必要に応じてデットレターキュー設定(deadLetterConfig)
      },
      groupName: "default",
    });
    // 省略...
```

https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_scheduler-readme.html

## AWS CDKによるリソースのデプロイ

これまで作成したリソースをデプロイして、作成されたリソースを確認します。

### Step Functions

ECS タスクに入力パラメータを渡して実行する Step Functions と、その Step Functions を呼び出す Step Functions が作成されていることを確認できました。

![](/images/cdk-scheduled-sfn-run-task/result_step_functions.png)

### Event Bridge Scheduler

スケジューラーも意図したとおりに作成されていました。`スケジュール` タブを確認すると cron でスケジュールも設定されていることを確認できます。

![](/images/cdk-scheduled-sfn-run-task/result_eventbridge.png)

スケジューラーに設定したスケジュールに応じて実行されているかは、Step Functions の実行履歴から確認することができます。

![](/images/cdk-scheduled-sfn-run-task/result_schduled_execute.png)

## さいごに

ECR タスクを定期的に実行するだけだと、EventBridge Scheduler から直接 ECR タスクを呼び出して実行することもできます。ですが、今回のようにバッチ１を実行してからバッチ２を実行する。といったように実行の順序などを制御したい場合や、バッチの起動前後に処理をはさみたい、バッチの処理結果によって異なるバッチを実行したい場合など、ECR タスクの実行を Step Functions を挟んで実行するのが良さそうかなと思いました。
