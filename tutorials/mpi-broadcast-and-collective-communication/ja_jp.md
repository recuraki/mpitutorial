---
layout: post
title: MPIブロードキャストと集団通信 - MPI Broadcast and Collective Communication
author: Wes Kendall
categories: Beginner MPI
tags: MPI_Barrier, MPI_Bcast, MPI_Wtime
redirect_from: '/mpi-broadcast-and-collective-communication/'
---

ここまでの[チュートリアル]({{ site.baseurl }}/tutorials/)では、2プロセス間のポイントツーポイント通信を説明してきました。このレッスンから集団通信(Collective Communication)について学びます。集団通信はコミュニケータ内の*すべての*プロセスが関係する通信です。このレッスンでは、最初に集団通信の意味を確認し、標準的な集団通信のルーチンであるブロードキャスト(Broadcast)について説明します。

> **Note** - このサイトのコードはすべて[GitHub]({{ site.github.repo }})にあります。このレッスンのコードは[tutorials/mpi-broadcast-and-collective-communication/code]({{ site.github.code }}/tutorials/mpi-broadcast-and-collective-communication/code)にあります。

## 集団通信と同期 - Collective communication and synchronization points
集団通信を学んでいくために、最初に覚えておくべきことの1つは、プロセス間の同期が必要になるということです。つまり、すべての関連するプロセスがコード内の特定の集団通信を完了しなければ、すべてのプロセスが再び実行を開始できません。

同期についてさらに詳しく説明します。MPIにはプロセスの同期専用の特別な関数があります。


```cpp
MPI_Barrier(MPI_Comm communicator)
```

バリアという非常にわかりやすい名前がついています。この関数は、コミュニケータ内のすべてのプロセスがこの関数を呼ぶまで全てのプロセスはこの関数でブロック（訳注：つまり、あるプロセスだけが先に進むことをバリア）します。下図の横軸がプログラム実行の時間を示し、各円はプロセスを示します。
![MPI_Barrier example](../barrier.png)

それぞれの図を見ていきましょう。T1では、プロセス0が`MPI_Barrier`に達しました。T2ではプロセス0はバリア関数でブロックされ、この間にプロセス1と3がバリア関数に到達しました。T3でプロセス2がようやくバリアに到達します。この結果、T4のようにすべてのプロセスが再び実行を進められます。

`MPI_Barrier`の利用用途はいくつかあります。最も主な使途は並列プログラムで正確な時間測定(timed accurately)をするためです。

`MPI_Barrier` はどのように実装されているのでしょうか？[sending and receiving tutorial]({{ site.baseurl }}/tutorials/mpi-send-and-receive/)のレッスンで学んだリングプログラムを思い出してください。トークンをリングのようにすべてのプロセスに渡すプログラムでした。この実装は全てのプロセスが処理を終えなければこの処理は終わらないのでバリアを実装するの１つの方式です。

繰り返しになりますが、すべての集団通信は同期されている。言い換えると、そのルーチンを`MPI_Barrier`とした時にバリアが完了しないような状態ができてしまうと集団通信は正常に完了できません。`MPI_Barrier`や集団通信の関数をコミュニケータ内の全てのプロセスが呼び出すことを保証せずに呼び出そうとするとプログラムはブロック状態のまま先に進めません。これは初学者にとって非常に分かりにくいので覚えておいてください！

## MPI_Bcast によるブロードキャスト - Broadcasting with MPI_Bcast
ブロードキャスト（broadcast)は最も基本的な集団通信の1つです。ブロードキャストはある1つのプロセスがコミュニケータ内のすべてのプロセスに同じデータを送信します。ユーザ入力や構成パラメータをすべてのプロセスに送信するために使うことができます。

ブロードキャストの例を次に示します。

![MPI_Bcast pattern](../broadcast_pattern.png)

この例はプロセス0がルートプロセスでとなり、オリジナルのデータを保持しています。他のすべてのプロセスはデータのコピーを受け取ります。

