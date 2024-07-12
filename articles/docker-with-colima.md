---
title: "Colima で Docker と Docker Compose を使ってみた"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker", "dockercompose", "colima"]
published: true
---

[Colima](https://github.com/abiosoft/colima) は、Dockerコンテナを管理するためのコマンドラインインターフェース（CLI）です。
Colima をセットアップして、`docker` コマンドや `docker-compose` コマンドを実行してみようと思います。

## 実行環境

- macOS: `Ventura 13.3.1`
- チップ: `Apple M1 Pro`
- Homebrew: `4.0.18`

## インストール

まずは必要なパッケージをインストールしていきます。今回は、[Homebrew](https://brew.sh/index_ja) を利用して、インストールしてきます。

### Docker と Docker Compose をインストールする

[Docker Desktop](https://www.docker.com/products/docker-desktop/) などを利用していて、Docker や Docker Compose を使っていた環境だとすでにインストールされている可能性があるので、スキップしてください。
以下のコマンドを実行して、インストールします。

```
$ brew install docker docker-compose
```

今回インストールしたバージョンは以下の通りとなりました。

```
$ docker --version
$ Docker version 23.0.6, build ef23cbc431
$ docker-compose --version
$ Docker Compose version 2.17.3
```

※ 公式のバイナリからインストールしたい場合は、以下の手順でインストールすることができます。

https://qiita.com/ore88ore/items/d3fe5da8ddf49e60cee6

### Colima をインストールする

次のコマンドを実行してColimaをインストールできます。インストールの詳細については[公式ドキュメント](https://github.com/abiosoft/colima#installation)を確認ください。

```
$ brew install colima
```

今回インストールしたバージョンは以下の通りとなりました。

```
$ colima --version
colima version 0.5.4
```

## Colimaを起動する

Colima のインストールが完了したので、起動してみます。

```
$ colima start
```

特にオプションをつけずに起動した際は、デフォルト設定で起動されます。
主に利用しそうなオプションには以下のようなものがあり、必要に応じて設定して起動することができます。

| Flag | Description | Default |
|---|---|---|
| -c, --cpu | CPUの数 | 2 |
| -d, --disk | GiB単位のディスク容量 | 60 |
| -e, --edit | 起動前に構成ファイルを編集します。 | false |
| --env | VMの環境変数 | [] |
| -h, --help | ヘルプ | N/A |
| -m, --memory | GiB単位のメモリ | 2 |
| -V, --mount | マウントするディレクトリ。書き込み可能の場合は接尾辞「:w」を付けます。 | [] |
| -r, --runtime | コンテナランタイム（containerd, docker） | "docker" |
| -s, --ssh-agent | VMにSSHエージェントを転送します。 | false |

なお、以下のコマンドでオプションを確認することができます。

```
$ colima start --help
```

:::message
すでに Docker がインストールしてた場合にうまく起動できない場合は、Docker のコンテキストを確認し、`colima` が選択されているか確認してください。
```
$ docker context ls
NAME       DESCRIPTION                               DOCKER ENDPOINT                                       ERROR
colima *   colima                                    unix:///Users/xxx/.colima/default/docker.sock

# colima が選択されていない場合以下のコマンドで切り替える
$ docker context use colima
```
:::

## docker コマンドを実行してみる

Colima の起動ができたら、`docker` コマンドを実行できるようになるので、実行してみます。

```
docker run -d -p 8080:80 nginx
```

このコマンドは、[nginx のイメージ](https://hub.docker.com/_/nginx)を起動して、localhost:8080 をコンテナの 80 ポートにマッピングするコマンドです。
nginx のイメージがローカルにない場合は、イメージを pull してから起動されます。

![](/images/docker-with-colima/docker-confirm.png)

`docker` コマンドでイメージやコンテナを確認してみます。

```
$ docker image ls
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
nginx        latest    6405d9b26faf   12 days ago   135MB

$ docker container ls
CONTAINER ID   IMAGE     COMMAND                   CREATED         STATUS         PORTS                                   NAMES
41697e0c736f   nginx     "/docker-entrypoint.…"   8 minutes ago   Up 8 minutes   0.0.0.0:8080->80/tcp, :::8080->80/tcp   adoring_gauss
```

特に問題なく `docker` コマンドを実行することができました。

## docker compose コマンドを実行してみる

`docker` が実行できたので、続いて `docker compose` コマンドを実行してみます。
まずは、設定ファイルを以下の通り作成し、カレンドディレクトリに保存します。
[postgres コンテナ](https://hub.docker.com/_/postgres)とDB接続確認のために [Adminerコンテナ](https://hub.docker.com/_/adminer/)を起動します。

```yml:docker-compose.yml
version: '3'

services:
  db:
    image: postgres
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: example
  adminer:
    image: adminer
    restart: always
    ports:
      - 8080:8080
volumes:
  db_data:
```

起動中のコンテナやポートが重複していないことを確認して、以下のコマンドを実行して起動します。

```
docker-compose up -d
```

起動完了後、ブラウザにて確認してみます。

![](/images/docker-with-colima/docker-compose-confirm2.png)

問題なく起動できていそうですね。
コマンドで起動しているコンテナやボリュームを確認してみます。

```
$ docker container ls 
CONTAINER ID   IMAGE      COMMAND                   CREATED         STATUS         PORTS                                       NAMES
b3b6e3e8b010   postgres   "docker-entrypoint.s…"   5 seconds ago   Up 4 seconds   5432/tcp                                    temp-db-1
653ee34374b8   adminer    "entrypoint.sh php -…"   5 seconds ago   Up 4 seconds   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   temp-adminer-1

$ docker volume ls
DRIVER    VOLUME NAME
local     temp_db_data
```

想定していた動きになっているようでした🙌

## さいごに

Colima を使って、Docker, Docker Compose コマンドを実際に試してみました。これまで多く使っていた [Docker Desktop](https://www.docker.com/products/docker-desktop/)とシンプルな構成だと遜色なく利用できると感じました。
セットアップも簡単にできますので、GUI はなくてもいいかな、Docker Desktop から乗り換えたい！など考えている人の選択肢の一つにいかがでしょうか。

## 参考
https://github.com/abiosoft/colima
https://docs.docker.jp/engine/reference/commandline/index.html
https://docs.docker.jp/compose/reference/docker-compose.html
