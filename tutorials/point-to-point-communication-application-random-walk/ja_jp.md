---
layout: post
title: ポイントツーポイント通信アプリケーション ランダムウォーク - Point-to-Point Communication Application - Random Walk
author: Wes Kendall
categories: Beginner MPI
tags:
redirect_from: '/point-to-point-communication-application-random-walk/'
---

[sending and receiving tutorial]({{ site.baseurl }}/tutorials/mpi-send-and-receive/)と[MPI_Probe and MPI_Status lesson]({{ site.baseurl }}/tutorials/dynamic-receiving-with-mpi-probe-and-mpi-status/) の レッスンで紹介された概念のいくつかを使用して実際のアプリケーションを作成しましょう。これは「ランダムウォーク」と呼ばれるシミュレートションです。


> **Note** - このサイトのコードはすべて[GitHub]({{ site.github.repo }})にあります。このレッスンのコードは[tutorials/point-to-point-communication-application-random-walk/code]({{ site.github.code }}/tutorials/point-to-point-communication-application-random-walk/code)にあります。
 
ランダムウォークの問題設定を説明します。この問題ではある数直線の*Min*、*Max*とおよびランダムウォーカー*W*が与えられます。ウォーカー*W*は適当なの長さのランダムウォークを右に*S*回実行します。右端に到達したら、プロセスは左端に戻ります(訳注: If the process goes out of bounds, it wraps back around. で折り返すとあるが、スタートに戻る実装です)。*W*は一度に右か1ユニットしか移動できません(訳注: to the right or left at a timeとありますが、右にだけ動く実装です)。

![Random walk illustration](../random_walk.png)

この動作はとても基本的ですが、ランダムウォークの並列化というのはさまざまな並列アプリケーションのシミュレートで用いられる手法です。このことは最後に詳しく説明します。まずは、ランダムウォークの問題を並列化する方法を考えます。

## ランダムウォーキング問題の並列化 - Parallelization of the random walking problem
多くの並列プログラミングにとって最初に考えなければいけないのは、各プロセスがが持つ領域の分割です。ランダムウォークは*Max - Min + 1* の大きさの1次元の領域を考えます(*Max*と*Min*は含まれて良い値です)。ウォーカーのステップを整数とて考えると、各領域をほぼ等しいサイズに分割することができるでしょう。例えば*Min*が0、*Max*が20で、分割するプロセスが4つの場合はドメインを次のように分割できます。

![Domain decomposition example](../domain_decomp.png)

最初の3プロセス(プロセス0,1,2)は5つのユニットを管理します。最後のプロセス3は合計6つのユニットを管理します。これは、同じように分割された5つのユニットに加えて残りの1つのユニットです。ドメインを分割したらアプリケーションはウォーカーを初期化します。ウォーカーはランダムなウォークサイズでS回のウォークを実行します。今説明した分割においてウォーカーがプロセス0からサイズ6のウォークを実行すると次のようになります。（訳注：サイズ6のウォークとは、６ユニット移動する動作です）

1. プロセス0からウォーカーは6のステップします。値が4になった時プロセス0の境界に達しています。プロセス0はプロセス1にウォーカーを渡します。

2. プロセス1はウォーカーを受け取ったあと、合計が6に達するまでウォークを続けます。このサイクルを１つとしてウォーカーは新しいランダムウォークを試行します。

![Random walk, step one](../walk_step_one.png)

*W*はプロセス0からプロセス1に1回だけ移動しましたが*W*がさらに長い距離を移動する時はドメイン内のより多くのプロセスを通過する必要がある可能性があります。

## CMPI_Send と MPI_Recv を使用したコーディング - oding the application using MPI_Send and MPI_Recv
この動作を`MPI_Send` と`MPI_Recv`を使用して書いていきましょう。まず、前にプログラムの動きを整理します。

