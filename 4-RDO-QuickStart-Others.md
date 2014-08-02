#RDO Neutron Quickstart Plus Novaの設定変更とインスタンスイメージの登録

最終更新日: 2014/8/2

##この文書について
この文書はとりあえず1台に全部入りのOpenStack Icehouse環境をさくっと構築する場合の手順を説明しています。

この文書は以下の公開資料を元にしています。

RDO Neutron Quickstart

<http://openstack.redhat.com/Neutron-Quickstart>
<http://openstack.redhat.com/Neutron_with_existing_external_network>

##Step 10: Novaの設定変更

デフォルト設定のままインストールした場合、Novaは仮想化基盤にqemuを利用するようになっています。パフォーマンスを上げるには以下のように設定を変更します。

KVMモジュールが読み込まれていることを確認します。
またスケジューラーの設定も行っておきます。

````
# lsmod | grep kvm
kvm_intel              54285  6
kvm                   332980  1 kvm_intel
(Intel CPUの場合)
kvm                   332980  1 kvm_amd
(AMD CPUの場合)

# vi /etc/nova/nova.conf
（略）
scheduler_default_filters=AllHostsFilter
（略）
virt_type=kvm
#virt_type=qemu
（設定を変更）

# service openstack-nova-scheduler restart
# service openstack-nova-compute restart
````


##Step 11: インスタンスイメージの登録

OpenStack環境でインスタンスを実行するため、イメージの登録を行ってみます｡
ここでは動作テスト用としてしばしば利用される、CirrOSを登録してみます｡

CirrOSをダウンロードしてOpenStackに登録するには、次のように行います。

````
# curl -OL http://download.cirros-cloud.net/0.3.2/cirros-0.3.2-x86_64-disk.img

# glance image-create --name="CirrOS 0.3.2" --disk-format=qcow2 \
--container-format=bare --is-public=true \
< cirros-0.3.2-x86_64-disk.img
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 64d7c1cd2b6f60c92c14662941cb7913     |
| container_format | bare                                 |
| created_at       | 2014-03-31T02:08:50                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | d1b69800-3b30-42b2-a0b4-3c4a97bcdb1b |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | CirrOS 0.3.2                         |
| owner            | 67962937e5f843d0985d23c01a644011     |
| protected        | False                                |
| size             | 13167616                             |
| status           | active                               |
| updated_at       | 2014-03-31T02:08:50                  |
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

Cirros

<http://download.cirros-cloud.net>

※0.3.1以上のバージョンを利用してください｡

CentOS

<http://repos.fedorapeople.org/repos/openstack/guest-images/>

Fedora

<http://fedoraproject.org/en/get-fedora#clouds>

Ubuntu

<http://cloud-images.ubuntu.com>

- Floating IPをインスタンスに割り当て
- ホストにSSHアクセスしてそこからインスタンスにアクセス
- (Cinderを構築したのであれば)ボリューム作成とインスタンスへの割り当て


##Step 12: QA

- 負荷状況を確認したい

dstatコマンドなどで確認してみましょう。

````
# dstat -cdn --top-cpu
````

- 高負荷でOpenStackホストが落ちる

例えばこう設定して、様子を見てみる。

````
# ulimit -n 5000
````

最適値は、以下のコマンドを実行して一番左に出てきた数字より多い値を設定する。

````
# cat /proc/sys/fs/file-nr
3328	0	811180
````

詳細は<http://mikio.github.io/article/2013/03/02_.html>を参照。

- インスタンスで外部ネットワークにアクセスしようとすると応答がなくなったり切断される

RDO Packstack Icehouse版アンサーファイルのデフォルトはvxlanモードに設定されており、このままインストールするとMTU溢れの問題が発生します。
その結果、インスタンスとのSSH接続が切断されたり、Pingの応答がされなくなるなどの問題が発生します。

VXLANとMTUについては以下に少々記述があり、参考になります。
<http://www.cisco.com/cisco/web/support/JP/docs/SW/DCSWT/Nex1000VSWT/CG/026/b_VXLAN_Configuration_4_2_1SV_2_1_1_chapter_010.html?bid=0900e4b182e40102>

CirrOSなどパスワード認証できるイメージから起動して、インスタンスでファイルのコピーを実行してみてください。
一度目はダウンロードが行えず、二回目のwgetで正常にダウンロードできればこの問題にあたっている可能性があります。

````
$ wget hogehoge
$ sudo ifconfig eth0 mtu 1450
$ wget hogehoge
````

インスタンスの起動のさいに対処する方法として、カスタマイズスクリプトでcloud-configを書く方法があります。
この方法はLinuxインスタンスでのみ有効です。もしくはインスタンス起動後に/etc/rc.localに直接記述してもよいです。

````
#cloud-config
bootcmd:
 - echo "ifconfig eth0 mtu 1450" > /etc/rc.local
````
