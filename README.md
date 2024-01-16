# Nextcloud

## 調査したいこと

- sso,saml認証のやり方

## 初期環境構築

### docker環境構築

```text
docker-compose build
```

```text
docker-compose up -d
```

アクセス：http://localhost:8000/login

### 環境設定

**可能ならdockerfileなどに設定を記載したい（https://hub.docker.com/_/nextcloud）**

```text
docker exec -it NextCloudのコンテナID /bin/bash
```

```text
apt-get update
```

```text
apt-get install vim
```

```text
vi config/config.php
```

最後に以下を追加（デフォルト言語とデフォルトロケールの設定）

```text
'default_language' => 'ja',
'default_locale' => 'ja',
```

### 管理者権限のログイン情報

ID

```text
admin
```

PASSWORD

```text
admin
```

### DB

データベース：myswql/mariaDB

ユーザー名 ：docker-compose.ymlの「MYSQL_USER」
パスワード ：docker-compose.ymlの「MYSQL_PASSWORD」
データベース名 ：docker-compose.ymlの「MYSQL_DATABASE」
ホスト名 ：docker-compose.ymlのデータベースのservice名

### 必要なアプリ

- Group folders
  - 右上の[①ユーザ] > 「②アプリ」>「③group」で検索し、Group foldersの「④ダウンロードして有効にする」をクリック
  - 右上の[①ユーザ] > 「②管理者設定」> サイドメニューの「グループフォルダー」から利用可能

### 必要な設定

- パスワードポリシー
  - 右上の[①ユーザ] > 「②管理者設定」> サイドメニューの「セキュリティ」
  - AI-PHARMAのパスワードポリシーに合わせるならトグルは全部OFFにする

## API

### [グループ管理](https://docs.nextcloud.com/server/latest/admin_manual/configuration_user/instruction_set_for_groups.html)

#### ■ グループ新規作成

`curl -X POST http://admin:admin@localhost:8000/ocs/v1.php/cloud/groups -d groupid="新規施設1" -H "OCS-APIRequest: true"`

`groupid`： 文字列、新しいグループ名、グループ情報を取得する際に必要な値

※ 途中で施設名を変更されても`groupid`は変更されないみたい
※ 別のアカウント管理システムからnextcloud側のグループを作成する場合、`groupid`をアカウント管理システムに保持させる必要あり

#### ■ [グループフォルダの作成](https://github.com/nextcloud/groupfolders?tab=readme-ov-file#apis)

`curl -X POST http://admin:admin@localhost:8000/apps/groupfolders/folders -d mountpoint="施設1" -H "OCS-APIRequest: true"`

`mountpoint`: 新しいフォルダーの名前

※ xml形式のレスポンスにグループフォルダのidがあるためアカウント管理システム側で保存しておく必要がある

#### ■ [グループフォルダへのアクセス権限を付与](https://github.com/nextcloud/groupfolders?tab=readme-ov-file#apis)

`curl -X POST http://admin:admin@localhost:8000/apps/groupfolders/folders/$folderId/groups -d group="新規施設1" -H "OCS-APIRequest: true"`

`group`: アクセス権限を付与したい施設のID

#### ■ [グループフォルダへの容量制限を付与](https://github.com/nextcloud/groupfolders?tab=readme-ov-file#apis)

`curl -X POST http://admin:admin@localhost:8000/apps/groupfolders/folders/$folderId/quota -d quota="209715200" -H "OCS-APIRequest: true"`

`quota`: フォルダーの新しいクォータ (バイト単位)、ユーザー-3は無制限（209715200=200MB）



### [ユーザー管理](https://docs.nextcloud.com/server/latest/admin_manual/configuration_user/index.html)

#### ■ ユーザー新規登録

`curl -X POST http://admin:admin@localhost:8000/ocs/v1.php/cloud/users -d userid="demo1" -d password="demo1#12345" -d displayName="デモ1" -d email="demo1@stad.kit-rd.com" -d groups[]="新規施設1" -d quota="1B" -d language="ja" -H "OCS-APIRequest: true"`

`-d userid="demo1"`：文字列、新規ユーザーのユーザー名。
`-d password="demo1#12345"`： 文字列、新規ユーザーのパスワード、ウェルカムメールを送信する場合は空のままにする。
`-d displayName="デモ1`：文字列、新規ユーザーの表示名
`-d email="demo1@stad.kit-rd.com"`：文字列、新規ユーザーのEメール、パスワードが空の場合は必須
`-d groups[]="新規施設1`： 配列、新規ユーザーのグループ
`-d subadmin=""`：配列、新しいユーザーがサブ管理者になるグループ
`-d quota="1B"`： 文字列、新しいユーザーのクォータ数、**好き勝手個人で登録できないように最小限にしておく**
`-d language="ja"`： 文字列、新しいユーザーの言語、言語の指定は[こちら](https://explore.transifex.com/languages/)を参照

**ssoまたはsamlで認証した場合、ユーザー情報どうなるか気になる**

#### ■ ユーザーをグループに追加

`curl -X POST http://admin:admin@localhost:8000/ocs/v1.php/cloud/users/{userid}/groups -d groupid="デモ1" -H "OCS-APIRequest: true"`

`-d groupid="デモ1"`：文字列、所属させたいグループのgroupid

#### ■ ユーザーをグループから削除

`curl -X DELETE http://admin:admin@localhost:8000/ocs/v1.php/cloud/users/{userid}/groups -d groupid="デモ1" -H "OCS-APIRequest: true"`

`-d groupid="デモ1"`：文字列、所属させたいグループのgroupid


#### ■ ユーザーを無効にする

`curl -X PUT http://admin:admin@localhost:8000/ocs/v1.php/cloud/users/{userid}/disable -H "OCS-APIRequest: true"`

※ nextcloud側にアカウントは残り続けるがログインできなくなる
※ 削除するとファイルも削除される可能性があるため検討が必要