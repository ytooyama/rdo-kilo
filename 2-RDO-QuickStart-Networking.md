#RDO Neutron ネットワークの設定

最終更新日: 2015/04/27

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

登録済みのテナントを確認して、ネットワーク作成時に指定するテナントを検討します｡

````
# keystone tenant-list
+----------------------------------+----------+---------+
|                id                |   name   | enabled |
+----------------------------------+----------+---------+
| 2b0260a2580842abab33b56dae6a145f |  admin   |   True  |
| 20a6abed5e8549f29a76fa26b2b1c8db | services |   True  |
+----------------------------------+----------+---------+
````

- パブリックネットワークの作成

本例ではtenant-idはadminのものを指定します｡

````
# neutron net-create public --router:external --tenant-id 2b0260a2580842abab33b56dae6a145f
````

net-createコマンドの先頭にはまずネットワーク名を記述します｡
tenant-idは「keystone tenant-list」で出力される中から「テナント」を指定します。
router:external=Trueは外部ネットワークとして指定するかしないかを設定します｡
プライベートネットワークを作る場合は指定する必要はありません｡

- プライベートネットワークの作成
本例ではtenant-idはserviceのものを指定します｡

````
# neutron net-create demo-net --shared --tenant-id 20a6abed5e8549f29a76fa26b2b1c8db
````

ネットワークを共有するには--sharedオプションを付けて実行します｡

###◆ネットワークサブネットの登録
ネットワークで利用するサブネットを定義します｡

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
qrouter-97b749cb-83af-401b-9c99-12be68cb7528

# ip netns exec `ip netns|grep qrouter` ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
7: qg-518ec30b-e1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether fa:16:3e:7b:45:d7 brd ff:ff:ff:ff:ff:ff
    inet 172.16.214.241/16 brd 172.16.255.255 scope global qg-518ec30b-e1
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe7b:45d7/64 scope link
       valid_lft forever preferred_lft forever
8: qr-5aff4ca2-49: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether fa:16:3e:8c:2d:70 brd ff:ff:ff:ff:ff:ff
    inet 192.168.4.1/24 brd 192.168.4.255 scope global qr-5aff4ca2-49
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe8c:2d70/64 scope link
       valid_lft forever preferred_lft forever

# ip netns exec `ip netns|grep qrouter` \
 ping -c 3 -I qg-518ec30b-e1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) from 172.16.214.241 qg-518ec30b-e1: 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=53 time=9.92 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=53 time=2.22 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=53 time=2.17 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 2.179/4.778/9.927/3.641 ms

# ip netns exec `ip netns|grep qrouter` \
 ping -c 3 -I qr-5aff4ca2-49 192.168.2.1
...
````

pingコマンドが通れば、外部ネットワークと接続がうまくいっていると判断できます。
nova bootコマンドなどでインスタンスを起動するとqdhcpが作られます。同様に確認してみてください。
