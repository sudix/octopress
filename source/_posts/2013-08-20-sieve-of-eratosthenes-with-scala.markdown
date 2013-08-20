---
layout: post
title: "Scalaでエラトステネスの篩を実装してみる"
date: 2013-08-20 12:26
comments: true
categories: scala
---

こちらのエントリ
[もしぼくが採用するなら](http://udzura.hatenablog.jp/entry/2013/08/17/021446)
を読んで、エラトステネスの篩は知ってはいるものの実装してみたことないなぁと思い、
Scalaで実装してみました。

[Wikipedia](http://ja.wikipedia.org/wiki/%E3%82%A8%E3%83%A9%E3%83%88%E3%82%B9%E3%83%86%E3%83%8D%E3%82%B9%E3%81%AE%E7%AF%A9)の例題を参考に作ってみます。
考え方としては自然数の集合に対して、小さい順からその倍数を削除していって、
残ったものが素数、という考えです。


最初、こんな感じで実装してみました。
```scala 末尾再帰じゃないバージョン
def sieveOfEratosthenes(numbers: List[Int]): List[Int] = {
  numbers match {
    case Nil =>
      Nil
    case 1 :: tail =>
      sieveOfEratosthenes(tail)
    case head :: tail =>
      head :: sieveOfEratosthenes(tail.filterNot(_ % head == 0))
  }
}

// 実行
println(sieveOfEratosthenes(Range(1, 121).toList))
```

まあこれで動くには動くのですが、最後で再帰の結果に対して演算しているので末尾再帰されてません（@tailrecつけるとコンパイルエラーになる）。
なので数が多くなるとOutOfMemoryで落ちます。

いろいろ考えて、素数の結果リストを引数で渡すようにしました。

```scala 末尾再帰バージョン
import scala.annotation.tailrec

@tailrec
def sieveOfEratosthenes(numbers: List[Int], primes: List[Int]): (List[Int], List[Int]) = {
  numbers match {
    case Nil =>
      (numbers, primes.reverse)
    case 1 :: tail =>
      sieveOfEratosthenes(tail, primes)
    case head :: tail =>
      sieveOfEratosthenes(tail.filterNot(_ % head == 0), head :: primes)
  }
}
  
// 実行
sieveOfEratosthenes(Range(1, 120).toList, Nil)._2
```

末尾再帰になった！しかし実行時に引数にNilを渡したり、結果がタプルなので結果取得にひと手間かかったり、かっこ悪すぎ。  
なので隠蔽を図った結果がこれ。

```scala 末尾再帰バージョン（呼び出しをシンプルに）
import scala.annotation.tailrec

def sieveOfEratosthenes(numbers: List[Int]): List[Int] = {
  @tailrec
  def sieve(numbers: List[Int], primes: List[Int]): (List[Int], List[Int]) = {
    numbers match {
      case Nil =>
        (numbers, primes.reverse)
      case 1 :: tail =>
        sieve(tail, primes)
      case head :: tail =>
        sieve(tail.filterNot(_ % head == 0), head :: primes)
    }
  }
  sieve(numbers, Nil)._2
}

// 実行
println(sieveOfEratosthenes(Range(1, 121).toList))
```

できたけどなんかゴチャゴチャしてる感じがありますね・・・。きっともっと素敵な書き方があるはず。  
ということでググってみると、  
[Sieve of Eratosthenes in Scala](http://mogproject.blogspot.jp/2013/05/sieve-of-eratosthenes-in-scala.html)  
こちらの記事を発見。おお、Stream使うとこんなにもスッキリ書けるんですね。確かに衝撃的。  
まだまだScala力が足りません。





