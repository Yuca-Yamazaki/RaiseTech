# 第5回課題
## 課題内容
-  EC2上にサンプルアプリケーションをデプロイ
   -  組み込みサーバーのみで起動
   -  Webサーバー（Nginx）とAPサーバー（Unicorn）で起動
-  ELB（ALB）を追加
-  S3を追加
-  構成図の作成

### EC2上にサンプルアプリケーションをデプロイ

#### 1.組み込みサーバーのみで起動
- Railsアプリを動かせるように環境構築をする
- 動作環境

|Name    |Version |
|--------|--------|
|ruby    |3.1.2   |
|Bundler |2.3.14  |
|Rails   |7.0.4   |
|Node    |17.9.1  |
|yarn    |1.22.19 |


- パッケージのアップデート


```
$ sudo yum -y update
```

- 必要なパッケージをインストール

```
$ sudo yum -y install git curl make gcc-c++ patch openssl-devel libyaml-devel libffi-devel libicu-devel libxml2-devel libxslt-devel zlib-devel readline-devel ImageMagick-devel
```

- Node(17.9.1)をインストール

```
#ノードバージョンマネージャー(nvm)をインストール
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash

#nvmを有効化
$ . ~/.nvm/nvm.sh

#任意のバージョンのNode.jsをインストール
$ nvm install 17.9.1
```

- yarnをインストール

```
$ npm install --global yarn
```

- Ruby(3.1.2)をインストール

```
rbenvをインストール
$ git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
$ source ~/.bash_profile

ruby-buildをインストール
$ git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build

rubyをインストール
$ rbenv install -v 3.1.2
$ rbenv global 3.1.2
$ rbenv rehash
```

- Rails(7.0.4)をインストール

```
$ gem install rails -v 7.0.4
```

- bundler(2.3.14)をインストール

```
$ gem install bundler -v 2.3.14
```


- サンプルアプリケーションのクローン

```
#クローンしてくるアプリの保存先を作成
$ sudo mkdir /var/www/

#作成したディレクトリの権限をec2-userに変更
$ sudo chown ec2-user /var/www/

#アプリをクローンするディレクトリに移動
$ cd /var/www

#サンプルアプリケーションのリポジトリをクローン
$ git clone https://github.com/yuta-ushijima/raisetech-live8-sample-app
```

- database.ymlファイルを作成

```
#database.ymlファイルを作成
$ cd raisetech-live8-sample-app/config/
$ cp config/database.yml.sample config/database.yml

#username、password、host、socketを任意の設定にする
$ vim database.yml
username: RDSのユーザー名
password: RDSのパスワード
host:     RDSのエンドポイント

development:
  <<: *default
  database: raisetech_live8_sample_app_development
  socket: /var/lib/mysql/mysql.sock
```


- MySQLをインストール

```
#MariaDBを削除
$ sudo yum remove -y mariadb-libs

#MySQLyumリポジトリを追加
$ sudo yum localinstall https://dev.mysql.com/get/mysql80-community-release-el7-9.noarch.rpm

#MySQLに必要なパッケージを取得
$ sudo yum install --enablerepo=mysql80-community mysql-community-server
$ sudo yum install --enablerepo=mysql80-community mysql-community-devel
```

- EC2のセキュリティグループのインバウンドルールにTCP ポート3000番を追加可

![TCP3000](lecture05/images_lec5/lec05_security1.png)

- 環境構築

```
$ bin/setup
```

- Rails サーバーの起動

```
$ bundle exec rails server -b 0.0.0.0 -p 3000
```


- EC2インスタンスのパブリックIPアドレスを使用してブラウザからアプリケーションにアクセス

![Puma1](lecture05/images_lec5/lec05_puma1.png)

![Puma2](lecture05/images_lec5/lec05_puma2.png)


#### 2.Webサーバー（Nginx）とAPサーバー（Unicorn）で起動

**Nginxをインストールして起動**


- EC2のセキュリティグループのインバウンドルールにHTTP ポート80番を追加可

![HTTP80](lecture05/images_lec5/lec05_security2.png)

- Nginxをインストール

```
#amazon-linux-extrasパッケージがインストールされていることを確認
$ which amazon-linux-extras

#パッケージの確認
$ amazon-linux-extras list | grep nginx

#インストール
$ sudo amazon-linux-extras install -y nginx1

#Nginxの起動
$ sudo systemctl start nginx

#起動確認
$ sudo systemctl status nginx

#起動停止
$ sudo systemctl stop nginx
```

