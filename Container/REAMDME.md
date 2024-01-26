# プライベートWebサーバを使用しないバージョンのコンテナ版IRIS利用時のWebサーバ設定例

コミュニティエディションを除くInterSystems製品のバージョン2023.2以降では、プライベートWebサーバを使用した管理ポータル／Webアクセスを非推奨に変更しました。

> [開発者コミュニティ](https://jp.community.intersystems.com)の記事：[Apache Webサーバ(プライベートWebサーバ: PWS)インストレーションの廃止](https://jp.community.intersystems.com/node/532436)も併せてご参照ください。

バージョン2023.2以降のコンテナ版IRISを利用する場合、以下いずれかの方法で管理ポータルを含めたWebアクセスを行うための設定が必要です。

- 任意の場所にWebサーバを用意する
- Webゲートウェイ用コンテナを利用する

このリポジトリでは、【Webゲートウェイ用コンテナを利用する】方法についてご紹介します。

【任意の場所にWebサーバを用意する】場合に必要となる情報については、下記ドキュメントをご参照ください。

- Webゲートウェイのインストールについては、ドキュメント：[Web ゲートウェイのインストール](https://docs.intersystems.com/irislatestj/csp/docbook/DocBook.UI.Page.cls?KEY=GCGI_install)をご参照ください。
- IRISへの接続設定については、ドキュメント：[サーバ・アクセスの構成](https://docs.intersystems.com/irislatestj/csp/docbook/DocBook.UI.Page.cls?KEY=GCGI_config_serv)、[アプリケーション・アクセスの構成](https://docs.intersystems.com/irislatestj/csp/docbook/DocBook.UI.Page.cls?KEY=GCGI_config_app)をご参照ください。


---
## Webゲートウェイコンテナを利用したWebアクセス例

- [事前準備](#事前準備)

    このサンプルを使用して動作させるために必要な準備を説明します。

- [開始するコンテナ](#開始するコンテナ)

    開始する2つのコンテナがどのように接続しているか説明します。

    - [IRISのコンテナ](#irisのコンテナ)

        オリジナルのコンテナイメージの作成の流れを説明します。

    - [Webゲートウェイのコンテナ](#webゲートウェイのコンテナ)

        Webゲートウェイコンテナに設定を追加するために使用している２つのファイルについて説明します。

- [コンテナ開始手順](#コンテナの開始手順)

    [docker-compose.yml](/Container/docker-compose.yml)を使ったコンテナビルド、開始、停止、削除の手順を説明します。

### 事前準備

[InterSystemsコンテナレジストリ](https://containers.intersystems.com/contents)から、必要なイメージをPull、またはイメージ名：タグを確認します。
- コンテナ版Webゲートウェイ用イメージ
- コンテナ版IRISまたはIRIS for Health用イメージ

>コンテナレジストリの使い方については、「[InterSystemsコンテナレジストリの使い方とコンテナ開始までの流れ（解説ビデオ付き）](https://jp.community.intersystems.com/node/545786)」（ビデオ　～2:44まで）をご参照ください。

> ※ コンテナ版Webゲートウェイ用イメージをpullする際、コンテナレジストリへのログインは不要です。

また、使用するIRISまたはIRIS for Healthのライセンスキーを準備します。

この例の中では、ライセンスキーを [iris](/Container/iris) ディレクトリに iris.key の名称で配置してください。

使用するコンテナのイメージは以下の通りです。

- webgateway2023.3のイメージ

    `containers.intersystems.com/intersystems/webgateway:2023.3`

- IRIS 2023.3のイメージ

    `containers.intersystems.com/intersystems/iris:2023.3`



### 開始するコンテナ

IRISコンテナとWebゲートウェイコンテナを同一のdockerネットワークに参加させ、開始するようにします。（説明では、[docker-compose.yml](/Container/docker-compose.yml)を使用してコンテナを開始します）

![](/assets/IRIS-WG.png)

[docker-compose.yml](/Container/docker-compose.yml)のIRISに関する記述は以下の通りです。

```
services:
  iris:
    #image: containers.intersystems.com/intersystems/iris:2023.3
    build:
      context: ./iris
      dockerfile: Dockerfile
    init: true
    container_name: nopwsiris
    hostname: nopwsiris
    ports:
       - "8082:1972"
    environment:
      - TZ=JST-9
    volumes:
      - ./iris:/ISC
    command: --key /ISC/iris.key
```

IRISのコンテナ名を **nopwsiris** としています。WebゲートウェイのコンテナからIRISにアクセスする際、このコンテナ名を使用してアクセスできます。


以下、IRISとWebゲートウェイのコンテナの中身について説明します。


#### IRISのコンテナ

コンテナ版IRISのデフォルト設定を少し変更したいため、[Dockerfile](/Container/iris/Dockerfile)を使用しています。

Dockerfileの中では、コンテナの元となるイメージの指定と、変更内容のメソッド実行を記述したファイル：[iris.script](/Container/iris/iris.script)を使用した処理の実行が記載されています。

コンテナビルド中に実行されるメソッドの内容は以下の通りです。

- 「次回ログイン時にパスワード変更」のオプションを無効に変更

    WebゲートウェイコンテナからIRISコンテナにアクセスする際に使用する、IRISの事前定義ユーザ：**CSPSystems** のパスワードを初回アクセス時に変更しなくていいように、IRIS内ユーザに対する「次回ログイン時にパスワード変更」のオプションを無効にします。（初期設定としてパスワード「SYS」が設定されています。）

    ```
    set $namespace="%SYS"
    // 事前定義ユーザのパスワードを無期限に設定する（デフォルトパスワードはSYS）
    Do ##class(Security.Users).UnExpireUserPasswords("*")
    ```

- 日本語ロケールへの変更

    ```
    // 日本語ロケールに変更（コンテナがUbuntu英語版のためデフォルトは英語ロケール）を利用
    Do ##class(Config.NLS.Locales).Install("jpuw")
    ```


続いて、Webゲートウェイのコンテナについて説明します。

#### Webゲートウェイのコンテナ

設定に使うファイルは、[webgateway](/Container/webgateway/)以下にあります。

```
~/NoPWS/Container/webgateway$ tree
.
├── CSP.conf
└── CSP.ini
```
[docker-compose.yml](/Container/docker-compose.yml)のWebゲートウェイに関する記述は以下の通りです。

```
  webgw:
    image: containers.intersystems.com/intersystems/webgateway:2023.3
    container_name: webgw1
    init: true
    ports:
      - 8080:80
      - 8443:443
    environment:
    - ISC_CSP_CONF_FILE=/webgateway-shared/CSP.conf
    - ISC_CSP_INI_FILE=/webgateway-shared/CSP.ini
    volumes:
    - ./webgateway:/webgateway-shared
```

`environment`に設定されている環境変数は、Webゲートウェイの構成に使用するファイル名を指定できる予め決められた名称です。

- ISC_CSP_CONF_FILE

    Apacheのconfファイルに追加したい設定を記述できます。例：[CSP.conf](/Container/webgateway/CSP.conf)

    例では、Webサーバに / のパスが渡された場合全てのURLをWebゲートウェイに渡し、IRISに渡す設定を記述しています。

    詳しはドキュメントもご参照ください：「[Location による Apache の構成](https://docs.intersystems.com/irislatestj/csp/docbook/DocBook.UI.Page.cls?KEY=GCGI_apacheconfig_filetypes#GCGI_apacheconfig_filetypes_location)」

    - 以下ご参考
        証明書ファイルの準備があればhttpsでの通信に変更することもできます。

        コメント化されている行のコメントを外し、証明書ファイル類を server.*の名称で [webgateway](/Container/webgateway/)以下に配置すればコンテナ開始時に設定します。

        ```
        # SSL SECTION #
        # Enable SSL/TLS (https://) on the Apache web server.
        # The user is responsible for providing valid SSL certificates.
        #LoadModule ssl_module /usr/lib/apache2/modules/mod_ssl.so
        #<VirtualHost *:443>
        #SSLEngine on
        #SSLCertificateFile "/webgateway-shared/server.crt"
        #SSLCertificateKeyFile "/webgateway-shared/server.key"
        #</VirtualHost>
        ```
    
- ISC_CSP_INI_FILE

    Webゲートウェイ管理画面で設定する[サーバ・アクセスの構成](https://docs.intersystems.com/irislatestj/csp/docbook/DocBook.UI.Page.cls?KEY=GCGI_config_serv)と、[アプリケーション・アクセスの構成](https://docs.intersystems.com/irislatestj/csp/docbook/DocBook.UI.Page.cls?KEY=GCGI_config_app)、[追加クライアント・アドレスのアクセスの有効化](https://docs.intersystems.com/irislatestj/csp/docbook/DocBook.UI.Page.cls?KEY=GCGI_mgmtportal) を指定したiniファイルを指定します。

    例：[CSP.ini](/Container/webgateway/CSP.ini)

    このファイルを使用せず、コンテナを開始してからWebゲートウェイ管理画面で手動で作成することもできます。

    管理画面は以下URLで開きます。

    http://localhost:8080/csp/bin/systems/module.cxw

    > ポート番号はWebゲートウェイコンテナがホストに割り当てている番号を指定してください。）


### コンテナの開始手順

1. オリジナルの設定を加えたオリジナルのIRISイメージをビルドします。

    ```
    docker-compose build
    ```
2. コンテナを開始します。

    ```
    docker-compose up -d
    ```

    開始できたら、以下URLにアクセスし、管理ポータルが起動するかご確認ください。

    http://localhost:8080/csp/sys/UtilHome.csp


3. コンテナを停止・削除する方法
    
    停止
    ```
    docker-compose stop
    ```
    再開始
    ```
    docker-compose start
    ```

    削除 
    ```
    docker-compose down
    ```








