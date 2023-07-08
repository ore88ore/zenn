---
title: "AWS CDKでS3にAssetsを配置する"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "awscdk"]
published: true
---

CDK でアプリケーションやコンテナなどで利用する設定ファイルをアクセス可能な場所に配置したいと思って調べてみたところ、 [Assets](https://docs.aws.amazon.com/cdk/v2/guide/assets.html) が利用できそうでしたので、Amazon S3 assets を利用して、S3 バケットに Assets （今回の例だとファイル）を配置してみます。

## 実行環境

- Node.js: `18.16.1`
- TypeScript: `5.1.6`
- CDK: `2.86.0`

## Assets 配置する

今回は Assets として `sample.txt` を配置します。実行環境のディレクトリは以下のようなイメージとなります。

```
project-root/
├── lib/
│   ├── cdk-s3-assets-stack.ts 
│   └── sample.txt              // このローカルファイルを Assets として S3 に配置する
├── package.json
├── tsconfig.json
├── cdk.json
└── README.md
```

[公式のサンプル](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3_assets-readme.html) を参考に、Assets の配置と配置先の情報を出力してみます。

```typescript
import * as cdk from "aws-cdk-lib";
import { Construct } from "constructs";
import { aws_s3_assets as assets } from "aws-cdk-lib";
import * as path from "path";

export class CdkS3AssetsStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const asset = new assets.Asset(this, "SampleAsset", {
      path: path.join(__dirname, "sample.txt"),
    });

    new cdk.CfnOutput(this, "S3BucketName", { value: asset.s3BucketName });
    new cdk.CfnOutput(this, "S3ObjectKey", { value: asset.s3ObjectKey });
    new cdk.CfnOutput(this, "S3HttpURL", { value: asset.httpUrl });
    new cdk.CfnOutput(this, "S3ObjectURL", { value: asset.s3ObjectUrl });
  }
}
```

## Assets の確認

デプロイ後に Assets がどのように配置されているかを確認してみます。

### 配置先

出力した配置先情報を確認すると以下のようになっていました。

```
Outputs:
CdkS3AssetsStack.S3BucketName = cdk-xxxxx-assets-99999999-ap-northeast-1
CdkS3AssetsStack.S3HttpURL = https://s3.ap-northeast-1.amazonaws.com/cdk-xxxxx-assets-99999999-ap-northeast-1/7454228aab84e4fb5ef947c94daa6dc864ceaf6fb8250100418cfec152ae7cd0.txt
CdkS3AssetsStack.S3ObjectKey = 7454228aab84e4fb5ef947c94daa6dc864ceaf6fb8250100418cfec152ae7cd0.txt
CdkS3AssetsStack.S3ObjectURL = s3://cdk-xxxxx-assets-99999999-ap-northeast-1/7454228aab84e4fb5ef947c94daa6dc864ceaf6fb8250100418cfec152ae7cd0.txt
```

`sample.txt` というローカルファイルがリネームされて配置されているようでした。バケット名なども、自動で作成されるので、特に考慮する必要はなさそうでした。
なおバケットはリージョン毎に別のバケットが作成されるようでした。

![](/images/cdk-s3-assets/s3-assets.png)

### ファイルの内容

アップロードしたファイルをダウンロードして内容を確認してみました。今回はシンプルなテキストファイル１ファイルでしたが、ファイル名はリネームされていましたが、中身は同様となっていました。

## さいごに

ECS のコンテナに設定ファイルを配置したいと思っていて、できれば Docker ビルドしないで配置する方法を探していたところ、Assets 使えばできそうかなぁと、実際に試してみました。
ほんの数行でローカルファイルやディレクトリを S3 に配置できるので、とても簡単でした。Assets に配置したオブジェクトに利用（参照）する際は、アクセス権限が必要になるので、ここらへんの考慮は漏れないように注意が必要ですね。