MPIにはブロードキャストを実現する`MPI_Bcast`関数があります。

```cpp
MPI_Bcast(
    void* data,
    int count,
    MPI_Datatype datatype,
    int root,
    MPI_Comm communicator)
```

ルートプロセスは送信をし、他のプロセスは受信を行うのですが共通して`MPI_Bcast`関数を呼びます。ルートプロセス（この例ではプロセス0）が`MPI_Bcast`を呼ぶと`data`が他のすべてのプロセスに送信されます。すべての受信プロセスは`MPI_Bcast`でルート・プロセスからの`data`を受け取ります。

## MPI_Send と MPI_Recv によるブロードキャスト - Broadcasting with MPI_Send and MPI_Recv
`MPI_Bcast`は`MPI_Send`と`MPI_Recv`のラッパーなのでしょうか？実際、このラッパーはsendとrecvを使って簡単に実装することもできます。[my_bcast.c]({{ site.github.code }}/tutorials/mpi-broadcast-and-collective-communication/code/my_bcast.c)に示す`my_bcast`という関数は、`MPI_Bcast`と同じ引数をとる自作のブロードキャスト関数です。

```cpp
void my_bcast(void* data, int count, MPI_Datatype datatype, int root,
              MPI_Comm communicator) {
  int world_rank;
  MPI_Comm_rank(communicator, &world_rank);
  int world_size;
  MPI_Comm_size(communicator, &world_size);

  if (world_rank == root) {
    // ルートプロセスはforで各プロセスにデータを送る
    int i;
    for (i = 0; i < world_size; i++) {
      if (i != world_rank) {
        MPI_Send(data, count, datatype, i, 0, communicator);
      }
    }
  } else {
    // ルートでないプロセスはルートプロセスからのデータを受け取る
    MPI_Recv(data, count, datatype, root, 0, communicator,
             MPI_STATUS_IGNORE);
  }
}
```

コメントの通りで、ルートプロセスが他のプロセスにデータを送り、他のプロセスはルートプロセスからデータを受け取る。とても簡単ですね。[レポジトリ]({{ site.github.code }}/tutorials/mpi-broadcast-and-collective-communication/code/)のチュートリアルディレクトリからmy_bcastプログラムを実行してみましょう。

```
>>> cd tutorials
>>> ./run.py my_bcast
mpirun -n 4 ./my_bcast
Process 0 broadcasting data 100
Process 2 received data 100 from root process
Process 3 received data 100 from root process
Process 1 received data 100 from root process
```

動作はしますが、この自作関数は非常に非効率的です。各プロセスには送信/受信ネットワークのリンクが1つしかないのです。つまりプロセス0から常に1つのネットワーク リンクのみを使用することを繰り返してすべてのデータを送信します。ネットワークリンクを一度に多く使用できる賢い方法を考えましょう。ツリー(木)ベースの通信アルゴリズムです。

![MPI_Bcast tree](../broadcast_tree.png)

この図を説明します。最初のステップでプロセス0はデータをプロセス1に送信します。次のステップでプロセス0もデータをプロセス2に送信します。さらにプロセス1はプロセス3にデータを送信します。他のプロセスがルートプロセスを助けて2つのネットワーク接続がブロードキャストに使用されています。このようにすべてのプロセスがデータを受信するまで各ステップごとにネットワークの使用率は2倍になります。