* プロセスはドメイン内の自分の領域を識別します
* 各プロセスは*N*個のウォーカーを初期化します。これらはすべてそのプロセスの最初の値から開始されます。（訳注: 5-9を管理するプロセスであれば、このプロセスに割り当てられた全てのウォーカーは5にいることにします）
* 各ウォーカーはオブジェクトです。ウォーカーの現在位置と残歩数という 2 つのintを持ちます。
* ウォーカーたちはドメイン内でウォークを開始し、ウォークを完了するまで他のプロセスへの移動を行います。
* すべてのウォーカーが終了するとそのターンは終了します。

まずはプロセスが領域を分割するコードを書きます。この関数はドメインの合計サイズを受け取りMPIプロセスが担当する適切なサイズの分割された領域を決定します。そして残ってしまった領域を最後のプロセスに担当させます。簡単のためエラーが見つかった場合は`MPI_Abort`を呼び出すことにします。この関数`decompose_domain`を示します。

```cpp
void decompose_domain(int domain_size, int world_rank,
                      int world_size, int* subdomain_start,
                      int* subdomain_size) {
    if (world_size > domain_size) {
        // 領域の数が用意できるプロセスの数より大きいならエラーとします
        MPI_Abort(MPI_COMM_WORLD, 1);
    }
    *subdomain_start = domain_size / world_size * world_rank;
    *subdomain_size = domain_size / world_size;
    if (world_rank == world_size - 1) {
        // 残りを最後のプロセスに割り当てます
        *subdomain_size += domain_size % world_size;
    }
  }
```

関数は余りがある場合を配慮しながらドメインを均等な領域に分割します。そして関数はその領域の開始位置とサブドメインのサイズを返します。

ウォーカーは次の構造体で定義します。

```cpp
typedef struct {
    int location;
    int num_steps_left_in_walk;
} Walker;
```

次に`initialize_walkers`という初期化関数を考えます。この関数はそのプロセスの領域を取得し、そのプロセスが担当するウォーカーを配列`incoming_walkers`に追加します (このアプリケーションは C++ で書かれています)。

```cpp
void initialize_walkers(int num_walkers_per_proc, int max_walk_size,
                        int subdomain_start, int subdomain_size,
                        vector<Walker>* incoming_walkers) {
    Walker walker;
    for (int i = 0; i < num_walkers_per_proc; i++) {
        // Initialize walkers in the middle of the subdomain
        // 訳注: "in the mid"とあるが、ウォーカーは領域の最初から開始する
        walker.location = subdomain_start;
        walker.num_steps_left_in_walk =
            (rand() / (float)RAND_MAX) * max_walk_size;
        incoming_walkers->push_back(walker);
    }
}
```

初期化の次はウォーカーを前進させる関数です。この関数はウォーカーが歩行を完了するまで前進させます。ウォーカーが自分の管理する領域の端より進もうとしたら、そのプロセスの配列`outgoing_walkers`に追加します。

```cpp
void walk(Walker* walker, int subdomain_start, int subdomain_size,
          int domain_size, vector<Walker>* outgoing_walkers) {
    while (walker->num_steps_left_in_walk > 0) {
        if (walker->location == subdomain_start + subdomain_size) {
            // Take care of the case when the walker is at the end
            // of the domain by wrapping it around to the beginning
            if (walker->location == domain_size) {
                walker->location = 0;
            }
            outgoing_walkers->push_back(*walker);
            break;
        } else {
            walker->num_steps_left_in_walk--;
            walker->location++;
        }
    }
}
```

初期化関数 (`incoming_walkers`にデータを追加する) とウォーキング関数 (`outgoing_walkers`にデータを追加する)ができたので、あとは`outgoing_walkers`を次のプロセスに送る関数と受け取る関数の2つがあれば良いです。まずは`outgoing_walkers`を次に送る関数を見ていきます。

```cpp
void send_outgoing_walkers(vector<Walker>* outgoing_walkers, 
                           int world_rank, int world_size) {
    // 配列のデータをMPI_BYTEsのバイトデータとして次のプロセスに送ります
    // 右端のプロセスはプロセス0にデータを送ることに注意します
    MPI_Send((void*)outgoing_walkers->data(), 
             outgoing_walkers->size() * sizeof(Walker), MPI_BYTE,
             (world_rank + 1) % world_size, 0, MPI_COMM_WORLD);

    // 次のプロセスにデータを送ったので配列はクリアします
    outgoing_walkers->clear();
}
```

