# DB定義指針

DB定義をする際に発生する以下の問題を解消するため、本書をまとめています。

- テーブル名・カラム名がバラバラ
- テーブル名・カラム名から内容が推測できない
- テーブル・カラムの命名のルールが解らない
- 2択にならないフラグがある
- 日付系カラムが全部datetimeになっている
- 大文字・小文字を区別して検索できない
- 意図しないデータが将来的に混入するかもしれない

## ローカル環境構築

```sh
# git cloneする
git clone https://github.com/treetips/treetips-db-definition-example.git
# node_modulesをインストールする
cd treetips-db-definition-example
npm i
# gitbookを起動する
npm start
```

## productionデプロイ

```sh
# gitbookをビルドしてhtmlを出力する
npm build
```
htmlを出力し、AmzonS3等にアップロードします。