この実装コードを書くことはレッスンの目的から少し外れます。詳細が気になる場合は[Parallel Programming with MPI](http://www.amazon.com/gp/product/1558603395/ref=as_li_qf_sp_asin_tl?ie=UTF8&tag=softengiintet-20&linkCode=as2&camp=217145&creative=399377&creativeASIN=1558603395)を参照してください。コードの問題の完全な例が掲載されている優れた本です。

## MPI_Bcast と MPI_Send および MPI_Recv との比較 - Comparison of MPI_Bcast with MPI_Send and MPI_Recv
`MPI_Bcast`はネットワーク利用効率の改善のために、今紹介したようなツリー型のブロードキャストアルゴリズムを利用しています。自作ブロードキャストを`MPI_Bcast`と比較してみましょう。このための`compare_bcast`を実行します。`compare_bcast`は、レッスンコードに含まれているサンプルプログラムです([compare_bcast.c]({{ site.github.code }}/tutorials/mpi-broadcast-and-collective-communication/code/compare_bcast.c))。コードの説明の前にまずはMPIのタイミング(timing)関数の1つである`MPI_Wtime`の説明をします。`MPI_Wtime`は引数を取らず、過去の実行の秒数を浮動小数点数で返します。うまく使用することでCの`time`関数と同じようにプログラム中で複数の`MPI_Wtime`関数を呼び出して、差分を引くことでセグメントごとの時間を取得することができる関数です。

`my_bcast` と `MPI_Bcast` を比較するコードを書きます。

```cpp
for (i = 0; i < num_trials; i++) {
  // Time my_bcast
  // バリアをして事前時間を同期する
  MPI_Barrier(MPI_COMM_WORLD);
  total_my_bcast_time -= MPI_Wtime();
  my_bcast(data, num_elements, MPI_INT, 0, MPI_COMM_WORLD);
  // Synchronize again before obtaining final time
  MPI_Barrier(MPI_COMM_WORLD);
  total_my_bcast_time += MPI_Wtime();

  // Time MPI_Bcast
  MPI_Barrier(MPI_COMM_WORLD);
  total_mpi_bcast_time -= MPI_Wtime();
  MPI_Bcast(data, num_elements, MPI_INT, 0, MPI_COMM_WORLD);
  MPI_Barrier(MPI_COMM_WORLD);
  total_mpi_bcast_time += MPI_Wtime();
}
```

`num_trials`は実行する回数を示す変数です。 ２つの関数ごとに実行時間を計算します。そして平均時間をプログラムの最後で出力します。 コード全体は、[compare_bcast.c]({{ site.github.code }}/tutorials/mpi-broadcast-and-collective-communication/code/compare_bcast.c)を見てください。[レッスンコード]({{ site.github.code }}/tutorials/mpi-broadcast-and-collective-communication/code)で確認できます。

[レポジトリ]({{ site.github.code }})の compare_bcast プログラムを実行すると出力は次のようになります。

```
>>> cd tutorials
>>> ./run.py compare_bcast
/home/kendall/bin/mpirun -n 16 -machinefile hosts ./compare_bcast 100000 10
Data size = 400000, Trials = 10
Avg my_bcast time = 0.510873
Avg MPI_Bcast time = 0.126835
```

スクリプトは、16個のプロセッサにおいて1ブロードキャスト呼び出しあたり100,000 個の整数を送ることを10回試行します。Ethernet経由で接続された16プロセッサを使用した私の環境ではmy_bcastとMPI実装の間に大きな実行時間の差が生まれました。以下は、さまざまなプロセッサ数での結果です。

| Processors | my_bcast | MPI_Bcast |
| --- | --- | --- |
| 2 | 0.0344 | 0.0344 |
| 4 | 0.1025 | 0.0817 |
| 8 | 0.2385 | 0.1084 |
| 16 | 0.5109 | 0.1296 |

2つのプロセッサでは2つの実装の時間差はありません。これは`MPI_Bcast`がツリー実装を使ったとしても、2つのプロセッサの場合はネットワーク使用率には違いがないためです。ただし16プロセッサまで増やすと違いがはっきりとわかります。

試してみてください！

## Conclusions / up next
集団通信に慣れてきましたか？次は[ScatterとGather]({{ site.baseurl }}/tutorials/mpi-scatter-gather-and-allgather/)を学びましょう！

その他のレッスンは[MPIチュートリアル]({{ site.baseurl }}/tutorials/)をみてください。
