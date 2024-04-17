---
title: "タグエディタを利用してリソースを検索する"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws"]
published: true
publication_name: "yumemi_inc"
---

## はじめに

時間が経つと、自分で作成したリソースを忘れがちになりますよね。リソース作成時にタグ付けをしたり、IaC でリソースを管理している場合は比較的簡単に追跡できますが、試しに作成したリソースは特に忘れやすいものです。特に複数人が利用する環境では、自分のリソースを識別しやすいように ID や名前などの識別子をリソース名として設定することがあります。今回は、そうしたリソースをタグエディタを使って検索し、不要なリソースを削除してみました。

## タグエディタを使ったリソース検索

### マネジメントコンソールでタグエディタを開く

`Resource Groups & Tag Editor` というサービス名なので、サービス名で検索するか、サービス一覧から [管理とガバナンス] の中から `Resource Groups & Tag Editor` を開きます。
ナビゲーションの中に [タグ付け] - [タグエディタ] があるので、ここからタグエディタを開くことができます。

![](/images/search-resource-by-tag-editor/open-tag-editor.png)

### 検索条件を指定する

リソースの検索条件を指定します。検索条件としては、`リージョン`、`リソースタイプ`、`タグ` を指定することができます。今回はすべてのリージョンですべてのリソースを検索対象としてみました。検索対象が多いためか、検索には少し時間がかかりました。
なお、`リソースタイプ` は、すべてのリソースがサポートされているわけではなく、以下に記載されているサポートされているリソースタイプが対象となります。

https://docs.aws.amazon.com/ja_jp/ARG/latest/userguide/supported-resources.html

![](/images/search-resource-by-tag-editor/search-resource-condition.png)

### リソース名でフィルタリングする

検索条件だけだと、リソース名で絞り込むことができないので、上記の検索条件で取得した結果をリソース名でフィルタリングします。
今回の例だと識別子(`Identifier`)に特定の文字列が含まれる(`Conteins`)リソースをフィルタリングしたいので、`Identifier : [文字列]` のようにフィルターテキストを指定します。
なお識別子以外のプロパティも対象にしたい場合は、`[文字列]` と指定することで、[部分一致検索となる](https://docs.aws.amazon.com/ja_jp/tag-editor/latest/userguide/find-resources-to-tag.html)ようです。

![](/images/search-resource-by-tag-editor/search-result.png)

### 検索結果を確認

今回は消し忘れたリソースを削除する目的で検索を行いました。検索結果では、各リソース名がリンクとして表示されており、リンク先で編集することができます。事前に自分で作成した CloudFormation のスタックを削除しましたが、予想以上に多くのリソースが残っていると感じました。これは、CloudFormation で `DeletionPolicy` が `Retain` に設定されているリソースが削除されていないためかもしれません。

![](/images/search-resource-by-tag-editor/search-result-link.png)

また、検索結果から対象のリソースを選択し、一括でタグを付与することも可能です。

![](/images/search-resource-by-tag-editor/search-result-tag.png)

## さいごに

リソースを横断的に検索できないかなと思い、今回タグエディタを使った検索を試してみました。料金がかかるリソースはほとんどなかったのですが、不要なリソースが邪魔になることもあるため、定期的に整理整頓しておきたいですね。また、後から自分が作成したリソースであるとわかるように、タグ付けを忘れずに行いましょう！！（自戒の念を込めて）

https://docs.aws.amazon.com/ja_jp/tag-editor/latest/userguide/tagging.html#tag-best-practices
