# commentary of training002
training001で作成したファイルを流用しつつ、いくつかの具体的な設定を行ってみました。  
実際に操作しながら、調べたことなどをメモしていきます。  

## nginx.conf
training001ではスルーしてしまったnginx.confの記述内容について、  
今回は設定できるディレクティブの種類と設定方法まで、ゼロベースで調べて記述してみました。  
nginx.conf.with_explainに、各ディレクティブについて調べた内容を記載しています。  
ディレクティブの種類はとても多くて、すべては記載しきれませんでしたが…  
よく利用されるディレクティブについては、おおむね理解できました。

## basic認証設定
training002では、Webサーバーでよく行わせるいくつかの設定を試してみることにしました。  
basic認証は公開前のページの確認などでしょっちゅう設定しますので、まずはこれをやってみることに。  
hostsに記載した、nginx-training-01.localに対して設定を行います。

### auth_basic, auth_basic_user_file を追記
nginx-training01.local用のconfファイルのserverディレクティブに、下記を追加します。  
~~~nginx

    root /var/www/public/nginx-training-01.local;

	location / { } # ここに設定していたroot ディレクティブを外に出す

	#user001用のbasic認証設定を行う
	location /user001/ {
		auth_basic            "auth as user001.";
		auth_basic_user_file  /var/www/settings/.htpasswd_user001;
	}

	#user002用のbasic認証設定を行う
	location /user002/ {
		auth_basic            "auth as user002.";
		auth_basic_user_file  /var/www/settings/.htpasswd_user002;
	}

	#user003用のbasic認証設定を行う
	location /user003/ {
		auth_basic            "auth as user003.";
		auth_basic_user_file  /var/www/settings/.htpasswd_user003;
	}
~~~
server全体に rootを記載しておけば、各location ディレクティブでrootをそれぞれ設定する必要がありません。  
あとは、 `/var/www/settings/.htpasswd_*`のファイルをそれぞれ作って、volumesでホストと共有するようにすればOKです。  

### 認証情報が記載されたファイル(.htpasswd)を作成する
.htpasswdの書式は、Apacheと同様です。今回はopenssl()コマンドで作成してみました。  
~~~bash
$ printf "user001:$(openssl passwd -crypt USER001)\n" > ./path/to/.httpasswd_user001
~~~
作成されたファイルを、volumesで共有したディレクトリに入れます。今回は、./var/www/settings/にしました。  

http://nginx-training-01:18080/user001/ にアクセスしてみて、Basic認証が表示されれば設定は完了です。

## SSL設定(自己証明書利用)
nginx-training-02のドメインをSSL/TLSに対応してみます。  
じつは自己証明書での認証ってやったことがなかったので、ついでにその練習も兼ねました。  

やるべきことは以下の3つです。  
- confファイルの設定を変更  
- openssl()で秘密鍵を作成し、それを利用してCSRとCRTを作成  
- docker-compose.ymlファイルを変更  

### confファイルの設定を変更
まずはnginxの設定ファイルを変更します。nginx-training-02.local.confを以下のように変えました。
~~~nginx
server{
    listen       80;
    listen  [::]:80;
    server_name  nginx-training-02.local;

    # http接続は、httpsへリダイレクトする
    return  301 https://nginx-training-02.local:10443$request_uri;
}

# https接続を443ポートで待ち受け
server{
    listen       443 ssl;
    listen       [::]:443 ssl;
    server_name  _;

    root /var/www/public/nginx-training-02.local;

    # 証明書と鍵ファイルのパスを記載する
    ssl_certificate      /etc/nginx/ssl/nginx-training-02.local.crt;
    ssl_certificate_key  /etc/nginx/ssl/nginx-training-02.local.key;
}
~~~

### openssl()で秘密鍵を作成し、それを利用してCSRとCRTを作成
ここでも、openssl()を利用していきます。まずは秘密鍵を作成。
~~~bash
$ openssl genrsa -out nginx-training-02.local.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
.+++++
.......................+++++
~~~

この秘密鍵ファイルを利用して、CSR(証明書署名要求)を作成します。
~~~bash
$ openssl req -new  -key ./nginx-training-02.local.key -out ./nginx-training-02.local.csr
~~~
CSRファイルに記載される情報を聞かれますが、とりあえずC(Conutry)とST(State)、そしてCommon Name(サーバーのドメイン名)だけ入力します。  
それ以外のものは空Enterです。  

出力されたCSRの内容で、CRT(サーバー証明書)を作成します。
~~~bash
$ openssl x509 -days 3650 -req -signkey ./nginx-training-02.local.key -in ./nginx-training-02.local.csr -out ./nginx-training-02.local.crt
Signature ok
subject=C = JP, ST = Tokyo, O = Internet Widgits Pty Ltd, CN = nginx-training-02.local
Getting Private key
$
$ ls -ls
total 12
4 -rw-r--r-- 1 uchiida uchiida 1204 Nov  3 19:08 nginx-training-02.local.crt
4 -rw-r--r-- 1 uchiida uchiida  997 Nov  3 19:02 nginx-training-02.local.csr
4 -rw------- 1 uchiida uchiida 1679 Nov  3 18:53 nginx-training-02.local.key
~~~

key, csr, crtファイルが作成されています。  
必要なファイルはこれですべてです。

### docker-compose.ymlファイルを変更