次は受け取る関数です。受け取るウォーカーの数が事前にわからないため`MPI_Probe`を使用する必要があります。

```cpp
void receive_incoming_walkers(vector<Walker>* incoming_walkers,
                              int world_rank, int world_size) {
    MPI_Status status;

    // 前のプロセスからのデータを受け取ります
    // プロセス0は右端のプロセスからのデータを受け取ります
    int incoming_rank =
        (world_rank == 0) ? world_size - 1 : world_rank - 1;
    MPI_Probe(incoming_rank, 0, MPI_COMM_WORLD, &status);

    // そして受け取るべきデータのサイズ分のメモリを確保します
    int incoming_walkers_size;
    MPI_Get_count(&status, MPI_BYTE, &incoming_walkers_size);
    incoming_walkers->resize(
        incoming_walkers_size / sizeof(Walker));
    MPI_Recv((void*)incoming_walkers->data(), incoming_walkers_size,
             MPI_BYTE, incoming_rank, 0, MPI_COMM_WORLD,
             MPI_STATUS_IGNORE); 
}
```

これでプログラムの主な機能を実装しました。次のように処理を作っていきましょう。

1. ウォーカーを初期化する。
2. `walk` 関数でウォーカーを進める。
3. `outgoing_walkers`にあるウォーカーを次に送る
4. 新しいウォーカーは`incoming_walkers`に入れる。
5. すべてのウォーカーが終了するまで、ステップ2から4を繰り返す。

以下の通りとなるが、まずは5で行いたいすべてのウォーカーの終了判定は気にしないものとします。このコードには誤りがあるのでそれに留意してご覧ください。

```cpp
// このプロセスの領域を決める
decompose_domain(domain_size, world_rank, world_size,
                 &subdomain_start, &subdomain_size);

// このプロセスのウォーカーを配置する(incoming_walksに入れる)
initialize_walkers(num_walkers_per_proc, max_walk_size,
                   subdomain_start, subdomain_size,
                   &incoming_walkers);

while (!all_walkers_finished) { // 全てのウォーカーが終了するまで
    // このプロセス内全てのウォーカーの動きを実施
    for (int i = 0; i < incoming_walkers.size(); i++) {
        walk(&incoming_walkers[i], subdomain_start, subdomain_size,
             domain_size, &outgoing_walkers); 
    }

    // 外に出ていくウォーカーの処理
    send_outgoing_walkers(&outgoing_walkers, world_rank,
                          world_size);

    // 新しく入ってきたウォーカーの処理
    receive_incoming_walkers(&incoming_walkers, world_rank,
                             world_size);
}
```

良さそうに見えますか？しかし、これはデッドロックが起こりやすいコードとなっています。

## デッドロックとその予防方法 - Deadlock and prevention
Wikipediaによれば、デッドロックとは「2つ以上のプロセスがそれぞれ他のプロセスのリソース解放を待つ。あるいは2つ以上のプロセスが循環的にリソースを待っている。こういった特定の状態のこと」です。上記のコードは`MPI_Send`の循環的なチェーンが発生します。

![Deadlock](../deadlock-1.png)

