---
title: "CDK で Permissions Boundary を設定するよ"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "awscdk", "iam"]
published: false
publication_name: "yumemi_inc"
---

:::message
※本記事は、[AWS CDK Advent Calendar 2023](https://qiita.com/advent-calendar/2023/aws-cdk) の 13 日目の記事となります
:::

## はじめに

ガバナンスを強化し、セキュリティのガードレールとして機能させるために、[Permissions Boundary](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies_boundaries.html) が設定された AWS アカウントを利用することがありませんか？ Permissions Boundary は、IAM Role や IAM User の操作に制限を加えることで、権限昇格を防ぐ役割を果たすことができます。このような環境で CDK を使用してリソースを作成する際、多くのリソースで IAM Role の作成が必要になりますが、適切な権限設定がないと権限エラーが発生し、作成ができないことがあります。
今回の記事では、このような制約のある環境下で CDK を利用する方法について紹介します。

![](/images/cdk-with-permissions-boundary/architecture.png)

### 前提条件

組織のルールとして AWS の利用者に IAM Role（または IAM User）を提供する際は必ず `role-permissions-boundary` ポリシーが Permissions Boundary に設定している。この設定により `role-permissions-boundary` が Permissions Boundary に設定されていない IAM Role の作成を拒否しています。
実際の運用だとこのポリシーで細かい権限の制御を設定する必要がありますが、今回は IAM Role 関連のポリシーのみ設定しています。

```json:role-permissions-boundary
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AdministratorAccess",
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        },
        {
            "Sid": "DenyRole",
            "Action": [
                "iam:UpdateAssumeRolePolicy",
                "iam:DetachRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:PutRolePermissionsBoundary",
                "iam:CreateRole",
                "iam:AttachRolePolicy",
                "iam:UpdateRole",
                "iam:PutRolePolicy"
            ],
            "Resource": "*",
            "Effect": "Deny",
            "Condition": {
                "StringNotEquals": {
                    "iam:PermissionsBoundary": "arn:aws:iam::xxxxx:policy/role-permissions-boundary"
                }
            }
        }
    ]
}
```

## Bootstrapping

新しい AWS アカウントや新たなリージョンで CDK を利用する場合は、はじめに [Bootstrap](https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/bootstrapping.html) を行う必要があります。
特に制約がない環境だと `cdk bootstrap` コマンドを実行するだけで Bootstrap できるのですが、今回のような環境の場合、以下のようなエラーとなります。

```
$ cdk bootstrap
...
21:44:48 | CREATE_FAILED        | AWS::IAM::Role
CloudFormationExecutionRole Resource handler returned message: "User: arn:aws:sts::xxxxx:assumed-role/ore88ore-permission-boundary-test/techlead is not authorized to perform: iam:CreateRole on resource: arn:aws:iam::xxxxx:role/cdk-yyyyy-cfn-exec-role-xxxxx-us-west-2 with an explicit deny in a permissions boundary (Service: Iam, Status Code: 403, Request ID: zzz-zzz)"(RequestToken: zzz-zzz, HandlerErrorCode: AccessDenied)
...
```

Bootstrap では CDK でデプロイする時などに利用するロール作成します。そのロールに Permissions Boundary(`role-permissions-boundary` ポリシー) が設定されていないので、作成できないというエラーとなります。Bootstrap で作成されるロールについては、以下のようなイメージとなります。

![](/images/cdk-with-permissions-boundary/bootstrap-role.png)
（[CDK Security and Safety Dev Guide](https://github.com/aws/aws-cdk/wiki/Security-And-Safety-Dev-Guide) より）

### Bootstrap 時に作成するロールに Permissons Boundary を設定する

今回の環境だと IAM Role を作成するには Permissions Boundary(`role-permissions-boundary` ポリシー) が設定されている必要があります。ですので、Bootstrap 時に作成するロールに Permissons Boundary を設定する必要があります。

#### テンプレートをカスタマイズ

`cdk bootstrap` コマンドで、Bootstrap 時に利用する CFn テンプレートを出力することができます。

```
$ cdk bootstrap --show-template > custom-template.yaml
```

出力した CFn テンプレートの `AWS::IAM::Role` リソースに [Permissons Boundary](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html#cfn-iam-role-permissionsboundary) が設定されるように、テンプレートを修正します。以下のテンプレートは、一部のみですが、IAM Role は複数あるので、すべてに設定する必要があります。一部のリソースは、`PermissionsBoundary` プロパティを CFn のインプットから設定できるようなっているリソースがあるので、定義が重複しないように注意が必要です。

```
...
  FilePublishingRole:
    Type: AWS::IAM::Role
    Properties:
      PermissionsBoundary: arn:aws:iam::xxxxx:policy/cdk-delegated-permissions-boundary  # ここを追加
      AssumeRolePolicyDocument:
        Statement:
          - Action: sts:AssumeRole
            Effect: Allow
            Principal:
              AWS:
                Ref: AWS::AccountId
          - Fn::If:
              - HasTrustedAccounts
              - Action: sts:AssumeRole
                Effect: Allow
                Principal:
                  AWS:
                    Ref: TrustedAccounts
              - Ref: AWS::NoValue
      RoleName:
        Fn::Sub: cdk-${Qualifier}-file-publishing-role-${AWS::AccountId}-${AWS::Region}
      Tags:
        - Key: aws-cdk:bootstrap-role
          Value: file-publishing
...
```

Permissions Boundary を設定した CFn テンプレートを利用して Bootstrap を実行します。

```
$ cdk bootstrap --template custom-template.yaml
...
✅  Environment aws://xxxxx/us-west-2 bootstrapped.
```

この手順で任意の CFn テンプレートを利用して Bootstrap を行うことができます。

#### bootstrap コマンドの custom-permissions-boundary オプションは使えないの？

`custom-permissions-boundary` で、Bootstrap 時に設定する Permissions Boundary を設定することができます。ただし、指定した Permissions Boundary が設定されるのは、`CloudFormationExecutionRole` のみとなります。
今回のサンプルでは、すべての IAM Role に Permissions Boundary を設定する必要がありましたので、合致しませんでした。。。（残念！）要件に合致するようなら、以下のようなコマンドで簡単に Permissions Boundary を設定することができます。

```
$ cdk bootstrap --custom-permissions-boundary role-permissions-boundary
```

## デプロイ

Bootstrap が完了したので、続いてデプロイします。CDK で IAM Role が作成できるかを確認するために、以下のように IAM Role を作成してみます。

```typescript:cdk-permissions-boundary-stack.ts
export class CdkPermissionsBoundaryStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const iamRole = new iam.Role(this, 'IamRole', {
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
    });

    iamRole.addToPolicy(new iam.PolicyStatement({
      actions: ['s3:*'],
      resources: ['*'],
    }));
  }
}
```

リソースを定義したのでデプロイしてみます。

```
$ cdk deploy
...
7:19:16 | CREATE_FAILED        | AWS::IAM::Role
IamRoleA750FF82 Resource handler returned message: "User: arn:aws:sts::xxxxx:assumed-role/cdk-hnb659fds-cfn-exec-role-xxxxx-us-west-2/AWSCloudFormation is not authorized to perform: iam:CreateRole on resource: arn:aws:iam::xxxxx:role/CdkPermissonsBoundaryStack-IamRoleA750FF82-GIwAOEzyt0Et with an explicit deny in a permissions boundary (Service: Iam, Status Code: 403, Request ID: zzz-zzz)" (RequestToken: zzz-zzz, HandlerErrorCode: AccessDenied)
...
```

Bootstrap の時と同様に IAM Role の作成で AccessDenied のエラーとなりましたね。意図した動きになっています。

### CDK で作成する IAM Role に Permissions Boundary を設定する

CDK で作成する IAM Role に Permissions Boundary を設定するには、`cdk.json` で、作成するすべての IAM Role に Permissons Boundary を設定することができます。
この設定をすることで、全ての IAM Role に `name` で指定したポリシーの Permissions Boundary が設定されます。

```json:cdk.json
...
"context": {
    ...
    "@aws-cdk/core:permissionsBoundary": {
      "name": "role-permissions-boundary"
    }
}
...
```

他にも一部のリソースに設定したい、この環境のみ設定したいなど柔軟に設定をすることができます。詳細は[こちらのドキュメント](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam-readme.html#custom-permissions-boundary)が参考になります。

実際にデプロイされている IAM Role を確認すると、Permissions Boundary が設定されていることを確認できました。

![](/images/cdk-with-permissions-boundary/deploy-result.png)

## さいごに

比較的かんたんに CDK で Permissions Boundary を設定することができました。Permissions Boudnary を設定できるように機能が提供されているのが大きいですね。
開発者のロールやデプロイする環境などに応じて、最大の権限はどこまでかを明確に指定することができるので、セキュリティのガードレールとして機能させるのが良さそうですね。
大きなチームや組織で CDK を利用する際は、導入を検討するのがいいかと思いました。

## 参考

https://aws.amazon.com/jp/blogs/mt/how-to-deploy-cdk-v2-to-an-account-that-requires-boundary-policies/