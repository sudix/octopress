---
layout: post
title: "Goで重い処理を並列実行して実行時間を短くするサンプル"
date: 2014-03-18 20:59
comments: true
categories: golang goroutine concurrent
---

時間のかかる処理を複数実行して、最後に各処理の結果をがっちゃんこしたい。
そのまま行うと各処理の実行時間の合計だけかかってしまうので、
golangで並列で実行する場合のサンプルを書いてみた。

goroutineについてはこちらを参考にさせていただいた。

[Go の並行処理 - Block Rockin’ Codes](http://jxck.hatenablog.com/entry/20130414/1365960707)

##コード

main.go

```go
package main

import (
        "fmt"
        "time"
)

func wait(waitSec time.Duration) <-chan float64 {
        ch := make(chan float64)
        go func() {
                start := time.Now()
                time.Sleep(waitSec * time.Second)
                end := time.Now()
                ch <- end.Sub(start).Seconds()
        }()
        return ch
}

func main() {

        cpus := runtime.NumCPU()
        runtime.GOMAXPROCS(cpus)
        
        start := time.Now()

        ch1 := wait(5)
        ch2 := wait(10)
        ch3 := wait(3)

        ret1 := <-ch1
        ret2 := <-ch2
        ret3 := <-ch3

        fmt.Printf("各処理時間合計 %f sec\n", ret1+ret2+ret3)
        end := time.Now()

        fmt.Printf("実時間 %f sec \n", end.Sub(start).Seconds())
}
```

channelからの結果取り出しでブロックされてるけど、重い処理自体はブロックされずに
別goroutineで実行されているので、今回の目的ならこれで問題ないはず。

## 実行結果

```bash
$ go run main.go
各処理時間合計 18.002096 sec
実時間 10.000841 sec
```

指定秒数だけ待つ関数を呼んでいる。それぞれ5秒、10秒、3秒なので、
普通に実行すれば全体で18秒かかるけど、実際には10秒で終わっている。
今回はSleepしてるだけだからあまり関係ないけど、実際に重い計算をさせる場合は、
マルチコア環境なら並行に処理が行われて、CPUリソースを活かせる。
こんなにお手軽に並列処理ができて、goroutineは面白い。


