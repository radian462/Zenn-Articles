---
title: "EC2+Elastic BeanstalkでDiscord Botを無料で常時起動"
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

要はめんどいデプロイの作業を自動でやってくれるサービスです。
Elastic BeanstalkはGo、Java、.NET、Node.js、PHP、Python、RubyのほかにもDockerコンテナも動かせるみたいです。

https://aws.amazon.com/jp/elasticbeanstalk/



