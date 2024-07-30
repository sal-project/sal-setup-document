演習支援ツール(secfarm)の環境構築について
========================================

# 概要

演習支援ツールsecfarmは組織・団体内で閉じた実機演習を支援するために開発したツールである。
研修用資料の配布や閲覧を本ツール経由で行うことができ、管理者は受講者の進捗状況などの
チェックを行うこともできる。

# 基盤構築手順

本節では、任意のOS上で演習支援ツールsecfarm動作させる手順を説明する。

### 動作環境について

本ツールはJava環境上で動作する。実行に必要な主なソフトウェアを挙げる。
ツールが依存するライブラリなどはleiningenによって実行時に自動的にダウンロードされるため
個別に準備する必要はなく、本表では記載を省いている。

| 名前       | 推奨バージョン  |
|------------|-----------------|
| Java       | 21 or later     |
| PostgreSQL | 13 or later     |
| leiningen  | 2.10.0 or later |

### 動作環境構築

ツールを実行するための環境の構築例を記載する。本説明ではAlmaLinux（RHELクローン）上に環境を構築する
手順を説明するが、他のOSであっても必要なソフトウェアを導入することでツールを実行することができる。

また本説明にて記載しているコマンドはシャープ記号（＃）で始まる場合はroot権限が必要なものを示し、
ドル記号（＄）で始まるものはroot以外の権限で実行するコマンドを示す。また、それ以外については
特定プログラムの対話型プロンプトなどを示す。

#### 必要パッケージのインストール

はじめに必要なソフトウェアをパッケージマネージャからインストールする。
インストールはAlmaLinuxの場合は下記のようにdnfコマンドを用いて行う。

```
# dnf install postgresql-server java-21-openjdk git
```

#### PostgreSQLのセットアップ

secfarmではRDBMSとしてPostgreSQLを利用する。本節ではPostgreSQLのセットアップを行う。
ここではAlmaLinuxにおいて最小限利用できるPostgreSQL環境を構築するために、下記の手順を実施する。
本節では最小限の設定のみ行うので利用者の状況に応じて必要な設定等を追加したり、手順を変更すること。

1. PostgreSQLの初期化
2. postgresユーザーのパスワード設定
3. TCPによる接続について認証設定を変更(pg_hba.conf)
4. secfarm用の新規ユーザーと新規データベースの作成

また、secfarmでは別ホスト上のDBを参照することも可能であるためすでに
外部にPostgreSQLサーバーがある場合はそれを利用することもできる。

##### PostgreSQLの初期化

はじめに下記のコマンドでデータベースを格納するディレクトリの初期化を行う。

```
# postgresql-setup --initdb
```

PostgreSQLへ接続できるようにするために、サービスを起動する。

```
# systemctl start postgresql
```

##### postgresユーザーのパスワード設定

postgresユーザーに切り替えて作業を行う。

```
# su - postgres
```

postgresユーザーのパスワードを変更する。例では"Passw0rd!"に変更しているが、
任意のパスワードを設定する。

```
$ psql
postgres=# ALTER ROLE postgres WITH PASSWORD 'Passw0rd!';
```

psqlコマンドを終了して、postgresユーザーからもログアウトする。

```
$ exit
```


##### TCPによる接続について認証設定を変更(pg_hba.conf)

pg_hba.confを編集して、TCPを用いた接続時に認証を要求するように設定する。

```
# vi /var/lib/pgsql/data/pg_hba.conf
```

