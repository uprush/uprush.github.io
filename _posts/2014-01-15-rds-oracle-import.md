---
layout: post
title: "EC2にOracle XEをインストールしデータインポート"
description: "EC2にOracle XEをインストールしデータインポート"
category: "cloud"
tags: [aws, rds, oracle]
---
{% include JB/setup %}

オンプレミスでダンプしたOracle DatabaseのデータをAmazon Relational Database Service (Amazon RDS) for Oracleにインポートする方法をまとめてみます。無償で利用可能のOracle XEをEC2にインストールし、付属のimpコマンドでデータインポートします。

## Oracle XE用のEC2を起動する

以下の条件でOracle XEインストール用のEC2をManagement Consoleから起動する。

- AMI: Amazon Linux AMI 2013.09.2
- 64bit
- Size: m1.small 以上（Swap領域が2GB以上必要のためt1.microは不可）
- Swap領域を増やすため、エフェメラルストレージもつけて起動して頂く必要あります
  - 方法：EC2のラウンチ画面の「Step 4: Add Storage」にて、「Add New Volume」をクリック、「Type」欄のドロップボックスより「Instance Store 0」を選んで次へ進みます


EC2起動が完了したら以下のようにログインしディスクを確認

    $ ssh ec2-user@c2-xx-xxx-xxx-xxx.us-west-2.compute.amazonaws.com
    $ df -h
    Filesystem            Size  Used Avail Use% Mounted on
    /dev/xvda1            7.9G  3.3G  4.6G  42% /
    tmpfs                 829M  395M  435M  48% /dev/shm
    /dev/xvdb             147G  4.2G  136G   4% /media/ephemeral0


## Oracle XEをダウンロード
[Oracle XE 11g release 2 for Linux x64](http://www.oracle.com/technetwork/jp/products/express-edition/downloads/index.html) をダウンロード。

ダウンロードしたzipファイルをEC2にアップ

    $ scp oracle-xe-11.2.0-1.0.x86_64.rpm.zip ec2-user@ec2-xx-xxx-xxx-xxx.us-west-2.compute.amazonaws.com


## OSセットアップ
EC2のSwap領域を増やす。一時的作業なので、ファイルベースのswap領域を作成します。

    $ ssh ec2-user@c2-xx-xxx-xxx-xxx.us-west-2.compute.amazonaws.com
    $ sudo su
    $ dd if=/dev/zero of=/media/ephemeral0/swapfile bs=1M count=4096
    $ chmod 600 /media/ephemeral0/swapfile
    $ mkswap /media/ephemeral0/swapfile
    $ echo /media/ephemeral0/swapfile none swap defaults 0 0 | tee -a /etc/fstab
    $ swapon -a


必要なパッケージをインストール

    $ yum install -y binutils


## Oracle XEインストール

以下のインストールコマンドを実行する。

    $ cd /usr/local/src
    $ unzip oracle-xe-11.2.0-1.0.x86_64.rpm.zip
    $ cd Disk1/
    $ rpm -hiv oracle-xe-11.2.0-1.0.x86_64.rpm
    $ /etc/init.d/oracle-xe configure

Oracle user設定

    $ su - oracle
    $ cd /u01/app/oracle
    $ cp /etc/skel/.bash_profile ./

以下の内容を /u01/app/oracle/.bashrc に追加

    if [ -f /etc/bashrc ]; then
      . /etc/bashrc
    fi
    export ORACLE_ROOT='/u01/app/oracle/product/11.2.0/xe'
    . $ORACLE_ROOT/bin/oracle_env.sh
    export LD_LIBRARY_PATH="$ORACLE_HOME/lib/:/lib:/usr/lib"

Oracle clientの設定(tns nameやSID)をRDS Oracle用に下記ファイルを編集（オンプレミスと同じ、詳細は割愛します）。

- $ORACLE_ROOT/network/admin/tnsnames.ora
- $ORACLE_ROOT/bin/oracle_env.sh

設定を反映する。

    $ source /u01/app/oracle/.bashrc

## データインポート

Impコマンドが実行可能になりますので、EC2からRDS Oracleに接続可能か確認したうえ、ダンプしたファイルをEC2にアップして、以下のようにインポートを実行して頂く。

    $ scp ora.dmp ec2-user@c2-xx-xxx-xxx-xxx.us-west-2.compute.amazonaws.com:~/
    $ ssh ec2-user@c2-xx-xxx-xxx-xxx.us-west-2.compute.amazonaws.com
    $ sudo su - oracle
    $ imp myuser/mypassword file=/home/ec2-user/ora.dmp fromuser=user01 touser=user02

コピーを飲みながらインポート完了を待ちます…