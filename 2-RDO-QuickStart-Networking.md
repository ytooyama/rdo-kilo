#RDO Neutron ネットワークの設定

最終更新日: 2016/05/24

##この文書について
この文書は構築したOpenStack環境にNeutronネットワークを作成する手順と作成したネットワークの確認方法の一例を説明しています。

##Step 1: ネットワークの追加
br-exにeth1を割り当てて、仮想マシンをハイパーバイザー外部と通信できるようにする為の経路が確保されていることを確認します。

````
# ovs-vsctl show
d5305735-c3db-410a-9182-f5ec63823c56
    Bridge br-ex
        Port phy-br-ex
            Interface phy-br-ex
        Port "eth1"
            Interface "eth1"
        Port br-ex
            Interface br-ex
                type: internal
    Bridge br-int
        Port int-br-ex
            Interface int-br-ex
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "1.11.0"
````

OSやハードウェア側の設定が終わったら、OpenStackが利用するネットワークを作成してみましょう｡OpenStackにおけるネットワークの設定は以下の順で行います｡

1. ルーターを作成
2. ネットワークを作成
3. ネットワークサブネットを作成

OpenStackの環境構成をコマンドで実行する場合は、/root/keystonerc_adminファイルをsourceコマンドで読み込んでから実行してください｡

````
# source keystonerc_admin
````

それでは順に行っていきましょう｡

###◆ルーターの作成
ルーターの作成は次のようにコマンドを実行します。

````
# neutron router-create router1
````

###◆ネットワークの作成
ネットワークの作成は次のようにコマンドを実行します。

- テナントリストを確認

登録済みのテナントを確認して、ネットワーク作成時に指定するテナントを検討します｡openstackコマンドでテナントの一覧を見るには、"openstack project list"を実行します。

````
# openstack project list
+----------------------------------+----------+
| ID                               | Name     |
+----------------------------------+----------+
| 08114e2588b14949bcf20115e7ae8a41 | admin    |
| 30f2cc33d5f8405895e5558b077a86c6 | services |
+----------------------------------+----------+
````

- パブリックネットワークの作成


本例ではtenant-idはadminのものを指定します｡

````
# neutron net-create public --router:external --tenant-id 08114e2588b14949bcf20115e7ae8a41
````

net-createコマンドの先頭にはまずネットワーク名を記述します｡
tenant-idは「keystone tenant-list」で出力される中から「テナント」を指定します。
router:externalは外部ネットワークとして指定するかしないかを設定します｡
プライベートネットワークを作る場合は指定する必要はありません｡

- プライベートネットワークの作成
本例ではtenant-idはserviceのものを指定します｡

````
# neutron net-create demo-net --shared --tenant-id 30f2cc33d5f8405895e5558b077a86c6
````

ネットワークを共有するには--sharedオプションを付けて実行します｡

###◆ネットワークサブネットの登録
publicネットワークで利用するサブネットを定義します｡

````
# neutron subnet-create --name public_subnet --enable_dhcp=False \
--allocation-pool=start=192.168.1.241,end=192.168.1.254 --gateway=192.168.1.1 public 192.168.1.0/24
Created a new subnet:
（略）
````

これでpublic側ネットワークにサブネットなどを登録することができました｡
次にdemo-net（プライベート）側に登録してみます。

````
# neutron subnet-create --name demo-net_subnet --enable_dhcp=True \
--allocation-pool=start=192.168.2.100,end=192.168.2.254 --gateway=192.168.2.1 \
--dns-nameserver 8.8.8.8 demo-net 192.168.2.0/24
Created a new subnet:
（略）
````

###◆ゲートウェイの設定
作成したルーター(router1)とパブリックネットワークを接続するため、「ゲートウェイの設定」を行います｡

````
# neutron router-gateway-set router1 public
Set gateway for router router1
````


