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
- mysqlのrootのパスワードを設定する
  - `mysql> SET PASSWORD FOR root@localhost=PASSWORD('hoge'); `
- mysqlにrootでログインする
  - `mysql -u root -p`
  - passwordは設定したもの
- 環境ごとにmysqlにDBを作成(development/test)
  - 手元で確認するだけなので、productionは不要

```
mysql> CREATE DATABASE db_development DEFAULT CHARACTER SET utf8;
mysql> CREATE DATABASE db_test DEFAULT CHARACTER SET utf8;
```

  - `/usr/local/var/mysql`にmysqlのDBが作成される
    - `mysql> show variables like 'datadir'`で`datadir`(DBの保存先)を確認できる
  - sqlit3ではRails_Appの`db/`に作成されていた(railsが直接見に行く仕様のため)
- 環境ごとにmysqlにユーザを作成

```
mysql> GRANT ALL PRIVILEGES ON db_development.* TO user_development@localhost IDENTIFIED BY 'pass_development' WITH GRANT OPTION;
mysql> GRANT ALL PRIVILEGES ON db_test.* TO user_test@localhost IDENTIFIED BY 'pass_test' WITH GRANT OPTION;
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

mysqlでは`/usr/local/var/mysql`に作成されているデータにadapterがsocketを通してアクセスする。
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

### VMにRuby2.2をインストールする
- `wget`を使えるようにする
  - `yum install wget`
- インストール先のディレクトリへ移動

```
wget http://cache.ruby-lang.org/pub/ruby/2.2/ruby-2.2.1.tar.gz
tar zxvf ruby-2.2.2.tar.gz
cd ruby-2.2.2
yum install gcc
./configure
make
make install
```
`ruby -v`を実行して`ruby 2.2.2p95 (2015-04-13 revision 50295) [x86_64-linux]`が表示されればok。  
configureスクリプトによりソフトウェアをビルドする場合、`./configure --prefix=ディレクトリ`とすることでインストール先のディレクトリを指定できる

### VagrantにRails環境を用意する

- 不足しているものをインストールする
```
yum install zlib-devel 
yum install opennssl-devel
cd /opt/ruby-2.2.2/
./configure
make
make install
```

ここで`gem install rails`するとnokogiri周りでエラーが起きるので、以下を行う。

```
yum install libxml2 libxslt libxml2-devel libxslt-devel
gem install nokogiri -- --use-system-libraries
```
- Railsをインストール
  - `gem install rails`

### Vagrantでsample_appを動かす

- githubからsample_appをインストールするためgitをインストールする
  - `yum install git`
- sample_appをcloneする
  - `git clone https://github.com/komaji/sample_app.git`
- mariadbを使えるようにする
  - `yum install mariadb-server mariadb-devel`
- postgresqlのgemをインストールする
  - `yum gem install postgresql-devel`
- これで`bundle install`が通る
- mariadbを起動する
  - `service mariadb start`
- mysqlのrootのパスワードを設定する
  - `mysql> SET PASSWORD FOR root@localhost=PASSWORD('hoge'); `
- mysqlにrootでログインする
  - `mysql -u root -p`
  - passwordは設定したもの
- 各環境のDBとユーザを作成

```
mysql> CREATE DATABASE db_production DEFAULT CHARACTER SET utf8;
mysql> CREATE DATABASE db_development DEFAULT CHARACTER SET utf8;
mysql> CREATE DATABASE db_test DEFAULT CHARACTER SET utf8;

mysql> GRANT ALL PRIVILEGES ON db_development.* TO user_development@localhost IDENTIFIED BY 'pass_development' WITH GRANT OPTION;
mysql> GRANT ALL PRIVILEGES ON db_test.* TO user_test@localhost IDENTIFIED BY 'pass_test' WITH GRANT OPTION;
mysql> GRANT ALL PRIVILEGES ON db_production.* TO user_production@localhost IDENTIFIED BY 'pass_production' WITH GRANT OPTION;
```

