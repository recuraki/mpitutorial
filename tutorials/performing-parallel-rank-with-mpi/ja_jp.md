---
layout: post
title: 並列ランク問題 - Performing Parallel Rank with MPI
author: Wes Kendall
categories: Beginner MPI
tags: MPI_Type_size
translations: zh_cn
redirect_from: '/performing-parallel-rank-with-mpi/'
---

[前のレッスン]({{ site.baseurl }}/tutorials/mpi-scatter-gather-and-allgather/)では
 `MPI_Scatter`, `MPI_Gather`,  `MPI_Allgather`を紹介しました。このレッスンでは、MPIの便利なルーチンである並列ランク(parallel rank)を実装して集団通信への理解を深めていきましょう。

> **Note** - 全てのコードは[GitHub]({{ site.github.repo }})にあります。このレッスンのコードは [tutorials/performing-parallel-rank-with-mpi/code]({{ site.github.code }}/tutorials/performing-parallel-rank-with-mpi/code)にあります。

## 問題概要 - Parallel rank - problem overview
すべてのプロセスがローカル メモリにただ1つの数値を持っています。各プロセスは自分の持っている数値が全てのプロセスの中で何番目に小さい値を持っているのか知りたいです。このシナリオは、ユーザーがMPIクラスター内のプロセッサのベンチマークを行っており自分のプロセッサが他のプロセッサと比較してどのくらい高速であるかを知りたいケースなどに利用されます。他にもタスクのスケジュール設定などに使用できるでしょう。数値がプロセス全体に分散していると全ての数値の中での順序を見つけるのは難しいように思えます。この問題を並列ランク問題と呼び、このレッスンで解決しましょう。

具体例を図で示します。

![Parallel Rank](../parallel_rank_1.png)

4つのプロセスがあり、0から3のラベルでそれぞれを区別します。順番に5、2、7、4の4つの数字を持っているとしましょう。並列ランクアルゴリズムを実行すると全体での順番がわかるのでそれぞれは2,0,3,1の4つの数字を持つことになります。（訳注:0-indexedです）

## 並列ランクAPI定義
これを実現するためにがなにが必要かを考えましょう。すべての数値を集めること、その数値の順番を返す必要があります。さらに使用されているコミュニケータやランク付けされる数値のデータ型なども必要になります。ランク関数のプロトタイプを次のようにしましょう。

```cpp
TMPI_Rank(
    void *send_data,
    void *recv_data,
    MPI_Datatype datatype,
    MPI_Comm comm)
```

`TMPI_Rank`は、データ型の数値を1つ含む`send_data`バッファを受け取ります。`recv_data`は、`send_data`の各プロセスの順位を受け取る変数です。`comm`変数はプロセスのコミュニケータです。

> **Note** - 
> MPI標準では、ユーザー関数と MPI関数が混同しないように、MPI関数には`MPI_<something>`という名前づけがされています。したがって、このレッスンではさらに`T`を追加しています。

## 並列ランク問題の解き方　- Solving the parallel rank problem
API定義ができたので、どのように解くかを考えましょう。並列ランク問題を解決するためには全てのプロセスから集めた数値のソートが必要です。このソートは各数字の順番を知るために必要で、いくつかのアプローチで実現できます。最も簡単な方法は各プロセスの数字を1つのプロセスに集め、そのプロセスでソートすれば良いでしょう。サンプル コード ([tmpi_rank.c]({{ site.github.code }}/tutorials/performing-parallel-rank-with-mpi/code/tmpi_rank.c)) では`gather_numbers_to_root`関数はすべての数字をルート プロセスに集める役割を担います。