![Nginx1](lecture05/images_lec5/lec05_nginx1.png)


**Nginxの設定ファイルを作成**

```
#/etc/nginx/conf.d配下にrails.confを新規作成して以下を追加
$ cd /etc/nginx/conf.d
$ sudo vim rails.conf

upstream unicorn {
  server unix:/var/www/raisetech-live8-sample-app/tmp/unicorn.sock;
}

server {
  listen 80;
  server_name localhost;
  root /var/www/raisetech-live8-sample-app/public;

  location @unicorn {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://unicorn;
  }

  try_files $uri/index.html $uri @unicorn;
  error_page 500 502 503 504 /500.html;
}  
```

**Unicornの設定**

- Gemfile を開き`gem unicorn`があることを確認
- `unicorn.rb`ファイルを編集

```
#configディレクトリにあるunicorn.rbファイルを編集
$ cd /var/www/raisetech-live8-sample-app/config/
$ vim unicorn.rb

# 以下の内容に変更
listen '/var/www/raisetech-live8-sample-app/unicorn.sock'
pid    '/var/www/raisetech-live8-sample-app/unicorn.pid'
```

**NginxとUnicornを起動**

```
#Nginxの起動
$ sudo systemctl start nginx

#Unicornの起動
$ bundle exec unicorn -c config/unicorn.rb -E development
```

![Nginx_unicorn1](lecture05/images_lec5/lec05_nginx_unicorn1.png)

- EC2インスタンスのパブリックIPアドレスを使用してブラウザからアプリケーションにアクセス

![Nginx_unicorn2](lecture05/images_lec5/lec05_nginx_unicorn2.png)



### ELB（ALB）を追加

- ターゲットグループの作成

![target](lecture05/images_lec5/lec05_target.png)

- ELB(ALB)用セキュリティグループの作成

![ELB1](lecture05/images_lec5/lec05_elb1.png)

- ロードバランサーの作成

![ELB2](lecture05/images_lec5/lec05_elb2.png)

- ヘルスステータスが healthy になっていることを確認

![ELB3](lecture05/images_lec5/lec05_elb3.png)

-  `development.rb`にELB(ALB)のDNS名を追加
 
```
config.hosts << "ELB(ALB)のDNS名"  
```

- ELB(ALB)のDNS名を使用してブラウザからアプリケーションにアクセス

![ELB4](lecture05/images_lec5/lec05_elb4.png)

### S3を追加

- S3バケットの作成

![bucket](lecture05/images_lec5/lec05_s3_bucket.png)


- アプリ用のIAMユーザーを作成

![IAM1](lecture05/images_lec5/lec05_s3_iam1.png)


- Gemfile を開き `aws-sdk-s3`があることを確認

**S3の設定**
- アクセスキーとシークレットアクセスキーを取得する

![IAM2](lecture05/images_lec5/lec05_s3_iam2.png)

- `development.yml.enc`ファイルを作成してアクセスキーを設定する

```
#ファイルのあるディレクトリへ移動
$ cd /var/www/raisetech-live8-sample-app/config/credentials/

#設定ファイルの削除
$ rm development.yml.enc

#設定ファイルの作成
$ EDITOR=vim rails credentials:edit --environment development


以下の情報をdevelopment.yml.encファイルに保存

aws:
  access_key_id: 作成したアクセスキーID
  secret_access_key: 作成したシークレットアクセスキー
  active_storage_bucket_name: 作成したバケットの名前

```

- `development.rb`を編集

```
#ファイルのあるディレクトリに移動
$ cd /var/www/raisetech-live8-sample-app/config/environments

#development.rbを編集
$ vim development.rb

#localからamazonへ変更
vconfig.active_storage.service = :amazon
```

- NginxとUnicornを再起動
- ELB(ALB)のDNS名を使用してブラウザからアプリケーションにアクセス

![s3_1](lecture05/images_lec5/lec05_s3_1.png)

- バケットに画像が保存されていることを確認

![s3_2](lecture05/images_lec5/lec05_s3_2.png)

### 構成図の作成

![architecture](lecture05/images_lec5/lec05_architecture.png)
