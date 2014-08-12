#RDO Neutron Quickstart Plus 単体構成編

最終更新日: 2014/8/3

##この文書について
この文書はとりあえず1台に全部入りのOpenStack Icehouse環境をさくっと構築する場合の手順を説明しています。

この文書は以下の公開資料を元にしています。

RDO Neutron Quickstart

<http://openstack.redhat.com/Neutron-Quickstart>
<http://openstack.redhat.com/Neutron_with_existing_external_network>

##Step 0: 要件

Software:
- Red Hat Enterprise Linux (RHEL) 6.5以降
- CentOS, Scientific Linux 6.5以降
- Fedora 20

Hardware:
- CPU 3Core以上
- メモリー6GB以上
- 最低1つのNIC
※All-in-oneの構成を作る場合は、Privateネットワーク用はloインターフェイスを利用できます。

- ネットワーク
本書では次のネットワーク構成を利用します。

Private Network | Public Network
--------------  | -------------
192.168.2.0/24  | 192.168.1.0/24

- OpenStackホスト

eth0            | eth1
--------------  | --------------
192.168.0.10/24 | 192.168.1.10/24

- カーネルパラメータの設定
以下のように設定を変更します。

````
# vi /etc/sysctl.conf

net.ipv4.ip_forward = 1              #変更
net.ipv4.conf.default.rp_filter = 0  #変更

net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0

net.ipv4.conf.all.rp_filter = 0     #追記
net.ipv4.conf.all.forwarding = 1    #追記

# sysctl -e -p /etc/sysctl.conf
（設定を反映）
````

##Step 1: SELinuxの設定変更
SELinuxの設定を変更します｡

下記公式サイトのコメントにあるようにSELinuxをpermissiveに設定する<br />
↓<br />
ただしdisableにするとpackstackが想定通り動かない模様。

<http://openstack.redhat.com/SELinux_issues>

Note: Due to the quantum/neutron rename, SELinux policies are currently broken for Havana, so SELinux must be disabled/permissive on machines running neutron services, edit /etc/selinux/config to set SELINUX=permissive.

##Step 2: ソフトウェアリポジトリの追加

ソフトウェアパッケージのインストールとアップデートを行う｡
Neutron環境の構築にはSELinuxの設定変更が必要なので設定完了後、一旦再起動する｡

次のコマンドを実行(Fedora 21では不要):

````
# yum install -y http://rdo.fedorapeople.org/openstack-icehouse/rdo-release-icehouse.rpm
````

システムアップデートの実施:

````
# yum -y update
# reboot
````

##Step 3: Packstackおよび必要パッケージのインストール

以下のようにコマンドを実行します｡

````
# yum install -y openstack-packstack python-netaddr
````


##Step 4:アンサーファイルを生成

以下のようにコマンドを実行してアンサーファイルを作成します｡

````
# packstack --gen-answer-file=answer.txt
(answer.txtという名前のファイルを作成する場合)
````

アンサーファイルを使うことで定義した環境でOpenStackをデプロイできます｡

作成したアンサーファイルは1台のマシンにすべてをインストールする設定が行われています｡IPアドレスや各種パスワードなどを適宜設定します｡

##Step 5:アンサーファイルを自分の環境に合わせて設定

OpenStack環境を作るには最低限以下のパラメータを設定します。

- CONFIG_NOVA_COMPUTE_HOSTSにコンピュートノードを設定

複数のコンピュートノードを追加するにはコンマでIPアドレスを列挙します｡

- 1つ指定する例

CONFIG_NOVA_COMPUTE_HOSTS=192.168.1.10

- 複数指定する例

CONFIG_NOVA_COMPUTE_HOSTS=192.168.1.10,192.168.1.11

- NICを利用したいものに変更する

（例）eth1がゲートウェイに接続されている場合

CONFIG_NOVA_COMPUTE_PRIVIF=lo

CONFIG_NOVA_NETWORK_PRIVIF=lo

CONFIG_NOVA_NETWORK_PUBIF=eth1

NICが複数ある場合は次のように指定しても良い｡

（例）eth1がゲートウェイに接続されている場合

CONFIG_NOVA_COMPUTE_PRIVIF=eth0

CONFIG_NOVA_NETWORK_PRIVIF=eth0

CONFIG_NOVA_NETWORK_PUBIF=eth1

- Dashboardにアクセスするパスワード

CONFIG_KEYSTONE_ADMIN_PW=admin

- テスト用demoユーザーとかネットワークを作らないようにする

CONFIG_PROVISION_DEMO=n

- 単体構成なので、ローカルモードで実行する設定を行う

CONFIG_NEUTRON_ML2_TYPE_DRIVERS=local

CONFIG_NEUTRON_ML2_TENANT_NETWORK_TYPES=local

CONFIG_NEUTRON_LB_TENANT_NETWORK_TYPE=local

CONFIG_NEUTRON_OVS_TENANT_NETWORK_TYPE=local


##Step 6: Packstackを実行してOpenStackのインストール

設定を書き換えたアンサーファイルを使ってOpenStackを導入するには、次のようにアンサーファイルを指定して実行します。

````
# packstack --answer-file=/root/answer.txt
````

##Step 7: ネットワーク設定の変更

次に外部と通信できるようにするための設定を行います。
http://openstack.redhat.com/Neutron_with_existing_external_network

###◆public用として使うNICの設定を確認
コマンドを実行して、アンサーファイルに設定したpublic用NICを確認します。
以降の手順ではeth1であることを前提として解説します。

````
# less {packstack-answers-*,answer.txt}|grep CONFIG_NOVA_NETWORK_PUBIF
CONFIG_NOVA_NETWORK_PUBIF=eth1
````

###◆public用として使うNICの設定ファイルを修正
packstack実行後、eth1をbr-exにつなぐように設定をします(※BOOTPROTOは設定しない)

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
IPADDR=192.168.1.10
NETMASK=255.255.255.0  # netmask
GATEWAY=192.168.1.1    # gateway
DNS1=8.8.8.8           # nameserver
DNS2=8.8.4.4
NM_CONTROLLED=no
````

###◆プライベートインターフェイスの設定
プライベートインターフェイスとしてanswer.txtに指定したNICの設定を変更します。loデバイスを指定した場合は特に設定変更する必要はありません。
ただし、loデバイスを指定した場合はall-in-one構成のみしか構成できません。

- answer.txtファイルの設定を確認します。

````
# less {packstack-answers-*,answer.txt}|grep CONFIG_NOVA_NETWORK_PRIVIF
CONFIG_NOVA_NETWORK_PRIVIF=eth0
# less {packstack-answers-*,answer.txt}|grep CONFIG_NOVA_COMPUTE_PRIVIF
CONFIG_NOVA_COMPUTE_PRIVIF=eth0
````

- IP設定を行います。IPアドレスとサブネットマスクの設定を行います。

````
# vi /etc/sysconfig/network-scripts/ifcfg-eth0

DEVICE=eth0
HWADDR=xx:xx:xx:xx:xx:xx # Your eth0's hwaddr
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
IPADDR=192.168.0.10
NETMASK=255.255.255.0
NM_CONTROLLED=no
````

- FedoraではここでNetworkManagerからnetworkサービスへの切り替え設定を実行します｡再起動後networkサービスが使われます。

```
# systemctl disable NetworkManager
# chkconfig enable network
```

ここまでできたらいったんホストを再起動します。

````
# reboot
````