とはいえ、実施には上記コードはデッドロックはほぼ発生*しません*。`MPI_Send`はブロッキング関数ですが、[MPI specification](http://www.amazon.com/gp/product/0262692163/ref=as_li_tf_tl?ie=UTF8&tag=softengiintet-20&linkCode=as2&camp=217145&creative=399377&creativeASIN=0262692163)では、`MPI_Send` は*送信バッファが確保できるまで*ブロックすると記載されています。MPI_Sendはネットワークがメッセージをバッファできるようになったときにブロッキングが終わります。つまりネットワークがバッファできなければそれに対応するreceiveが呼ばれるまでsendはブロックされるということです。今回のケースでは非常に小さな送信に対して非常に頻繁に受信を呼び出すコードであるためデッドロックはほぼないでしょう。しかし、全てのケースでネットワークバッファが十分に大きいと想定して並列プログラミングを書いてはいけません。

このレッスンでは `MPI_Send` と `MPI_Recv` だけに焦点をあてています。送受信のデッドロックを回避するベストの方法は送信と受信が一致するようにメッセージングを順序付けることです。いくつかの方法が考えられますが、１つとしては偶数番目のプロセスが受信の前に送信を送り、奇数番目のプロセスがその逆を行うようにすることです。もし、2つの実行ステージで考えると次のようなイメージとなります。

![Deadlock prevention](../deadlock-2.png)

> **Note** - これを1プロセスの環境下で実行するとデッドロックが発生する例外があります。

奇数個のプロセスでもこれは機能するのでしょうか？3つのプロセスで同様の図をもう一度見てみましょう。

![Deadlock solution](../deadlock-3.png)

3つのパターン全てにおいて、少なくとも1つの`MPI_Send`と`MPI_Recv`が存在するパスがあるので、デッドロックの発生を心配する必要はないとわかりました。

## 終了の判断 - Determining completion of all walkers
それでは、すべてのウォーカーが終了したかを判断するステップを考えます。ウォーカーはランダムな距離をいどうするため、あるウォーカーが移動を終了することは全てのプロセスで起こり得ることに注意します。そのため、何らかの追加通信を行わずに全プロセスがすべての歩行者が終了を知ることは困難です。考えられる解決策の1つはプロセス0がすべての歩行者を追跡し、他のすべてのプロセスに終了を伝えることが考えられます。ただし、この解決策は各移動においてプロセス0以外のプロセスはプロセス0に完了した歩行者を報告し、さらにさまざまな種類の受信メッセージを処理する必要があるため非常に面倒ですね。

このレッスンではもっとシンプルに考えます。どのウォーカーも移動できる最大距離と、ある送受信のプロセスのペアで移動できる最小の合計サイズ（サブドメインのサイズ）がわかっています。このため、各プロセスが終了までに行うべき送受信の量が定まります（訳注：全てのプロセスが最大の距離を行うとき、というのがこれに当てはまります）。この特徴とデッドロックを回避する戦略を用いると以下のように考えられます。

```cpp
// このプロセスの領域を決める
decompose_domain(domain_size, world_rank, world_size,
                 &subdomain_start, &subdomain_size);

// このプロセスのウォーカーを配置する(incoming_walksに入れる)
initialize_walkers(num_walkers_per_proc, max_walk_size,
                  subdomain_start, subdomain_size,
                  &incoming_walkers);

// 全てのウォーカーが終了するのに必要なsend/recv数を計算
int maximum_sends_recvs =
    max_walk_size / (domain_size / world_size) + 1;
for (int m = 0; m < maximum_sends_recvs; m++) {
    // このプロセス内全てのウォーカーの動きを実施
    for (int i = 0; i < incoming_walkers.size(); i++) {
        walk(&incoming_walkers[i], subdomain_start, subdomain_size,
             domain_size, &outgoing_walkers); 
    }

    // 偶数・奇数に基づいた順序で送受信を行う
    if (world_rank % 2 == 0) {
        send_outgoing_walkers(&outgoing_walkers, world_rank,
                              world_size);
        receive_incoming_walkers(&incoming_walkers, world_rank,
                                 world_size);
    } else {
        receive_incoming_walkers(&incoming_walkers, world_rank,
                                 world_size);
        send_outgoing_walkers(&outgoing_walkers, world_rank,
                              world_size);
    }
}
```

## Running the application

レッスンのコードは[ここ]({{ site.github.code }}/tutorials/point-to-point-communication-application-random-walk/code)で見ることができます. 他のレッスンとは異なり、このコードはC++を使用しています。[installing MPICH2]({{ site.baseurl }}/tutorials/installing-mpich2/)の際に、C++ MPIコンパイラもインストールしました(明示的に設定した場合を除く)。MPICH2をローカル・ディレクトリにインストールした場合はMPICXX環境変数が正しいmpicxxコンパイラーを指すように設定されていることを確認してください。

私のコードでは、アプリケーションの実行スクリプトにプログラムのデフォルト値を設定しています。ドメインサイズは 100、最大ウォークサイズは 500、プロセスあたりのウォーカー数は 20 です。random_walkプログラムを[レポジトリ]({{ site.github.code }})の*tutorials*ディレクトリから実行すると、5つのプロセスが生成され、このような出力が得られます。

```
>>> cd tutorials
>>> ./run.py random_walk
mpirun -n 5 ./random_walk 100 500 20
Process 2 initiated 20 walkers in subdomain 40 - 59
Process 2 sending 18 outgoing walkers to process 3
Process 3 initiated 20 walkers in subdomain 60 - 79
Process 3 sending 20 outgoing walkers to process 4
Process 3 received 18 incoming walkers
Process 3 sending 18 outgoing walkers to process 4
Process 4 initiated 20 walkers in subdomain 80 - 99
Process 4 sending 18 outgoing walkers to process 0
Process 0 initiated 20 walkers in subdomain 0 - 19
Process 0 sending 17 outgoing walkers to process 1
Process 0 received 18 incoming walkers
Process 0 sending 16 outgoing walkers to process 1
Process 0 received 20 incoming walkers
```

プロセスはすべてのウォーカーの送受信を終える(と想定される回数)まで出力を続けます。

## So what's next?
いかがでしょうか。もしこのレッスンを心地よいと感じたなら良いことです。このアプリケーションは、初めての実際のアプリケーションとしてはかなり発展的なものです。

次回のレッスンからはMPIでの*集団通信*について学習します。まず、[MPI Broadcast]({{ site.baseurl }}/tutorials/mpi-broadcast-and-collective-communication/)を学習します。その他のレッスンは[MPI tutorials]({{ site.baseurl }}/tutorials/)をみてください。

冒頭で、このプログラムのコンセプト(ランダムウォーク)は多くの並列プログラムにも応用できることをお伝えしました。もっと学びたい人のために、以下に追加資料を掲載したのでお楽しみください :-)

