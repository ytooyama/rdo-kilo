#RDO Kilo-Neutron Quickstart 3台構成編

最終更新日: 2015/06/04

##この文書について
この文書はとりあえず3台構成のOpenStack Kilo環境を構築する場合の手順を説明しています。

この文書は以下の公開資料を元にしています。

RDO Neutron Quickstart

- <https://www.rdoproject.org/Quickstart>
- <https://www.rdoproject.org/Neutron_with_existing_external_network>

##Step 0: 要件

ソフトウェア:

- <https://www.rdoproject.org/Quickstart> を参照。

ワークアラウンドのページも確認してください。

- <https://www.rdoproject.org/Workarounds>

ハードウェア:

- CPU: 3Core以上
- メモリー: 6GB以上

- NIC:
  - Multi Node構成: 2

- OpenStack ネットワーク:

本書では次のネットワーク構成を利用します。

Instance Network | Private Network | Public Network
---------------- | --------------  | --------------
192.168.2.0/24   | 192.168.0.0/24  | 192.168.1.0/24
gw: 192.168.2.1  | -               | gw: 192.168.1.1
ns: 8.8.8.8      | -               | ns: 192.168.1.1


##Step 1: IPアドレスなどの設定

- OpenStack controllerホスト:

eth0            | eth1
--------------  | --------------
192.168.0.10/24 | 192.168.1.10/24

- OpenStack networkホスト:

eth0            | eth1
--------------  | --------------
192.168.0.11/24 | 192.168.1.11/24

- OpenStack computeホスト:

eth0            | eth1
--------------  | --------------
192.168.0.12/24 | 192.168.1.12/24


- カーネルパラメータの設定:

この設定を書き込み、後述のコマンドで反映させます。

````
# vi /usr/lib/sysctl.d/00-system.conf

net.ipv4.ip_forward = 1
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.all.forwarding = 1

# sysctl -e -p /usr/lib/sysctl.d/00-system.conf
（設定を反映）
````

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


##Step 2: ソフトウェアリポジトリーの追加

ソフトウェアパッケージのインストールとアップデートを行います｡

次のコマンドを実行してリポジトリーを有効化:

````
# yum install http://rdoproject.org/repos/openstack-kilo/rdo-release-kilo.rpm
````

各ノードでシステムアップデートの実施:

````
# yum -y update
# reboot
````

##Step 3: Packstackおよび必要パッケージのインストール

コントローラーノードで以下のようにコマンドを実行します｡

````
# yum install -y openstack-packstack python-netaddr
````


##Step 4:アンサーファイルを生成

コントローラーノードで以下のようにコマンドを実行してアンサーファイルを作成します｡

````
# packstack --gen-answer-file=answer.txt
(answer.txtという名前のファイルを作成する場合)
````

アンサーファイルを使うことで定義した環境でOpenStackをデプロイできます｡

作成したアンサーファイルは1台のマシンにすべてをインストールする設定が行われています｡IPアドレスや各種パスワードなどを適宜設定します｡

##Step 5:アンサーファイルを自分の環境に合わせて設定

OpenStack環境を作るには最低限以下のパラメータを設定します。項目についてはPackstackのヘルプを確認してください。

- デフォルトパスワードを指定

````
CONFIG_DEFAULT_PASSWORD=password
````

- コンポーネントのインストール可否を指定

インストールする(y)と設定した場合、追加の設定を行う必要があるものもあります。

````
CONFIG_GLANCE_INSTALL=y
CONFIG_CINDER_INSTALL=n
CONFIG_NOVA_INSTALL=y
CONFIG_NEUTRON_INSTALL=y
CONFIG_HORIZON_INSTALL=y
CONFIG_SWIFT_INSTALL=n
CONFIG_CEILOMETER_INSTALL=n
CONFIG_HEAT_INSTALL=n
CONFIG_NAGIOS_INSTALL=y
````

- controllerノードを指定

````
CONFIG_CONTROLLER_HOST=192.168.1.10
````

- networkノードを指定

````
CONFIG_NETWORK_HOSTS=192.168.1.11
````

- computeノードを指定(カンマ区切りで複数可)

複数のコンピュートノードを追加するにはカンマでIPアドレスを列挙します｡

````
CONFIG_COMPUTE_HOSTS=192.168.1.12
````

- NICを利用したいものに変更する

eth1がゲートウェイに接続されている場合の記述例。

````
CONFIG_NOVA_COMPUTE_PRIVIF=eth0
CONFIG_NOVA_NETWORK_PRIVIF=eth0
CONFIG_NOVA_NETWORK_PUBIF=eth1
````

- そのほか、適宜設定を変更する

````
#Message Service
# 'qpid' or 'rabbitmq'
CONFIG_AMQP_BACKEND=rabbitmq
CONFIG_AMQP_HOST=192.168.1.10
CONFIG_AMQP_SSL_PORT=5671
#DB Host
CONFIG_MARIADB_HOST=192.168.1.10
#Region
CONFIG_KEYSTONE_REGION=RegionOne
#MongoDB Host
CONFIG_MONGODB_HOST=192.168.1.10
#ノード間通信用のNICを指定
CONFIG_NEUTRON_OVS_TUNNEL_IF=eth1
````

- Dashboardにアクセスするパスワード

````
CONFIG_KEYSTONE_ADMIN_PW=password
````

- テスト用demoユーザーとかネットワークを作らないようにする

