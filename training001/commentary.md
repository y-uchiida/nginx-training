# commentary of training001
実際に操作しながら、思考の整理のためにメモしていたことをまとめました。

## コンテナの操作
alpineのコンテナを起動して`docker-compose exec nginx bash`を実行すると、コマンドがないといわれてしまいます。  
bash入ってないんだなと思いつつ、コンテナ内を直接操作するためにシェルを探します。
~~~bash
$ docker-compose exec nginx which sh
/bin/sh
~~~
shはあるので、あまり使ったことはないけどshでコンテナ内に入っていきます。
~~~bash
$ docker-compose exec nginx sh
/ # nginx -v
nginx version: nginx/1.19.
/ #
/ # hostname
172e22a317a2
# ↑コンテナのIDと一致します
/ #
/# exit
~~~
終了は普通にexitコマンドでOKでした。コンテナも起動したままホスト側に戻ってこられました。

## Nginx の起動状態の確認
コンテナ環境でなければ `sudo systemctl status nginx`などでやるのですが、  
コンテナ内だと `systemctl` とか `service` のコマンドが使えません。。。  
というわけで、プロセスが生きてるかどうかで起動しているかの確認を行うことにしました。  
~~~sh
/ # ps ax | grep nginx
	  1 root      0:00 nginx: master process nginx -g daemon off;
	101 nginx     0:00 nginx: worker process
	102 nginx     0:00 nginx: worker process
	103 nginx     0:00 nginx: worker process
	104 nginx     0:00 nginx: worker process
	106 root      0:00 grep nginx
~~~
上記例では全てのプロセスを確認するため `ps ax` を利用しましたが、  
実行中のプロセスの一覧だけを表示して、nginxがなければ起動していない、という判断もできそうです。
その場合のコマンドは `ps r | grep nginx` です。

## 環境の永続化
これは厳密にはDockerの話ですが、、、コンテナ内のNginxに設定した内容を次回にも引き継ぐための設定をどうするか考えました。  
とりあえずvolumesで設定ファイルをコンテナ外と同期させる対応を行うことにしました。
~~~yml
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/conf.d:/etc/nginx/conf.d
~~~
同期の対象は、`nginx.conf` ファイルと、 `conf.d` ディレクトリです。
設定ファイルを追加する場合は、 `conf.d` ディレクトリに追加していけばOKです。

## 設定ファイルの構文確認
`nginx -t` で、設定ファイルに記述した内容に誤りがないか確認してくれます。
~~~sh
# / nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
~~~

`nginx -T` は、読み込んでいる設定ファイルの内容をすべて出力します。  
かなりの行数が吐き出されるので注意が必要です。

## nginx.confの構造
Apacheと同じく、ディレクティブにたいして設定を行っていきます。  
Apacheの設定ファイルはhtmlのような見た目で、Nginxの方はスクリプト言語のような見た目です。  
初期設定のnginx.confはこういう感じです。  
プログラミングやっている人にはこっちのほうがしっくりくるのかな。
~~~nginx
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
~~~


### モジュール
トップレベルのコンテキストに置かれる `http` とか `events` などは、モジュールというnginxの機能ごとの設定をまとめるものです。  
トップレベルコンテキストのなかで1度だけ記述できます。  
training001では、主にhttpモジュールに設定されるコンテキストの設定について見ていくことにします。

### ブロックとコンテキスト
`{ }`で囲まれている範囲は、それを囲んでいるモジュールやディレクティブ内の設定を行っている部分です。  
`{ }`によって設定の階層構造を表現します。囲まれている範囲内を「コンテキスト」と呼ぶようです。  

### コメント
#記号の右以降をコメントとして扱います。#記号の左の文字はコメントとみなされません。  
複数行コメントには対応してないようです。

### ディレクティブへの文字列の設定
クオート囲みあり/なしどちらでも大丈夫でした。  
`" "` と `' '`の区別もありません。

## default.conf
default.confの内容をサラッと見ていきます。

### server ブロック
このブロックの内部が、ひとつのバーチャルホストとして認識されます。  
default.confの設定内容はすべてserver ブロックで囲んであるので、default.confは初期設定のバーチャルホストということになります。  

