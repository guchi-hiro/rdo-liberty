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
=============== ============ ============

前提条件
--------

VM上で構築するため以下の通り
Nested KVMが有効であることを確認


Nested VM設定確認（ホストマシン上）::

 # cat /sys/module/kvm_intel/parameters/nested
 Y

VM設定確認（ホストマシン上）::

 # virsh dumpxml <vm名> | grep host-passthrough
 <cpu mode='host-passthrough'>
