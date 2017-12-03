---
layout: post
title: "Hadoop in the Cloud"
description: "Hadoop meets AWS"
category: "big-data"
tags: [hadoop, aws, emr, big data]
---
{% include JB/setup %}

この投稿は[Hadoop Advent Calendar](http://qiita.com/advent-calendar/2013/hadoop#)の17日目の記事です。
Elastic MapReduceを宣伝するような投稿になりそうですが、私自身がHadoopをオンプレミスで導入・運用したときの煩雑さを思いながら書きました。技術ネタが少ないので、後はWebでということで。。。

Hadoopはビッグデータを扱うシステムの開発コストを劇的に下げました。では、ビッグデータシステムの運用コストはどうでしょうか？

## Hadoopの導入と運用は簡単ではない

Hadoopを導入する場合のシナリオを考えてみましょう。キャパシティ・プランニング、見積もり、サーバー調達、ラックへ設置、電源とネットワークを設定、OSインストールと基本設定、Hadoopインストールとクラスタ構築、この一連のプロセスを通して、一ヶ月ぐらいはかかるだろうか、やっとHadoopを使い始められるわけですね。その中でも、特に難しいのはキャパシティ・プランニングではないかと思います。最適なサーバースペックや数をどう決めるか、投資に見合った効果は本当に出せるか、Hadoop導入を検討する際最初のハードルとなります。

一方で、ビッグデータの分析そのものが複雑であり、すば速いレスポンスでトライ・アンド・エラーをもっとしたい。収集、保存、解析、アクションのサイクルをもっと早くしたい、このようなユーザーの要望はよく聞きます。

また、Hadoopを導入した後も、冗長化、故障したHWの交換、データバックアップなど、煩雑な運用作業が残ります。本来、煩雑な運用作業ではなく、付加価値の高いコアビジネス（データ分析）に集中したい。

## Hadoop in the Cloud

このようなHadoop運用管理の煩雑さとコストを削減すべく、Hadoopをクラウド上でデプロイ、あるいはフルマネージドのサービスとして提供するケースが出ています。

クラウド上でHadoopを使う利点として：

- Hadoopを評価するコストが抑える。
- リソース調達しやすい。Hadoopは大量リソースが必要となる場合が多いが、クラウドなら素早くプロビジョニングできます。
- 定期的に実行するバッチシステムでは、一時的にクラスタを拡大／縮小することでコストを抑える。
- ハードウェア運用から開放される。

こんな背景の中で、AWSのHadoop as a ServiceとしてAmazon Elastic MapReduce (EMR)があります。

## Amazon Elastic MapReduce (EMR)

EMRはHadoopをオンデマンドでいつでも実行可能にしています。使った分だけ料金かかりますので、一日数回しかジョブを起動しないようなバッチシステムでは、オンプレミスとくらべて低コストの場合があります。また、コンソール画面から数クリックするだけで、数分後クラスタを起動できますので、開発者は分析・解析に集中できます。

EMRはApache HadoopやMapR２つのディストリビューションから選べます。それぞれ、Hadoop、HBase、HiveやPigを対応しています。また、先週ですね、あのCloudera Impalaもサポートするようになりました。


![EMRの全体的なアーキテクチャ](https://s3-us-west-2.amazonaws.com/yifeng-public/images/emr-architecture.png)

EMRはMaster Node、Core NodeやTask Node３つのコンポネントからなります。

### Master Node
EMRのmaster nodeはNamdNode、JobTrackerやEMRのservice endpointの役割を果たします。EMRでHBaseを使う場合、HMaserとHBase managed ZooKeeperもmaster nodeに起動します。Master nodeは一台のみ起動し、HA機能は今のところサポートしていません。そのかわりに、S3にデータを保持するか、定期的にデータを自動的にS3にバックアップする(Apache HBase with EMRの場合)アプローチを取っています。

Master nodeにSSHでログインして、ジョブを実行するイメージとなりあす。

    $ ssh hadoop@xxxx.compute.amazonaws.com
    $ hive

### Core Node
Core NodeはHadoop DataNodeやTaskTrackerに相当します。データを持ちづつmap reduce taskも実行します。EMRクラスタを起動するときに、スペックと台数を指定します。

### Task Node
Task NodeはHadoop TaskTrackerのみの機能です。データを持っていないので、ジョブ実行中も含めて、オンラインでノード追加・削除できます。例えば、何時もより「ジョブが遅い、結果早く出してくれ」って言われた場合ですね、Task Nodeを追加すればよいかと思います。

### AWSサービスとの連携

EMR最大の利点とも言えるのはS3との連携です。Hadoopは多いの場合HDFSにデータを保存しますが、この場合のHDFS上の大量のデータの可用性の確保やバックアップなどの運用が必要です。特に、NamenodeのHAは重要です。EMRの場合、S3をinput/outputにできます。S3は可用性が非常に高く、また容量も無制限なので、データをロストすることなく保護します。Hadoop adminにとっては非常に嬉しい機能ではないかと思います。

一方で、map reduce job実行時、常にリモート（S3）からデータをとってきて処理しますので、datanodeとtask trackerのデータのローカリティによる性能アップは期待できません。同じデータを何回も処理する場合、一旦S3から、EMRのHDFSにコピーして処理したほうがよりでしょう。

S3以外にも、AWSのその他ビッグデータ系サービス、例えばNoSQLのDynamoDB、データウェアハウスのRedShift、EMRが連携しやすい形となっています。全体的に、こんなイメージになります。

![Big data on AWS](https://s3-us-west-2.amazonaws.com/yifeng-public/images/big-data-on-aws.png)

### S3DistCp

S3DistCpはHadoop DistCpをS3に最適化したツールです。S3とHDFSの間、またはS3のbucketの間、大量のデータをコピーするときにぜひ使って貰いたいです。DistCpと同様、map reduceでパラレルにコピーしますが、S3アクセスのHTTP部分の最適化をした結果、S3のデータコピーのスループットは高い。

また、データをコピーしながら、複数の小さいファイルを圧縮し１つに結合できます、結果的に結合したファイルに対してhadoopの処理が早くなります。よく利用するシーンは、大量のサーバーからS3に集めたログファイル、一個一個は小さいが、S3DistCpを使って、1.5GB~2GBの少ない圧縮ファイルに結合したうえ、EMRで分析します。

    ./elastic-mapreduce --jobflow j-T1ACLO7Y39R2 --jar \
    /home/hadoop/lib/emr-s3distcp-1.0.jar \
    --arg --src --arg 's3://rosetta-logs/apache/' \
    --arg --dest --arg 's3://rosetta-logs/archive/' \
    --arg --outputCodec --arg 'lzo' \
    --arg --groupBy --arg '.*\.ip-.*\.([0-9]+-[0-9]+-[0-9]+)T.*\.txt' \
    --arg --targetSize --arg 12288

こんな形で、同じ日付（`--groupBy`）のデータを、LZOで圧縮し、大きいサイズ（`--targetSize`）に結合します。

## まとめ

気軽にHadoopクラスタをクラウド上に立ち上げ、データ分析をすぐ始められる、使った分だけ料金がかかる、このようなElastic MapReduceサービスのご紹介でした。

さて、明日の Hadoop アドベントカレンダーは [@hamaken](https://twitter.com/hamaken) さんです。お楽しみに！

