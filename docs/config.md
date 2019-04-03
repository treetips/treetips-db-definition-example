# DB設定

## 📝サーバ側

AmazonRDSやAuroraであったとしても、基本的に以下の項目を設定します。

### lower_case_table_namesを指定する

デフォルト値が無効になっているので、以下のように有効にします。
```ini
lower_case_table_names = 1
```
これを指定しないと大文字・小文字が区別されるようになってしまい、DDL・DMLが非常に不便になります。

### 文字コードはUTF8MB4で統一する

my.cnfの設定に以下を追加し、UTF8MB4に統一する事ができます。
```ini
character_set_client = utf8mb4
character_set_connection = utf8mb4
character_set_database = utf8mb4
character_set_results = utf8mb4
character_set_server = utf8mb4
```

### 照合順序の設定

ut8mb4で使用できるf照合順序には以下があります。

| 照合順序名 | 大文字小文字 | 全角半角 |
| ------------------ | -------- | --------|
| utf8mb4_unicode_ci | 区別しない | 区別しない |
| utf8mb4_general_ci | 区別しない | 区別する  |
| utf8mb4_bin        | 区別する   | 区別する  |

基本的に `utf8mb4_bin` を推奨します。

DB側で大文字小文字・全半角の区別が付かなくなると、アプリケーション側で対応が不能になりますが、`utf8mb4_bin` のしておくと、アプリケーション側で同一とみなすかの実装を自由に行う事ができるようになります。

```ini
collation_connection = utf8mb4_bin
collation_server = utf8mb4_bin
```

### タイムゾーン

タイムゾーンを指定しておきます。
```
time_zone = Asia/Tokyo
```

## 📝クライアント側

mysqlコマンドで接続する場合、パスワードを指定して接続すると、以下のようにコマンド履歴にパスワードが保存されてしまい危険である、と警告されます。

```
mysql: [Warning] Using a password on the command line interface can be insecure.
```

これを回避する方法は2種類あります。

### --defaults-group-suffix を使う

ホームディレクトリに `.my.cnf` を配置し、以下のようにパーミッション `0600` を指定します。

```sh
$ ls -lha .my.cnf
-rw------- 1 ec2-user ec2-user 560  3月 25 13:50 .my.cnf
```
`.my.cnf` は以下のように接続毎にセクションを分けて指定します。
```ini
[clientREADONLY]
default-character-set = "utf8mb4"
host = "localhost"
database = "test"
port = 3306
user = "readonly"
password = "readonly"
local-infile = 1

[clientAPP]
default-character-set = "utf8mb4"
host = "localhost"
database = "test"
port = 3306
user = "app"
password = "app"
local-infile = 1
```

`~/.my.cnf` を定義した状態で以下のコマンドを実行すると、接続情報を指定せず接続する事ができ、 `password` や `接続先` 情報を隠蔽する事ができ、セキュリティリスクを低減させる事に繋がります。
```sh
# readonly接続する場合
mysql --defaults-group-suffix=READONLY
# 更新可能接続する場合
mysql --defaults-group-suffix=APP
```

もしこの定義を流用しつつportだけ変えたい場合、以下のように一部設定を上書きする事ができます。
```sh
# readonly接続する場合
mysql --defaults-group-suffix=READONLY -P3307
```

### --defaults-extra-file を使う

`--defaults-group-suffix` が `.my.cnf` のいちがホームディレクトリ固定であるのに対し、 `--defaults-extra-file` を使うと `.my.cnf` の位置を自由に指定する事ができます。

```sh
mysql --defaults-extra-file=/var/app/.my.cnf -P3307
```
この場合、グループ名を指定する事はできません。
