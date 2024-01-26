# プライベートWebサーバを使用しないバージョンの新規インストール／アップグレードについて

コミュニティエディションを除くInterSystems製品のバージョン2023.2以降では、プライベートWebサーバを使用した管理ポータル／Webアクセスを非推奨に変更しました。

> [開発者コミュニティ](https://jp.community.intersystems.com)の記事：[Apache Webサーバ(プライベートWebサーバ: PWS)インストレーションの廃止](https://jp.community.intersystems.com/node/532436)も併せてご参照ください。


- [概要](#概要)
    - [Webサーバ自動設定を選択した場合](#webサーバ自動設定を選択した場合)
    - [VSCodeからアクセスするときの注意点](#vscodeからアクセスするときの注意点)
- [IISをWebサーバとする場合](#iisをwebサーバとする場合)
- [ApacheをWebサーバとする場合](#apacheをwebサーバとする場合)

※コンテナ版IRISの利用例については、[Container/REAMDME.md](/Container/REAMDME.md)をご参照ください。

## 概要
新規インストール／アップグレードインストールによるプライベートWebサーバの利用可否やインストール時の選択項目の違いについての概要は以下表をご参照ください。

![](/assets//NoPWS-NewInstVSUpgrade.png)

最新情報は下記英語ドキュメントに記載されます。正確な情報についてはドキュメントもご参照ください。
- [InterSystems IRIS for Health](https://docs.intersystems.com/irisforhealthlatest/csp/docbook/DocBook.UI.Page.cls?KEY=GCGI_private_web)
- [InterSystems IRIS](https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GCGI_private_web)

インストール方法は全製品で共通です。例ではInterSystems IRIS（以降IRIS）での例を記述します。

### Webサーバ自動設定を選択した場合

Webサーバを事前にインストールした状態で新規インストール／アップグレードインストールを行うと、Webサーバに対する必要な設定を自動的に行うか選択できます。

Linux系：インストーラーから以下のメッセージが表示されます。

```
Local web server detected. Would you like to use the web server to connect to this installation <Yes>? 
```
IIS：以下画面が表示されます。

![](/assets/IIS-installer.png)

この設定でインストールを行うと、Webサーバに対してインスタンス名（**小文字で指定**）のパスを自動的に設定します。

インスタンス名が**IRISTEST**である場合、以下のパスでアクセスできます。

**http://Webサーバ名/iristest**

REST 用のウェブアプリケーションパスや管理ポータルはこのURLの後ろにパスを続けて指定することでアクセスできます。

例）管理ポータル：**http://Webサーバ名/iristest/csp/sys/UtilHome.csp**


### VSCodeからアクセスするときの注意点

#### 1. pathPrefixの指定

Webサーバをインストール時の自動設定で構成した場合、Web経由のアクセスに使用するURLは「http://Webサーバ名/インスタンス名」　から始まるURLになります。

VSCodeからIRISにアクセスする際に指定する接続文字列にもインスタンス名を含めたパスを指定する必要があるため、**"pathPrefix"** を使用してパス名を指定します。（インスタンス名の先頭に / の指定が必要です）


例）インスタンス名：IRISの場合
```
{
    "objectscript.conn": {
        "active": true,
        "server": "iristest",
        "ns": "USER"
    },
    "intersystems.servers": {
        "iristest":{
            "webServer": {
                "host": "localhost",
                "scheme": "http",
                "pathPrefix": "/iris",
                "port":80
            },
            "username": "SuperUser"
        }
    }
}
```

### 2. デバッグ時の注意点

VSCodeからObjectScriptエクステンションを使用してデバッグを行う場合、WebサーバのWebソケットモジュールを有効にする必要があります。

有効化しないと以下のようなエラーメッセージを出力します。

![](/assets/VSCode-Debug-Error.png)

#### IISの場合

**「WebSocketプロトコル」** を有効にします。

![](/assets/IIS-WebSocketEnabled.png)

詳細な手続きは下図の通りです（Windowsサーバの例）。

[コントロールパネル] > [プログラム] > [プログラムと機能] > [Windowsの機能の有効化または無効化] の選択、または「サーバーマネージャ」を起動します。

[サーバの役割] から [Webサーバ(IIS)] をチェックし [機能の追加] ボタンをクリックします。（クリック後、[Webサーバ(IIS)]にチェックが付きます）

![](/assets/IIS-add-wizard.png)

[アプリケーション開発] > [Web Socket プロトコル] をチェックします。また、[HTTP共有機能] > [HTTPリダイレクト]も有効にします。

![](/assets/IIS-add-module.png)

オプション全体の設定は、以下の通りです。

![](/assets/IIS-option.png)

> Windows10上のIISの設定は下図の通りです。
![](/assets/Win10-WebSocketEnabled.png)


#### Apacheの場合

WebSocketモジュールを有効化し、Apacheを再起動します。

```
sudo a2enmod proxy_wstunnel
sudo systemctl restart apache2
```
---

## IISをWebサーバとする場合

同一サーバ上にIISとInterSystems製品をインストールする場合、事前にIISを有効化しておくとIISに必要なWebゲートウェイのインストールと設定をIRISのインストーラーが自動で行います。

以降の説明では、以下のインストール方法について解説します。

- [1. IISを事前に準備しない状態での新規インストール](#1-iisを事前に準備しない状態での新規インストール)

- [2. IISを事前に準備しない状態でのアップグレード](#2-iisを事前に準備しない状態でのアップグレード)

- [3. IISをインストールした後の新規／アップグレードインストール](#3-iisをインストールした後の新規アップグレードインストール)

- [4. IISの設定](#4iisの設定)

### 1. IISを事前に準備しない状態での新規インストール

インストーラーを起動し、新規インストールではセットアップタイプを選択した後、アップグレード時は対象インスタンスを選択した後、ローカルにWebサーバ（IIS）を検出できないため、以下の選択肢を表示します。

1. About the installtion

    インストールを中止する
2. Continue the instllation without Web Gateway

    Webゲートウェイのインストール無しでインストールを進める（インストール後Webアクセスが行えません）

【新規インストール時のインストーラーの表示】

![](/assets/NewInstallWithoutIIS.png)


【アップグレードインストール時のインストーラーの表示】

![](/assets/UpgradeWithoutIIS.png)


#### 2. 新規インストールで「Continue the instllation without Web Gateway」 を選択した場合

Webゲートウェイをインストールしないでインストールを継続するオプションとなるため、**Webアクセスが行えません（＝管理ポータルを開く方法がありません）。**

- **バージョン2023.2以降、新規インストール時にプライベートWebサーバはインストールされません。** そのため、このオプションを選択した場合、Webサーバがローカルの構成に存在しないためWebアクセスに必要な設定（Webゲートウェイのインストールと設定）が行われないため、Webアクセスが行えません。

Webアクセスを行うためには、IISなどのWebサーバをインストールし、別途Webサーバに対してWebゲートウェイのインストールとIRISへの接続設定が必要となります。

詳細はドキュメント：[Web ゲートウェイのみのインストール](https://docs.intersystems.com/irislatestj/csp/docbook/DocBook.UI.Page.cls?KEY=GCI_windows#GCI_windows_web)をご参照ください。


### 2. IISを事前に準備しない状態でのアップグレード

[1. IISを事前に準備しない状態での新規インストール](#1-iisを事前に準備しない状態での新規インストール)と同様に、インストールを中止するか、Webゲートウェイのインストール無しでインストールを進めるか、選択肢が表示されます。

2023.1以前のバージョンからのアップグレードする場合は、プライベートWebサーバがインストールされているため、アップグレード前と同じ方法でWebアクセスが利用できます。

管理ポータルのアドレス：　http://localhost:52773/csp/sys/UtilHome.csp

IRISのランチャーからアクセスする場合は、インスタンス名（例ではiris）のパスが付いたアドレスで管理ポータルが開きます。

例）http://localhost:52773/iris/csp/sys/UtilHome.csp

アップグレード前と同じURLでランチャーからアクセスしたい場合は、接続編集画面の「CSPサーバインスタンス」に追記されているインスタンス名を消してください。

![](/assets/ConnectionSettings-Launcher.png)


### 3. IISをインストールした後の新規／アップグレードインストール

事前にIISをインストールした場合、インストーラーがWebサーバを検出し、以下の選択肢を表示します。

1. Configure local IIS web server for thie instance

    インストール中インスタンス用にローカルのIIS Webサーバ構成します。

2. Do not configure local IIS web server for this instance

    インストール中インスタンス用にローカルのIIS Webサーバを構成しません。

![](/assets/Installer-WithIIS.png)



#### 1. Configure local IIS web server for thie instance を選択した場合 

IISを経由した管理ポータルのアクセスが行えます。

URLは以下のルールで指定します。

**http://Webサーバ/インスタンス名/csp/sys/UtilHome.csp**

例）http://localhost/iris/csp/sys/UtilHome.csp

> IRISのランチャーからもこのアドレスで管理ポータルが開きます。


#### 2. アップグレード時に「Do not configure local IIS web server for this instance」を選択した場合

アップグレード前に使用していたプライベートWebサーバが利用できます。

Webアクセス方法について詳細は、[2. IISを事前に準備しない状態でのアップグレード](#2-iisを事前に準備しない状態でのアップグレード) の記述をご参照ください。

#### 2. 新規インストール時に「Do not configure local IIS web server for this instance」を選択した場合

Webゲートウェイをインストールしないでインストールを継続するオプションとなるため、**Webアクセスが行えません（＝管理ポータルを開く方法がありません）。**

IISを事前準備しないときの新規インストールと同様です。詳しくは[こちらの記述](#2-新規インストールでcontinue-the-instllation-without-web-gateway-を選択した場合)をご参照ください。




### 4.IISの設定

IISをWebサーバとして利用し、VSCodeでデバッグを行う場合、IISのWebSocketサービスを有効化する必要があります。

詳細は、[デバッグ時の注意点](#2-デバッグ時の注意点)をご参照ください。


## ApacheをWebサーバとする場合

同一サーバ上にApacheとInterSystems製品をインストールする場合、事前にApacheをインストールしておくとApacheに必要なWebゲートウェイのインストールとIRISへの接続設定をインストーラーが自動で行います。

以降の説明では、以下のインストール方法について解説します。

- [1. Apacheを事前に準備しない状態での新規インストール](#1-apacheを事前に準備しない状態での新規インストール)

- [2. Apacheを事前に準備しない状態でのアップグレード](#2-apacheを事前に準備しない状態でのアップグレード)

- [3. Apacheをインストールした後の新規／アップグレードインストール](#3-apacheをインストールした後の新規アップグレードインストール)

- [4. Apacheの設定](#4apacheの設定)

### 1. Apacheを事前に準備しない状態での新規インストール

ローカルにWebサーバ（Apache）を検出できないため、以下の選択肢を表示します。

**No local web server found. Would you like to abort the installation \<Yes\>?**

実行時の表示は以下の通りです（Ubuntuを利用しています）。

```
$ sudo ./irisinstall
 
Your system type is 'Ubuntu 22.04 LTS (x64)'.

Enter instance name <IRIS>: IRIS

Enter a destination directory for the new instance.
Directory: /usr/iris
Directory '/usr/iris' does not exist.
Do you want to create it <Yes>?

Select installation type.
    1) Development - Install InterSystems IRIS server and all language bindings
    2) Server only - Install InterSystems IRIS server
    3) Custom
Setup type <1>? 1

How restrictive do you want the initial Security settings to be?
"Minimal" is the least restrictive, "Locked Down" is the most secure.
    1) Minimal
    2) Normal
    3) Locked Down
Initial Security settings <1>? 2

What user should be the owner of this instance? root
An InterSystems IRIS account will also be created for user root.

Install will create the following InterSystems IRIS accounts for you:
_SYSTEM, Admin, SuperUser, root and CSPSystem.
Please enter the common password for _SYSTEM, Admin, SuperUser and root:
Re-enter the password to confirm it:

Please enter the password for CSPSystem:
Re-enter the password to confirm it:

What group should be allowed to start and stop
  this instance? root

Do you want to install IRIS Unicode support <Yes>?

No local web server found. Would you like to abort the installation <Yes>?
```

**Yes**と回答するとインストールを中止します。
```
No local web server found. Would you like to abort the installation <Yes>?`
 
** Installation aborted **
 
 
No packages will be installed.
```

**No**と回答した場合、Webゲートウェイをインストールしないでインストールを継続するオプションとなるため、**Webアクセスが行えません（＝管理ポータルを開く方法がありません）。**


- **バージョン2023.2以降、新規インストール時にプライベートWebサーバはインストールされません。** そのため、このオプションを選択した場合、Webサーバがローカルの構成に存在しないためWebアクセスに必要な設定（Webゲートウェイのインストールと設定）が行われないため、Webアクセスが行えません。


Webアクセスを行うためには、ApacheなどのWebサーバをインストールし、別途Webサーバに対してWebゲートウェイのインストールとIRISへの接続設定が必要となります。

詳細はドキュメント：[Web ゲートウェイのインストール](https://docs.intersystems.com/irislatestj/csp/docbook/DocBook.UI.Page.cls?KEY=GCGI_install)をご参照ください。

実際の表示は以下の通りです。
```
No local web server found. Would you like to abort the installation <Yes>?  no

InterSystems IRIS did not detect a license key file

Do you want to enter a license key <No>?

Please review the installation options:
------------------------------------------------------------------
Instance name: IRIS
Destination directory: /usr/iris
InterSystems IRIS version to install: 2023.3.0.254.0
Installation type: Development
Unicode support: Y
Initial Security settings: Normal
User who owns instance: root
Group allowed to start and stop instance: root
Effective group for InterSystems IRIS processes: irisusr
Effective user for InterSystems IRIS SuperServer: irisusr
SuperServer port: 1972
JDBC Gateway port: 53773
Web Gateway: not using local web server
------------------------------------------------------------------

Confirm InterSystems IRIS installation <Yes>?

Starting installation
Starting up InterSystems IRIS for loading...
../bin/irisinstall -s . -B -c c -C /usr/iris/iris.cpf*IRIS -W 1 -g2
Starting Control Process
Allocated 498MB shared memory
32MB global buffers, 80MB routine buffers
Creating a WIJ file to hold 32 megabytes of data
IRIS startup successful.
System locale setting is 'C.UTF-8'
This copy of InterSystems IRIS has been licensed for use exclusively by:
No local key detected, trying license server.
Copyright (c) 1986-2023 by InterSystems Corporation
Any other use is a violation of your license agreement

^^/usr/iris/mgr/>

^^/usr/iris/mgr/>
Start of IRIS initialization
Loading system routines
Updating system TEMP and LOCALDATA databases
Loading system classes
Installing National Language support

Setting IRISTEMP default collation to IRIS standard (5)
Updating Security database
Loading system source code
Building system indices
Updating Audit database
Updating Journal directory
Updating User database
Updating Interoperability databases
Scheduling inventory scan
IRIS initialization complete
259 lines written to /usr/iris/mgr/filecheck.isc
See the 'iboot.log' file for a record of the installation.

Starting IRIS
Using 'iris.cpf' configuration file

Starting Control Process
Global buffer setting requires attention.  Auto-selected 25% of total memory.
Allocated 1529MB shared memory
976MB global buffers, 97MB routine buffers
Creating a WIJ file to hold 99 megabytes of data
This copy of InterSystems IRIS has been licensed for use exclusively by:
No local key detected, trying license server.
Copyright (c) 1986-2023 by InterSystems Corporation
Any other use is a violation of your license agreement

1 alert(s) during startup. See messages.log for details.

Installation completed successfully
$
```


### 2. Apacheを事前に準備しない状態でのアップグレード

インストール時、「[1. Apacheを事前に準備しない状態での新規インストール](#1-apacheを事前に準備しない状態での新規インストール)」と同様に、

**No local web server found. Would you like to abort the installation \<Yes\>?**

と表示されます。

**No** を選択するとアップグレードが継続されます。

ローカルにWebサーバがない状況ですが、**アップグレード前から使用しているプライベートWebサーバを使用して、アップグレード前と同じ方法で管理ポータルやWebアクセスが行えます。**

管理ポータルのアドレス：　http://localhost:52773/csp/sys/UtilHome.csp

また、インスタンス名（例ではiris）のパスが付いたアドレスも用意されるため、以下のURLでも管理ポータルにアクセスできます。

例）http://localhost:52773/iris/csp/sys/UtilHome.csp


### 3. Apacheをインストールした後の新規／アップグレードインストール

事前にApacheをインストールした場合、インストーラーがWebサーバを検出し、以下の選択肢を表示します。

**Local web server detected. Would you like to use the web server to connect to this installation \<Yes\>?**


```
$ sudo ./irisinstall

Your system type is 'Ubuntu 22.04 LTS (x64)'.


Currently defined instances:


Enter instance name: IRIS
Do you want to create InterSystems IRIS instance 'IRIS' <Yes>?

Enter a destination directory for the new instance.
Directory: /usr/iris
Directory '/usr/iris' does not exist.
Do you want to create it <Yes>?

------------------------------------------------------------------
NOTE: Users should not attempt to access InterSystems IRIS while
      the installation is in progress.
------------------------------------------------------------------

Select installation type.
    1) Development - Install InterSystems IRIS server and all language bindings
    2) Server only - Install InterSystems IRIS server
    3) Custom
Setup type <1>? 1

How restrictive do you want the initial Security settings to be?
"Minimal" is the least restrictive, "Locked Down" is the most secure.
    1) Minimal
    2) Normal
    3) Locked Down
Initial Security settings <1>? 2

What user should be the owner of this instance? root
An InterSystems IRIS account will also be created for user root.

Install will create the following InterSystems IRIS accounts for you:
_SYSTEM, Admin, SuperUser, root and CSPSystem.
Please enter the common password for _SYSTEM, Admin, SuperUser and root:
Re-enter the password to confirm it:

Please enter the password for CSPSystem:
Re-enter the password to confirm it:

What group should be allowed to start and stop
  this instance? root

Do you want to install IRIS Unicode support <Yes>?

Local web server detected. Would you like to use the web server to connect to this installation <Yes>?
```
#### 3-1. 「Yes」を選択した場合
Webサーバに必要な設定を自動的に行います。
```
Local web server detected. Would you like to use the web server to connect to this installation <Yes>?

InterSystems IRIS did not detect a license key file

Do you want to enter a license key <No>?

Please review the installation options:
------------------------------------------------------------------
Instance name: IRIS
Destination directory: /usr/iris
InterSystems IRIS version to install: 2023.3.0.254.0
Installation type: Development
Unicode support: Y
Initial Security settings: Normal
User who owns instance: root
Group allowed to start and stop instance: root
Effective group for InterSystems IRIS processes: irisusr
Effective user for InterSystems IRIS SuperServer: irisusr
SuperServer port: 1972
WebServer port: 80
JDBC Gateway port: 53773
Web Gateway: installed into /opt/webgateway
Apache web server will be configured for Web Gateway
Apache web server configuration file: /etc/apache2/apache2.conf
------------------------------------------------------------------

Confirm InterSystems IRIS installation <Yes>?

Starting installation
    Updating Apache configuration file ...
    - /etc/apache2/apache2.conf

Starting up InterSystems IRIS for loading...
../bin/irisinstall -s . -B -c c -C /usr/iris/iris.cpf*IRIS -W 1 -g2
Starting Control Process
Allocated 498MB shared memory
32MB global buffers, 80MB routine buffers
Creating a WIJ file to hold 32 megabytes of data
IRIS startup successful.
System locale setting is 'C.UTF-8'
This copy of InterSystems IRIS has been licensed for use exclusively by:
No local key detected, trying license server.
Copyright (c) 1986-2023 by InterSystems Corporation
Any other use is a violation of your license agreement

^^/usr/iris/mgr/>

^^/usr/iris/mgr/>
Start of IRIS initialization
Loading system routines
Updating system TEMP and LOCALDATA databases
Loading system classes
Installing National Language support

Setting IRISTEMP default collation to IRIS standard (5)
Updating Security database
Loading system source code
Building system indices
Updating Audit database
Updating Journal directory
Updating User database
Updating Interoperability databases
Scheduling inventory scan
IRIS initialization complete
260 lines written to /usr/iris/mgr/filecheck.isc
See the 'iboot.log' file for a record of the installation.

Starting IRIS
Using 'iris.cpf' configuration file

Starting Control Process
Global buffer setting requires attention.  Auto-selected 25% of total memory.
Allocated 1529MB shared memory
976MB global buffers, 97MB routine buffers
Creating a WIJ file to hold 99 megabytes of data
This copy of InterSystems IRIS has been licensed for use exclusively by:
No local key detected, trying license server.
Copyright (c) 1986-2023 by InterSystems Corporation
Any other use is a violation of your license agreement

1 alert(s) during startup. See messages.log for details.

You can now access InterSystems IRIS, to access the management portal point your browser to:
http://localhost/iris/csp/sys/UtilHome.csp

Installation completed successfully
$
```

管理ポータルに「http://Webサーバ/インスタンス名/csp/sys/UtilHome.csp」 でアクセスできることが最後に案内されます。

（アップグレードした場合は、アップグレード前まで利用していたプライベートWebサーバは無効化されます。）

#### 3-2. アップグレード時「No」を選択した場合

**プライベートWebサーバを使用した環境が残るため、アップグレード前と同じ方法で管理ポータルやWebアクセスが行えます。**

管理ポータルのアドレス：　http://localhost:52773/csp/sys/UtilHome.csp


#### 3-3. 新規インストール時「No」を選択した場合

**インストールは完了しますが、Webアクセスが行えません＝管理ポータルを開く方法がありません。**

Webアクセスを行うためには、手動でApacheにWebゲートウェイをインストールし、IRISに接続するための設定が必要となります。

詳細はドキュメント：[Web ゲートウェイのインストール](https://docs.intersystems.com/irislatestj/csp/docbook/DocBook.UI.Page.cls?KEY=GCGI_install)をご参照ください。

以下、「No」を選択した後のインストール表示例です。

```
Local web server detected. Would you like to use the web server to connect to this installation <Yes>?  no

InterSystems IRIS did not detect a license key file

Do you want to enter a license key <No>?

Please review the installation options:
------------------------------------------------------------------
Instance name: IRIS
Destination directory: /usr/iris
InterSystems IRIS version to install: 2023.3.0.254.0
Installation type: Development
Unicode support: Y
Initial Security settings: Normal
User who owns instance: root
Group allowed to start and stop instance: root
Effective group for InterSystems IRIS processes: irisusr
Effective user for InterSystems IRIS SuperServer: irisusr
SuperServer port: 1972
JDBC Gateway port: 53773
Web Gateway: not using local web server
------------------------------------------------------------------

Confirm InterSystems IRIS installation <Yes>?

Starting installation
Starting up InterSystems IRIS for loading...
../bin/irisinstall -s . -B -c c -C /usr/iris/iris.cpf*IRIS -W 1 -g2
Starting Control Process
Allocated 498MB shared memory
32MB global buffers, 80MB routine buffers
Creating a WIJ file to hold 32 megabytes of data
IRIS startup successful.
System locale setting is 'C.UTF-8'
This copy of InterSystems IRIS has been licensed for use exclusively by:
No local key detected, trying license server.
Copyright (c) 1986-2023 by InterSystems Corporation
Any other use is a violation of your license agreement

^^/usr/iris/mgr/>

^^/usr/iris/mgr/>
Start of IRIS initialization
Loading system routines
Updating system TEMP and LOCALDATA databases
Loading system classes
Installing National Language support

Setting IRISTEMP default collation to IRIS standard (5)
Updating Security database
Loading system source code
Building system indices
Updating Audit database
Updating Journal directory
Updating User database
Updating Interoperability databases
Scheduling inventory scan
IRIS initialization complete
260 lines written to /usr/iris/mgr/filecheck.isc
See the 'iboot.log' file for a record of the installation.

Starting IRIS
Using 'iris.cpf' configuration file

Starting Control Process
Global buffer setting requires attention.  Auto-selected 25% of total memory.
Allocated 1529MB shared memory
976MB global buffers, 97MB routine buffers
Creating a WIJ file to hold 99 megabytes of data
This copy of InterSystems IRIS has been licensed for use exclusively by:
No local key detected, trying license server.
Copyright (c) 1986-2023 by InterSystems Corporation
Any other use is a violation of your license agreement

1 alert(s) during startup. See messages.log for details.

Installation completed successfully
$
```


### 4.Apacheの設定

ApacheをWebサーバとして利用し、VSCodeでデバッグを行う場合、ApacheのWebSocket用モジュールを有効化する必要があります。

詳細は、[デバッグ時の注意点](#2-デバッグ時の注意点)の[Apacheの場合](#apacheの場合)をご参照ください。をご参照ください。