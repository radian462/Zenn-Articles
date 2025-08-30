---
title: "EC2+Elastic BeanstalkでDiscord Botを(ほぼ)無料で常時起動"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Elastic Beanstalk", "Discord", "CI/CD", "EC2"]
published: false
---

EC2のt4g.smallが無料で使えるという情報を聞いて、Discord Botを動かす方法をまとめました。
EC2で直接起動してもいいのですが、Elastic Beanstalkというのを使うと楽にデプロイできたので今回はこれ経由でデプロイしてみます。

# 用意するもの
- AWSアカウント
- Githubアカウント
- Dockerでコンテナ化したDiscord Bot

# テストコード

https://github.com/radian462/Test-Bot-AWS

# そもそもElastic Beanstalkってなんぞや
>AWS Elastic Beanstalk は、AWS でウェブアプリケーションを立ち上げて稼動させるのに最も速い方法です。アプリケーションのコードをアップロードするだけで、リソースのプロビジョニング、ロードバランシング、オートスケーリング、モニタリングなどの細かい作業はサービスが自動的に処理します。(原文ママ)

要はめんどい環境構築を自動でやってくれるサービスです。
Elastic Beanstalk(以下EBと表記します)はGo、Java、.NET、Node.js、PHP、Python、RubyのほかにもDockerコンテナも動かせるみたいです。

https://aws.amazon.com/jp/elasticbeanstalk/


# 無料枠
## Elastic Beanstalk
無料ですが、EBが使うリソースには課金が発生します。
課金が発生するリソースはEC2とS3、あと使うならデータベースサービスなどがあるみたいです。

https://aws.amazon.com/jp/elasticbeanstalk/pricing/

## Amazon EC2
2025年12月31日までt4g.smallが月750時間使えるみたいです。
この無料枠はしれっと毎年伸ばされているみたいです。多分来年も延ばされます。(希望的観測)
t4g.smallのスペックはこんな感じです。

| [**t4g.small**](https://aws.amazon.com/jp/ec2/instance-types/t4/) |  | 
| ---- | ---- | 
| vCPU | 2vCPU |
| メモリ | 2GiB |

https://aws.amazon.com/jp/ec2/faqs/?nc1=h_ls#t4g-instances

↓過去のアーカイブ
https://web.archive.org/web/20240305041107/https://aws.amazon.com/jp/ec2/faqs/?nc1=h_ls#t4g-instances

## Amazon S3
無料枠はありません。タイトルにほぼってあるのはこれが理由です。
ただ、超でかく見積もって合計5MBのBotだったとしても、USD 0.025/GB×約0.005=USD 0.000125(≒0.018円)と1円にも満たないです。
あと、AWSサービス内の通信は無料だったり、受信は無料だったりといろいろあるのですが話が長くなるので下のリンクから確認してください。

https://aws.amazon.com/jp/s3/pricing/

# ユーザー作成
## ユーザーを作成する
まずユーザーを作成します。CI/CDを構築しない人やGUIで全て済ませる人はやらなくても大丈夫ですが、後々のことを考えて作成することをおすすめします。

ユーザーの名前は好きなものを入れてください。今回は`Beanstalk_User`とします。
![](/images/cd4b014c4eebea/user_1.png)

次にユーザーに権限を付与していきます。
`ポリシーに直接アタッチする`を選んで、`AdministratorAccess-AWSElasticBeanstalk`を付与してください。

![](/images/cd4b014c4eebea/user_2.png)

これだけだとeb cliでアクセスしたときに、S3の書き込み権限がないってエラーが出たので、S3の権限も付与します。S3のフルアクセスは過剰なので、EBだけのファイルにアクセスできるようにします。(これでも過剰かもしれないけど)

上の画像のポリシーを作成から、ポリシーエディタをJSONに変えてこれを書き込んでください。
```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": "s3:*",
			"Resource": [
				"arn:aws:s3:::elasticbeanstalk-*",
				"arn:aws:s3:::elasticbeanstalk-*/*"
			]
		}
	]
}
```

![](/images/cd4b014c4eebea/user_3.png)

ポリシー名と説明には分かりやすい名前を付けてください。今回は`S3-Beanstalk-FullAccess`にします。
作成したら忘れずに付与してください。
これで権限は十分なのでユーザーを作成してください。

![](/images/cd4b014c4eebea/user_4.png)

## アクセスキーを取得する
ユーザーを作成したらアクセスキーを取得します。作成したユーザーをクリックして、`セキュリティ認証情報`にアクセスしてください。
そこのアクセスキーを作成をクリックしてください。

![](/images/cd4b014c4eebea/access_key_1.png)

ユースケースは`コマンドラインインターフェイス (CLI)`を選択します。
推奨された代替案と出ると思いますが、`上記のレコメンデーションを理解し、アクセスキーを作成します。`にチェックを入れても大丈夫(なはず)です。

![](/images/cd4b014c4eebea/access_key_2.png)

![](/images/cd4b014c4eebea/access_key_3.png)

説明は分かりやすいのを入れてください。
![](/images/cd4b014c4eebea/access_key_4.png)

次に進むとアクセスキーが生成されます。この画面を閉じるとアクセスキーは**二度と表示されないので**忘れずにメモしてください。
忘れた場合アクセスキーを作り直すことになります。あとこのアクセスキーをGithubに上げたりしないよう保管してください。
このアクセスキーは後で使います。

![](/images/cd4b014c4eebea/access_key_5.png)

## アクセスキーをチェックする
ユーザーで正しくeb cliを操作できるかチェックしてみましょう。これは今はそこまで重要でないので面倒くさいなら一旦飛ばしても構いません。
pipでaws cliとeb cliをインストールしてみましょう。

```bash
pip install awscli awsebcli -U
```

アクセスキーを使ってログインした後、`eb init`してリージョンを選ぶように言われたら正しく設定されています。
この記事では解説しませんが、CLIで作成するならこのまま続けても構いません。

```bash
$ aws configure
AWS Access Key ID [None]: [アクセスキーのID] 
AWS Secret Access Key [None]: [アクセスキーのシークレット]
Default region name [None]: [リージョン]
Default output format [None]: json

$ eb init

Select a default region
1) us-east-1 : US East (N. Virginia)
2) us-west-1 : US West (N. California)
3) us-west-2 : US West (Oregon)
4) eu-west-1 : EU (Ireland)
5) eu-central-1 : EU (Frankfurt)
6) ap-south-1 : Asia Pacific (Mumbai)
7) ap-southeast-1 : Asia Pacific (Singapore)
8) ap-southeast-2 : Asia Pacific (Sydney)
9) ap-northeast-1 : Asia Pacific (Tokyo)
10) ap-northeast-2 : Asia Pacific (Seoul)
11) sa-east-1 : South America (Sao Paulo)
12) cn-north-1 : China (Beijing)
13) cn-northwest-1 : China (Ningxia)
14) us-east-2 : US East (Ohio)
15) ca-central-1 : Canada (Central)
16) eu-west-2 : EU (London)
17) eu-west-3 : EU (Paris)
18) eu-north-1 : EU (Stockholm)
19) eu-south-1 : EU (Milano)
20) ap-east-1 : Asia Pacific (Hong Kong)
21) me-south-1 : Middle East (Bahrain)
22) af-south-1 : Africa (Cape Town)
23) ap-southeast-3 : Asia Pacific (Jakarta)
24) ap-northeast-3 : Asia Pacific (Osaka)
25) il-central-1 : Israel (Tel Aviv)
26) me-central-1 : Middle East (UAE)
(default is 3): 
```