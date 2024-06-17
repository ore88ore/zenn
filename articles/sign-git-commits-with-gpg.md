---
title: "Mac で GPG を使って Git のコミットを署名する設定"
emoji: "💨"
type: "tech"
topics: []
published: true
---

一度設定してしまうとそれ以降何度も設定する事がないので、設定するたびに忘れてしまうので、設定する内容をメモしておきます。

## 実行環境

```sh
% sw_vers                                              
ProductName:		macOS
ProductVersion:		14.5
BuildVersion:		23F79

% brew --version
Homebrew 4.3.5
```

## インストール

```sh
% brew install gnupg
...
% brew install pinentry-mac
```

## キーを生成

https://docs.github.com/ja/authentication/managing-commit-signature-verification/generating-a-new-gpg-key

```sh
% gpg --full-generate-key
```

キーの生成は、基本的にはデフォルトで進めて、ユーザー名、メールアドレス、パスワードなどは適切に設定する。

```sh
% gpg --list-secret-keys --keyid-format=long
/Users/hubot/.gnupg/secring.gpg
------------------------------------
sec   4096R/3AA5C34371567BD2 2016-03-10 [expires: 2017-03-10]
uid                          Hubot <hubot@example.com>
ssb   4096R/4BB6D45482678BE3 2016-03-10
```

上記の出力の場合、GPG Key は `3AA5C34371567BD2`。

```sh
% gpg --armor --export 3AA5C34371567BD2
-----BEGIN PGP PUBLIC KEY BLOCK-----
...
-----END PGP PUBLIC KEY BLOCK-----
```

## キーを GitHub に設定

https://docs.github.com/ja/authentication/managing-commit-signature-verification/adding-a-gpg-key-to-your-github-account

生成したキーをコピーして、[こちら](https://github.com/settings/keys)から設定する。

## Git でコミット時に署名されるように設定

https://docs.github.com/ja/authentication/managing-commit-signature-verification/telling-git-about-your-signing-key

```sh
% git config --global user.signingkey 3AA5C34371567BD2
% git config --global commit.gpgsign true
% echo "pinentry-program $(which pinentry-mac)" >> ~/.gnupg/gpg-agent.conf
% gpgconf --kill gpg-agent
```

done 🙌