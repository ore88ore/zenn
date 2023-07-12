---
title: "FireLens for Amazon ECS を利用してコンテナのログをルーティングしてみる"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "awscdk", "fargate", "fluentbit"]
published: false
---

Fargate で実行しているコンテナのログを、FireLens を利用して CloudWatch Logs と S3(Kinesis Data Firehose を経由させて) にログをルーティングしてみました。ログのルーティングは AWS が提供してくれている [Fluent Bit イメージ](https://gallery.ecr.aws/aws-observability/aws-for-fluent-bit)を利用し、サイドカーパターンで配置します。これらのリソースの実装を CDK を利用して実装していきます。

![](/images/cdk-ecs-firelens/architecture.png)

## 実行環境

- Fluent Bit: `1.9.10`
- Node.js: `18.16.1`
- TypeScript: `5.1.6`
- CDK: `2.87.0`

## Fluent Bit の設定ファイルを作成
ログのルーティングをカスタマイズするためには、設定ファイルをイメージに取り込む必要があります。この設定ファイルには、どのようなログ情報をどの場所に送信するかなど、ルーティングに関する具体的な情報が含まれます。

https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/yaml/configuration-file

今回は CloudWatch Logs と Kinesis Data Firehose から S3 へ出力したいので以下のような設定にしました。

```text:extra.conf
[OUTPUT]
    Name   cloudwatch
    Match  *
    region ${AWS_REGION}
    log_group_name /aws/ecs/${ECS_CLUSTER}
    log_stream_prefix ecs-fluentbit-
    auto_create_group true

[OUTPUT]
    Name   firehose
    Match  *
    region ${AWS_REGION}
    delivery_stream log-delivery-stream
```

Fluent Bit の公式ドキュメントに、CloudWatch Logs や Kinesis Data Firehose への出力する場合のパラメータについて記載されているので、詳細はこちらをご確認ください。

https://docs.fluentbit.io/manual/pipeline/outputs/cloudwatch

https://docs.fluentbit.io/manual/pipeline/outputs/firehose

### Fluent Bit の設定ファイルを読み込む方法について

FireLens でカスタム設定ファイルを読み込ませる方法として、`config-file-type` に `s3` または `file` を設定することができます。ですが、Fargate の場合は、`file` のみのサポートとなるようです。（執筆時点）

> AWS Fargate でホストされるタスクは、file 設定ファイルタイプのみをサポートします。
> 引用元: https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/firelens-taskdef.html

今回は Fargate を利用するので、イメージに設定ファイルを追加したコンテナイメージを作成する必要がありそうです。少し試したいだけだったので、良い方法がないかと探してみたところ init プロセスを活用すると S3 から Fluent Bit の設定ファイルを読み込むことができるというブログを見つけました！！！
この init プロセスを利用するには、`init` というキーワードを含んだイメージタグを利用すれば良さそうです。（感謝🙏）

https://kakakakakku.hatenablog.com/entry/2023/05/29/094701

読み込ませ方の詳細はイメージの README にも記載されていました。

https://github.com/aws/aws-for-fluent-bit/tree/mainline/use_cases/init-process-for-fluent-bit

https://github.com/aws-samples/amazon-ecs-firelens-examples/tree/mainline/examples/fluent-bit/multi-config-support

読み込ませる設定ファイルは S3 に配置する必要があるので、S3 にアセットとして配置することにしました。

https://zenn.dev/ore88ore/articles/cdk-s3-assets

## CDK の実装

必要なファイルが準備できたので、必要なリソースの作成を CDK を利用して実装していきます。
なお紹介させてもらうソースは一部ですので、ソース一式をこちらで確認することができます。

https://github.com/ore88ore/cdk-ecs-firelens-sample/tree/main

### S3 にアセットを作成

設定ファイルを S3 に配置するためにアセットを作成します。また、配置したファイルを読み込むために[タスク IAM ロール](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/task-iam-roles.html)に読み取り権限を付与します。（タスク実行 IAM ロールではありません）

```typescript
    // アセットを作成
    const asset = new assets.Asset(this, "asset", {
      path: path.join(__dirname, "extra.conf"),
    });

    // アセットに配置したファイルを読み込むための権限が必要
    taskRole.addToPolicy(
      new iam.PolicyStatement({
        actions: [
          "logs:CreateLogStream",
          "logs:CreateLogGroup",
          "logs:DescribeLogStreams",
          "logs:PutLogEvents",
          "s3:GetObject",
          "s3:GetBucketLocation",
          "firehose:PutRecordBatch",
        ],
        resources: ["*"],
        effect: Effect.ALLOW,
      })
    );
```

### Kinesis Data Firehose を作成

ここで作成した Kinesis Data Firehose にログをルーティングするように設定ファイルで定義しているので、設定ファイルで指定した `delivery_stream` とストリームの名前を合わせる必要があります。

```typescript
    // Firehose から配信される S3 バケット
    const logBucket = new s3.Bucket(this, "logBucket", {
      removalPolicy: RemovalPolicy.DESTROY,
      autoDeleteObjects: true,
    });
    new firehose.DeliveryStream(this, "logDeliveryStream", {
      deliveryStreamName: "log-delivery-stream",        // ここの名前を設定ファイルと合わせる
      destinations: [new destinations.S3Bucket(logBucket)],
    });
```

### FireLens を作成
FireLens をタスク定義に追加します。これにより、ログをルーティングするためのコンテナがタスクに追加されます。
設定ファイル指定方法については、[こちら](https://github.com/aws/aws-for-fluent-bit/tree/mainline/use_cases/init-process-for-fluent-bit)に詳細が記載されています。複数の設定ファイルを指定することもできます。

```typescript
    taskDefinition.addFirelensLogRouter("firelensLogRouter", {
      firelensConfig: {
        type: FirelensLogRouterType.FLUENTBIT,      // Fluent Bit を指定
      },
      environment: {
        aws_fluent_bit_init_s3_1: `arn:aws:s3:::${asset.s3BucketName}/${asset.s3ObjectKey}`,    // S3 に配置した設定ファイルの ARN を指定
      },
      image: ecs.ContainerImage.fromRegistry(
        "public.ecr.aws/aws-observability/aws-for-fluent-bit:init-latest"   // "init" がついたタグを利用
      ),
      logging: ecs.LogDrivers.awsLogs({     // Fluent Bit が出力するログは awsLogs を指定して CloudWatch Logs へ出力
        streamPrefix: "log-router",
      }),
    });
```


### アプリケーションコンテナを作成
今回はなんらかの Web アプリケーションで、アクセスしてログが出力されれば良いので、ECR のパブリックイメージに登録されている [nginx](https://gallery.ecr.aws/nginx/nginx) を利用させてもらいました。

```typescript
    taskDefinition.defaultContainer = taskDefinition.addContainer(  // defaultContainer に設定
      "nginxContainer",
      {
        image: ecs.ContainerImage.fromRegistry(
          "public.ecr.aws/nginx/nginx:latest"
        ),
        logging: ecs.LogDrivers.firelens({
          options: {},
        }),
        portMappings: [{ containerPort: 80 }],
      }
    );
```

:::message
[公式ドキュメントにも書かれています](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs-readme.html#task-definitions)が、タスクに複数のコンテナを定義する際は、特に指定がなければ最初に追加したコンテナが `デフォルトコンテナ` となります。

もしデフォルトコンテナの指定をせずに最初に FireLens を追加していた場合は、FireLens のコンテナがデフォルトコンテナとなり、FireLens のコンテナにポートマッピングが存在しないというエラーでデプロイできませんでした。
１タスクに複数のコンテナを起動する場合は、明示的に[デフォルトコンテナを指定する](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs.FargateTaskDefinition.html#defaultcontainer)のがよいのかもしれませんね。
:::

## デプロイ
一通り CDK の実装が完了したら、デプロイします。

```
cdk deploy
```

デプロイが完了したら、ALB が発行した DNS でアクセスすることができます。

![](/images/cdk-ecs-firelens/web-nginx.png)

## ログを確認
デプロイして何度かアクセスすることで、ログが出力されているはずなので、ログの出力が正しく行われていることを確認するため、CloudWatch Logs と S3 を確認します。

### CloudWatch Logsを確認
CloudWatch Logs でログが正しく出力されていることを確認します。出力する場所は、Fluent Bit の設定ファイルで、`/aws/ecs/${ECS_CLUSTER}` と指定しているので、該当のロググループを確認してみます。

![](/images/cdk-ecs-firelens/result-cloudwatch.png)

JSON で出力されて、`log` というキーの値が出力したログ出力メッセージとなっているようです。その他、コンテナ情報や ECS タスクの情報のようなメタ情報も合わせて出力されていました。出力内容については、必要に応じて変更しておくと良いかと思います。

### S3を確認
S3 にも同様にログが正しく保存されていることを確認します。

![](/images/cdk-ecs-firelens/result-s3.png)

日時(UTC)でフォルダがきられて、ログファイルが配置されていました。[ログファイルの出力の頻度](https://docs.aws.amazon.com/ja_jp/firehose/latest/dev/basic-deliver.html#frequency)については、Kinesis Data Firehose のバッファサイズとバッファ間隔に応じて出力されます。対象のファイルをダウンロードして中身を確認すると、CloudWatch Logs で出力されていた内容と同じ内容のログがファイルに出力されているようでした。

```json
{"container_id":"dummy","container_name":"nginxContainer","ecs_cluster":"yus-sakai-ecs-firelens-yyyyy","ecs_task_arn":"arn:aws:ecs:us-west-2:xxxxx:task/yus-sakai-ecs-firelens-yyyyy/xxx","ecs_task_definition":"taskdef:12","log":"2023/07/11 23:05:26 [error] 29#29: *8681 open() \"/usr/share/nginx/html/favicon.ico\" failed (2: No such file or directory), client: x.x.x.x, server: localhost, request: \"GET /favicon.ico HTTP/1.1\", host: \"dummy.amazonaws.com\", referrer: \"http://dummy.amazonaws.com/\"","source":"stderr"}
{"container_id":"dummy","container_name":"nginxContainer","ecs_cluster":"yus-sakai-ecs-firelens-yyyyy","ecs_task_arn":"arn:aws:ecs:us-west-2:xxxxx:task/yus-sakai-ecs-firelens-yyyyy/xxx","ecs_task_definition":"taskdef:12","log":"x.x.x.x - - [11/Jul/2023:23:05:26 +0000] \"GET /favicon.ico HTTP/1.1\" 404 555 \"http://dummy.amazonaws.com/\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 Safari/537.36\" \"x.x.x.x\"","source":"stdout"}
```

## さいごに
これまでは CloudWatch Logs だけにログ出力していましたが、複数箇所へのルーティングやログのフィルタリングを実装したい際に FireLens を利用して出力するのも良さそうでした。設定によってはいらないログを破棄してコスト削減をしたり、ログを外部のサービスに連携して一元管理したりといろいろな用途に利用できそうです。
Fluent Bit は AWS が公式イメージとして提供してくれているので、こういったログルーティングの実装を検討している場合は使ってみてはいかがでしょうか。


## 参考
https://mazyu36.hatenablog.com/entry/2023/01/04/201631#3AWS-CDK%E3%81%AE%E5%AE%9F%E8%A3%85
https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/firelens-taskdef.html
https://www.sbcr.jp/product/4815607654/