### listen と server_name
Apacheと似たようなディレクティブ名です。  
listenはnginxの待ち受けポート、server_nameはこの設定内容が適用されるドメイン名を設定するディレクティブです。  
半角スペース区切りで複数のドメイン名を並べることができるようです。  
というわけで、初期値の `localhost` に加えて、デフォルト用のドメインにも割り当てておきます。
~~~nginx
    server_name  localhost  nginx-training001.default;
~~~

### location ディレクティブ
URIのパスに対して、条件に合致する場合に設定を反映します。  
`location /` の場合はルート以下すべてのURIに適用されるので、そのドメインに対するデフォルトの設定を行っていることになります。

### root ディレクティブと index ディレクティブ
root ディレクティブで、サーバー上のパスを設定します。  
default.confでは
~~~nginx
location / {
	root   /usr/share/nginx/html;
    index  index.html index.htm;
}
~~~
と設定されていますので、httpリクエストに対して `/usr/share/nginx/html` ディレクトリの内部が参照されます。  
今回は `/var/www/public/nginx-training001.default` を指定しました。  
web公開用によく使う`/var/www` 配下に、ドメインごとに分けて公開ディレクトリを持てるように設定します。  
ついでに、docker-compose.ymlのvolumesにもこれに対応するディレクトリを追加しておきます。

### error_page ディレクティブ
40x系、50x系のステータスコードに対して、それぞれオリジナルのエラーページを指定できるようです。
~~~nginx
error_page  500 502 503 504 /50x.html;
location = /50x.html{
    root   /usr/share/nginx/html;
}
~~~

## バーチャルホストとマルチドメイン設定
つづいて、マルチドメイン対応の方法を調べます。  
Debian系ディストリビューションのApacheでよく用いられている方法が紹介されていました。  
 - sites-available ディレクトリにバーチャルホストごとのconfファイルを作成する  
 - sites-enabled ディレクトリにバーチャルホストのconfファイルへのシンボリックリンクを貼る  
 - nginx.confのhttpディレクティブに、sites-enable内のconfファイルをインクルードする設定を記述する

こうすることで、設定ファイルを分割しつつ現在運用中のバーチャルホストが一目でわかる、ということのようです。  
ちょいと回りくどい感じもしますが、せっかくなのでやってみます。

### docker-composeのvolumesを追加
まずは`docker-compose.yml` に、バーチャルホストの設定ファイルを保存するためのディレクトリを追記します。
~~~yml
volumes:
  - ./nginx/nginx.conf:/etc/nginx/nginx.conf
  - ./nginx/conf.d:/etc/nginx/conf.d
  - ./nginx/sites-available:/etc/nginx/sites-available  #  <- これを追加
  - ./nginx/sites-enable:/etc/nginx/sites-enable        #  <- これを追加
  - ./var/www/public:/var/www/public
~~~

つづいて、ホストの `sites-available` ディレクトリにバーチャルホスト用のconfファイルを作成します。  
hostsファイルに設定してある `nginx-training001-1.local` `nginx-training001-2.local` `nginx-training001-3.local` のファイルを用意します。  
例として `nginx-training001-1.local.conf`はこんな感じにしました。
~~~nginx
server{
    listen       80;
    listen  [::]:80;
    server_name  nginx-training001-1.local;

	# /var/www/public 内に作成したディレクトリをルートに設定
	location / {
		root /var/www/public/nginx-training001-1.local;
	}
}
~~~

設定ファイルで指定したディレクトリと、その中にindex.htmlファイルを作っておくのも忘れずに。  

それから、`sites-enabled` ディレクトリにシンボリックリンクを作ります。
~~~bash
$ ln -s ../sites-available/nginx-training001-1.local.conf nginx-training001-1.local.conf
$ ls -l
0 lrwxrwxrwx 1 uchiida uchiida 49 Nov  1 18:18 nginx-training001-1.local.conf -> ../sites-available/nginx-training001-1.local.conf
~~~

最後に、作成した設定ファイルを`nginx.conf` からincludeすれば設定は完了です。
~~~nginx
    # バーチャルホスト用の設定ファイルをインクルード
    include /etc/nginx/sites-enabled/*.conf;
~~~

これで、 ブラウザからhttp://nginx-training001-1.local:18080/ にアクセスして、作成したindex.htmlが表示できれば成功です。  

## お疲れさまでした！
`server` ブロックでバーチャルホスト設定ができたところまでで、training001は完了としました。  
今後はbasic認証とか、SSLとか、Apacheでもよく扱う設定を試してみたいと考えています。