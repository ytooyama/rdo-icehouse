#RDO Neutron Quickstart Plus ユーザーの追加

最終更新日: 2014/9/11

##この文書について
この文書はとりあえず1台に全部入りのOpenStack Icehouse環境をさくっと構築する場合の手順を説明しています。

この文書は以下の公開資料を元にしています。

RDO Neutron Quickstart

<http://openstack.redhat.com/Neutron-Quickstart>
<http://openstack.redhat.com/Neutron_with_existing_external_network>

##Step 9: ユーザーの追加
次に、ユーザーの追加を行います｡
ユーザーを作成する前にユーザーをどのテナント、つまりどのグループに追加するか考えます。作成したテナントにロール、つまり権限を割り振ることで指定した権限を持つユーザーをテナントに追加できます。

ここでは例として、demoというテナントを作り、そのテナントにdemoユーザーを追加する流れを説明します。demoユーザーには利用に最低限必要なMemberの権限を割り当てます。

まず、登録済みのテナントを確認します。

````
# keystone tenant-list
+----------------------------------+----------+---------+
|                id                |   name   | enabled |
+----------------------------------+----------+---------+
| 2b0260a2580842abab33b56dae6a145f |  admin   |   True  |
| 20a6abed5e8549f29a76fa26b2b1c8db | services |   True  |
+----------------------------------+----------+---------+
（テナントリストの表示）
````

次にテナントを登録します。

````
# keystone tenant-create --name demo --description demo-tenant --enable true
（テナントdemoの作成）
````

追加したテナントが登録されていることを確認します。

````
# keystone tenant-list
+----------------------------------+----------+---------+
|                id                |   name   | enabled |
+----------------------------------+----------+---------+
| 2b0260a2580842abab33b56dae6a145f |  admin   |   True  |
| f1217f04d6f94a7ca3df1f4d6122322d |   demo   |   True  |
| 20a6abed5e8549f29a76fa26b2b1c8db | services |   True  |
+----------------------------------+----------+---------+
（テナントの確認）
````

demoユーザーを作成してみます。パスワードはdemoにします。

````
# keystone user-create --name demo --pass demo --tenant demo --email demo@example.com --enabled true
````

ユーザー作成コマンドはkeystone user-createです。
パラメータはいくつかあるので--helpで確認。--nameがユーザー名、--passがパスワード、--tenantはテナント(Horizonではプロジェクト)名、--enabledは有効化の可否を指定します｡

tenantはHorizonで登録されている「プロジェクト」を確認するか、コマンドではkerstone tenant-listで確認できます。
roleはkeystone role-listコマンドで確認できます。

例えば作成したdemoユーザーにMemberロール（権限）を割り当てるには次のように行います。

````
# keystone user-role-add --user demo --tenant demo --role Member
(demoユーザーをdemoテナントに追加してMemberロールを割り当て)
````

作成したテナント、ユーザーを最後に確認しましょう。

````
# keystone tenant-list
（テナントリストの確認）
# keystone user-list
（ユーザーリストの確認）
````

以上で、ユーザーdemoでOpenStack環境を利用可能になります。
ユーザーはadminユーザーが設定した共有ネットワークを使ってインスタンスでネットワーク接続できます。また、Floating IPを割り当てることで外部PCからインスタンスにアクセスできます。

もちろん、ユーザーが独自のネットワークを作ることもできます。その場合は次のように行います。

- ルーターを作成
- サブネットを作成
- 作成したサブネットとルーターをpublicネットワークとつなぐ
