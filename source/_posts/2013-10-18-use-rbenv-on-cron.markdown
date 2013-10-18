---
layout: post
title: "CentOSでrbenvを使ってバッチのRubyバージョンを切り替える"
date: 2013-10-18 20:47
comments: true
categories: 
---

# Motivation

Ruby1.8.7で作った大量のバッチがある。
それを1.9（さらには2.0）に移行していきたいんだけど、
一気に移行するまで待ってると時間がかかるので、
少しずつ移行したい。また、同じサーバ内で並行運用したい。
そこで、rbenvで複数のRuby環境を用意し、切り替えて運用したい。
バッチは全てcronで動いているので、cron環境でのPATHをうまく設定すればいけるはず。

# 環境
CentOS6.4

# 事前準備
## gitをインストール

```sh
yum install git
```

## epelをインストール

後で必要になるライブラリを入れるため、EPELを追加しておきます。

```sh
cd /usr/local/src
wget http://ftp-srv2.kddilabs.jp/Linux/distributions/fedora/epel/6/x86_64/epel-release-6-8.noarch.rpm
rpm -ivh epel-release-6-8.noarch.rpm
```

# rbenv

参考：[CentOSでsystem wideなrbenv+ruby-build環境を構築する](http://nomnel.net/blog/centos-system-wide-rbenv-and-ruby-build/)

## rbenvをインストール

rbenvとruby-buildを/usr/localの下にインストールします。

```sh
su -
cd /usr/local
git clone git://github.com/sstephenson/rbenv.git rbenv
mkdir rbenv/shims rbenv/versions
groupadd rbenv
chgrp -R rbenv rbenv
chmod -R g+rwxXs rbenv
git clone git://github.com/sstephenson/ruby-build.git ruby-build
cd ruby-build
./install.sh
```

## 環境変数を設定

グローバルな環境変数にrbenvへのパスを追加。  

vi /etc/profile.d/rbenv.sh

```sh
export RBENV_ROOT="/usr/local/rbenv"
export PATH="/usr/local/rbenv/bin:$PATH"
eval "$(rbenv init -)"
```

書いたら読み込みます。

```sh
source /etc/profile.d/rbenv.sh
```


## インストール可能なruby一覧を見てみる

```sh
rbenv install --list
```

## ruby install

### 依存ライブラリのインストール

```sh
yum install --enablerepo=epel make gcc zlib-devel openssl-devel readline-devel ncurses-devel gdbm-devel db4-devel libffi-devel tk-devel libyaml-devel
```

### ruby自体のインストール

ここでは以下の3つのバージョンを入れてみます。

```sh
rbenv install 1.8.7-p374
rbenv install 1.9.3-p448
rbenv install 2.0.0-p247
```

#### （Dockerの場合のみ）fdのシンボリックリンク作成

Docker上のCentOS6.4で試していたらエラーが出るので、以下のコマンド実行

参考:[unable to set global or or local ruby on freebsd](https://github.com/sstephenson/rbenv/issues/401)

```sh
ln -s /proc/self/fd /dev/fd
```

### インストールされたrubyの確認

```sh
rbenv versions
```

### globalのrubyバージョン設定

```sh
rbenv global 1.9.3-p448
rbenv rehash
```

### localのrubyバージョン設定

```sh
rbenv local 2.0.0-p247
rbenv rehash
```

### rbenv-rehash,bundlerのインストール

いちいちrehashするのも面倒なので、自動でrehashしてくれるrbenv-rehashを入れます。
ついでに後々必要になるbundlerも入れておきましょう。
インストールした全てのバージョンで行なっておく必要があります。

参考:[crontabでrbenvのrubyを使う](http://shokai.org/blog/archives/7258)



```sh
su -

rbenv global 1.8.7-p374
gem install rbenv-rehash
gem install bundler
rbenv rehash

rbenv global 1.9.3-p448
gem install rbenv-rehash
gem install bundler
rbenv rehash

rbenv global 2.0.0-p247
gem install rbenv-rehash
gem install bundler
rbenv rehash
```

# Rubyバージョン設定

## それぞれのディレクトリでrbenv local

以下のようなディレクトリ構成で、batch_18はRuby1.8、
batch_19はRuby1.9で動かしたいとします。
exec_batch_a.shはバッチを起動するためのshellスクリプトです。

```sh
/home/sudix
└── batches
    ├── batch_18
    │   ├── batch_a.rb
    │   ├── exec_batch_a.sh
    │   └── Gemfile
    └── batch_19
        ├── batch_a.rb
        ├── exec_batch_a.sh
        └── Gemfile

```

この場合、それぞれのディレクトリに入って、rbenv localでRubyのバージョンを指定します。
これで、それぞれのディレクトリに.ruby-versionが作成されるはずです。
同時にbundlerでgemを入れておきますが、ここではpathを指定せず、グローバルに入れていしまいます。
このあたりは必要に応じて変えてください。

```sh
cd /home/sudix/batches/batch_18
rbenv local 1.8.7-p374
su
cd /home/sudix/batches/batch_18
bundle install
exit

cd /home/sudix/batches/batch_19
rbenv local 1.9.3-p448
su
cd /home/sudix/batches/batch_19
bundle install
exit
```

Gemfileはここでは普通に作成していますが、クックパッドさんでは
バージョンごとのGemfileを作成して管理しているようで、参考になります。

[Cookpad の本番環境で使用している Ruby が 2.0.0-p0 になりました](http://techlife.cookpad.com/2013/04/09/ruby200/)

## 起動shell

バッチ実行前に意図した.ruby-versionが有効となるディレクトリに移動したいので、
起動shellは以下のような感じにしました。

```sh
#!/bin/sh

dir=`dirname $0`
cd $dir

ruby -e "require './batch_a.rb'; BatchA.new.main"

exit
```

##確認

使いたいRubyが本当に使えているか、確認します。
BatchAのmainは、RUBY_VERSIONをputsするだけのメソッドです。

```sh
$ cd
$ /home/sudix/batches/batch_18/exec_batch_a.sh 
1.8.7
$ /home/sudix/batches/batch_19/exec_batch_a.sh 
1.9.3
```

# cronの設定

参考:[cronでのversion変更について](http://shokai.org/blog/archives/7258)

rbenvをglobalにインストールしているので、上記参考先とは
rbenvのbinとshimsのpathが異なるので注意してください。
PATHに/usr/local/rbenv/binと/usr/local/rbenv/shimsを追加します。

crontab -e

```sh
SHELL=/bin/sh
PATH=/usr/local/rbenv/bin:/usr/local/rbenv/shims:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin

0 * * * * /home/sudix/batches/batch_18/exec_batch_a.sh
0 * * * * /home/sudix/batches/batch_19/exec_batch_a.sh
```

以上でcronから実行されるバッチのRubyバージョンを切り替えることができました。



