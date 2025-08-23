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

https://aws.amazon.com/jp/elasticbeanstalk/pricing/

## Amazon EC2
2025年12月31日までt4g.smallが月750時間使えるみたいです。
この無料枠はしれっと毎年伸ばされているみたいです。多分来年も延ばされます。(希望的観測)

https://aws.amazon.com/jp/ec2/faqs/?nc1=h_ls#t4g-instances

↓過去のアーカイブ
https://web.archive.org/web/20240305041107/https://aws.amazon.com/jp/ec2/faqs/?nc1=h_ls#t4g-instances

