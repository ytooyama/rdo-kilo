# Kilo-Swift Quickstart

最終更新日: 2015/10/22

## この文書について

この文書は1台のマシンにOpenStack KiloバージョンのSwift環境をさくっと構築する場合の手順を説明しています。


## Step 0: 要件

ソフトウェア:

- CentOS 7 (Release+ Repo)
- Fedora 23 (Beta+updates-testing Repo)

ハードウェア:

- CPU: 2Core以上
- メモリー: 2GB以上
- Disk: 16GB以上(推奨はSystem用,Swift1用,Swift2用に分ける)
- NIC: 1以上

                  
## Step 1: IPアドレスなどの設定

- ホスト:

固定IPアドレスを設定します。

- ホスト名の設定:

[hostnamectlコマンド](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/sec_Configuring_Host_Names_Using_hostnamectl.html)を使ってホスト（コンピューター）名を設定します。ちなみに従来のように、hostnameコマンドと/etc/hostnameの編集でも可能です。

````
# hostnamectl set-hostname rdo-kilo
````

hostnameに設定したホスト名を、hostsファイルの127.0.0.1のエントリーに追加します。

````
# vi /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
↓
127.0.0.1   rdo-kilo localhost localhost.localdomain localhost4 localhost4.localdomain4
````


## Step 2: ソフトウェアリポジトリーの追加

ソフトウェアパッケージのインストールとアップデートを行います｡

次のコマンドを実行してOpenStack用リポジトリーを有効化:

※Fedora 23を使う場合は不要(Fedora 23はkiloのパッケージが標準リポジトリーにある)。

````
# yum install centos-release-openstack-kilo
# ls /etc/yum.repos.d/ |grep OpenStack
CentOS-OpenStack-kilo.repo
````

<!--
````
# yum install http://rdoproject.org/repos/openstack-kilo/rdo-release-kilo.rpm
````
-->

システムアップデートの実施:

````
# yum -y update
# reboot
````

##Step 3: Packstackおよび必要パッケージのインストール

ノードで以下のようにコマンドを実行します｡

````
# yum install -y openstack-packstack openstack-packstack-doc python-netaddr
````

## Step 4:DryRunモードでPackstackコマンドの実施

ノードで以下のようにコマンドを実行してマニフェストファイルを作成します。

````
# packstack --dry-run --allinone --default-password=password --provision-demo=n
...
Welcome to the Packstack setup utility
...
 **** Installation completed successfully ******
Additional information:
 * A new answerfile was created in: /root/packstack-answers-20150612-162316.txt
````

処理が終わると、/rootディレクトリーにアンサーファイル（本例ではpackstack-answers-20150612-162316.txt）が作られます。
アンサーファイルを使うことで定義した環境でOpenStackをデプロイできます｡

作成したアンサーファイルは1台のマシンにすべてをインストールする設定が行われています｡IPアドレスや各種パスワードなどを適宜設定します｡

## Step 5:アンサーファイルを自分の環境に合わせて設定

Swift環境を作るには最低限以下のパラメータを設定します。項目についてはPackstackのヘルプを確認してください。

- コンポーネントのインストール可否を指定

インストールする(y)と設定した場合、追加の設定を行う必要があるものもあります。ここではMariaDB、SwiftとHorizonをインストールします。

````
CONFIG_MARIADB_INSTALL=y
CONFIG_GLANCE_INSTALL=n
CONFIG_CINDER_INSTALL=n
CONFIG_NOVA_INSTALL=n
CONFIG_NEUTRON_INSTALL=n
CONFIG_HORIZON_INSTALL=y
CONFIG_SWIFT_INSTALL=y
CONFIG_CEILOMETER_INSTALL=n
CONFIG_HEAT_INSTALL=n
CONFIG_NAGIOS_INSTALL=n
````

## Step 6: Packstackを実行してOpenStackのインストール

実行前に、各ノードでsetenforce 0を実行してください。
これはSELinuxのモードを一時的に許容モードに設定するものです。再起動後は元のモードに戻ります。

````
# setenforce 0
````

設定を書き換えたアンサーファイルを使ってOpenStackを導入するには、次のようにアンサーファイルを指定して実行します。

````
# packstack --answer-file=packstack-answers-20150612-162316.txt
...
 **** Installation completed successfully ******
````

エラーが出ず、インストールが正常に完了すれば「Installation completed successfully」と表示されます。


## Swiftを使う前に

### CentOS 7の場合

CentOS 7では以下のコマンドを実行してSwift関連のSELinux設定を変更します。permissiveにしたり、disabledにする必要はありません。

````
# setsebool -P swift_can_network on
# setsebool -P httpd_use_openstack on
# reboot
````

### Fedora 23の場合

上記の設定を行ってもSwiftのサービスが動かないのでSELinuxが発動しないように設定変更します(Fedora 23 Betaでのテストなので、正式リリース時はオフにする必要がないかもしれません)。

````
# sed -i s/SELINUX=enforcing/SELINUX=disabled/g /etc/selinux/config
# touch /.autorelabel
# reboot
````


## Swiftを使う

- コマンドでSwiftを利用
 - \<container>はコンテナー名
 - \<filename>はアップロードしたいファイル名
 - \<object>はコンテナーからダウンロードしたいオブジェクト名

````
# keystonerc_admin
(環境設定ファイルを読み込み)
# openstack container create <container>
(コンテナの作成)
# openstack container delete <container>
(コンテナの削除)
# openstack container list
(コンテナの一覧)
# openstack container show <container>
(コンテナの詳細情報の表示)
# openstack object create <container> <filename>
(コンテナにオブジェクトをアップロード)
# openstack object save <container> <object>
(コンテナからオブジェクトをダウンロード)
# openstack object list <container>
(コンテナー内のオブジェクトを一覧表示)
# openstack object delete <container> <object>
(コンテナー内のオブジェクトを削除)
````

- HorizonでSwiftを利用

![Dashboard Login](./images/login.png)

1. admin/passwordでログイン。
2. 左のメニューで「プロジェクト→オブジェクトストア→コンテナー」をクリック。
3. 「コンテナーの作成」ボタンをクリックしてコンテナーを作成。
4. 「オブジェクトのアップロード」ボタンをクリックしてコンテナーにアップロード。

※Novaなどがインストールされていないとエラーが出るけど気にするな。


##Swiftサービスなどの確認

次のようなシェルスクリプトを作成する。

````
#!/bin/bash
systemctl $1 rabbitmq-server.service openstack-swift-account-auditor.service \
openstack-swift-account-reaper.service openstack-swift-account-replicator.service \
openstack-swift-account.service openstack-swift-container-auditor.service \
openstack-swift-container-replicator.service openstack-swift-container-updater.service \
openstack-swift-container.service openstack-swift-object-auditor.service \
openstack-swift-object-replicator.service openstack-swift-object-updater.service \
openstack-swift-object.service openstack-swift-proxy.service
````

次のように実行して`active (running)`になっていることを確認。

````
# bash swift-service.sh status
````


## このあとやること

デフォルトではループバックデバイスが一つ登録されているだけなので、データの冗長性を保つには複数のストレージを追加してSwiftのストレージとして追加する必要があります（説明は省略）。