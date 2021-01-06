---
title: "Host xxx is not allowed to connect to this MySQL server の対応"
emoji: "🚨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mysql"]
published: true
---

Docker で立てた MySQL に外部から接続しようとした際に、エラーが発生して詰まったので備忘録として書きます。

# 事象

Docker で立てた MySQL サーバーの MySQL に、ホスト側で起動したアプリケーションからアクセスしようとした際に、以下エラーが発生しました。

```
Mysql2::Error: Host '172.22.0.1' is not allowed to connect to this MySQL server
```

ポート番号やユーザー名、パスワードに間違いはなく、コンテナに入り MySQL へアクセスすると正常に起動しています。なぜ🤔


# 原因

指定したユーザーに、外部からのアクセス権がなかったのことが原因でした。
実際にコンテナ上の MySQL にて以下を実行すると指定ユーザー（sample_user）の host が `localhost` のみになっています。
これだと MySQL サーバーのコンテナ内からしかアクセスができません。

```
mysql> select user, host from mysql.user;
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
| sample_user      | localhost |
+------------------+-----------+
```

# 解決策
外部アクセスを許可するユーザーを作成し、権限を設定します。
コンテナの MySQL に入り、以下 SQL を実行します。

```sql
CREATE USER 'sample_user' IDENTIFIED BY '';
GRANT ALL PRIVILEGES ON *.* TO 'sample_user'@'%' WITH GRANT OPTION;
```

ユーザーの状態を確認すると sample_user が新たに追加され、host が`%`（ワイルドカード）になっています。

```
mysql> select user, host from mysql.user;
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
| sample_user      | localhost |
| sample_user      | %         |
+------------------+-----------+
```

これで sample_user に対してホスト側のアプリケーションからのアクセスが可能となります。

:::message
上の SQL だとセキュリティ的にはあまりよくないかもです。本番で使う場合には host 名を指定の IP にしたり DB 操作権限を制限したり適宜修正してください。
:::

# その他
`docker-compose.yml` または `Dockerfile`にて、`MYSQL_ROOT_HOST`の環境変数で root ユーザーの host を指定できるようです。
以下のように設定しておくと初回起動時に host が`%`の root ユーザーが作成されます。
この状態で root ユーザーにアクセスすれば、前述の対応は不要になります。

```yml
environment:
  MYSQL_ROOT_HOST: '%'
```

https://dev.mysql.com/doc/refman/8.0/en/docker-mysql-more-topics.html#docker_var_mysql-root-host

# 参考
- [mysqlのユーザー権限を管理する - Qiita](https://qiita.com/harada4atsushi/items/a36d26c798967e1f2cd9)
- [Host 'xxx.xx.xxx.xxx' is not allowed to connect to this MySQL server - Stack Overflow](https://stackoverflow.com/questions/1559955/host-xxx-xx-xxx-xxx-is-not-allowed-to-connect-to-this-mysql-server)