- `config/databases.yml`のsocketとDB名やユーザー名やパスワードやを書き換える。
dbをmigrationをしようとすると`xecJS::RuntimeUnavailable: Could not find a JavaScript runtime.`となるので以下をgemfileに追加して`bundle install`。
```
gem 'execjs'
gem 'therubyracer', :platforms => :ruby
```
しかし、therubyracerがインストールできないと言われるので、`yum install gcc-c++`を実行して、もう一度`bundle install`。  
がしかし、`There was an error while trying to write to Gemfile.lock. `となるので、所有者を`chown`コマンドで変更して、`bundle install`。  
ここでdbをmigrationしようとしても`LoadError: cannot load such file -- readline`となるので、先にgemfileに`gem 'rb-readline'`を追加して`bundle install`。
- DBのマイグレーションをする
  - `bundle exec rake db:migrate`
- DBをシードする
  - `bundle exec rake db:seed`
- loalhost:8080でアクセスできるようにするためにVagrantのForwarded portsを使う
  - クライアントのVagrantfileの`config.vm.network "forwarded_port", guest: 80, host: 8080`の箇所のコメントを外す
  - `vagrant reload`を実行する  
現段階ではcentosのfirewallによって、ポート8080が開いておらず、また、httpが許可されていない状態なので以下を実行する。  
```
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --add-port=8080/tcp
```
- mariadbを再起動
  - `systemctl restart mariadb`
- localhot:8080にアクセスして動作確認

### その他
- `brew`では`/usr/local/Cellar`にインストールされる
- `brew --cellar mysql`で`datadir/local/Cellar/mysql` とかが取得できる
  - `ls -l $(brew --cellar mysql)`でmysqlの中が見られる
- `Permission denied @ rb_sysopen - mkmf.log (Errno::EACCES)`が起きた場合
  - mkmf.logの所有者が実行したファイルと異なっている場合があるので、`ls -l`で所有者を確認し、異なる場合は`chown`コマンドで変更する必要がある。
firewall-cmd --add-p- `yum`コマンドの`-y`オプションは、問い合わせがあったときに全て「y」と答える
- `yum list installed | grep <パッケージ名>`は`yum`でインストールしたパッケージを確認する
- `mysql> show databases`で作成されたDBの一覧を確認できる
- `/var/log/`にいろいろなログがあるよ
- `curl`でコマンドラインからurlにアクセスする
- `curl -IL`でアクセスした時の情報が見れる
- HTTPステータスコード
  - 1xx Informational 情報
  - 2xx Success 成功
  - 3xx Redirection リダイレクション
  - 4xx Client Error クライアントエラー
  - 5xx Server Error サーバエラー
- コマンドラインで複数ファイルから特定の文字列を検索する
  - find . -type f -name "*.rb" | xargs grep 'keyword' -n  

## WEBrickからNginx + Unicornに切り替える
### Nginxをインストールする
- `yum install nginx`
- `/etc/yum.repos.d/nginx.repo`を作成する。中身は以下にする。
```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1
```
- `service nginx start`を実行し、`http://localhost:8080/`にアクセスしてWelcome to nginx!と表示されればok。
- `service nginx stop`で停止

### Unicornの設定
- gemfileを以下のように書き換えて`bundle install`する  
```
#gem 'puma'
gem 'unicorn'
```
- `config/unicorn.rb`を作成する。中身は以下にする。  
```
#config/unicorn.rb
worker_processes Integer(ENV["WEB_CONCURRENCY"] || 3)
timeout 15
preload_app true

before_fork do |server, worker|
  Signal.trap 'TERM' do
    puts 'Unicorn master intercepting TERM and sending myself QUIT instead'
    Process.kill 'QUIT', Process.pid
  end

  defined?(ActiveRecord::Base) and
    ActiveRecord::Base.connection.disconnect!
end

after_fork do |server, worker|
  Signal.trap 'TERM' do
    puts 'Unicorn worker intercepting TERM and doing nothing. Wait for master to send QUIT'
  end

  defined?(ActiveRecord::Base) and
    ActiveRecord::Base.establish_connection
end
```

## その他
- stdin
  - コンソールからの入力
- stdout
  - コンソールへの出力  
- stderr
  - コンソールへのエラーログ出力  
