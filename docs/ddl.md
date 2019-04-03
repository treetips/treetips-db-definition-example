# DDL定義指針

## 📝データベース・テーブル生成時に必ずオプションを指定する

基本的にエンジンは文字コードは指定して、データベース・テーブルを作成します。
```sql
-- データベース作成テンプレート
create database hoge charset=utf8mb4 collate=utf8mb4_bin;

-- テーブル名作成テンプレート
drop table if exists hoge;
create table hoge(
  name varchar(10) comment 'ほげほげ'
)engine=innodb charset=utf8mb4 collate=utf8mb4_bin comment='ほげほげ';
```
指定しない場合はデフォルト指定値で作成されてしまい、環境に左右されてしまうので、オプションは必ず固定し、immutableになるようにします。

## 📝極力NOT NULLにする

null値は検索の効率が悪くなったり、 `null` `空文字` の2種類のemptyが発生してしまうので、極力NOT NULLを付けます。

## 📝フラグ値はboolean型を使う

フラグ値は必ずboolean型でNOT NULLとし、デフォルト値を指定する事で、フラグ値であるにも関わらずnullや2が入るといった `2択にならないフラグカラムの出現を抑制する` 事ができます。
```sql
hoge_flag boolean not null default false
```
尚、MySQLにはbool・booleanという型は実は存在せず、内部的には `tinyint(1)へのalias` になっています。
```sql
drop table if exists hoge;
create table hoge(
  b1 boolean not null default false,
  b2 bool not null default false
)engine=innodb charset=utf8mb4;
show create table hoge \G
*************************** 1. row ***************************
       Table: hoge
Create Table: CREATE TABLE `hoge` (
  `b1` tinyint(1) NOT NULL DEFAULT '0',
  `b2` tinyint(1) NOT NULL DEFAULT '0'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
```

[ `メリット` ] MySQL側をboolean型で定義する事で、ORMによってはBoolean・boolean型にマッピングしてくれる事もあり、便利に取り扱う事が可能になります。

[ `デメリット` ] boolean型は厳密には0・1以外の値が混入します。

### 厳密なbooleanカラム定義をするには

```sql
drop table if exists hoge;
create table hoge(
  b bit(1) not null default b'0'
)engine=innodb charset=utf8mb4;
insert into hoge values (2);
ERROR 1406 (22001): Data too long for column 'b' at row 1
insert into hoge values (-1);
ERROR 1406 (22001): Data too long for column 'b' at row 1
insert into hoge values (0), (1);
```
厳密にはなりますが、理解し難いエラーメッセージが表示されるのと、ORMとのマッピング相性が悪いため、ORM側でカラム名から型を決定する等のカスタマイズが別途必要になる可能性があります。

### DMLでもboolean値を使う

selectやupdateでもboolean型に対してboolean値を使う事が可能です。
```sql
select * from hoge where hoge_flag = true;
update hoge set hoge_flag = true;
insert into hoge values(true);
```
DMLでboolean値を指定する事で、0・1といったマジックナンバーを防ぐと共に、2や3といったフラグ値ではない値の混入を防ぐ事も可能になります。

## 📝固定長コード値はchar型を使う

`01` といった固定長の場合はchar型で定義します。
```sql
hoge_cd char(2)
```

## 📝現状のMySQLは数値型の桁数を制限する事はできません

例えば以下のように添え字を指定しても、3桁に制限されません。
```sql
hoge int(3)
```
この添字は `zerofill` オプションを付けた場合にのみ有効になります。
```sql
drop table if exists hoge;
create table hoge(
  i int(3) zerofill
)engine=innodb charset=utf8mb4;
insert into hoge values(1);
select * from hoge;
+------+
| i    |
+------+
|  001 |
+------+
```

## 📝機械的な作成日時・更新日時は自動更新を使う

以下のように定義すると、 `insert時` には `create_time` と `update_time` に現在日時が入り、 `update時` には `update_time` が現在日時で更新されます。
```sql
drop table if exists hoge;
create table hoge(
  cd char(10),
  create_time datetime not null default current_timestamp,
  update_time datetime not null default current_timestamp on update current_timestamp
)engine=innodb charset=utf8mb4;
insert into hoge (cd) values ('1');
select * from hoge;
+------+---------------------+---------------------+
| cd   | create_time         | update_time         |
+------+---------------------+---------------------+
| 1    | 2019-04-03 02:08:06 | 2019-04-03 02:08:06 |
+------+---------------------+---------------------+
update hoge set cd = '2' where cd = '1';
select * from hoge;
+------+---------------------+---------------------+
| cd   | create_time         | update_time         |
+------+---------------------+---------------------+
| 2    | 2019-04-03 02:08:06 | 2019-04-03 02:08:31 |
+------+---------------------+---------------------+
```

## 📝日付型を使う

全てdatetime型にするのではなく以下のような型を使う事でカラムの意味を明確化し、不正な値の混入を防ぎ、取扱いを楽にする事ができます。また、ORMによっては適切なマッピングが行われる事があります。

