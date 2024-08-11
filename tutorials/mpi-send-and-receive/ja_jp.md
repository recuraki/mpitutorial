---
layout: post
title: MPIのsendとreceive - MPI Send and Receive
author: Wes Kendall
categories: Beginner MPI
tags: MPI_Recv, MPI_Send
redirect_from: '/mpi-send-and-receive/'
---

送信と受信(send,recv)は、MPIもっとも基本的な概念です。MPIのほぼすべての機能はsend,recvをによって実装できます。このレッスンではブロッキング送受信について説明し、MPIのデータ送信の基本的な概念を説明します。

> **Note** - このサイトのコードはすべて[GitHub]({{ site.github.repo }})にあります。このレッスンのコードは[tutorials/mpi-send-and-receive/code]({{ site.github.code }}/tutorials/mpi-send-and-receive/code)にあります。

## MPI による送受信の概要: Overview of sending and receiving with MPI
MPIの基本的な送受信を見ていきましょう。まず、プロセス*A*は、プロセス*B*にメッセージを送信したいとします。プロセス*A*はプロセス*B*に送信したいデータをすべて１つにまとめて(パックして)バッファに格納します。このバッファは、データを1つのメッセージにまとめているのでエンベロープ(*envelopes*)と呼ばれることがあります(郵便の手紙が封筒にパックされるのと同じです)。データがバッファにパックされた後は通信デバイス(多くはネットワークでしょう)がメッセージを適切にルーティングします。メッセージにおける宛先はプロセスのランクによって定義されます。

メッセージは*B*にむけてルーティングされます。次にプロセス*B*は*A*からデータを受信する意思があることを明確に通知(acknowledge)する必要があります。通知が完了するとデータは送信されたことになり、プロセス*A*に対してデータが送信されたことが通知され、Aのブロッキングは完了し、次の処理に移れます。

次はAがBにいくつかのタイプ種類のメッセージを送信することを考えます。Bがこれらのメッセージを区別するために特別な方法を取らずとも、MPIはメッセージにID (タグ - tagsと呼ばれる)をつけた送信と受信ができます。プロセスBは特定のタグのメッセージのみを要求でき、その他の異なるタグのメッセージはBがそれを受け取る要求をするまでネットワークレイヤによってバッファリングされています。

MPI send関数とrecv関数の定義を見てみましょう。

```cpp
MPI_Send(
    void* data,
    int count,
    MPI_Datatype datatype,
    int destination,
    int tag,
    MPI_Comm communicator)
```

```cpp
MPI_Recv(
    void* data,
    int count,
    MPI_Datatype datatype,
    int source,
    int tag,
    MPI_Comm communicator,
    MPI_Status* status)
```

複雑で長い引数に見えるかもしれませんが、多くのMPI関数では同様の構文が使用されるためご安心ください。最初の引数はデータバッファのアドレスです。2番目と3番目の引数は、バッファー内にある要素の数とタイプです。`MPI_Send`は正確な数の要素を送信し、`MPI_Recv`は"最低でも"count個の要素を受信します(これについては次のレッスンで詳しく説明します)。4番目と5番目の引数は、sendあるいはrecvするプロセスのランクとタグを指定します。6番目の引数はコミュニケータを指定します。recvのみに含まれる最後の引数`status`は、受信したメッセージに関する情報のポインタです。

## 基本的な MPI データ型 - Elementary MPI datatypes
`MPI_Send`と`MPI_Recv`関数は、メッセージのデータ構造をC言語のデータ型ではなくて、より高いレベルのデータ型で指定できます。たとえば、プロセスが1つの整数の送信する場合、カウント1と`MPI_INT`データ型を使用します。基本的なMPIデータ型と、それに相当するC言語のデータ型を以下に示します。

