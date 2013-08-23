---
layout: post
title: "[coffeescript] 標準偏差を使って異常っぽい値を検知してみる"
date: 2013-08-23 11:02
comments: true
categories: 
---

日々変わる数字があって、たまに異常に減ったり増えたりする場合があり、
それが起きたときにすぐわかるようにしたい。
という要望があったので、標準偏差を使ってエラーっぽい値が分かるようにしてみました。
ブラウザ上で実行する必要があったのでcoffeescriptです。

## コード

checkRangeListメソッドにデータを与えると、下限以下なら-1、上限以上なら1を返します。

```coffeescript
######################
# 異常値計算用クラス
######################

class ErrorDataDetector

  self = undefined
  
  # 平均からどれぐらい離れたら異常とみなすかの定数。
  # ±（この値×標準偏差）から外れていれば異常とします。
  # 2であればおよそ95%範囲の外の値になります。
  normalRange = 2

  constructor: -> self = this

  #平均を求める
  average: (data) ->
    data.reduce((acc, n) -> acc + n) / data.length

  #分散を求める
  #((データ－平均値)の２乗)の総和÷ 個数
  variance: (data, avg) ->
    data.reduce((acc, n) -> acc + Math.pow(n - avg, 2)) / data.length

  #標準偏差を求める
  #分散の平方根
  standardDeviation: (data, avg) ->
    Math.sqrt(self.variance(data, avg))

  #外れ値かどうかを調べ、リストに結果を詰めて返す
  #-1:範囲外(下) 0:正常 1:範囲外(上)
  checkRangeList: (data) ->
    avg = self.average(data)
    sd = self.standardDeviation(data, avg)
    lb = self.lowerBound(avg, sd)
    ub = self.upperBound(avg, sd)
    data.map (n) -> self.checkRange(n, avg, lb, ub)

  # 下限境界
  lowerBound: (avg, sd) -> avg - (sd * normalRange)

  # 上限境界
  upperBound: (avg, sd) -> avg + (sd * normalRange)

  checkRange: (n, avg, lb, ub) ->
    if avg == 0
      0
    else if n <= lb
      -1
    else if n >= ub
      1
    else
      0

```

## 実行結果

```coffeescript
l = [826,751,833,905,888,950,880,868,935,1293,1315,1555,1445,1732,1351,1157,1268,1201,733,2000,100]
d = new ErrorDataDetector()
console.log(d.checkRangeList(l))
 # 結果 =>　[0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, -1]
```

## オチ
さっそく実装して試したところ異常っぽい値を検知できたので意気揚々と担当者に見せたところ、
よくわかんないからもっと簡単なロジックにしてと言われましたとさ・・・。