## 追加資料 - ランダムウォーキングと並列粒子追跡の類似性 Additional reading - Random walking and its similarity to parallel particle tracing
ランダムウォーク問題は、一見単純なものに見えますが実は多くの種類の並列アプリケーションのシミュレーションの基礎となります。科学分野の一部の並列アプリケーションでは、多くの種類のランダムな送受信が必要です。1つのアプリケーション例は並列粒子追跡(parallel particle tracing)です。

![Flow visualization of tornado](../tornado.png)

並列粒子追跡は流れ場を可視化するための主要な手法の1つです。粒子を流れ場に仮定して数値積分技術(Runge-Kutta法など)を用いて流れに沿って追跡します。この経路は、可視化のために描画することができます。描画の一例として上のトルネード画像があります。

効率的な並列粒子追跡というのは非常に困難しいです。主な理由は粒子の移動方向が積分の各増分ステップの後にしか決定できないためです。したがってプロセスがすべての通信と計算を調整してバランスをとるのは困難です。より理解するために、粒子追跡の一般的な並列化を見てみましょう。

![Parallel particle tracing illustration](../parallel_particle_tracing.png)

この図はドメインを6つのプロセスに分割していることがわかります。粒子（時にはシードと呼ばれる）が各サブドメインに配置され（ウォーカーをそれぞれの領域に配置した方法に似ています）、その後トレースを開始します。粒子が境界を越えると、適切なサブドメインを持つプロセスと情報を交換します。このプロセスは粒子が領域から離れるか最大トレースの回数に達するまで繰り返されます。

並列粒子追跡の問題は、先ほどコーディングしたアプリケーションと同様に`MPI_Send`、`MPI_Recv`、`MPI_Probe`を使用して解決できます。より効率的に作業を行うために、もっと洗練されたMPIルーチンも存在するのでそれは次のレッスンでお話しします :-)

ランダムウォークの問題が他の並列アプリケーションとどのように似ているかを示す例を少なくとも1つは確認できたと思います。