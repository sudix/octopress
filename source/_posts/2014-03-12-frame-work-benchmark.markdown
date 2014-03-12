---
layout: post
title: "GoとかScalaとかNode.JSとかでベンチマーク"
date: 2014-03-12 19:59
comments: true
categories: go scala finagle spray node.js
---

最近、色々API化したいものがあって、何で書けばいいかなぁと考えている。
とにかく速く、複数のDBから並列でデータをもってきて集約して返す、ようなのを作りたい。
そんな事考えているとき、こんなブログをみつけた。

[Go as an alternative to Node.js for Very Fast Servers | Safari Flow Blog](http://blog.safariflow.com/2013/02/22/go-as-an-alternative-to-node-js-for-very-fast-servers/)

なかなか興味深いので、GoとNode.jsの他にも、気になっていたFinagleとかも入れて、
自分も同じようにベンチマークを取ってみた。

***

# disclaimer(いいわけ)

ベンチマークのとり方ははそんなに厳密にとっているわけでありません。カジュアルな感じで。
サーバもクライアント(Apache Bench)も同一マシンで実行しちゃってます。
もっとこうすべきとかあれば教えてください。
あと、計測結果は長くなりすぎたので、最後に置いています。

***

# 測定について

## 実行方法

元ブログと同じように、httpで1MBのコンテンツを返すだけのAPIを作り、
Apache Benchで測定する。測定結果は5回連続実行後の最後のものを使用。
メモリの使用量を見るため、[procfs](http://d.hatena.ne.jp/naoya/20080727/1217119867)で
最大物理メモリ使用量も見る。

## マシンスペック

<table>
  <tbody>
    <tr>
      <td>CPU&nbsp;</td>
      <td>Core i7-2600</td>
    </tr>
    <tr>
      <td>メモリ&nbsp;</td>
      <td>８GB</td>
    </tr>
    <tr>
      <td>OS&nbsp;</td>
      <td>Ubuntu 13.10 64bit</td>
    </tr>
  </tbody>
</table>


## バージョン等

<table>
  <thead>
    <tr>
      <th>言語・FW名</th>
      <th>バージョン</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Go&nbsp;</td>
      <td>go1.2 linux/amd64</td>
    </tr>
    <tr>
      <td>Node.js&nbsp;</td>
      <td>v0.10.26</td>
    </tr>
    <tr>
      <td>Scala&nbsp;</td>
      <td>2.10.3</td>
    </tr>
    <tr>
      <td>Finagle&nbsp;</td>
      <td>6.12.1</td>
    </tr>
    <tr>
      <td>Spray&nbsp;</td>
      <td>1.1.0</td>
    </tr>
  </tbody>
</table>


***

# 登場選手たち

## Go

* [The Go Programming Language](http://golang.org/)

最近噂のGoogle生まれのあいつ。C++の代替を狙っていたのに、
なぜかPythonistaやRubyistからばかり注目されてるとか。
小さな言語仕様、お手軽な並列処理などが特徴。

※テストコードは元ブログのコードをそのまま流用。

```go
package main

import "net/http"

func main() {
	bytes := make([]byte, 1024*1024)
	for i := 0; i < len(bytes); i++ {
		bytes[i] = 100
	}

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write(bytes)
	})
	http.ListenAndServe(":8000", nil)
}
```


## Go/Martini

* [Martini - Classy web development in Go.](http://martini.codegangsta.io/)

気になっていたGo用軽量WEBフレームワーク。ついでに試してみた。
Sinatra風味。なんせ公式サイトがかっこいい。

```go
package main

import (
	"github.com/codegangsta/martini"
	"net/http"
)

func main() {
	bytes := make([]byte, 1024*1024)
	for i := 0; i < len(bytes); i++ {
		bytes[i] = 100
	}

	m := martini.Classic()
	m.Get("/", func() []byte {
		return bytes
	})
	//m.Run() これで起動するとポートは3000
	http.ListenAndServe(":8000", m) //ポートを指定したい場合はこっち
}
```


## Node.js

* [node.js](http://nodejs.org/)

ブラウザ界の住人だったはずのあいつが、サーバサイドにもやってきた。
自分はJavaScriptは苦手だしあまり好きじゃないと思っていたけれど、少し触ってみたらなかなか楽しい。
ブラウザ側でJavaScriptを極めた方々からすれば、
サーバサイドまで全て同一言語で完結できるのはまさに夢のようだろうなぁと思う。

※テストコードは元ブログのコードをそのまま流用。

```javascript
http = require('http')
Buffer = require('buffer').Buffer;
n = 1024*1024;
b = new Buffer(n);
for (var i = 0; i < n; i++) b[i] = 100;

http.createServer(function (req, res) {
  res.writeHead(200);
  res.end(b);
}).listen(8000);
```


## Scala/Finagle

* [Finagle](http://twitter.github.io/finagle/)

FinagleはTwitterのバックエンドで使われてるすごいやつらしい。

[Finagle: A Protocol-Agnostic RPC System | Twitter Blogs](https://blog.twitter.com/2011/finagle-a-protocol-agnostic-rpc-system)

[The Twitter stack - Braindump](http://blog.oskarsson.nu/post/40196324612/the-twitter-stack)

あのサイトのトラフィックをさばいてるんだからさぞかし凄いんだろう。
ZooKeeprと連携したり、各種Metricsを取れるようになってたり、ただものではない感がある。

```scala
package jp.sudix.finagle.benchmark

import com.twitter.finagle.Service
import com.twitter.finagle.http.Http
import com.twitter.util.Future
import com.twitter.finagle.builder.ServerBuilder
import org.jboss.netty.handler.codec.http.{DefaultHttpResponse, HttpVersion, HttpResponseStatus, HttpRequest, HttpResponse}
import org.jboss.netty.buffer.ChannelBuffers.copiedBuffer
import java.net.{SocketAddress, InetSocketAddress}

object Main {

  def main(args: Array[String]) {

    val bytes: Array[Byte] = Array.fill(1024*1024)(100:Byte)
    val cb = copiedBuffer(bytes)
    val response = new DefaultHttpResponse(HttpVersion.HTTP_1_1,
                                           HttpResponseStatus.OK)
    response.setContent(cb)

    val service = new Service[HttpRequest, HttpResponse] {
      def apply(request: HttpRequest) = {
        Future.value(response)
      }
    }

    val address: SocketAddress = new InetSocketAddress(8000)
    ServerBuilder()
      .codec(Http())
      .bindTo(address)
      .name("HttpServer")
      .build(service)
  }
}
```


## Scala/Spray

* [spray | REST/HTTP for your Akka/Scala Actors](http://spray.io/)

これもScala用フレームワーク。AkkaというActorを使った非同期フレームワーク。
Finagleのなんでもできるよ感と違って、REST APIを作るのに特化してるっぽい。

*サンプルプロジェクトをちょっといじっただけ

```scala
package com.example

import akka.actor.{ActorSystem, Props}
import akka.io.IO
import spray.can.Http

object Boot extends App {
  implicit val system = ActorSystem("on-spray-can")
  val service = system.actorOf(Props[MyServiceActor], "demo-service")
  IO(Http) ! Http.Bind(service, interface = "localhost", port = 8000)
}
```

```scala
package com.example

import akka.actor.Actor
import spray.routing._
import spray.http._
import MediaTypes._

class MyServiceActor extends Actor with MyService {
  def actorRefFactory = context
  def receive = runRoute(myRoute)
}

trait MyService extends HttpService {
  val bytes: Array[Byte] = Array.fill(1024*1024)(100:Byte)
  val myRoute =
    path("") {
      get {
        respondWithMediaType(`text/html`) {
          complete {
            bytes
          }
        }
      }
    }
}
```


***

# 結果一覧

下記のコマンドを実行した結果

```bash
$ ab -c 100 -n 10000 http://localhost:8000/
```

<table>
  <thead>
    <tr>
      <th>言語・FW</th>
      <th>実行時間(秒)</th>
      <th>処理数/秒</th>
      <th>応答時間(ミリ秒)</th>
      <th>物理メモリ使用量(KB)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Go&nbsp;</td>
      <td>4.312</td>
      <td>2319.10</td>
      <td>43.120</td>
      <td>8084</td>
    </tr>
    <tr>
      <td>Go/Martini&nbsp;</td>
      <td>4.757</td>
      <td>2102.27</td>
      <td>45.475</td>
      <td>7928</td>
    </tr>
    <tr>
      <td>Node.js&nbsp;</td>
      <td>4.346</td>
      <td>2300.74</td>
      <td>43.464</td>
      <td>56892</td>
    </tr>
    <tr>
      <td>Scala/Finagle&nbsp;</td>
      <td>7.682</td>
      <td>1301.69</td>
      <td>76.823</td>
      <td>807656</td>
    </tr>
    <tr>
      <td>Scala/Spray&nbsp;</td>
      <td>6.184</td>
      <td>1146.31</td>
      <td>61.843</td>
      <td>833040</td>
    </tr>
  </tbody>
</table>

***

# 感想

GoやNode.jsと比べるとScala勢が遅い感じがするけど、FinagleやSprayは重量級で機能豊富なフレームワークだと思うので、
同じ土俵で比べるのは酷だった気がする。実際はDBからデータ取ってきたりというステップが入るので、
速度差は埋まっていくだろうし、性能的にはどれ使っても問題無い気がしている。
Scalaのメモリ使用量が多すぎる気がするけど、JVMアプリのメモリ測定方法は、今回のでは正確に出ないのかも。
この程度のベンチマークでどれを採用するかまでは決められないけれど、Goのメモリフットプリントの少なさはいいなと思う。
どれか一つ選べと言われたら、Go/Martiniあたりがいいかなぁ。
なんかGoはサクサク書けて気持ちいい。

***

# 結果詳細

## Go

### 結果

#### ab
```bash
$ ab -c 100 -n 10000 http://localhost:8000/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /
Document Length:        1048576 bytes

Concurrency Level:      100
Time taken for tests:   4.312 seconds
Complete requests:      10000
Failed requests:        0
Write errors:           0
Total transferred:      10486730000 bytes
HTML transferred:       10485760000 bytes
Requests per second:    2319.10 [#/sec] (mean)
Time per request:       43.120 [ms] (mean)
Time per request:       0.431 [ms] (mean, across all concurrent requests)
Transfer rate:          2374973.45 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   0.5      1      11
Processing:    13   42   2.6     42      74
Waiting:        0    1   2.2      1      33
Total:         14   43   2.7     43      76

Percentage of the requests served within a certain time (ms)
  50%     43
  66%     43
  75%     43
  80%     44
  90%     44
  95%     45
  98%     46
  99%     48
 100%     76 (longest request)

```

#### procfs
```bash
$ cat /proc/`pgrep -lf 'go_server' | awk '{print $1}'`/status
Name:	go_server
State:	S (sleeping)
Tgid:	14952
Pid:	14952
PPid:	6925
TracerPid:	0
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
FDSize:	256
Groups:	4 24 27 30 46 108 109 1000 
VmPeak:	  239304 kB
VmSize:	  185236 kB
VmLck:	       0 kB
VmPin:	       0 kB
VmHWM:	    8084 kB
VmRSS:	    8084 kB
VmData:	  174376 kB
VmStk:	     140 kB
VmExe:	    1452 kB
VmLib:	    2012 kB
VmPTE:	      80 kB
VmSwap:	       0 kB
Threads:	5
SigQ:	0/63671
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	0000000000000000
SigIgn:	0000000000000000
SigCgt:	ffffffffffc1feff
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	0000001fffffffff
Seccomp:	0
Cpus_allowed:	ffffffff,ffffffff
Cpus_allowed_list:	0-63
Mems_allowed:	00000000,00000001
Mems_allowed_list:	0
voluntary_ctxt_switches:	9951
nonvoluntary_ctxt_switches:	7772
```


## Go/Martini

### 結果

#### ab
```bash
$ ab -c 100 -n 10000 http://localhost:8000/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /
Document Length:        1048576 bytes

Concurrency Level:      100
Time taken for tests:   4.757 seconds
Complete requests:      10000
Failed requests:        0
Write errors:           0
Total transferred:      10486730000 bytes
HTML transferred:       10485760000 bytes
Requests per second:    2102.27 [#/sec] (mean)
Time per request:       47.568 [ms] (mean)
Time per request:       0.476 [ms] (mean, across all concurrent requests)
Transfer rate:          2152920.42 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   0.5      1       6
Processing:     5   47   8.5     47      88
Waiting:        0   14  14.6      8      54
Total:          8   47   8.5     48      88

Percentage of the requests served within a certain time (ms)
  50%     48
  66%     50
  75%     52
  80%     53
  90%     57
  95%     62
  98%     67
  99%     71
 100%     88 (longest request)

```

#### procfs
```bash
$ cat /proc/`pgrep -lf 'martini_server' | awk '{print $1}'`/status
Name:	martini_server
State:	S (sleeping)
Tgid:	14745
Pid:	14745
PPid:	6925
TracerPid:	0
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
FDSize:	256
Groups:	4 24 27 30 46 108 109 1000 
VmPeak:	  242100 kB
VmSize:	  185852 kB
VmLck:	       0 kB
VmPin:	       0 kB
VmHWM:	    9064 kB
VmRSS:	    9064 kB
VmData:	  174376 kB
VmStk:	     140 kB
VmExe:	    1632 kB
VmLib:	    2012 kB
VmPTE:	      80 kB
VmSwap:	       0 kB
Threads:	5
SigQ:	0/63671
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	0000000000000000
SigIgn:	0000000000000000
SigCgt:	ffffffffffc1feff
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	0000001fffffffff
Seccomp:	0
Cpus_allowed:	ffffffff,ffffffff
Cpus_allowed_list:	0-63
Mems_allowed:	00000000,00000001
Mems_allowed_list:	0
voluntary_ctxt_switches:	985
nonvoluntary_ctxt_switches:	17704
```

## Node.js

### 結果

#### ab

```bash
$ ab -c 100 -n 10000 http://localhost:8000/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /
Document Length:        1048576 bytes

Concurrency Level:      100
Time taken for tests:   4.346 seconds
Complete requests:      10000
Failed requests:        0
Write errors:           0
Total transferred:      10486510000 bytes
HTML transferred:       10485760000 bytes
Requests per second:    2300.74 [#/sec] (mean)
Time per request:       43.464 [ms] (mean)
Time per request:       0.435 [ms] (mean, across all concurrent requests)
Transfer rate:          2356128.80 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.4      1       5
Processing:    16   43   6.2     45      60
Waiting:        0   14  12.9     13      41
Total:         17   43   6.5     46      60
ERROR: The median and mean for the initial connection time are more than twice the standard
       deviation apart. These results are NOT reliable.

Percentage of the requests served within a certain time (ms)
  50%     46
  66%     49
  75%     49
  80%     49
  90%     50
  95%     51
  98%     54
  99%     55
 100%     60 (longest request)
```

#### procfs

```bash
$ cat /proc/`pgrep -lf 'nodejs node_server.js' | awk '{print $1}'`/status
Name:	nodejs
State:	S (sleeping)
Tgid:	14860
Pid:	14860
PPid:	6925
TracerPid:	0
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
FDSize:	256
Groups:	4 24 27 30 46 108 109 1000 
VmPeak:	  720076 kB
VmSize:	  672132 kB
VmLck:	       0 kB
VmPin:	       0 kB
VmHWM:	   56892 kB
VmRSS:	   56892 kB
VmData:	  645248 kB
VmStk:	     140 kB
VmExe:	    8132 kB
VmLib:	    4168 kB
VmPTE:	     264 kB
VmSwap:	       0 kB
Threads:	2
SigQ:	0/63671
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	0000000000000000
SigIgn:	0000000000001000
SigCgt:	0000000188004202
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	0000001fffffffff
Seccomp:	0
Cpus_allowed:	ffffffff,ffffffff
Cpus_allowed_list:	0-63
Mems_allowed:	00000000,00000001
Mems_allowed_list:	0
voluntary_ctxt_switches:	6389
nonvoluntary_ctxt_switches:	7180
```

## Scala/Finagle

### 結果

#### 起動スクリプト

sbt-assemblyで固めて実行。

```bash
java -jar ./target/scala-2.10/FinageBenchmark-assembly-1.0.jar
```

#### ab
```bash
$ ab -c 100 -n 10000 http://localhost:8000/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /
Document Length:        1048576 bytes

Concurrency Level:      100
Time taken for tests:   7.682 seconds
Complete requests:      10000
Failed requests:        0
Write errors:           0
Total transferred:      10485950000 bytes
HTML transferred:       10485760000 bytes
Requests per second:    1301.69 [#/sec] (mean)
Time per request:       76.823 [ms] (mean)
Time per request:       0.768 [ms] (mean, across all concurrent requests)
Transfer rate:          1332958.99 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   0.5      1       5
Processing:     7   75   5.9     75     153
Waiting:        1    3   4.4      2      82
Total:         11   77   5.9     76     155

Percentage of the requests served within a certain time (ms)
  50%     76
  66%     77
  75%     78
  80%     79
  90%     80
  95%     82
  98%     86
  99%     92
 100%    155 (longest request)
```

#### procfs

```bash
$ cat /proc/`pgrep -lf 'FinageBenchmark-assembly-1.0.jar' | awk '{print $1}'`/status
Name:	java
State:	S (sleeping)
Tgid:	15132
Pid:	15132
PPid:	15017
TracerPid:	0
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
FDSize:	256
Groups:	4 24 27 30 46 108 109 1000 
VmPeak:	 3993828 kB
VmSize:	 3936516 kB
VmLck:	       0 kB
VmPin:	       0 kB
VmHWM:	  807656 kB
VmRSS:	  726864 kB
VmData:	 3878388 kB
VmStk:	     140 kB
VmExe:	       4 kB
VmLib:	   15552 kB
VmPTE:	    1868 kB
VmSwap:	       0 kB
Threads:	25
SigQ:	0/63671
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	0000000000000000
SigIgn:	0000000000000000
SigCgt:	2000000181005ccf
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	0000001fffffffff
Seccomp:	0
Cpus_allowed:	ffffffff,ffffffff
Cpus_allowed_list:	0-63
Mems_allowed:	00000000,00000001
Mems_allowed_list:	0
voluntary_ctxt_switches:	5
nonvoluntary_ctxt_switches:	6
```

## Scala/Spray

### 結果

#### 起動スクリプト

sbt-assemblyで固めて実行。

```bash
java -jar ./target/scala-2.10/spray_benchmark-assembly-0.1.jar
```

#### ab
```bash
$ ab -c 100 -n 10000 http://localhost:8000/
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:        spray-can/1.1.0
Server Hostname:        localhost
Server Port:            8000

Document Path:          /
Document Length:        1048576 bytes

Concurrency Level:      100
Time taken for tests:   6.184 seconds
Complete requests:      10000
Failed requests:        0
Write errors:           0
Total transferred:      10487070000 bytes
HTML transferred:       10485760000 bytes
Requests per second:    1617.00 [#/sec] (mean)
Time per request:       61.843 [ms] (mean)
Time per request:       0.618 [ms] (mean, across all concurrent requests)
Transfer rate:          1656013.76 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   0.4      1       5
Processing:     7   61   8.4     60     105
Waiting:        1    6   7.2      3      52
Total:         11   62   8.4     61     106

Percentage of the requests served within a certain time (ms)
  50%     61
  66%     63
  75%     65
  80%     66
  90%     70
  95%     76
  98%     82
  99%     85
 100%    106 (longest request)
```

#### procfs

```bash
$ cat /proc/`pgrep -lf 'spray_benchmark-assembly-0.1.jar' | awk '{print $1}'`/status
Name:	java
State:	S (sleeping)
Tgid:	15264
Pid:	15264
PPid:	15017
TracerPid:	0
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
FDSize:	256
Groups:	4 24 27 30 46 108 109 1000 
VmPeak:	 4130404 kB
VmSize:	 4130400 kB
VmLck:	       0 kB
VmPin:	       0 kB
VmHWM:	  833040 kB
VmRSS:	  832932 kB
VmData:	 4069920 kB
VmStk:	     140 kB
VmExe:	       4 kB
VmLib:	   15804 kB
VmPTE:	    1948 kB
VmSwap:	       0 kB
Threads:	22
SigQ:	0/63671
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	0000000000000000
SigIgn:	0000000000000000
SigCgt:	2000000181005ccf
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	0000001fffffffff
Seccomp:	0
Cpus_allowed:	ffffffff,ffffffff
Cpus_allowed_list:	0-63
Mems_allowed:	00000000,00000001
Mems_allowed_list:	0
voluntary_ctxt_switches:	5
nonvoluntary_ctxt_switches:	2
```






