---
title: "macを買ったのでbaserCMSの開発環境を最速で構築する"
emoji: "💬"
type: "tech"
topics:
  - "docker"
  - "vagrant"
  - "mac"
  - "lamp"
  - "basercms"
published: true
published_at: "2021-02-12 12:33"
---

# macを買いました

今までずっと欲しかったのでやっと手に入れられてとても嬉しいです。
M1と迷ったんですが、今回の手順のどこかで絶対つまづくだろうと思ったので安定のintelにしました。
先にいっとくと結果としてはM1まだ必要ないかなって思うくらい満足してます。買ってよかった。

今回はそのmacに自分なりに最速でbasercms(というかlamp環境)の開発環境を整えたので今後の備忘録というかまあそういう感じです。

# やることリスト
- 必要な物をいろいろインストールする
- githubからbasercmsとローカル環境を落とす、そのための設定
- シェルスクリプト叩く

これだけです
では行きます。

## 必要な物をインストールする

その他にもまずmacを買った時にやった方がいいことや、必要なものもありますが、とりあえず最短ルートで行きます。

### Command Line Tools
まずはこれですね。適当にターミナルを開いて`git`と打つと勝手にインストールしてくれます。便利。
多分少しだけ時間がかかるので次いきましょう

### VirtualBox
仮装マシンを起動するために必要です。
[こちら](https://www.virtualbox.org/)からダウンロードしてインストールまでやりましょう。
docker for macを使うパターンもありますが、docker on vagrantの方が早いのでそっちで行きます。ここはお好みで。

### vagrant
上のvirtualboxで作った仮想マシンをコマンドラインから扱いやすくする感じのやつですね。
dockerが出る前はこれが主流だったんじゃないですか？知らんけど

### VScode
basercmsの初期設定をするために必要です。
[こちら](https://azure.microsoft.com/ja-jp/products/visual-studio-code/)からダウンロードしてこれもインストール！！

### Chrome
これは必須ではないです。だけど今後の開発は100%必要なのでこの際に入れましょう。safariの役目はこれで終わり。

この辺でもうCommand Line Toolsのインストールは終わってるとおもうので次いきましょう

## githubからbasercmsとローカル環境を落とす、そのための設定

はい。まずgithubからssh経由でレポジトリを落としてくるために、sshの共有鍵をgithubにアップロードする必要があります。
なのでその鍵を作りましょう。
```
% ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/〇〇/.ssh/id_rsa): github-key
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in github-key.
Your public key has been saved in github-key.pub.
The key fingerprint is: ...
```
これでsshの共有鍵は完成です。これをgithubで追加しましょう。ググれば出てくるのでやり方は割愛します。

追加出来たら、sshの設定を書き込みましょう。
```
% echo 'Host github github.com
    HostName github.com
    User git
    Identityfile ~/.ssh/github-key' >> ~/.ssh/config
```
このコマンドで一発でできると思います。
一応`ssh github`で繋いだかどうかの確認ができます。不安な人はやりましょう。

ここまでできたらgithubから必要な物が落とせる様になったと思いますので、必要な物をどんどん落としていきましょう。
まずローカル環境を落としましょう。
私のレポジトリにあるvagrant-lampを入れると早いです。（当社比）

macのディレクトリ配置は以下の様にします。
```
~ 
> tree
├── vagrant-lamp
│   ├── Vagrantfile
│   └── docker-lamp (lamp本体へのシンボリックリンク)
│
└── Projects
    ├── basercms
    └── その他lamp環境で使うもの
```

以下のコマンドを順に実行することで
この環境を作ることができます。
```
cd 
git clone git@github.com:Diwamoto/vagrant-lamp.git
touch Projects
cd Projects
git clone git@github.com:baserproject/basercms.git
```
これで必要な物はインストールできました。では、あとはシェルスクリプトに頑張ってもらいましょう。

```
cd
cd vagrant-lamp
./init.sh
```
もし、init.shの実行権限がないようであれば、
```
chmod +x ./init.sh
```
と打つと実行権限を与えることができます。
このシェルスクリプトを叩くだけでvagrantの仮想マシン上にdockerコンテナが起動するので、ブラウザから
`basercms.localhost`と叩きましょう。

できましたね。
vagrant-lampはvagrantの仮想マシン上にbargeというdockerを立てるためだけみたいなlinuxディストリビューションを使用しておりますので、他のdockerで作成されたプロジェクトもこちらで立てることができます。docker for mac を使用して開発するより数段早いと思います。


お疲れ様でした。よりよいbasercmsとmacライフを!
