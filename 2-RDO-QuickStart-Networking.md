#RDO Neutron Quickstart Plus ネットワークの設定

最終更新日: 2014/7/23

##この文書について
この文書はとりあえず1台に全部入りのOpenStack Icehouse環境をさくっと構築する場合の手順を説明しています。
サーバーとは別途クライアントを用意して、同じネットワーク側(192.168.0.0/24)に接続します｡

この文書は以下の公開資料を元にしています。

RDO Neutron Quickstart

<http://openstack.redhat.com/Neutron-Quickstart>
<http://openstack.redhat.com/Neutron_with_existing_external_network>

##Step8: ネットワークの追加
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
| f032057fb6044faa81a30ad81231d9a8 | alt_demo |   True  |
| 93edefe5c7304ef9b268b1459740b7d8 |   demo   |   True  |
| 20a6abed5e8549f29a76fa26b2b1c8db | services |   True  |
+----------------------------------+----------+---------+
````

- パブリックネットワークの作成
本例ではtenant-idはadminのものを指定します｡

````
# neutron net-create public --router:external=True --tenant-id 2b0260a2580842abab33b56dae6a145f
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
