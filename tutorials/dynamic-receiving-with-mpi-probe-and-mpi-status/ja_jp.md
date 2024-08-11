---
layout: post
title: MPI Probeを用いた可変長メッセージの受信 - Dynamic Receiving with MPI Probe (and MPI Status)
author: Wes Kendall
categories: Beginner MPI
tags: MPI_Get_count, MPI_Probe
redirect_from: '/dynamic-receiving-with-mpi-probe-and-mpi-status/'
---

[前回のレッスン]({{ site.baseurl }}/tutorials/mpi-send-and-receive/)ではMPI_SendとMPI_Recvを使用した基本的なポイントツーポイント通信を学びました。前回はメッセージ長が固定である場合のみを説明しました。可変長メッセージを送る方法として１つ目のメッセージ長をsend/recvするという方法もとれます。しかし、MPIには追加の関数呼び出しだけで可変長のメッセージをサポートすることが可能です。このレッスンではこの方法を学びます。

> **Note** - このサイトのコードはすべて[GitHub]({{ site.github.repo }})にあります。このレッスンのコードは[tutorials/dynamic-receiving-with-mpi-probe-and-mpi-status/code]({{ site.github.code }}/tutorials/dynamic-receiving-with-mpi-probe-and-mpi-status/code)にあります。

## MPI_Status構造体 - The MPI_Status structure
[前回のレッスン]({{ site.baseurl }}/tutorials/mpi-send-and-receive/)で説明したように、`MPI_Recv`は受信したメッセージに関する情報を`MPI_Status`として受け取ります(`MPI_STATUS_IGNORE`で無視することもできます)。`MPI_Recv`関数に`MPI_Status`を渡していると受信操作が完了した後に次の3つの追加情報を得ることができます。

1. **送信者のランク**: 送信者のランクは`MPI_SOURCE`構造体に格納されます。`MPI_Status stat`と宣言すると、ランクには`stat.MPI_SOURCE`でアクセスできます。
2. **メッセージタグ**: メッセージのタグは`MPI_TAG`に格納されます。`MPI_SOURCE`と同じようにアクセスできます。
3. **メッセージ長**: メッセージ長は、ステータス構造内には含まれません。そこで`MPI_Get_count`を使用してメッセージの長さを調べる必要があります。

```cpp
MPI_Get_count(
    MPI_Status* status,
    MPI_Datatype datatype,
    int* count)
```

`MPI_Get_count`関数に`MPI_Status`渡すとメッセージの`datatype`と`count`が返されます。`count`には受信した要素の合計数が入ります。

なぜこの２つの情報が必要になるのかを説明します。`MPI_Recv`受信のために`MPI_ANY_SOURCE`を送信者のランクに、`MPI_ANY_TAG`をメッセージのタグとして受信動作を行えます。この場合には`MPI_Status`が送信者とメッセージタグを知る唯一の手がかりになります（訳注：rankとtagを指定してrecvしていない場合、という意味です。）。また、`MPI_Recv`関数は引数として指定した要素数を全て受信することが保証されていないことに注意します。受信した要素数を得ることができます。しかしながら、指定した受信できる数以上の要素が送信された場合はエラーになることに注意してください。`MPI_Get_count`は実際に受信する量を決定するのに使われます。

## MPI_Status構造体をクエリする例 - An example of querying the MPI_Status structure

`MPI_Status`構造体をクエリするプログラム[check_status.c]({{ site.github.code }}/tutorials/dynamic-receiving-with-mpi-probe-and-mpi-status/code/check_status.c)を見ていきます。プログラムは数字を適当な個数送信します。受信者は送信された数字の数を調べます。コードのメイン部分は次のようになります。

```cpp
const int MAX_NUMBERS = 100;
int numbers[MAX_NUMBERS];
int number_amount;
if (world_rank == 0) {
    // プロセス1に送る数字の数を決定する
    srand(time(NULL));
    number_amount = (rand() / (float)RAND_MAX) * MAX_NUMBERS;

    // その分の数をプロセス1に送る
    MPI_Send(numbers, number_amount, MPI_INT, 1, 0, MPI_COMM_WORLD);
    printf("0 sent %d numbers to 1\n", number_amount);
} else if (world_rank == 1) {
    MPI_Status status;
    // 最大でMAX_NUMBERS個のMPI_INTをプロセス0から受け取る
    MPI_Recv(numbers, MAX_NUMBERS, MPI_INT, 0, 0, MPI_COMM_WORLD,
             &status);

    // メッセージを受信した後、そのメッセージにいくつの整数が含まれていたかを
    // 取得する
    MPI_Get_count(&status, MPI_INT, &number_amount);

    // 含まれていた数字の数、ランク、タグを出力する。
    printf("1 received %d numbers from 0. Message source = %d, "
           "tag = %d\n",
           number_amount, status.MPI_SOURCE, status.MPI_TAG);
}
```

