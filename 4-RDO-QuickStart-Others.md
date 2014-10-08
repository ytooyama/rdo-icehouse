#RDO Neutron Quickstart Plus Novaの設定変更とインスタンスイメージの登録

最終更新日: 2014/10/8


##この文書について
この文書はとりあえず1台に全部入りのOpenStack Icehouse環境をさくっと構築する場合の手順を説明しています。

この文書は以下の公開資料を元にしています。

RDO Neutron Quickstart

<http://openstack.redhat.com/Neutron-Quickstart>
<http://openstack.redhat.com/Neutron_with_existing_external_network>

##Step 10: Novaの設定変更

デフォルト設定のままインストールした場合、Novaは仮想化基盤にqemuを利用するようになっています。パフォーマンスを上げるには以下のように設定を変更します。

事前にKVMモジュールが読み込まれていることを確認します。

````
# lsmod | grep kvm
kvm_intel              54285  6
kvm                   332980  1 kvm_intel
(Intel CPUの場合)
kvm                   332980  1 kvm_amd
(AMD CPUの場合)

# vi /etc/nova/nova.conf
（略）
virt_type=kvm
#virt_type=qemu
（設定を変更）

# service openstack-nova-compute restart
````


##Step 11: インスタンスイメージの登録

OpenStack環境でインスタンスを実行するため、イメージの登録を行ってみます｡
ここでは動作テスト用としてしばしば利用される、CirrOSを登録してみます｡

CirrOSをダウンロードしてOpenStackに登録するには、次のように行います。

````
# curl -OL http://download.cirros-cloud.net/0.3.3/cirros-0.3.3-x86_64-disk.img

# glance image-create --name="CirrOS 0.3.3" --disk-format=qcow2 \
--container-format=bare --is-public=true \
< cirros-0.3.3-x86_64-disk.img
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 64d7c1cd2b6f60c92c14662941cb7913     |
| container_format | bare                                 |
| created_at       | 2014-09-22T02:08:50                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | d1b69800-3b30-42b2-a0b4-3c4a97bcdb1b |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | CirrOS 0.3.3                         |
| owner            | 67962937e5f843d0985d23c01a644011     |
| protected        | False                                |
| size             | 13167616                             |
| status           | active                               |
| updated_at       | 2014-09-22T02:08:50                  |
+------------------+--------------------------------------+
````

以上で、テスト用のインスタンスイメージを登録できました｡

その他のOSを動かすには、公式ドキュメント
「OpenStack Virtual Machine Image Guide」をご覧ください｡

<http://docs.openstack.org/image-guide/content/index.html>


###■動作確認
- 「セキュリティグループ」でICMPとSSHを有効化
- インスタンスを起動

OpenStackで利用するクラウドイメージは以下からダウンロードできます。

CirrOS

<http://download.cirros-cloud.net>

※0.3.1以上のバージョンを利用してください｡

Fedora

<http://fedoraproject.org/en/get-fedora#clouds>

Ubuntu

<http://cloud-images.ubuntu.com>

- Floating IPをインスタンスに割り当て
- ホストにSSHアクセスしてそこからインスタンスにアクセス
- (Cinderを構築したのであれば)ボリューム作成とインスタンスへの割り当て

##Step 12: CentOS 6.xでNested KVM
CentOS 6.xのKernelは2.6.32と古いので、Nested KVMは動きません(正確に言うと、L-1でKVMモジュールを読み込めません)。
つまり、Linux KVM(L-0)上の仮想マシンでOpenStack環境を構築して、Computeノードで入れ子のKVM(L-1)を動かすことができません。

Nested KVMを利用するには、より新しいLinuxでNested KVM環境を構築するか、次の手順のようにXen向けのカーネルを流用して構築してください。

<http://wiki.centos.org/HowTos/NestedVirt>

ただし、上記手順のリポジトリー上のカーネルパッケージは古いので、次のようなリポジトリーファイルを作ってください。
このリポジトリーはenabled=1に絶対に設定しないでください。そう設定してyum updateしてしまうと再起動後、起動しなくなります。

````
[xen‒c6]
name=CentOS-$releasever - Xen
baseurl=http://isoredirect.centos.org/centos/6/xen4/$basearch/
gpgcheck=0
enabled=0
priority=1
````

新しいカーネルは次のようにインストールします。

````
# yum --enablerepo xen-c6 install kernel kernel-firmware
````

アップデートする場合は次のように実行してください。

````
# yum --enablerepo xen-c6 update kernel kernel-firmware
````

Nested KVMのL-0ホストはFedora 20,Ubuntu Server 14.04.1,CentOS 7などをオススメします。


##Step 13: QA

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

インスタンスへのログインはデフォルトでは鍵認証で行います。そのため、Dashboardのコンソールでは実質操作できません。
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

ログの日付が見づらくなるので日本時間(JST)に設定したいと思うかもしれません。カスタマイズ・スクリプトでcloud-configに次のようにrunコマンドで実行したいコマンドを定義することで可能です。

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
# service dnsmasq restart
# service neutron-dhcp-agent restart
````
