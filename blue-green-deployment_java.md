## Blue-Greenデプロイ

[Blue-Greenデプロイ](http://martinfowler.com/bliki/BlueGreenDeployment.html)はBlueとGreenとよばれる2つの環境を用意して、ルーティングを切り替えることによりダウンタイムリスクを低減させる手法です。

Cloud Foundryでは`cf map-route`、`umnap-route`コマンドによりルーティングの設定を行うことでBlue-Greenデプロイを実現できます。

はじめに作成した`hello-cf`アプリケーションを使ってBlue-Greenデプロイを試しましょう。
`hello-cf`のディレクトリの直下に以下のファイルを作成します。ファイル名は`manifest.yml`としてください。

``` yaml
---
applications:
  - name: hello-<STUDENT_ID>
    path: target/hello-cf-0.0.1-SNAPSHOT.jar
    buildpack: java_buildpack_offline
```

`HelloCfApplication.java`の`hello`メソッドを少しだけ修正してください。

``` java
    @GetMapping("/")
    String hello() {
        return "Hello World! V1"; // V1を追加
    }
```

これをビルドしてpushします。これはBlueに相当します。

``` console
$ ./mvnw package -Dmaven.test.skip=true
$ cf push
```

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/16d05b51-d04c-61a9-18e2-5ced07c21034.png)


現在のアプリケーションのバージョンを確認するために、`curl`を定期的に実行しましょう。

``` console
$ while true; do curl -s http://hello-<STUDENT_ID>.apps.pcflab.jp.io; echo; sleep 1;done
```

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/b47937f2-9d32-23fd-e386-de7018394372.png)

**ここまで完了したら進捗シートにチェックをしてください。**

`HelloCfApplication.java`の`hello`メソッドを少しだけ修正してください。

``` java
    @GetMapping("/")
    String hello() {
        return "Hello World! V2"; // V2に変更
    }
```

ビルドしてpushします。これはGreenに相当します。

``` console
$ ./mvnw package -Dmaven.test.skip=true
$ cf push hello-<STUDENT_ID>-green # manifest内のapplication nameをoverride
```

`cf apps`の結果は以下のようになります。

``` console
$ cf apps
Getting apps in org tmaki / space development as ****@gmail.com...
OK

name                requested state   instances   memory   disk   urls   
hello-tmaki         started           1/1         1G       1G     hello-tmaki.cfapps.io   
hello-tmaki-green   started           1/1         1G       1G     hello-tmaki-green.cfapps.io 
```

新バージョンにアクセスして、正常に動作していることを確認してください。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/6d89019e-59cd-4718-47c5-6115a97f29f3.png)

`cf map-route hello-<STUDENT_ID>-green <Domain> -n <Hostname>`で`hello-<STUDENT_ID>.apps.pcflab.jp`へのリクエストが`hello-<STUDENT_ID>-green`にルーティングされるようにします。

``` console
$ cf map-route hello-<STUDENT_ID>-green apps.pcflab.jp -n hello-<STUDENT_ID>
Creating route hello-tmaki.cfapps.io for org tmaki / space development as ****@gmail.com...
OK
Route hello-tmaki.cfapps.io already exists
Adding route hello-tmaki.cfapps.io to app hello-tmaki-green in org tmaki / space development as ****@gmail.com...
OK
```

2つのアプリケーションに対して`hello-<STUDENT_ID>.apps.pcflab.jp`でアクセスできるため、`curl`の結果は次のように`V1`と`V2`の両方が表示されるようになります。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/56c6deb5-1327-dd2f-f069-7cf0a0af6192.png)


`map-route`とは反対の`unmap-route`コマンドで`hello-<STUDENT_ID>`へのルーティングを外します。

``` console
$ cf unmap-route hello-<STUDENT_ID> apps.pcflab.jp -n hello-<STUDENT_ID>
Removing route hello-tmaki.cfapps.io from app hello-tmaki in org tmaki / space development as ****@gmail.com...
OK
```

これで`V2`のみが出力されるようになりました。