| MPIデータ型            | C言語のデータ型        |
| ---------------------- | ---------------------- |
| MPI_SHORT              | short int              |
| MPI_INT                | int                    |
| MPI_LONG               | long int               |
| MPI_LONG_LONG          | long long int          |
| MPI_UNSIGNED_CHAR      | unsigned char          |
| MPI_UNSIGNED_SHORT     | unsigned short int     |
| MPI_UNSIGNED           | unsigned int           |
| MPI_UNSIGNED_LONG      | unsigned long int      |
| MPI_UNSIGNED_LONG_LONG | unsigned long long int |
| MPI_FLOAT              | float                  |
| MPI_DOUBLE             | double                 |
| MPI_LONG_DOUBLE        | long double            |
| MPI_BYTE               | char                   |

これらのデータ型は初心者向けの次のMPIチュートリアルでのみ使用します。基礎を学習したら複雑なメッセージを扱うための独自のMPIデータ型を作成する方法も学習します。

## MPI send / recv program
それではコードを見ていきます。冒頭にあるように、このサイトのコードはすべて[GitHub]({{ site.github.repo }})にあります。このレッスンのコードは[tutorials/mpi-send-and-receive/code]({{ site.github.code }}/tutorials/mpi-send-and-receive/code)にあります。

最初に見るのは[send_recv.c]({{ site.github.code }}/tutorials/mpi-send-and-receive/code/send_recv.c). です。プログラムのメインとなる一部は以下の通りです。

```cpp
// Find out rank, size
int world_rank;
MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
int world_size;
MPI_Comm_size(MPI_COMM_WORLD, &world_size);

int number;
if (world_rank == 0) {
    number = -1;
    MPI_Send(&number, 1, MPI_INT, 1, 0, MPI_COMM_WORLD);
} else if (world_rank == 1) {
    MPI_Recv(&number, 1, MPI_INT, 0, 0, MPI_COMM_WORLD,
             MPI_STATUS_IGNORE);
    printf("Process 1 received number %d from process 0\n",
           number);
}
```

`MPI_Comm_rank`と`MPI_Comm_size`はワールドサイズとプロセスのランクを得るための関数です。プロセス0は整数データを-1に初期化し、この値をプロセス1に送信します。ステートメントでわかるように、プロセス1は数値を受信するために`else if`で分岐して、受信した値も出力します。送受信したいのは1つのINTだけなので、各プロセスは1つの`MPI_INT`の送信と受信を行います。また、メッセージを識別するためにタグ番号0を指定しました。送信されるメッセージの種類は1種類なのでプロセスはタグ番号に定義済みの定数`MPI_ANY_TAG`を使用してもよいです。

サンプルコードの全文は[GitHub]({{ site.github.repo }}) で確認できますし、`run.py` を使って実行することもできます。

```
>>> git clone {{ site.github.repo }}
>>> cd mpitutorial/tutorials
>>> ./run.py send_recv
mpirun -n 2 ./send_recv
Process 1 received number -1 from process 0
```

プロセス1はプロセス0から-1を受け取りましたことがわかります。

## MPI ping - MPI ping pong program
次の例はpingプログラムです。この例では、プロセスは`MPI_Send`と`MPI_Recv`を利用して停止するまでメッセージを送受信します。[ping_pong.c]({{ site.github.code }}/tutorials/mpi-send-and-receive/code/ping_pong.c)を参照してください。コードのメイン部分は次の通りです。

```cpp
int ping_pong_count = 0;
int partner_rank = (world_rank + 1) % 2;
while (ping_pong_count < PING_PONG_LIMIT) {
    if (world_rank == ping_pong_count % 2) {
        // Increment the ping pong count before you send it
        ping_pong_count++;
        MPI_Send(&ping_pong_count, 1, MPI_INT, partner_rank, 0,
                 MPI_COMM_WORLD);
        printf("%d sent and incremented ping_pong_count "
               "%d to %d\n", world_rank, ping_pong_count,
               partner_rank);
    } else {
        MPI_Recv(&ping_pong_count, 1, MPI_INT, partner_rank, 0,
                 MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        printf("%d received ping_pong_count %d from %d\n",
               world_rank, ping_pong_count, partner_rank);
    }
}
```

このコードは2つのプロセスのみで実行されることを想定します。プロセスは最初に簡単な演算で相手のランクを決定します。`ping_pong_count`は0で初期化し、プロセスの各送信ステップで+1します。そして、送信側と受信側を交互に担当します(訳註:一般的なpingと違い、互いに自発的にフレームを送ることに注意)。最後に、limitの回数に達すると(私のコードでは10)プロセスは動作を停止します。出力は次のようになります。

