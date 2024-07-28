---
layout: post
title: MPI Scatter, Gather, Allgather
author: Wes Kendall
categories: Beginner MPI
tags: MPI_Gather, MPI_Allgather, MPI_Scatter
redirect_from: '/mpi-scatter-gather-and-allgather/'
---

[前のレッスン]({{ site.baseurl }}/tutorials/mpi-broadcast-and-collective-communication/)では集団通信の基本である集団コミュニケーションルーチン`MPI_Bcast`を説明しました。今回のレッスンではさらに集団通信を学んでいきましょう。非常に重要なルーチン`MPI_Scatter`と`MPI_Gather`そして`MPI_Allgather`をみていきましょう。

> **Note** - 全てのコードは[GitHub]({{ site.github.repo }})にあります。このレッスンのコードは[tutorials/mpi-scatter-gather-and-allgather/code]({{ site.github.code }}/tutorials/mpi-scatter-gather-and-allgather/code)をみてください。

## MPI_Scatterとは - An introduction to MPI_Scatter
`MPI_Scatter`は`MPI_Bcast`に似た集団通信ルーチンです。ルートプロセスはコミュニケータ内のすべてのプロセスにデータを送信します。ただし、`MPI_Bcast`はルートの持つ同じデータと全く同じデータをすべてのプロセスに送信していたのに対し、`MPI_Scatter`はルートの持つ配列のチャンクをそれぞれ異なるプロセスに送信します。

![MPI_Bcast vs MPI_Scatter](../broadcastvsscatter.png)

`MPI_Bcast`はルートプロセス(赤)のデータを全てのプロセスにコピーしました。一方、`MPI_Scatter`は配布したい配列をプロセスランクの順に要素を分けて配布します。最初の要素 (赤) はプロセス0に配る, 2番目の要素 (緑) はプロセス1に配る、といったようにです。ルートプロセス(プロセス0)はデータの配列全体を持っており、`MPI_Scatter`は適切な要素をプロセスの受信バッファにコピーします。`MPI_Scatter`の関数定義は以下の通りです。

```cpp
MPI_Scatter(
    void* send_data,
    int send_count,
    MPI_Datatype send_datatype,
    void* recv_data,
    int recv_count,
    MPI_Datatype recv_datatype,
    int root,
    MPI_Comm communicator)
```

最初の引数`send_data`はルートプロセスにおける配布したい配列のポインタです。2番目と3番目の引数`send_count`と`send_datatype`は、各プロセスに対してどんなMPI Datatypeの要素を何個送信するかを指定します。`send_count` が1で`send_datatype`が MPI_INTならば、プロセス0は配列の最初の整数を受け取り、プロセス 1は2番目の整数を受け取ります。`send_count`が2なら、プロセス0は1,2番目の整数を担当し、プロセス1は3,4番目の整数を担当します。実際には`send_count`は配列の要素数をプロセス数で割ったものに等しいでしょう。要素数がプロセス数で割り切れない？ご心配なく。それは後で見ていきましょう。

後続の引数はほぼ同様です。`recv_data`は`recv_datatype`のデータ型を持つ`recv_count`個の要素を保持できるデータのバッファです。`root`と`communicator`はルートプロセスのランクと、コミュニケータです。

## MPI_Gatherとは - An introduction to MPI_Gather
`MPI_Gather`は`MPI_Scatter`の逆の動きをする関数です。要素を1つのプロセスから多数のプロセスに分散させるのではなく、`MPI_Gather`多数のプロセスから要素を取得して1つのプロセスに集めます。このルーチンは並列ソートや並列検索などの多くの並列アルゴリズムで使われます。

![MPI_Gather](../gather.png)

`MPI_Scatter` の逆と考えれば良いです。 `MPI_Gather` は各プロセスから要素を受け取ってルートプロセスに集めます。そして要素を受け取ったプロセスの順位で並べます。`MPI_Gather` の関数宣言は `MPI_Scatter` と同じです。

```cpp
MPI_Gather(
    void* send_data,
    int send_count,
    MPI_Datatype send_datatype,
    void* recv_data,
    int recv_count,
    MPI_Datatype recv_datatype,
    int root,
    MPI_Comm communicator)
```

`MPI_Gather` では、ルートプロセスだけが有効な受信バッファを持てば良いので、ルートプロセス以外は`recv_data` に `NULL` を渡して良いです。また、*recv_count* パラメータは、全プロセスからのカウントの合計ではなく、*プロセスごとに*受信した要素のカウントであることを忘れないでください。これはMPIを触ったばかりのプログラマを混乱させます。

## 数値の平均 - Computing average of numbers with MPI_Scatter and MPI_Gather
レッスンの[レポジトリ]({{ site.github.code }}/tutorials/mpi-scatter-gather-and-allgather/code)では、配列内の数値の平均を計算するサンプルプログラム([avg.c]({{ site.github.code }}/tutorials/mpi-scatter-gather-and-allgather/code/avg.c))を提供しています。MPIを使用してプロセスを分割し、プロセスごとに計算を実行し、それらの小さな結果を集約して最終的な答えを出すという簡単なプログラムです。まずは動作を確認しましょう。