| MySQLの型 | 書式 | java側の型 |
| --- | --- | --- |
| year | yyyy | java.time.Year |
| date   | yyyy-MM-dd | java.time.LocalDate |
| time | HH:mm:ss | java.time.LocalTime |
| datetime | yyyy-MM-dd HH:mm:ss | java.time.LocalDateTime |
| timestamp | yyyy-MM-dd HH:mm:ss | java.sql.Timestamp |

## 📝テーブル・カラムにcomment句を記述する

カラムとテーブルそれぞれにコメントを設定する事ができます。
```sql
drop table if exists employee;
create table employee(
  last_name varchar(100) not null comment '姓',
  first_name varchar(100) not null comment '名'
)engine=innodb charset=utf8mb4 comment='従業員';
```
コメントを定義する事で以下のメリットを教授する事ができます。

- DataGripやSequel ProやTeamSQL等のツール使用時にコメントを表示してくれる場合がある。
- show create tableした際に内容が把握しやすくなる。

但し、コード定義等を書いてしまうと、コード値が増えた場合等にメンテナスされなくなる可能性があるため、基本的に内容のみを記述します。

## 📝alter tableにcomment句を記述する

カラム・テーブルだけでなく、インデックスにもコメントを付与する事ができるので、可能ならコメントを記述しましょう。
```sql
drop table if exists hoge;
create table hoge(
  name varchar(10) comment 'ほげほげ'
)engine=innodb charset=utf8mb4 collate=utf8mb4_bin comment='ほげほげ';
alter table hoge add key idx1(name) comment 'ほげほげ';
show index from hoge\G
*************************** 1. row ***************************
        Table: hoge
   Non_unique: 1
     Key_name: idx1
 Seq_in_index: 1
  Column_name: name
    Collation: A
  Cardinality: 0
     Sub_part: NULL
       Packed: NULL
         Null: YES
   Index_type: BTREE
      Comment:
Index_comment: ほげほげ
```

## 📝固定長ではないコード値は極力数値型を使う

可変長なコード値が入るカラムを文字列型で定義してしまうと、 `a` や `A` 等、意図しない値が将来的に設定されてしまう可能性があります。

これを防ぐため、可変長コード値が入るカラムは数値型が望ましい場合がほとんどです。

## 📝数値型でunsigned（符号無し）を指定できる場合は極力設定する

unsignedを指定しない場合、以下の問題が起きる場合があります。

- 金額等、アプリケーションバグで本来入らない筈のマイナス値が入ってしまう可能性
- エラーを表すための `-1` や `-9999` といった特殊な意味を持つ数値が生まれる可能性
- アプリケーション側で本来不要な筈のマイナス値チェックが必要になる可能性

unsignedを付与する事でマイナス値が入らなくなる代わりに、プラス値で使用可能な範囲が2倍になるというメリットもあります。

| MySQLの型 | 符号有りの範囲 | 符号無しの範囲 |
| --- | --- | --- |
| tinyint | -128 〜 127 | 0 〜 255 |
| smallint | -32768 〜 32767 | 0 〜 65535 |
| mediumint | -8388608 〜 8388607 | 0 〜 16777215 |
| int | -2147483648 〜 2147483647 | 0 〜 4294967295 |
| bigint | -9223372036854775808 〜 9223372036854775807 | 0 〜 18446744073709551615 |

## 📝MySQLの数値型（符号無し）とORMのマッピングの問題

MySQLでunsignedを使用する場合、ORMのjava側との型マッピング時にjava側の範囲が足りないケースが発生します。

| MySQL側の型 | java側の型 |
| --- | --- |
| tinyint(-128 〜 127) | ○ byte(-128 ～ 127) |
| tinyint unsigned(0 〜 255) | ☓ byte(-128 ～ 127) |
| smallint(-32768 〜 32767) | ○ short(-32768 ～ 32767) |
| smallint unsigned(0 〜 65535) | ☓ short(-32768 ～ 32767) |
| mediumint(-8388608 〜 8388607) | ○ int(-2147483648 ～ 2147483647) |
| mediumint unsigned(0 〜 16777215) | ○ int(-2147483648 ～ 2147483647) |
| int(-2147483648 〜 2147483647) | ○ int(-2147483648 ～ 2147483647) |
| int unsigned(0 〜 4294967295) | ☓ int(-2147483648 ～ 2147483647) |
| bigint(-9223372036854775808 〜 9223372036854775807) | ○ byte(-9223372036854775808 ～ 9223372036854775807) |
| bigint unsigned(0 〜 18446744073709551615) | ☓ byte(-9223372036854775808 ～ 9223372036854775807) |

ライブラリ側で専用の符号無し型を提供していない場合、ORMのジェネレータをカスタマイズし、unsignedの場合はjava側の型を1つ上の型に指定する等で、数値の範囲オーバーを回避する事が可能になります。

## 📝特殊な意味を持つ数値型は設定しない

int型で `-1` は既知のエラー、 `-9999` は未知のエラー、というように数値に意味を持たせてしまうと、数値として使用できなくなる可能性があるため、意味を持たせたい場合は別途カラムを追加しましょう。
