# nginx-training
Nginxの設定ファイルの内容について、いろいろ試してみた記録を残すリポジトリです。

## training001(2020.11.01) 
Docker HubにあるNginxの公式イメージ(nginx-apline)を利用して、いろいろな設定を試してみました。  
なお、Nginxでのマルチドメイン対応をしてみたかったので、  
hostsファイルに以下の内容を追記した環境で演習を行いました。  
~~~bash
127.0.0.1 nginx-training.default nginx-training-01.local nginx-training-02.local nginx-training-03.local
~~~

