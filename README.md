# nginx-training
Nginxの設定ファイルの内容について、いろいろ試してみた記録を残すリポジトリです。

## training001(2020.11.01) 
Docker HubにあるNginxの公式イメージ(nginx-apline)を利用して、いろいろな設定を試してみました。  
なお、Nginxでのマルチドメイン対応をしてみたかったので、  
hostsファイルに以下の内容を追記した環境で演習を行いました。  
~~~bash
127.0.0.1 nginx-training.default nginx-training-01.local nginx-training-02.local nginx-training-03.local
~~~  

## training002(2020.11.04) 
training001の設定内容を引き継いで、basic認証の設定、SSL認証対応、`rewrite` ディレクティブによる実践的なリダイレクト設定までを行ってみました。  
また、traninig001では触れなかった`nginx.conf` の設定内容や個別のディレクティブの使いかたも確認しました。