![image](https://qiita-image-store.s3.amazonaws.com/0/1852/d3924903-283e-c923-1e9d-d1e28c648964.png)


`V2`に問題がなければ、旧バージョンを削除し、`hello-<STUDENT_ID>-green.apps.pcflab.jp`のルーティングも削除します。

``` console
$ cf delete hello-<STUDENT_ID>
$ cf unmap-route hello-<STUDENT_ID>-green apps.pcflab.jp -n hello-<STUDENT_ID>-green
```

`hello-tmaki.apps.pcflab.jp`にアクセスし続けていましたが、404エラーなどが発生することなく`V1`から`V2`へ移行することができました。

``` console
$ cf apps
Getting apps in org tmaki / space development as ****@gmail.com...
OK

name                requested state   instances   memory   disk   urls   
hello-tmaki-green   started           1/1         1G       1G     hello-tmaki.cfapps.io
```

アプリケーション名を`hello-tmaki`に戻しましょう。

``` console
$ cf rename hello-<STUDENT_ID>-green hello-<STUDENT_ID>
```

これで元の通りです。

``` console
$ cf apps
Getting apps in org tmaki / space development as ****@gmail.com...
OK

name          requested state   instances   memory   disk   urls   
hello-tmaki   started           1/1         1G       1G     hello-tmaki.cfapps.io
```


`cf delete hello-<STUDENT_ID>`する前に、もし新バージョン(green)で問題が発覚すれば、旧バージョン(blue)に切り戻しすればよいです。

``` console
$ cf map-route hello-<STUDENT_ID> apps.pcflab.jp -n hello-<STUDENT_ID>
$ cf unmap-route hello-<STUDENT_ID>-green apps.pcflab.jp -n hello-<STUDENT_ID>
```

を行えば`V1`に戻ります。

**ここまで完了したら進捗シートにチェックをしてください。**

### Autopilotプラグインを使う

[Autopilotプラグイン](https://github.com/contraband/autopilot)を使うと、上記の同様な操作をコマンド一つで実行できます。

プラグインのインストール方法は、[Releaseページ](https://github.com/contraband/autopilot/releases)からOSにあったバイナリをダウンロードして、



``` console
$ chmod +x path/to/downloaded/binary # (Linux/Macの場合)
$ cf install-plugin path/to/downloaded/binary
```

です。

`zero-downtime-push`コマンドが追加されます。

``` console
$ cf help -a | grep zero
   zero-downtime-push                     Perform a zero-downtime push of an application over the top of an old one
```

``` java
    @GetMapping("/")
    String hello() {
        return "Hello World! V3"; // V3に変更
    }
```

コンパイルして、`cf zero-downtime-push`しましょう。

``` console
$ ./mvnw clean package
$ cf zero-downtime-push hello-<STUDENT_ID> -f manifest.yml 
```

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/be7420f4-4497-48ed-212a-3b14be4f8de2.png)

しばらくすると、V3だけになります。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/91a44ead-877c-9296-35fd-9fc0ab6dd05a.png)

アプリケーションを止めずにバージョンアップを実現できました。
この場合は旧バージョンのアプリケーションが削除されてしまう点に注意する必要があります。

実際には最初に示した方法とは少し異なり、

1. 旧バージョンのアプリケーション名を<App>-venerableにrename
2. 新バージョンのアプリケーションを<App>でpush。この段階で両方のアプリケーションにリクエストがルーティングされる
3. 旧バージョンのアプリケーションを削除

という手順を取っています。

**ここまで完了したら進捗シートにチェックをしてください。**

### Scaleoverプラグインを使う

Autopilotプラグインの方法は旧バージョンと同じ数のインスタンスを新バージョンにも作り、一度に切り替えるため同時に使用するリソースが多くなってしまいます。

1. `V1`:10 - `V2`:0
1. `V1`:7 - `V2`:3
1. `V1`:5 - `V2`:5
1. `V1`:3 - `V2`:7
1. `V1`:0 - `V2`:10

というように少しずつ新バージョンのインスタンス数を増やしたい場合には、
[Scaleoverプラグイン](https://github.com/krujos/scaleover-plugin)が利用できます。

実際に10インスタンスのアプリケーションを新バージョンに移行してみましょう。ここではメモリ消費量の小さいStaticfile Buildpackを使って静的サイトをデプロイします。

``` console
$ mkdir -p try-scaleover
$ echo V1 > index.html
$ touch Staticfile
$ cf push try-scale-over-v1 -b staticfile_buildpack -m 32m
$ cf scale -i 10 try-scale-over-v1
$ cf map-route try-scale-over-v1 apps.pcflab.jp -n try-scale-over
```

次に新バージョンを作成しデプロイします。

``` console
$ echo V2 > index.html
$ cf push try-scale-over-v2 -b staticfile_buildpack -m 32m
```

新バージョンが動作することを確認したら一旦ストップし、本番環境へのルーティングを設定します。

```
$ cf stop try-scale-over-v2
$ cf map-route try-scale-over-v2 apps.pcflab.jp -n try-scale-over
```

この段階では`V1`:10 - `V2`:0になっています。

``` console
$ cf apps
Getting apps in org tmaki / space development as ****@gmail.com...
OK

name                requested state   instances   memory   disk   urls   
try-scale-over-v1   started           10/10       32M      1G     try-scale-over-v1.cfapps.io, try-scale-over.cfapps.io   
try-scale-over-v2   stopped           0/1         32M      1G     try-scale-over.cfapps.io, try-scale-over-v2.cfapps.io
```

現在のアプリケーションのバージョンを確認するために、`curl`を定期的に実行しましょう。

``` console
$ while true; do curl http://try-scale-over.apps.pcflab.jp; sleep 1;done
```

いよいよバージョンアップします。
`cf scaleover <Old> <New> <Period>`を実行して、少しずつ新バージョンに移行する様子をみてください。

``` console
$ cf scaleover try-scale-over-v1 try-scale-over-v2 10s
try-scale-over-v1 (started) <<<<<<<<<<  try-scale-over-v2 (stopped)
try-scale-over-v1 (started) <<<<<< >>>> try-scale-over-v2 (started)
try-scale-over-v1 (started) <<< >>>>>>> try-scale-over-v2 (started)
try-scale-over-v1 (stopped)  >>>>>>>>>> try-scale-over-v2 (started)
```

`curl`の結果が`V1`から`V2`へと変わっていくのを確認できるでしょう。

* **このプラグインはCLIの実行結果文字列をパースしているため、必ず英語環境で実行してください。CLIが日本語表示の場合は`LANG=C cf scaleover ...`としてください。**
* `CF CLI v6.17.0`以降では動作しないことを確認。`v6.16.1`では動作します。(2016-05-13)
* `v6.19.1`で修正されました (2016-06-14)