1. ルートプロセス(プロセス0) でランダムな内容のの配列を生成します。
2. 配列をすべてのプロセスを分散します。各プロセスに同じ数のデータを割り当てます。
3. 各プロセスは割り当てられた数値の平均を計算します。
4. 各プロセスは結果をルートプロセスに収集します。ルートプロセスはこれらの数値の平均を計算して最終的な平均を取得します。

```cpp
if (world_rank == 0) {
  rand_nums = create_rand_nums(elements_per_proc * world_size);
}

// 乱数の配列を作ります。長さは固定
float *sub_rand_nums = malloc(sizeof(float) * elements_per_proc);

// 配列を全てのプロセスにScatterする
MPI_Scatter(rand_nums, elements_per_proc, MPI_FLOAT, sub_rand_nums,
            elements_per_proc, MPI_FLOAT, 0, MPI_COMM_WORLD);

// 各プロセスは自分に割り当てられた配列の平均を計算する
float sub_avg = compute_avg(sub_rand_nums, elements_per_proc);
// ルートプロセスは各プロセスの値を集める
float *sub_avgs = NULL;
if (world_rank == 0) {
  sub_avgs = malloc(sizeof(float) * world_size);
}
MPI_Gather(&sub_avg, 1, MPI_FLOAT, sub_avgs, 1, MPI_FLOAT, 0,
           MPI_COMM_WORLD);

// ルートプロセスは集めた平均を計算する
if (world_rank == 0) {
  float avg = compute_avg(sub_avgs, world_size);
}
```

最初にルートプロセスは乱数の配列を作成します。 `MPI_Scatter`によって各プロセスには`elements_per_proc`のデータが配布されます。各プロセスは自分に割り当てられた配列の平均を計算し、ルートプロセスはそれぞれの平均を収集します。ルートプロセスでの全体の平均計算処理は、非常に小さい配列に基づいて計算すれば良いです。

[レポジトリ]({{ site.github.code }})のチュートリアルディレクトリから avg プログラムを実行すると、出力は次のようになります。数字はランダムなので、この出力結果と異なるでしょう。

```
>>> cd tutorials
>>> ./run.py avg
/home/kendall/bin/mpirun -n 4 ./avg 100
Avg of all elements is 0.478699
Avg computed across original data is 0.478699
```

## MPI_Allgatherと平均プログラムの修正 - MPI_Allgather and modification of average program
さて、多くのプロセスが1つのプロセスに対してある送受信を行う、つまり多対1または1対多の通信パターンを実行する2つのMPIルーチンをみてきました。ところで、複数のプロセルから多くのプロセスに要素を送信できることは便利です。そう、多対多の集団通信パターンです。`MPI_Allgather`というルーチンを説明します。

`MPI_Allgather`は全プロセスに分散している要素を全プロセスに配布するルーチンです。つまり、`MPI_Allgather`は`MPI_Gather`を行った後に`MPI_Bcast`しているような動作をします。下図は`MPI_Allgather`を呼び出したデータの動きです。

![MPI_Allgather](../allgather.png)

`MPI_Gather`と同じように、各プロセスに対して各プロセスが持っていた要素をランク順に集める。それだけです。`MPI_Allgather`の引数は`MPI_Gather`とほぼ同じですが、`MPI_Allgather`にはルートプロセスの引数は存在しません。

```cpp
MPI_Allgather(
    void* send_data,
    int send_count,
    MPI_Datatype send_datatype,
    void* recv_data,
    int recv_count,
    MPI_Datatype recv_datatype,
    MPI_Comm communicator)
```

`MPI_Allgather`を使うように平均計算コードを修正しました。このレッスンのコードからall_avg.c]({{ site.github.code }}/tutorials/mpi-scatter-gather-and-allgather/code/all_avg.c)のソースを見ることができます。コードの主な違いは次のとおりです。

```cpp
// 全てのプロセスに対して各プロセスで計算した平均を集める
float *sub_avgs = (float *)malloc(sizeof(float) * world_size);
MPI_Allgather(&sub_avg, 1, MPI_FLOAT, sub_avgs, 1, MPI_FLOAT,
              MPI_COMM_WORLD);

// 各プロセスは全ての平均を求める
float avg = compute_avg(sub_avgs, world_size);
```

各プロセスの平均を`MPI_Allgather`を使って全プロセスに集めます。そして、すべてのプロセスでその平均を計算して出力します。

```
>>> ./run.py all_avg
/home/kendall/bin/mpirun -n 4 ./all_avg 100
Avg of all elements from proc 1 is 0.479736
Avg of all elements from proc 3 is 0.479736
Avg of all elements from proc 0 is 0.479736
Avg of all elements from proc 2 is 0.479736
```

このようにall_avg.c では`MPI_Allgather`で全てのプロセスに各プロセスの平均を集めて表示します。

## Up next
次のレッスンでは `MPI_Gather`と`MPI_Scatter`を利用して[並列なランク計算]({{ site.baseurl }}/tutorials/performing-parallel-rank-with-mpi/)を説明します。

その他のレッスンは[MPI tutorials]({{ site.baseurl }}/tutorials/) にあります。