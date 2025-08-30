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
まずユーザーを作成します。CI/CDを構築しない人やGUIで全て済ませる人はやらなくても大丈夫ですが、後々のことを考えて作成することをおすすめします。

ユーザーの名前は好きなものを入れてください。今回は`Beanstalk_User`とします。
![](/images/cd4b014c4eebea/role_1.png)

次にユーザーに権限を付与していきます。
`ポリシーに直接アタッチする`を選んで、`AdministratorAccess-AWSElasticBeanstalk`を付与してください。

![](/images/cd4b014c4eebea/role_2.png)

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

![](/images/cd4b014c4eebea/role_3.png)

ポリシー名と説明には分かりやすい名前を付けてください。今回は`S3-Beanstalk-FullAccess`にします。(記入したスクショ取り忘れた。)
作成したら忘れずに付与してください。
これで権限は十分なのでユーザーを作成してください。

![](/images/cd4b014c4eebea/role_4.png)

# アクセスキー取得
作成したらアクセスキーを取得します。作成したユーザーをクリックして、`セキュリティ認証情報`にアクセスしてください。
そこのアクセスキーを作成をクリックしてください。

ユースケースは`コマンドラインインターフェイス (CLI)`を選択します。
推奨された代替案と出ると思いますが、`上記のレコメンデーションを理解し、アクセスキーを作成します。`にチェックを入れても大丈夫(なはず)です。

説明は分かりやすいのを入れてください。

次に進むとアクセスキーが生成されます。この画面を閉じるとアクセスキーは**二度と表示されないので**忘れずにメモしてください。
忘れた場合アクセスキーを作り直すことになります。
