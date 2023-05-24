---
title: "alpine + php7.3(バージョン選べる) + apacheなdocker image作った"
emoji: "🐕"
type: "tech"
topics:
  - "docker"
  - "php"
  - "apache"
published: true
published_at: "2021-01-27 15:01"
---

# TL;DR

```
docker pull diwamoto0118/php7.3-apache-alpine 
```
で利用できます。

# 概要
dockerの公式イメージの`php{PHP_VERSION}-apache`ってビルド長すぎませんか？
僕は結構環境をdockerfileからいじるので、毎回ビルドに悩まされてとても面倒でした。

そこで見つけたのがalpine Linuxです。癖がありますがビルド時間がとても早い！
ですがちょうどいいイメージが公式から提供されていないんですよね、php-apache-alpineみたいな。
なので作りました。という感じです。作りはとても簡単で、公式のphp-alpineイメージにapacheを突っ込んで動かしているだけです。これを使ってビルド時間を短縮しよう！

# リンク集
https://github.com/Diwamoto/php7.3-apache-alpine
https://hub.docker.com/repository/docker/diwamoto0118/php7.3-apache-alpine