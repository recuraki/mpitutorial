---
layout: post
title: MPI Reduce と Allreduce - MPI Reduce and Allreduce
author: Wes Kendall
categories: Beginner MPI
tags: MPI_Allreduce, MPI_Reduce
redirect_from: '/mpi-reduce-and-allreduce/'
---

前回のレッスンでは`MPI_Scatter`と`MPI_Gather`を使用してMPIで並列に順位計算を実行するアプリケーションを説明しました。このレッスンでは、`MPI_Reduce`と`MPI_Allreduce`を説明して集団通信への理解をさらに深めます。

> **Note** :  このサイトのコードはすべて [GitHub]({{ site.github.repo }})にあります。このチュートリアルのコードは [tutorials/mpi-reduce-and-allreduce/code]({{ site.github.code }}/tutorials/mpi-reduce-and-allreduce/code)にあります。

## Reduce - An introduction to reduce
*reduce*は関数型プログラミングで使われる基本的な概念です。データのreduceはある関数を使用して数値のセットを小さな数値のセットにします。たとえば、`[1, 2, 3, 4, 5]`というリストがあるとしましょう。これをsum 関数でreduceすると`sum([1, 2, 3, 4, 5]) = 15`が計算されることになります。掛け算でreduceすると`multiply([1, 2, 3, 4, 5]) = 120`となります。

分散されている数値の集合にreduceするのは非常に面倒です。さらに集合の順序を考慮しながらのreduceを効率的にプログラムするのも面倒です。MPIには `MPI_Reduce` 関数があり、プログラマが並列アプリケーションで行う必要のある一般的なreduceのほとんどを扱うことができます。

## MPI_Reduce
`MPI_Reduce` は`MPI_Gather` と同様、各プロセスで入力要素の配列を受け取って出力要素の配列をルートプロセスに返します。出力要素にはreduceの結果が含まれます。`MPI_Reduce`の関数定義は次のとおりです。

```cpp
MPI_Reduce(
    void* send_data,
    void* recv_data,
    int count,
    MPI_Datatype datatype,
    MPI_Op op,
    int root,
    MPI_Comm communicator)
```

`send_data`引数は、各プロセスがreduceしたい`datatype`型の要素の配列のポインタです。`recv_data`はルートプロセスだけの引数でreduceの結果が格納されて`sizeof(datatype) * count`のサイズを持もちます。`op` パラメータには、データに適用したい処理を指定します。MPIには一般的なreduce演算が用意されています。自作のreduceの関数を作成することもできますがこのレッスンでは扱いません。以下にMPIでサポートされている操作を紹介します。

* `MPI_MAX` - 最大値
* `MPI_MIN` - 最小値
* `MPI_SUM` - 合計
* `MPI_PROD` - 乗算
* `MPI_LAND` - 各要素の論理的なAND演算
* `MPI_LOR` - 各要素の論理的なOR演算
* `MPI_BAND` - 各要素のビットAND演算
* `MPI_BOR` - 各要素のビットOR演算
* `MPI_MAXLOC` - 最大値とそれを持つプロセスのランク
* `MPI_MINLOC` - 最小値とそれを持つプロセスのランク

`MPI_Reduce`のイメージを示します。

![MPI_Reduce](../mpi_reduce_1.png)

この例では各プロセスは整数を１つ持ちます。`MPI_Reduce`はプロセス0で呼び出され`MPI_SUM`をreduce演算として使用します。合計された結果がルートプロセスに格納されます。

次は各プロセスが整数を複数持った時のことを考えます。

![MPI_Reduce](../mpi_reduce_2.png)

この例では各プロセスは2つの整数を持ちます。ルートのプロセスではこの2つの数値を一緒にして合計を求めるのではなく、結果を入れる配列のi番目に対して、各プロセスのi番目の数の合計を集約することに注意してください。
つまり、すべての配列の要素を合計して1つの要素にするのではなく、各配列の要素を合計してプロセス0の結果配列のi番目の要素に格納します

`MPI_Reduce` がどのように見えるか理解できたので次のトピックに進みましょう。

## MPI_Reduceを使った平均の計算 - Computing average of numbers with MPI_Reduce
一つ前のレッスンでは`MPI_Scatter`と`MPI_Gather`で数値の平均を求めましたが、`MPI_Reduce`を使うことでより簡単に実装することができます。次に[reduce_avg.c]({{ site.github.code }}/tutorials/mpi-reduce-and-allreduce/code/reduce_avg.c)を示します。

```cpp
float *rand_nums = NULL;
rand_nums = create_rand_nums(num_elements_per_proc);

// 各のプロセスのsumを求める
float local_sum = 0;
int i;
for (i = 0; i < num_elements_per_proc; i++) {
  local_sum += rand_nums[i];
}

// 各のローカル変数の平均を合計と平均を表示する
printf("Local sum for process %d - %f, avg = %f\n",
       world_rank, local_sum, local_sum / num_elements_per_proc);

// 各プロセスで求めたsumをグローバルsumとしてreduceする
float global_sum;
MPI_Reduce(&local_sum, &global_sum, 1, MPI_FLOAT, MPI_SUM, 0,
           MPI_COMM_WORLD);

// 結果の表示
if (world_rank == 0) {
  printf("Total sum = %f, avg = %f\n", global_sum,
         global_sum / (world_size * num_elements_per_proc));
}
```