###◆外部ネットワークと内部ネットワークの接続
最後にプライベートネットワークを割り当てたインスタンスがFloating IPを割り当てられたときに外に出られるようにするために「ルーターにインターフェイスの追加」を行います｡

````
# neutron router-interface-add router1 subnet=demo-net_subnet
Added interface xxxx-xxxx-xxxx-xxxx-xxxx to router router1.
````

routerはneutron router-listコマンドで確認、サブネットはneutron subnet-listコマンドで確認することができます。


##Step 2: 仮想ルーターが作られたか確認
neutronコマンドでネットワークを定義したことで仮想ルーター(qrouter)が作られていることを確認します。neutron-l3-agentサービスを実行しているノードで確認します。

````
# ip netns
qrouter-580e8e2f-f99a-4c33-8dea-7369bcc12c8d
qdhcp-cdb0f492-cf59-40ac-8218-4fbabbbf9c08

# ip netns exec `ip netns|grep qrouter` ip r
default via 192.168.1.1 dev qg-fdf68416-44
192.168.1.0/24 dev qg-fdf68416-44  proto kernel  scope link  src 192.168.1.241
192.168.2.0/24 dev qr-416cae4f-17  proto kernel  scope link  src 192.168.2.1

# ip netns exec `ip netns|grep qrouter` \
 ping -c 3 -I qg-fdf68416-44 192.168.1.241

# ip netns exec `ip netns|grep qrouter` \
 ping -c 3 -I qg-fdf68416-44 192.168.1.1
 
# ip netns exec `ip netns|grep qrouter` \
 ping -c 3 -I qg-fdf68416-44 8.8.8.8

# ip netns exec `ip netns|grep qrouter` \
 ping -c 3 -I qr-416cae4f-17 192.168.2.1
````

pingコマンドが通れば、外部ネットワークと接続がうまくいっていると判断できます。


##番外:気をつける点など 
- CentOS 7のApache設定はデフォルトでKeepAlive Offなので、Onにしたほうがいいかもしれない。

````
# vi /etc/httpd/conf/httpd.conf
...
KeepAlive On
...
# systemctl restart httpd
````

- IP設定でMACアドレスの記述を忘れずに

NetworkManagerが動いている場合はHWADDRがなくてもうまく動きますが、networkサービスに切り替えた時に、運が悪いとNICデバイス名が変わって通信できなくなります。すべてのNIC設定にHWADDRを付加して再起動すると復旧できます。


````
...
HWADDR=xx:xx:xx:xx:xx:xx
````

- Unable to establish connection to http://xxx.xxx.xxx.xxx:5000/v2.0/tokensというエラーが発生する

KiloでUnable to establish connection to http://xxx.xxx.xxx.xxx:5000/v2.0/tokensといったエラーが出た場合はKeystoneが正常に動いていないので、httpdを再起動してみてください。その後、keystone token-getなどのコマンドで応答が返ってくれば問題ないです。

- openstack-statusの結果がなかなか出てこない

openstack-statusコマンドが含まれるopenstack-utilsパッケージは、更新日をみる限りおそらく更新されていないので使わないほうがいいです。パッケージ依存の関係で入っていますが、Kiloではうまく動かないようです。

- PublicゲートウェイにPingを飛ばしてもpingが通らない/インスタンスで外部ネットワークにpingが届かない

iptables-saveを仮想ルーター上で実行してみてください。

````
# ip netns exec `ip netns|grep qrouter` iptables-save
````

- ESXiで仮想マシンを作ってその環境でOpenStackを動かす場合、ESXi仮想マシンが接続しているネットワークのポートグループで「無差別モード(Promiscuous Mode)」を「承諾(Accept)」にする必要があります。

1. 仮想マシンが接続している「ポートグループ」を選択して「編集」ボタンをクリックします。
1. 「セキュリティ(Security)」タブをクリックします。
1. 「無差別モード(Promiscuous Mode)」の横のチェックマークをオンにして「承諾(Accept)」に上書きします。