---
layout: post
title: "CloudFrontとS3によるプライベートコンテンツ配信"
description: "S3とCloudFrontのSigned-URLを組み合わせてプライベートコンテンツ配信"
category: "CDN"
tags: [aws, cloudfront, cdn]
---
{% include JB/setup %}

Amazon CloudFrontはコンテンツ配信ネットワークサービスです。Amazon Simple Storage Service (S3)はオンラインストレージサービスです。この2つのAWSサービスを組み合わせて安全にプライベートコンテンツを配信することが簡単にできます。

今回は認証済みのユーザーにのみビデオを閲覧させるシステムを想定して、CloudFrontとS3によるプライベートコンテンツ配信を解説する。システムのアーキテクチャは以下のイメージです。ビデオをS3に保存しCloudFrontからのアクセスのみ許可する。認証サーバーにて認証済みのユーザーのみ、CloudFrontの署名付きURLを発行する。ユーザーは署名付きURLでCloudFrontにアクセス、CloudFrontは署名が確認できた場合のみビデオを返す仕組みです。
![cf-signed-url](https://s3-ap-northeast-1.amazonaws.com/static.uprush.net/images/cf_signedURL.jpg)

## 事前準備
S3やCloudFront Key Pairの事前準備が必要。

### S3事前準備
S3にサンプルビデオをアップロードし、非公開にする。

- Originコンテンツ用S3バケットがあること
  - 今回はサンプルとしてバケット名に static.example.net を利用する
- S3バケットは非公開であること

- サンプルビデオをS3にアップロード
  - static.example.net/videos/sample.mp4
  - 確認にブラウザからビデオをアクセスする
    - URL(東京リージョンの場合): https://s3-ap-northeast-1.amazonaws.com/static.example.net/videos/sample.mp4
    - 想定レスポンス: AccessDenied

### CloudFrontのキーペアを取得
署名付きURL作成のためにCloudFront Key Pairが必要です。EC2のKey Pairと異なるので間違いないように。

- AWSのWebサイトのAccounts画面のSecurity Credentialsをクリック。
- AWSアカウントでログインするとSecurity Credentials画面が表示される。
- CloudFront Key Pairsエリアで、Create New Key Pairをクリック。
- 表示に従い、鍵を作成

## CloudFrontの設定
AWS Management Console画面からCloudFrontの管理画面にて以下作業を行う。

### Origin Access Identityを作成
S3にアクセスするためのOrigin Access Identity(OAI)を作成する。最終的にはこのOAIからのアクセスのみS3のバケットのほうで許可する形になる。

- CloudFrontの左メニューのOrigin Access Identityをクリック
- 次の画面のCreate Origin Access Identityをクリック、コメントを入力しOAIを作成する。

### Distributionの作成
Create Distributionをクリックし、Web Distributionを新規作成。以下の通り設定する。

- Origin Settings
  - Origin Domain Name: S3バケット（static.example.net.s3.amazonaws.com）
  - Restrict Bucket Access: Yes
  - Origin Access Identity: Use an Existing Identity
  - Your Identities: 先ほど作成したOrigin Access Identityを選ぶ
  - Grant Read Permissions on Bucket: Yes, Update Bucket Policy
- Default Cache Behavior Settings
  - Restrict Viewer Access (Use Signed URLs): Yes
  - Trusted Signers: Self

- その他設定: 初期値のまま

### 非公開配信確認
上記設定で、DistributionのURLそのままビデオへのアクセスが禁止されていることを確認。URLはDistribution Settings画面のDomain Name項目値とビデオファイルのパスの組み合わせです。

- URL: http://xxxx.cloudfront.net/videos/sample.mp4
- 想定レスポンス: MissingKey

## Signed-URLによるプライベート配信
認証済みユーザーに署名付きURL（Signed-URL）を生成する方法を解説する。

### Signed-URL作成
[公式ドキュメントのサンプル](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateCFSignatureCodeAndExamples.html)を参考に、署名付きURLを作成する。

上記ドキュメントにPerl、PHP、C#、Javaのサンプルのなか、今回はPerlのほうを利用する。

[Amazon CloudFront Signed URLs helper tool](http://aws.amazon.com/code/3052?_encoding=UTF8&jiveRedirect=1)をダウンロードする。ダウンコートした cfsign.pl ファイルに実行権限を与える。

    $ chmod +x cfsign.pl

署名付きURLを生成。

    $ ./cfsign.pl --action encode --url http://xxxx.cloudfront.net/videos/sample.mp4 --private-key <your_cloudfront_secret_key.pem> --key-pair-id <your_cloudfront_access_key> --expires <timestamp_to_expire_the_URL>

上記コマンドのサンプル出力は以下の通りです。ユーザーにこのURLでビデオにアクセスさせるのでブラウザでアクセス可能かを確認する。

    Encoded URL:
    http://xxxx.cloudfront.net/videos/sample.mp4?Expires=1393947953&Signature=E5K4zhOVGrEOxueQ8-uHX4WOwfK4qWQBRZoSYVrTKUdPcgaLH~6xWBZmLa7BJEImXV~TFfcYIUFXkJo6TvUNSo2V-cxonyH9JunfbSn5HNAEICPe6WRbnZSWRLAEonsBeC2fX-X6PgttTVHfo~ZxQeBQb3G8zQqaG4q~gZg-q0Nk45RgWRa1A5ELFF7vLWqLgEcyuJMZ7Kej0vlyaswjsP97UZdTedAbzBZFfpYRC5d0617fSun1kTITfQMBjsmXiu3aY0hBHRVAWBzGzwpnYe0XllxeKwqcSCk36pBVjraYiScJ3jX8FskehqYs9eJAMk5c6FyfHoSRE1jV0En69A__&Key-Pair-Id=APKAIYQJDU3U6HJKX3XQ

想定レスポンス：ビデオが再生される。

## 次のステップ
CloudFrontとS3によるプライベートコンテンツ配信の簡単な紹介でした。
今回はサンプルとしてコマンドで署名付きURLを生成したが、実際のシステムは、プログラムで認証済みユーザーごとに署名付きURLを生成して渡し、CloudFrontからセキュアにビデオを配信する構成が多い。更に、ユーザーの認証情報や署名付きURLが盗聴されないためHTTPS通信が望ましいです。

また、今回は静的コンテンツの例でしたが、CloudFrontを使ったメディアのストリーミングのプライベート配信も可能です。