このコードでは各プロセスが(floatの)乱数を作成し、`local_sum`を計算します。次に、`local_sum`を`MPI_SUM`によってルートプロセスにreduceします。そして全部のプロセスの平均を`global_sum / (world_size * num_elements_per_proc)`で求めます。reduce_avgプログラムを[レポジトリ]({{ site.github.code }})の*tutorials*ディレクトリから実行しましょう。

```
>>> cd tutorials
>>> ./run.py reduce_avg
mpirun -n 4  ./reduce_avg 100
Local sum for process 0 - 51.385098, avg = 0.513851
Local sum for process 1 - 51.842468, avg = 0.518425
Local sum for process 2 - 49.684948, avg = 0.496849
Local sum for process 3 - 47.527420, avg = 0.475274
Total sum = 200.439941, avg = 0.501100
```

`MPI_Reduce` について学んだので次は`MPI_Allreduce`です！

## MPI_Allreduce
ほとんどの並列アプリケーションでは、ルートプロセスだけでなく全プロセスがreduceの結果にアクセスする必要があるでしょう。`MPI_Allgather`と`MPI_Gather`の関係のように`MPI_Allreduce`は値をreduceした結果を全プロセスに配布します。関数定義は以下の通りです。

```cpp
MPI_Allreduce(
    void* send_data,
    void* recv_data,
    int count,
    MPI_Datatype datatype,
    MPI_Op op,
    MPI_Comm communicator)
```

`MPI_Allreduce` は `MPI_Reduce` 同じ引数に見えますがルートプロセス IDは不要です (結果が全プロセスに配布されるため)。以下に `MPI_Allreduce` の通信パターンを図示します。

![MPI_Allreduce](../mpi_allreduce_1.png)

`MPI_Allreduce`は`MPI_Reduce`した後に`MPI_Bcast`しているのと同じです。

## MPI_Allreduce による標準偏差の計算 - Computing standard deviation with MPI_Allreduce
計算問題の多くは、問題を解くためにreduceを何度も呼び出します。ここで例に示す例は分散している数の標準偏差を求める計算です。標準偏差とは平均値からの数値のばらつきを表す尺度で、標準偏差が低ければ低いほど数値がより近くに集まっていることを意味し、標準偏差が高ければその逆です。

標準偏差を求めるには、まず全ての数値の平均が必要です。次に平均からの差の2乗和を計算します。この和の平均の平方根が最終結果となります。ということは2つの和が必要ということで、2つのreduceが必要になるわけです。レッスンコードの[reduce_stddev.c]({{ site.github.code }}/tutorials/mpi-reduce-and-allreduce/code/reduce_stddev.c)でこれを示します。

```cpp
rand_nums = create_rand_nums(num_elements_per_proc);

// プロセス内の合計を求める
float local_sum = 0;
int i;
for (i = 0; i < num_elements_per_proc; i++) {
  local_sum += rand_nums[i];
}

// 平均を求めるためのグローバル和を計算する
float global_sum;
MPI_Allreduce(&local_sum, &global_sum, 1, MPI_FLOAT, MPI_SUM,
              MPI_COMM_WORLD);
float mean = global_sum / (num_elements_per_proc * world_size);

// 平均からの差の2乗和をローカルで合計する。
float local_sq_diff = 0;
for (i = 0; i < num_elements_per_proc; i++) {
  local_sq_diff += (rand_nums[i] - mean) * (rand_nums[i] - mean);
}

// 2乗和をルートプロセスに集めて合計を取る
float global_sq_diff;
MPI_Reduce(&local_sq_diff, &global_sq_diff, 1, MPI_FLOAT, MPI_SUM, 0,
           MPI_COMM_WORLD);

//  標準偏差stddevは二乗差の平均の平方根。表示する。
if (world_rank == 0) {
  float stddev = sqrt(global_sq_diff /
                      (num_elements_per_proc * world_size));
  printf("Mean - %f, Standard deviation = %f\n", mean, stddev);
}
```

まず、各プロセスは要素の`local_sum`を計算します。そして`MPI_Allreduce` を使用してそれらを合計します。各プロセスは全ての平均がわかるようになったので、`local_sq_diff`を計算するための`mean` を計算します。そしてローカルで平均からの差の2乗和を計算すると、ルートプロセスは`MPI_Reduce`を使用して`global_sq_diff`を計算します。最後にルートプロセスはグローバル平方差の平均の平方根を取ることによって標準偏差を計算します。

サンプルプログラムの動作結果を示します。

```
>>> ./run.py reduce_stddev
mpirun -n 4  ./reduce_stddev 100
Mean - 0.501100, Standard deviation = 0.301126
```

## 次は？
`MPI_Bcast`, `MPI_Scatter`, `MPI_Gather`, `MPI_Reduce` といった一般的な集団計算の使い方に慣れたので、これらを利用した洗練された並列アプリケーションを構築してみましょう。次回のレッスンは[MPI groups and communicators]({{ site.baseurl }}/tutorials/introduction-to-groups-and-communicators/)です。

全てのレッスンは、[MPIチュートリアルセクション]({{ site.baseurl }}/tutorials/)を参照してください。