下記の例のようにファイルの"host all all 127.0.0.1/32"および"host all all ::1/128"の行について、METHODを"md5"に変更する。

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
```

設定を反映するためにPostgreSQLサービスを再起動する。

```
# systemctl restart postgresql
```

##### secfarm用の新規ユーザーと新規データベースの作成

PostgreSQLにツール用のユーザーsecfarmとデータベースsecfarmdbを作成する。
ユーザー名とデータベース名は任意の名前で設定しても問題ない。また、パスワードは任意のものを設定する。

```
$ createuser -h localhost -U postgres -W -d -S -R -P secfarm
```

```
$ createdb -h localhost -U secfarm -E UTF-8 secfarmdb
```

#### Leiningenをインストールする

leiningenはAlmaLinuxではパッケージマネージャからインストールすることができないため、
下記のコマンドを用いて直接ダウンロードして、任意のPATHの通ったディレクトリに配置する。

```
$ curl -O https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein
```

#### secfarmの配置

gitリポジトリからsecfarmをダウンロードして、任意のディレクトリに配置する。
別途ブラウザ等を用いてリポジトリからzip形式でダウンロードする等別の方法でダウンロードしても良い。

```
$ git clone https://github.com/sal-project/secfarm.git
```


### secfarmの実行

#### 研修用ドキュメントの配置

secfarm上で表示する研修用ドキュメントは所定の形式で作成し、secfarmを実行するプロセスから読み込める任意の
ディレクトリ内に設置することでツール上で閲覧が可能となる。今回の例では /srv/secfarm/lecdata に配置する。

本プロジェクトでは sal-lecture-document として、業務システムおよびWebアプリケーションに関する
研修用ドキュメントを作成している。このドキュメントをツールから読み込む場合は、下記のようにする。

```
# mkdir -p /srv/secfarm
# cd /srv/secfarm
# git clone https://github.com/sal-project/sal-lecture-document.git lecdata
```

#### secfarmの初期設定

secfarmの設定ファイルはsecfarmのプロジェクトディレクトリ以下のresouces/config.ednである。
本ファイルにDBの情報などを設定する。

```
$ cd secfarm
$ vi resources/config.edn
```

下記は設定ファイルの例である。このファイルはedn形式で記載できる。edn形式については
公式のドキュメントを参照すること。

https://clojuredocs.org/clojure.edn

```
{:server {:port 3000
          :lecdata ["/srv/secfarm/lecdata"]}
 :db {:datasource-option {:auto-commit        true
                          :read-only          false
                          :connection-timeout 30000
                          :validation-timeout 5000
                          :idle-timeout       600000
                          :max-lifetime       1800000
                          :minimum-idle       10
                          :maximum-pool-size  10
                          :pool-name          "db-pool"
                          :adapter            "postgresql"
                          :username           "secfarm"
                          :password           "secfarm"
                          :database-name      "secfarmdb"
                          :server-name        "localhost"
                          :port-number        5432
                          :register-mbeans    false}}}
```

基本的に設定が必要な主要なパラメータは次の通りである。

| パラメータ名   | 意味                                                                             |
|----------------|----------------------------------------------------------------------------------|
| :lecdata       | ツールがロードする研修ドキュメントを配置するディレクトリのリスト（複数指定も可能） |
| :username      | PostgreSQLへ接続するユーザー名                                                   |
| :password      | PostgreSQLへ接続するユーザーのパスワード                                         |
| :database-name | PostgreSQLで使用するデータベース名                                               |
| :server-name   | 接続するPostgreSQLのサーバー名（ドメイン名またはIPアドレス）                     |
| :port-number   | PostgreSQLサーバーのポート番号                                                                                 |

secfarmは初回起動前にデータベースの初期化や、初期データの投入を行う必要がある。
この初期化は開発モードで実施するため、設定ファイルを開発用設定ファイルのPATHにコピーしておく。

```
$ cp resources/config.edn dev/resources/config.edn
```

#### secfarmの初期化

はじめに、次のコマンドでデータベースの初期化を行う。
プロジェクトディレクトリ以下でleinコマンド（leiningen）を用いて、REPLを起動する。
REPL上ではclojureコードを入力することで、対応した操作を行うことができる。

初期化のためにREPL上でdev関数およびmigrate関数を続けて実行してデータベースのテーブルを初期化する。

```
$ lein repl
> (dev)
> (migrate)
```

次に、再度REPLを起動して、管理ユーザーの初期化をinitialize関数で実行する。
1つ目のパラメータが管理ユーザー名、2つ目のパラメータは管理ユーザーの連絡先、3つ目のパラメータは管理ユーザーのパスワードである。

```
$ lein repl
> (require 'secfarm.core)
> (secfarm.core/initialize "admin" "admin@example.com" "adminpassword")
```

#### secfarm起動

下記のコマンドでsecfarmを起動することができる。

```
$ lein with-profile production run
```