```
CONFIG_PROVISION_DEMO=n
```

##Step 6: Packstackを実行してOpenStackのインストール

実行前に、各ノードでsetenforce 0を実行してください。
これはSELinuxのモードを一時的に許容モードに設定するものです。再起動後は元のモードに戻ります。

````
# setenforce 0
````

設定を書き換えたアンサーファイルを使ってOpenStackを導入するには、次のようにアンサーファイルを指定して実行します。

````
# packstack --answer-file=answer.txt
Welcome to the Packstack setup utility
...
root@192.168.1.10's password:   #各ノードのパスワードを入れる
root@192.168.1.11's password:
root@192.168.1.12's password:
...
 **** Installation completed successfully ******
````

エラーが出ず、インストールが正常に完了すれば「Installation completed successfully」と表示されます。

インストール後に表示されるDashboardのURLにブラウザでアクセスしてみます。ユーザー、パスワードはanswer.txtに設定したものでログインできます。

/rootディレクトリー上にkeystonerc_hogeというユーザー別のRCファイルが作られており、そのファイルでも確認できます。

![Dashboard Login](./images/login.png)


##Step 7: ネットワーク設定の変更

次に外部と通信できるようにするための設定を行います。外部ネットワークとの接続を提供するノード(ネットワークノード、1台構成時はそのマシン)に仮想ネットワークブリッジインターフェイスであるbr-exを設定します。

###◆public用として使うNICの設定を確認
コマンドを実行して、アンサーファイルに設定したPublic用NIC(ゲートウェイとつながっている方)を確認します。
以降の手順ではeth1であることを前提として解説します。

````
# less {packstack-answers-*,answer.txt}|grep CONFIG_NOVA_NETWORK_PUBIF
CONFIG_NOVA_NETWORK_PUBIF=eth1
````

###◆public用として使うNICの設定ファイルを修正
Packstackコマンド実行後、eth1をbr-exにつなぐように設定をします(※BOOTPROTOは設定しない)

eth1からIPアドレス、サブネットマスク、ゲートウェイの設定を削除して次の項目だけを記述し、br-exの方に設定を書き込みます｡

````
# vi /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1
ONBOOT=yes
HWADDR=xx:xx:xx:xx:xx:xx # Your eth1's hwaddr
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
NM_CONTROLLED=no
````

###◆ブリッジインターフェイスの作成
br-exにeth1のIPアドレスを設定します。

````
# vi /etc/sysconfig/network-scripts/ifcfg-br-ex
DEVICE=br-ex
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSBridge
OVSBOOTPROTO=none
OVSDHCPINTERFACES=eth1
IPADDR=192.168.1.11
NETMASK=255.255.255.0  # netmask
GATEWAY=192.168.1.1    # gateway
DNS1=8.8.8.8           # nameserver
DNS2=8.8.4.4
NM_CONTROLLED=no
````

- Packstackインストーラー実行後に、「Warning: NetworkManager is active on xx.xx.xx.xx... OpenStack networking currently does not work on systems that have the Network Manager service enabled.」のようなメッセージが出た場合は、NetworkManagerからnetworkサービスへの切り替え設定を実行します｡再起動後networkサービスが使われます。

````
# systemctl disable NetworkManager && systemctl enable network
````

ここまでできたらいったんホストを再起動します。

````
# reboot
````

networkサービスに切り替え後は、NM_CONTROLLED=noは削除しても構いません（そのままでも構いません）。

###◆動作確認
Packstackインストーラーによるインストール時にエラー出力がされなければ問題はありませんが、念のためbr-exとNova、Neutronエージェントが導入されてかつ正しく認識されていることを確認しましょう。

まずは再起動後にbr-exが正しく動作し、外のネットワークとつながっていることを確認します。

````
# ip a s br-ex | grep inet
    inet 192.168.1.10/24 brd 192.168.1.255 scope global br-ex
    inet6 fe80::54d3:7dff:fee0:a046/64 scope link
# ping enterprisecloud.jp -c 3 -I br-ex | grep "packet loss"
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
````

パケットロスがないことを確認します。

つぎに、OpenStack NovaコンポーネントのステートがOKであることを確認します。

````
# source /root/keystonerc_admin
(adminユーザー認証情報を読み込む)
# nova-manage service list
Binary           Host      Zone             Status     State Updated_At
nova-consoleauth node1     internal         enabled    :-)   2015-04-28 02:22:19
nova-scheduler   node1     internal         enabled    :-)   2015-04-28 02:22:19
nova-conductor   node1     internal         enabled    :-)   2015-04-28 02:22:19
nova-compute     node1     nova             enabled    :-)   2015-04-28 02:22:19
nova-cert        node1     internal         enabled    :-)   2015-04-28 02:22:19
````

最後に、NeutronのエージェントがOKであることを確認します。

````
# neutron agent-list -c agent_type -c host -c alive
+--------------------+---------+-------+
| agent_type         | host    | alive |
+--------------------+---------+-------+
| Metadata agent     | node1   | :-)   |
| L3 agent           | node1   | :-)   |
| Open vSwitch agent | node1   | :-)   |
| DHCP agent         | node1   | :-)   |
+--------------------+---------+-------+
````

##この後の設定について

次にNeutron Networkを作成します。

[Neutron ネットワークの設定](2-RDO-QuickStart-Networking.md) をご覧ください。