#その他の設定メモ

最終更新日: 2015/10/7


##この文書について
この文書はその他の設定をまとめています。


##Step 1: KVMモードでインスタンスを起動するための設定

仮想マシン上にOpenStackを構築した場合、Novaは仮想化基盤にQEMUを利用するように設定されています。インスタンス起動のスピードやパフォーマンスを上げるには、以下のように設定を変更します。

事前にKVMモジュールが読み込まれていることを確認します。

````
# lsmod | grep kvm
kvm_intel              54285  6
kvm                   332980  1 kvm_intel
(Intel CPUの場合)
kvm                   332980  1 kvm_amd
(AMD CPUの場合)
# sed -i s/virt_type=qemu/virt_type=kvm/g /etc/nova/nova.conf
(KVMモードでインスタンスが起動するように設定を変更)
# modprobe vhost_net
# echo vhost_net >> /etc/modules-load.d/vhost_net.conf
(vhost_netモジュールの有効化と恒久的にモジュールを自動読み込みするように設定)
# systemctl restart openstack-nova-compute
(Computeサービスを再起動)
````


##Step 2: インスタンスイメージの登録

Glanceコマンドで--locationオプションを使うと、イメージをダウンロードせずに直接Glanceにアップロードできます。次のように実行します。

````
# glance image-create --name cirros --disk-format qcow2 --container-format bare --is-public True --location http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
````

以上で、テスト用のインスタンスイメージを登録できました｡

その他のOSを動かすには、公式ドキュメント
「OpenStack Virtual Machine Image Guide」をご覧ください｡

<http://docs.openstack.org/image-guide/content/index.html>


##Step 3: QA

### 負荷状況を確認したい

topコマンドやdstatコマンドなどで確認してみましょう。

````
# top
# dstat -cdn --top-cpu
````

### 高負荷でOpenStackホストが落ちる

例えばこう設定して、様子を見てみる。

````
# ulimit -n 5000
````

最適値は、以下のコマンドを実行して一番左に出てきた数字より多い値を設定する。

````
# cat /proc/sys/fs/file-nr
3328	0	811180
````

この設定をかえないと運用できない場合は、OpenStackのインストール構成が実際の負荷に対して適切ではない可能性が高いです。
サーバーやコンポーネントをできる限り分散してください。

ulimitについての詳細は<http://mikio.github.io/article/2013/03/02_.html>を参照。

### インスタンスでパスワード認証をしたい

インスタンスへのログインはデフォルトでは公開鍵認証で行います。そのため、Dashboardのコンソールでは実質操作できません。
しかし、カスタマイズ・スクリプトでcloud-configを書くと指定したインスタンスでパスワード認証が可能になります。

````
#cloud-config
password: vmpass
chpasswd: { expire: False }
ssh_pwauth: True
````

上記例はパスワードを"vmpass"にする例です。最低この4行があれば実現できます。

### アップデートしてインスタンスを起動したい

カスタマイズ・スクリプトでcloud-configに次のように「package_upgrade: True」を定義することで可能です。

````
#cloud-config
package_upgrade: True
````

### 日本時間(JST)に設定してインスタンスを起動したい

ログの日付が見づらくなるので日本時間(JST)に設定したいと思うかもしれません。カスタマイズ・スクリプトでcloud-configに次のようにruncmdで実行したいコマンドを定義することで可能です。

````
#cloud-config
runcmd:
 - [cp, /usr/share/zoneinfo/Asia/Tokyo, /etc/localtime]
````

### インスタンスで外部ネットワークにアクセスしようとすると応答がなくなったり切断される

RDO Packstack Icehouse版アンサーファイルのデフォルトはvxlanモードに設定されており、このままインストールするとMTU溢れの問題が発生します。
その結果インスタンスとのSSH接続が切断されたり、Pingに対する応答がなくなるなどの問題が発生します。

VXLANとMTUについては以下に少々記述があり、参考になります。
<http://www.cisco.com/cisco/web/support/JP/docs/SW/DCSWT/Nex1000VSWT/CG/026/b_VXLAN_Configuration_4_2_1SV_2_1_1_chapter_010.html?bid=0900e4b182e40102>

CirrOSなどパスワード認証できるイメージから起動して、インスタンスでファイルのコピーを実行してみてください。
一度目はダウンロードが行えず、二回目のwgetで正常にダウンロードできればこの問題にあたっている可能性があります。

````
$ wget hogehoge
$ sudo ifconfig eth0 mtu 1450
$ wget hogehoge
````

インスタンスの起動のさいに対処する方法として、OpenStack DashBoardを使ってインスタンスを起動する際に、カスタマイズ・スクリプトでcloud-configを書く方法があります。
この方法はLinuxインスタンスでのみ有効です。もしくはインスタンス起動後に/etc/rc.localに直接記述してもよいです。

````
#cloud-config
bootcmd:
 - echo "ifconfig eth0 mtu 1450" > /etc/rc.local
````

もう少し現実的な方法として、DHCPサーバーのdhcp-optionでMTU 1450などを渡す方法があります。
OpenStackのデフォルト構成ではdnsmasqを利用していますので、以下のように設定してください。

まずはdhcp_agentの設定を行います。

````
# vi /etc/neutron/dhcp_agent.ini
dnsmasq_config_file=/etc/dnsmasq.d/dnsmasq.conf
````

つぎにMTU 1450を設定します。

````
# vi /etc/dnsmasq.d/dnsmasq.conf
dhcp-option=26,1450
````

あとはdnsmasqとneutron-dhcp-agentサービスを再起動します。

````
# systemctl restart dnsmasq
# systemctl restart neutron-dhcp-agent
````

###日本語キーボードを使いたい
nova-computeの/etc/nova/nova.confに以下のように記述

````
# Keymap for VNC (string value)
#vnc_keymap=en-us
vnc_keymap=ja
````

Computeサービスを再起動します。

````
# systemctl restart openstack-nova-compute
````

次回起動したインスタンスから日本語キーボードが使えるようになります。ただし、日本語キーマップで使えるかどうかはOS次第です。