1. volumesに秘密鍵ファイルと証明書ファイルを追加
dockerを利用しているので、コンテナ側にファイルを受け渡ししなければなりません。  
csrファイルはSSL/TLS接続には必要ないので、秘密鍵ファイル(.key)とサーバー証明書ファイル(.crt)をvolumesに追記します。
~~~yml
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/sites-available:/etc/nginx/sites-available
      - ./nginx/sites-enabled:/etc/nginx/sites-enabled
      - ./nginx/ssl/nginx-training-02.local.key:/etc/nginx/ssl/nginx-training-02.local.key # <- これを追記
      - ./nginx/ssl/nginx-training-02.local.crt:/etc/nginx/ssl/nginx-training-02.local.crt # <- これを追記
      - ./var/www/public:/var/www/public
      - ./var/www/settings:/var/www/settings
~~~

2. portにhttps接続用のポート番号を追記
ホスト側の接続を、コンテナの443ポートに転送する設定を追記します。
~~~yml
    ports:
      - '18080:80'
      - '10443:443' # <- これを追記
~~~

docker-composeファイルの変更は以上で完了です。  

これで、  
https://nginx-training-02.local:10443/  
へアクセスしてみて、エラーにならずに表示ができればOKです。  

また、  
http://nginx-training-02.local:18080/  
にアクセスしても、httpsのURLにリダイレクトもされるようになっています。

## rewriteによるリダイレクト
最後に、リダイレクトの設定を試してみようと思います。  
SSL/TLSの設定ファイルでは、return ディレクティブでの転送をちょっと扱いましたが…
リダイレクト時のステータスコードの設定などをもう少し詳しく見ていこうと思います。

### rewtire ディレクティブ
Apacheのrewrite engineに対応する機能で、特定の条件に一致したURLへのアクセスに対して、  
別のURLを参照するように転送させます。  
書式は以下の通り。  

`rewrite [条件(正規表現)] [転送先] [フラグ]`

1. 条件

リクエストされたURLがこの条件に一致した場合、[転送先]で指定したURLに転送します。  
PCRE形式の正規表現が利用できます。  
利用できる特殊記号は以下のものが代表的。

| 記号 | 意味 | 例 | 一致する文字列 |
|----|----|----|----|
| ^ | 文字列の先頭 | ^/first-dir/ | /first-dir/ から始まる文字列 |
| $ | 文字列の末尾 | .*\\.css$ | .css で終わる文字列 |
| . | 任意の1文字| dir_. | 'dir_'とその後ろの1文字(何かがあるとき) |
| * | 0回以上の一致 | .* | 文字列の全体(空文字列も含む) |
| + | 1回以上の一致 | ^.+ | 文字列先頭の1文字 |
| [] | かっこ内の文字のいずれか | [abc] [a-z] [0-9] | はじめのパターンは 'a' 'b' 'c'のいずれか、2つ目は小文字アルファベット、3つ目は数字 |
| {n,m} | 指定回数の一致(n回以上m回以下) | [0-9]{1,5} | 連続する1つ～5つの数字。{n}や{,m}のような片方だけの指定も可能 |
| \ | 直前の特殊文字の打ち消し | \$ | '$' 1文字 |


他にもいくつかありますが、これくらい使えれば困ることはないです…  

2. 転送先  
nginxが定義している変数を利用できます。 $http_host とか $request_uri といった変数を利用することができます。  
一覧は以下に掲載されてます。  
http://mogile.web.fc2.com/nginx/http/ngx_http_core_module.html#variables  
また、正規表現中にかっこで区切った文字列を$1, $2 などで再利用することができます。  

3. フラグ
URLの書き換えを行った後の動作を指定するものです。  

| フラグ | 意味 |
|----|----|
| last | 書き換えたURLに対して、locationマッチからやり直す |
| break | 以降のrewriteの評価を行わず、書き換えたURLの内容を応答する |
| redirect | ステータスコード302 で転送する |
| permanent | ステータスコード301 で転送する |

last と break の使い分けが重要になります。
redirect_chain02 では、URLのクエリパラメータに"no-redirect"がある場合のみ、  
page01とpage02を表示できるように設定をしました。  

~~~nginx
    # no-redirectのパラメータがあるときは遷移させないパターン
    location /redirect_chain02 {
        rewrite ^(.*)$ http://$http_host/redirect_chain02/ last;
    }
    location /redirect_chain02/ {
        #no-redirect パラメータががついているとき、rewriteせずにリクエストされたページを応答する
        if ($args ~ "no-redirect(.*)") {
            rewrite (.*) $1 break; # breakなので、以降のrewrite ディレクティブの評価を行わない
        }

        rewrite page01\.html $scheme://$http_host/redirect_chain02/page02.html last; # page01 から page02 へ遷移させる
        rewrite page02\.html $scheme://$http_host/redirect_chain02/page03.html last; # page02 から page03 へ遷移させる
    }
~~~

`if` ディレクティブの中の `rewrite` ディレクティブは、フラグが `break` に設定されています。  
これにより、以降に記述されている `rewrite` ディレクティブの内容を評価せずにそのままレスポンスを返します。

# お疲れさまでした！
basic認証、SSL接続、`rewrite` ディレクティブによるリダイレクトまで、実際に設定ファイルを作成して動作を確認してきました。  
NginxのほうがApacheよりも設定がわかりやすいことと、意外と設定方法に関する日本語の情報がたくさん出てくることがわかりました。  
rewriteだけみても、Apacheの方はフラグの種類が多すぎて覚えきれないんですよね…違いも理解しづらいし…  
今後何かを作るときには、実際にNginxを利用してみたいと思います。