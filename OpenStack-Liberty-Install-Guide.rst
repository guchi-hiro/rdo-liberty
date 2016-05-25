================================
OpenStack Liberty Install Guide
================================

RDO(packstack)を利用してOpenStack環境を自動ビルドするためのガイド


構築環境
========

以下３台構成
今回はVMで構築

=============== ============ ============
controller node network node compute node
=============== ============ ============
10.0.0.200      10.0.0.201   10.0.0.202 
CentOS7.2       CentOS7.2    CentOS7.2
=============== ============ ============

0.前提条件
--------

VM上で構築するため以下の通り
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

すべてのVM上で以下の作業を実施の上、再起動

プロキシ設定（必要であれば）::

 以下設定を/root/.bashrc等に追加
 
 # export http_proxy=<proxy-server:port>
 # export https_proxy=<proxy-server:port>
 # export no_proxy=localhost,127.0.0.1,10.0.0.108,10.0.0.200,10.0.0.201,10.0.0.202

SELINUX無効化::

 # sed -i 's/SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
 # cat /etc/selinux/config

各種サービス起動設定::

 # systemctl disable firewalld
 # systemctl disable NetworkManager
 # systemctl enable network

リポジトリ設定用RPMインストール::

 # yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-liberty/rdo-release-liberty-3.noarch.rpm

パッケージ最新化::

 # yum -y update

2.PackStackインストール＆設定
--------

controller nodeのみでの作業

ssh鍵作成::

 # ssh-keygen 

packstackインストール::

 # yum install -y openstack-packstack

packstack設定ファイル作成::

 packstack --gen-answer=param.txt
