================================
OpenStack Liberty Install Guide
================================

RDO(packstack)を利用してOpenStack環境を自動ビルドするためのガイド


構築環境
========

以下4台構成
今回はVMで構築
web serverはproxy環境下でのみ必要

=============== ============ ============ ===========
controller node network node compute node web server
=============== ============ ============ ===========
10.0.0.200      10.0.0.201   10.0.0.202   10.0.0.108
CentOS7.2       CentOS7.2    CentOS7.2    CentOS7.2
=============== ============ ============ ===========

0.前提条件
--------

VM上で構築するため
Nested KVMが有効であることを確認


Nested VM設定確認（ホストマシン上）::

 # cat /sys/module/kvm_intel/parameters/nested
 Y

VM設定確認（ホストマシン上）::

 # virsh dumpxml <vm名> | grep host-passthrough
 <cpu mode='host-passthrough'>

設定されていない場合はvirsh edit <vm名>でxmlを直接修正


1.事前準備
--------

web server以外のすべてのVM上で以下の作業を実施

プロキシ設定（必要であれば）::

 以下設定を/root/.bashrc等に追加
 
 export http_proxy=<proxy-server:port>
 export https_proxy=<proxy-server:port>
 export no_proxy=localhost,127.0.0.1,10.0.0.108,10.0.0.200,10.0.0.201,10.0.0.202

SELINUX無効化::

 # sed -i 's/SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
 # cat /etc/selinux/config

各種サービス起動設定::

 # systemctl disable firewalld
 # systemctl disable NetworkManager
 # systemctl enable network

リポジトリ用RPMインストール::

 # yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-liberty/rdo-release-liberty-3.noarch.rpm

パッケージ最新化および再起動::

 # yum -y update
 # reboot

2.事前準備(web server)
------------

web serverにhttpdをインストールし
適当な場所にglanceに登録するimageを格納::

 $ ls -l /var/www/html/ubuntu.img .
 -rw-r--r--. 1 root   root   1173094400  5月 23 11:16 /var/www/html/ubuntu.img

とりあえず上記の通り、DocumentRootに格納
packstack設定ファイルにて、上記イメージへのpathを設定する


3.PackStackインストール＆設定
--------

controller nodeのみでの作業

ssh鍵作成::

 # ssh-keygen 

packstackインストール::

 # yum install -y openstack-packstack

packstack設定ファイル作成::

 # packstack --gen-answer=param.txt

packstack設定ファイル修正::

 不要なサービスのインストールを無効化

 # sed -i 's/CONFIG_NAGIOS_INSTALL=.*/CONFIG_NAGIOS_INSTALL=n/' param.txt
 # sed -i 's/CONFIG_SWIFT_INSTALL=.*/CONFIG_SWIFT_INSTALL=n/' param.txt
 # sed -i 's/CONFIG_CEILOMETER_INSTALL=.*/CONFIG_CEILOMETER_INSTALL=n/' param.txt

 All in oneを想定した設定になっているため
 network node, compute nodeのipアドレスを設定

 # sed -i 's/CONFIG_NETWORK_HOSTS=.*/CONFIG_NETWORK_HOSTS=10.0.0.201/' param.txt
 # sed -i 's/CONFIG_COMPUTE_HOSTS=.*/CONFIG_COMPUTE_HOSTS=10.0.0.202/' param.txt

 登録するimageのpathを設定する
 
 # sed -i 's/CONFIG_PROVISION_IMAGE_URL=.*/CONFIG_PROVISION_IMAGE_URL=http:\/\/10.0.0.108\/ubuntu.img/' param.txt

4.PackStack実行＆事後設定
--------

controller nodeでpackstack実行::

 # packstack --answer-file=param.txt

 以下の通り、対象マシンへのSSHパスワードを聞かれるので入力

 Welcome to the Packstack setup utility

 The installation log file is available at: /var/tmp/packstack/20160525-011514-Zw9j6b/openstack-setup.log

 Installing:
 Clean Up                                             [ DONE ]
 Discovering ip protocol version                      [ DONE ]
 root@10.0.0.201's password:
 root@10.0.0.200's password:
 root@10.0.0.202's password:
 Setting up ssh keys                                  [ DONE ]
 Preparing servers                                    [ DONE ]

 あとは自動で進んでいく
 完了すると以下の通り表示される

 Applying Puppet manifests                            [ DONE ]
 Finalizing                                           [ DONE ]

 **** Installation completed successfully ******

 Additional information:
 * Time synchronization installation was skipped. Please note that unsynchronized time on server instances might be problem for some OpenStack components.
 * File /root/keystonerc_admin has been created on OpenStack client host 10.0.0.200. To use the command line tools you need to source the file.
 * To access the OpenStack Dashboard browse to http://10.0.0.200/dashboard .
 Please, find your login credentials stored in the keystonerc_admin in your home directory.
 * To use Nagios, browse to http://10.0.0.200/nagios username: nagiosadmin, password: 59bbb3dd3dd644d4
 * The installation log file is available at: /var/tmp/packstack/20160525-011514-Zw9j6b/openstack-setup.log
 * The generated manifests are available at: /var/tmp/packstack/20160525-011514-Zw9j6b/manifests


 admin用のcredientialとdemo用のcredientialが出力されるので
 それを読み込み各種コマンドを実行する

 # source keystonerc_admin
 # nova list
 +----+------+--------+------------+-------------+----------+
 | ID | Name | Status | Task State | Power State | Networks |
 +----+------+--------+------------+-------------+----------+
 +----+------+--------+------------+-------------+----------+

compute node上でvirt-typeの設定を行う

virt-type設定::

 VM上でpackstackを実行すると自動でvirt_type=qemuとなってしまうので
 これをkvmに修正
 compute node上で実施

 # sed -i 's/^virt_type=.*/virt_type=kvm/' /etc/nova/nova.conf
 # openstack-service restart nova

network node上でbr-exの設定を行う

packstack構築完了時点ではbr-exはどのnicにもアタッチされていないため
手動で行う必要がある

br-ex設定::

 # cat <<EOF | tee /etc/sysconfig/network-scripts/ifcfg-br-ex
 DEVICE=br-ex
 DEVICETYPE=ovs
 TYPE=OVSBridge
 BOOTPROTO=static
 IPADDR=10.0.0.201
 NETMASK=255.255.255.0
 GATEWAY=10.0.0.1
 ONBOOT=yes
 EOF

eth0設定::

 # cat <<EOF | tee /etc/sysconfig/network-scripts/ifcfg-eth0
 DEFROUTE=yes
 PEERDNS=yes
 PEERROUTES=yes
 IPV4_FAILURE_FATAL=no
 NAME=eth0
 DEVICE=eth0
 ONBOOT=yes
 NM_CONTROLLED=yes
 TYPE=OVSPort
 OVS_BRIDGE=br-ex
 EOF

br-ex アタッチ::

 アタッチすると疎通が切れるのでそのまま再起動させる

 # ovs-vsctl add-port br-ex eth0 && sleep 5 && reboot
