---
layout: post
title: "[ruby] Rubyで関数合成できるLambdaDriverがかっこよすぎる"
date: 2013-10-23 15:30
comments: true
categories: ruby
---

Rubyで関数合成できると便利なのになぁという場面に出くわして、以前に見て知っていたけど試したことはなかったLambdaDriverを触ってみた。

[( ꒪⌓꒪) ゆるよろ日記 - Rubyで関数合成とかしたいので lambda_driver.gem というのを作った](http://yuroyoro.hatenablog.com/entry/2013/03/27/190640)

### install
gem install lambda_driver

### サンプル

```ruby
require 'lambda_driver'

add_hoge = lambda{|x| x + "hoge"}
add_fuga = lambda{|x| x + "fuga"}

# >>で合成
add_hoge_fuga = add_hoge >> add_fuga
# < で実行（callの別名)
add_hoge_fuga < "piyo"
=> "piyohogefuga"

# <<で逆順で合成
add_fuga_hoge = add_hoge << add_fuga
add_fuga_hoge < "piyo"
=> "piyofugahoge"
```

かっこよすぎるやろ！でも仕事のコードで使ったら顰蹙物だろうなぁ。