```
>>> ./run.py ping_pong
0 sent and incremented ping_pong_count 1 to 1
0 received ping_pong_count 2 from 1
0 sent and incremented ping_pong_count 3 to 1
0 received ping_pong_count 4 from 1
0 sent and incremented ping_pong_count 5 to 1
0 received ping_pong_count 6 from 1
0 sent and incremented ping_pong_count 7 to 1
0 received ping_pong_count 8 from 1
0 sent and incremented ping_pong_count 9 to 1
0 received ping_pong_count 10 from 1
1 received ping_pong_count 1 from 0
1 sent and incremented ping_pong_count 2 to 0
1 received ping_pong_count 3 from 0
1 sent and incremented ping_pong_count 4 to 0
1 received ping_pong_count 5 from 0
1 sent and incremented ping_pong_count 6 to 0
1 received ping_pong_count 7 from 0
1 sent and incremented ping_pong_count 8 to 0
1 received ping_pong_count 9 from 0
1 sent and incremented ping_pong_count 10 to 0
```

出力はプロセススケジューリングのため異なる可能性があります。ただし、出力が例と違ってもプロセス0とプロセス1は、交互に`ping_pong_count`ことが読み取れるでしょう。

## Ring Program
`MPI_Send`と`MPI_Recv`を使って2つ以上のプロセスで通信することを考えましょう。この例では、値をすべてのプロセス上をリング状に流れていきます。[ring.c]({{ site.github.code }}/tutorials/mpi-send-and-receive/code/ring.c)を見てください。コードのメイン部分は次のようになります。

```cpp
int token;
if (world_rank != 0) {
    MPI_Recv(&token, 1, MPI_INT, world_rank - 1, 0,
             MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    printf("Process %d received token %d from process %d\n",
           world_rank, token, world_rank - 1);
} else {
    // ランク0はトークンを-1で初期化する
    token = -1;
}
MPI_Send(&token, 1, MPI_INT, (world_rank + 1) % world_size,
         0, MPI_COMM_WORLD);

// ここでプロセス0はリンク上の最後の受信処理を行います
if (world_rank == 0) {
    MPI_Recv(&token, 1, MPI_INT, world_size - 1, 0,
             MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    printf("Process %d received token %d from process %d\n",
           world_rank, token, world_size - 1);
}
```

リングプログラムはプロセス0で値を初期化し、その値を次のプロセスに渡します。プロセス0が最後のプロセスから値を受信するとプログラムは終了します。デッドロックが発生しないように特別なロジックになっています。プロセス0は(最後のプロセスから)値をブロッキング受信する前に、最初の送信を完了していることが保証されます。他のすべてのプロセスはリングに沿って値を渡します。`MPI_Recv`(隣接するランクの低いプロセスから受信)で値を受け取り、次に`MPI_Send`(隣接するランクの高いプロセスに値を送信) を呼び出します。メッセージが送信されるまでブロックします。このため、printfは値が渡される順序で実行されます。5つのプロセスを使用する場合、出力は次のようになりました。

```
>>> ./run.py ring
Process 1 received token -1 from process 0
Process 2 received token -1 from process 1
Process 3 received token -1 from process 2
Process 4 received token -1 from process 3
Process 0 received token -1 from process 4
```

プロセス0は最初にプロセス1に-1を送信します。この値（トークン）はプロセス0に戻るまでリング状に渡されています。

## Up next

`MPI_Send`と`MPI_Recv`の基本を理解したのでこれらの関数についてもう少し詳しく見ていきます。次のレッスンは、[how to probe and dynamically receive messages]({{ site.baseurl }}/tutorials/dynamic-receiving-with-mpi-probe-and-mpi-status/)です。[MPI tutorials]({{ site.baseurl }}/tutorials/)をMPIレッスンの完全なリファレンスとして活用してください。

困っていますか?混乱していますか?お気軽に下記にコメントを残してください。私や他の読者がお役に立てるかもしれません。