```cpp
// ルートプロセスに数値を集めます。
// ルートプロセスは結果の（ランクの）配列を返します。他のプロセスにはNULLを返します。
void *gather_numbers_to_root(void *number, MPI_Datatype datatype,
                             MPI_Comm comm) {
  int comm_rank, comm_size;
  MPI_Comm_rank(comm, &comm_rank);
  MPI_Comm_size(comm, &comm_size);

  // ルートプロセスでは必要なバッファを用意します。データ型のサイズを配慮します  
  int datatype_size;
  MPI_Type_size(datatype, &datatype_size);
  void *gathered_numbers;
  if (comm_rank == 0) {
    gathered_numbers = malloc(datatype_size * comm_size);
  }

  // 全ての数値をルートプロセスに集めます
  MPI_Gather(number, 1, datatype, gathered_numbers, 1,
             datatype, 0, comm);

  return gathered_numbers;
}
```

`gather_numbers_to_root`関数はルートに送りたい数値（`send_data`変数）と数値の`datatype`および`comm`コミュニケータを引数で受け取ります。ルート・プロセスは`comm_size`の数値を受け取ればいいとわかるので`datatype_size * comm_size`の長さの配列をメモリ確保します。`datatype_size`変数は、このチュートリアルで初めて使う`MPI_Type_size`を使用して収集されます。私たちの今回のプログラムは`MPI_INT`と`MPI_FLOAT`のみを使用しますが、この関数を使うと様々なサイズのデータ型をサポートするように拡張することができます。これで数値が集まったので`MPI_Gather`でルートプロセスに数を集めます。次はルートプロセスで数をソートし、ランクを決定します。

## ソートとその所有権の維持 - Sorting numbers and maintaining ownership
ソートの処理自身は難しくありません。C標準ライブラリには`qsort` などの一般的なソートアルゴリズムが用意されています。ただし、並列ランク問題ソートでは数字を送信したプロセスのランクをルートプロセスのソートの間で維持する必要あることに注意します。ルートプロセス対して数字だけでは数字の順位をどのプロセスに返せばいいかわからないのです。

数字とそれに対応する送信元のプロセスのランク(つまり所有権)を保持する構造体を定義しましょう。

```cpp
// 値とコミュニケータランクを保持する
// この構造体はソートをされてもランクの情報を失わない
typedef struct {
  int comm_rank;
  union {
    float f;
    int i;
  } number;
} CommRankNumber;
```

構造体`CommRankNumber`はソートする数値を保持し、対応するプロセスのコミュニケータランクを保持します。ここで数値はfloatまたはintの可能性があるので、unionで両方を保持できるようにしていることに注意してください。次にこの構造体をソートする`get_ranks`関数を考えていきましょう。

```cpp
// この関数はルートプロセスに値を集めて、その順位を元のプロセスに返します
// この関数はルートでのみ実行されます。
int *get_ranks(void *gathered_numbers, int gathered_number_count,
               MPI_Datatype datatype) {
  int datatype_size;
  MPI_Type_size(datatype, &datatype_size);

  // 集めた値をCommRankNumbersに保持します。
  // これによりソートしてもコミュニケータランクを失いません。
  CommRankNumber *comm_rank_numbers = malloc(
    gathered_number_count * sizeof(CommRankNumber));
  int i;
  for (i = 0; i < gathered_number_count; i++) {
    comm_rank_numbers[i].comm_rank = i;
    memcpy(&(comm_rank_numbers[i].number),
           gathered_numbers + (i * datatype_size),
           datatype_size);
  }

  // ランクの情報を持ったまま値をソートする
  if (datatype == MPI_FLOAT) {
    qsort(comm_rank_numbers, gathered_number_count,
          sizeof(CommRankNumber), &compare_float_comm_rank_number);
  } else {
    qsort(comm_rank_numbers, gathered_number_count,
          sizeof(CommRankNumber), &compare_int_comm_rank_number);
  }

  // ソートが完了したので値を戻すための結果の配列を用意する。iにはプロセスiの順位が含まれる。
  int *ranks = (int *)malloc(sizeof(int) * gathered_number_count);
  for (i = 0; i < gathered_number_count; i++) {
    ranks[comm_rank_numbers[i].comm_rank] = i;
  }

  // comm_rank_numbersを解放する
  free(comm_rank_numbers);
  return ranks;
}
```
`get_ranks`は`CommRankNumber`構造体の配列を作成して、各プロセスのランクを設定します。`MPI_FLOAT`と`MPI_INT`である場合で違ったsort関数の呼び出しを行います。 ([tmpi_rank.c]({{ site.github.code }}/tutorials/performing-parallel-rank-with-mpi/code/tmpi_rank.c)を参照してください)。

