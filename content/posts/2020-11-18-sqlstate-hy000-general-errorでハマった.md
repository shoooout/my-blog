---
template: post
title: "SQLSTATE[HY000]: General errorでハマった"
slug: laravel-error-hy000
socialImage: /media/image-0.jpg
draft: false
date: 2020-02-28T12:30:21.470Z
description: Laravelでマイグレーションファイルを作成中に、その作成の順番でハマってしまったのでその備忘録です。
category: Laravel
tags:
  - Laravel
---
# ハマったこと

マイグレーションファイルを作成中のこと。外部制約キーを記述し、いざ`$ php artisan migrate`をすると、<font color="Red">SQLSTATE\[HY000]: General error</font>のエラーが。
参照元のテーブルや型の部分で間違いがあるのかと思い何度も確認したが間違いはなさそう。

# 原因

結論から述べると、<font color="Blue">マイグレーションファイルの順番（作成の順番）</font>がまずかったようです。
最初のマイグレーションファイルがこれ↓

<img width="383" alt="スクリーンショット 2020-02-28 17.12.51.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/308184/ed9411d2-131e-d50f-5d58-9fc8da7872b6.png">

`issues`から`categories`を参照したかったのにこの順番になっていたので、マイグレーションの際にエラーが出ていたようです。

コマンドから生成されるマイグレーションファイルは`YYYY_MM_DD_hhmiss_create_テーブル名s_table.php`のような名前になっているので頭の日付の部分をいじって順番を変えて上げると解決します。こんな感じ↓

<img width="383" alt="スクリーンショット 2020-02-28 17.12.28.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/308184/38beaa82-ff38-d067-c2e7-132f89b614c0.png">

前述したように、`issues`から`categories`を参照したいので、先にcategoriesをもってきます。
そうして`$ php artisan migrate:refresh`をするとうまくいきました。



というか、コマンドでマイグレーションファイルを作れば良いのにここのときだけなぜか手動で作っていたのでこんなことになりました。

# まとめ

とても初歩的なところだと思いますが、Laravel初心者の僕は引っかかってしまったので参考までに。
アドバイス・ご指摘など有りましたらよろしくおねがいします！