プロセス0は最大で`MAX_NUMBERS`個の整数をランダムに決めてプロセス1に送信します。プロセス1は最大で`MAX_NUMBERS`個の整数を読み込む`MPI_Recv`を実行します。プロセス1は`MPI_Recv` の引数として `MAX_NUMBERS` を渡しています。繰り返しになりますが、プロセス1が受け取ることができるのは**最大**この個数であることに注意してください。プロセス1では`MPI_Get_count`を呼び出して、実際に受信した`MPI_INT`の数をを調べます。プロセス1は受信したメッセージのサイズを表示すると同時に、`MPI_SOURCE`と`MPI_TAG`、つまり送信元ランクとタグも表示します。

ここで注意があります。`MPI_Get_count`で得られるのはデータ型の数でバイト数でないことです。もしユーザーが`MPI_CHAR`をデータ型として(recvを)使用したとすると、返されるデータ量は4倍になります(intを4バイト、charを1バイトと仮定した場合)。このプログラムを[レポジトリ]({{ site.github.code }})の*tutorials*ディレクトリから実行すると、出力はこのようになるでしょう。

```
>>> cd tutorials
>>> ./run.py check_status
mpirun -n 2 ./check_status
0 sent 92 numbers to 1
1 received 92 numbers from 0. Message source = 0, tag = 0
```

プロセス0はプロセス1にランダムな数の整数を送信して、プロセス1は受信したメッセージに関する情報を出力できました。

## MPI_Probeを使用してメッセージサイズを調べる　- Using MPI_Probe to find out the message size
`MPI_Status`への理解が深まってきました。前回の例では受信前にすべてのサイズのメッセージを処理できるように大きなバッファを用意しました。実は、`MPI_Probe`を使用すると受信前にメッセージ長を調べることができます。

```cpp
MPI_Probe(
    int source,
    int tag,
    MPI_Comm comm,
    MPI_Status* status)
```

`MPI_Probe`は`MPI_Recv`と似た呼び出しです。`MPI_Probe`は`MPI_Recv`で実際に受信する以外の処理を行うと考えて良いです。`MPI_Recv`と同様に`MPI_Probe`は指定したタグかつ指定した送信者ランクであるメッセージが来るまでブロッキングします。メッセージを受信するとstatus構造体に情報が格納されます。その後にユーザは(そのデータを受信するのに十分なバッファを用意して)`MPI_Recv`で実際にメッセージを受信すれば良いのです。

[レポジトリ]({{ site.github.code }}/tutorials/dynamic-receiving-with-mpi-probe-and-mpi-status/code/code)の[probe.c]({{ site.github.code }}/tutorials/dynamic-receiving-with-mpi-probe-and-mpi-status/code/probe.c)にこの例を示します。

```cpp
int number_amount;
if (world_rank == 0) {
    const int MAX_NUMBERS = 100;
    int numbers[MAX_NUMBERS];
    // プロセス1に送る数字の数を決定する
    srand(time(NULL));
    number_amount = (rand() / (float)RAND_MAX) * MAX_NUMBERS;

    // その分の数をプロセス1に送る
    MPI_Send(numbers, number_amount, MPI_INT, 1, 0, MPI_COMM_WORLD);
    printf("0 sent %d numbers to 1\n", number_amount);
} else if (world_rank == 1) {
    MPI_Status status;
    // プロセス0からのメッセージを"probe"する
    MPI_Probe(0, 0, MPI_COMM_WORLD, &status);

    // probeが完了した時、statusにはメッセージ長などの情報が含まれている
    // MPI_Get_countを使ってメッセージ長を得る
    MPI_Get_count(&status, MPI_INT, &number_amount);

    // probeで判明したメッセージ長文のメモリを確保する
    int* number_buf = (int*)malloc(sizeof(int) * number_amount);

    // そのバッファを使用してrecv処理を行う
    MPI_Recv(number_buf, number_amount, MPI_INT, 0, 0,
             MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    printf("1 dynamically received %d numbers from 0.\n",
           number_amount);
    free(number_buf);
}
```

先程の例と同様、プロセス0はプロセス1に送信する変数の数をランダムに決定します。変更点は、プロセス1が `MPI_Probe`を呼び出してプロセス0が送信しようとしている要素の数を(`MPI_Get_count`を使用して)得ることです。この後、プロセス1が適切なサイズのバッファを確保してから数値を受信します。

```
>>> ./run.py probe
mpirun -n 2 ./probe
0 sent 93 numbers to 1
1 dynamically received 93 numbers from 0
```

ここで示した例はシンプルなものです。`MPI_Probe`は多くのMPIアプリケーションで用いられる基本的な機能です。例えばマネージャ/ワーカプログラムは、可変サイズのメッセージを交換する際に`MPI_Probe`を多用します。練習として、`MPI_Probe`を使用した`MPI_Recv`のラッパーを作成し、動的なアプリケーションを書いてみてください。コードの見栄えが格段に良くなるでしょう :-)

## Up next
さて、標準的なブロック型ポイント・ツー・ポイント通信に抵抗はなくなってきましたか？そうなら、あなたはすでにいくらでも並列アプリケーションを書く能力を持っています！では、学んだルーチンを使ったより高度な例を見てみましょう。[the application example using MPI_Send, MPI_Recv, and MPI_Probe]({{ site.baseurl }}/tutorials/point-to-point-communication-application-random-walk/) をチェックしてください。

困っていますか?混乱していますか?お気軽に下記にコメントを残してください。私や他の読者がお役に立てるかもしれません。