ソートしたら順番を格納した配列を作成して各々のプロセスに正しく戻せるようにする必要があります。ソートされた`comm_rank_numbers`を順に見ていき、戻すための配列`ranks`を作ります。

## 全てを一緒にする - Putting it all together
さて、２つの重要なルーチンが実装できたのでこれらをまとめて `TMPI_Rank` を作りましょう。ルートプロセスに各プロセスの数字を集めてから、計算した順位を各プロセスに戻します。このコードを以下に示します。
```cpp
// datatype型のsend_dataの順位を取得する。
// ランクはrecv_dataで戻ってくるdatatype型である。
int TMPI_Rank(void *send_data, void *recv_data, MPI_Datatype datatype,
             MPI_Comm comm) {
  // datatypeがMPI_INTかMPI_FLOATであるかを確認します
  if (datatype != MPI_INT && datatype != MPI_FLOAT) {
    return MPI_ERR_TYPE;
  }

  int comm_size, comm_rank;
  MPI_Comm_size(comm, &comm_size);
  MPI_Comm_rank(comm, &comm_rank);

  // 順位を計算するために値をルートプロセスに集めます。
  // そして、ソートして結果を求め、Scatterで順位情報を戻します。
  // ルートプロセスにはプロセス0を使います
  void *gathered_numbers = gather_numbers_to_root(send_data, datatype,
                                                  comm);

  // 各プロセスの順位を計算する
  int *ranks = NULL;
  if (comm_rank == 0) {
    ranks = get_ranks(gathered_numbers, comm_size, datatype);
  }

  // Scatterする
  MPI_Scatter(ranks, 1, MPI_INT, recv_data, 1, MPI_INT, 0, comm);

  // clean up
  if (comm_rank == 0) {
    free(gathered_numbers);
    free(ranks);
  }
}
```

TMPI_Rank関数は`gather_numbers_to_root`と`get_ranks`の２つの関数を使用して各の順位を計算します。この関数は`MPI_Scatter`によって各プロセスに順位を戻します。

全体のデータフローを図解します。

![Parallel Rank](../parallel_rank_2.png)

もし、これに関する質問があれば以下を参照して質問してください。

## 実行 - Running our parallel rank algorithm
このアルゴリズムのサンプルプログラムがあります。[レッスンコード]({{ site.github.code }}/tutorials/performing-parallel-rank-with-mpi/code)の[random_rank.c file]({{ site.github.code }}/tutorials/performing-parallel-rank-with-mpi/code/random_rank.c)で確認できます。

このサンプルプログラムは各プロセスで乱数を生成した後、各順位を取得するために`TMPI_Rank`の呼び出しします。[レポジトリ]({{ site.github.code }})のチュートリアルディレクトリから random_rank プログラムを実行すると、出力は次のようになります。

```
>>> cd tutorials
>>> ./run.py random_rank
mpirun -n 4  ./random_rank 100
Rank for 0.242578 on process 0 - 0
Rank for 0.894732 on process 1 - 3
Rank for 0.789463 on process 2 - 2
Rank for 0.684195 on process 3 - 1
```

## Up next
次はさらに高度な集団通信について説明します。次のレッスンは、[using MPI_Reduce and MPI_Allreduce to perform number reduction]({{ site.baseurl }}/tutorials/mpi-reduce-and-allreduce/)です。

[MPI tutorials]({{ site.baseurl }}/tutorials/) に全てのレッスンの目次があります。