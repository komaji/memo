# Webオペレーション研修のあれこれ
## manualinastall
### VagrantでVM(Cent0S 7)を作る
- 作業するディレクトリへ移動
- 設定ファイル「Vagrantfile」の作成
- `vagrant init hfm4/centos7` でCentOS 7用のVagrantfileが作成される
- 仮想マシンの作成と起動
- `vagrant up`
- 仮想マシンにログイン
- `vagrant ssh`

### MySQLへ移行
- mysqlをインストール
- `brew install mysql`
- mysqlにrootでログインする
- `mysql -u root -p`
- passwordは何も入力しないでok
- 環境ごとにmysqlにDBを作成(development/test/production)

```
mysql> CREATE DATABASE db_development DEFAULT CHARACTER SET utf8;
mysql> CREATE DATABASE db_test DEFAULT CHARACTER SET utf8;
mysql> CREATE DATABASE db_production DEFAULT CHARACTER SET utf8;
```

- `usr/local/var/mysql`にmysqlのDBが作成される
- `show variables like 'datadir'`で`datadir`(DBの保存先)を確認できる
- sqlit3ではRails_Appの`db/`に作成されていた(railsが直接見に行く仕様のため)
- 環境ごとにmysqlにユーザを作成

```
mysql> GRANT ALL PRIVILEGES ON db_development.* TO user_development@localhost IDENTIFIED BY 'pass_development' WITH GRANT OPTION;
mysql> GRANT ALL PRIVILEGES ON db_test.* TO user_test@localhost IDENTIFIED BY 'pass_test' WITH GRANT OPTION;
mysql> GRANT ALL PRIVILEGES ON db_production.* TO user_production@localhost IDENTIFIED BY 'pass_production' WITH GRANT OPTION;
```

- Rails_Appの`config/database.yml`を以下のように修正する

```
default: &default
  adapter: mysql2
  encoding: utf8
  reconnect: false
  pool: 5
  socket: /var/lib/mysql/mysql.sock

development:
  <<: *default
  database: db_development
  username: user_development
  password: pass_development

test:
  <<: *default
  database: db_test
  username: user_test
  password: pass_test

production:
  <<: *default
  database: db_production
  username: user_production
  password: pass_production
```

mysqlでは`usr/local/var/mysql`に作成されているデータにadapterがsocketを通してアクセスする。
そのため、mysql2のadapterをインストールするために`Gemfile`を以下のように書き換えて`bundle install`する。

```
#gem 'sqlite3'
gem 'mysql2'
```

そして`adapter:`を書き換え、socketのファイルパスを`mysql_config --socket`で調べて`socket:` を書き換える。

- DBのマイグレーションをする
- `bundle exec rake db:migrate`
- DBをシードする
- `bundle exec rake db:seed`
- サーバを立ち上げて動作確認
- `bundle exec rails s -b 0.0.0.0`
- テストは落ちてしまうので、userとmicropostのymlの日付に`.to_s(:db)`をつける

## VMにRuby2.2をインストールする
- `wget`を使えるようにする
- `yum install wget`
- インストール先のディレクトリへ移動

```
wget http://cache.ruby-lang.org/pub/ruby/2.2/ruby-2.2.1.tar.gz
tar zxvf ruby-2.2.2.tar.gz
cd ruby-2.2.2
yum -y install gcc
./configure
make
make install
```
`ruby -v`を実行して`ruby 2.2.2p95 (2015-04-13 revision 50295) [x86_64-linux]`が表示されればok。

## その他
- `brew`では`/usr/local/Cellar`にインストールされる
