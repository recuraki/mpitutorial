---
layout: post
title: グループとコミュニケータ - Introduction to Groups and Communicators
author: Wesley Bland
categories: Advanced MPI
tags: MPI_Group, MPI_Comm
redirect_from: '/introduction-to-groups-and-communicators/'
---

これまでのレッスンでは`MPI_COMM_WORLD`を使用してきました。単純なプログラムの場合はプロセスの数は多くないので、一度に1つのプロセスと話すか、一度に全てのプロセスと話すかでしょうから問題はありませんでした。しかし、プログラムの規模が大きくなり始めると、限定的なプロセスとしか通信をしたくないケースが出てきます。このレッスンでは、元となるプロセスグループの集合のプロセスとだけ集団通信するために、新しいコミュニケータを作成する方法を紹介します。

> **Note** - チュートリアルのコードは[GitHub]({{ site.github.repo }})にあります。. このレッスンのコードは[tutorials/introduction-to-groups-and-communicators/code]({{ site.github.code }}/tutorials/introduction-to-groups-and-communicators/code)を参照してください。

## コミュニケータとは？ - Overview of communicators
集団通信のレッスンで見てきたように、MPIの集団通信はコミュニケータ内の全プロセスと一度に通信できます。`MPI_Scatter`は他のプロセスにデータを分配したり、`MPI_Reduce`ではreduceを実行できます。しかし、これまではデフォルトの `MPI_COMM_WORLD` しか使ってきませんでした。

単純なアプリケーションでは`MPI_COMM_WORLD`を使うことも珍しくないのですが、複雑なユースケースでは多くのコミュニケータがあった方が便利でしょう。例えば、グリッド内だけのプロセスのサブセットに対して計算を行いたい場合です。例として、各行の全プロセスの合計値を求めるような場合です。新しいコミュニケータを作成するための関数宣言を見てみましょう。


```cpp
MPI_Comm_split(
	MPI_Comm comm,
	int color,
	int key,
	MPI_Comm* newcomm)
```

`MPI_Comm_split`の名のとおり`color`と`key`に基づいて、あるコミュニケータをサブコミュニケータ群に"分割"して新しいコミュニケータを生成します。元のコミュニケータはなくならず、各プロセスに新しいコミュニケータが作成されることに注意してください。最初の引数`comm`は分割元のコミュニケータです。`MPI_COMM_WORLD`でもよいですし、他のコミュニケータでもかまいません。2番目の引数`color`は各プロセスがどの新しいコミュニケータに属するかを決定します。`color`に同じ値を渡したプロセスはすべて同じコミュニケータに割り当てられます。もし`color`が`MPI_UNDEFINED`であれば、そのプロセスは新しいコミュニケータには含まれません。3番目の引数`key`は、新しいコミュニケータ内の順序(ランク)を決定します。`key`の値が最も小さいプロセスがランク0になり、次に小さいプロセスがランク1になります。同順位の場合は、元のコミュニケーター内の順位が低いプロセスが最初になります。最後の引数`newcomm`は新しいコミュニケータです。

## 複数のコミュニケータを使用する例 - Example of using multiple communicators

単純な例として、1つのグローバルコミュニケータを複数のコミュニケータに分割してみます。元のコミュニケータには16個のプロセス存在しますが、これを4x4のグリッドに論理的にレイアウトし、グリッドを行ごとに分割したいというシナリオを考えます。それぞれの行にcolorをつけます。下の画像では左側の同じ色のプロセスグループが右側のそれぞれのコミュニケータに入る様子を示します。

![MPI_Comm_split example](../comm_split.png)

これを実現するコードです。

```cpp
// 元のコミュニケータのランクとサイズを得る
int world_rank, world_size;
MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
MPI_Comm_size(MPI_COMM_WORLD, &world_size);

int color = world_rank / 4; // 行をcolorとして使います

// colorと元のランクを利用して新しいコミュニケータを作成します
MPI_Comm row_comm;
MPI_Comm_split(MPI_COMM_WORLD, color, world_rank, &row_comm);

int row_rank, row_size;
MPI_Comm_rank(row_comm, &row_rank);
MPI_Comm_size(row_comm, &row_size);

printf("WORLD RANK/SIZE: %d/%d \t ROW RANK/SIZE: %d/%d\n",
	world_rank, world_size, row_rank, row_size);

MPI_Comm_free(&row_comm);
```

まず、オリジナルのコミュニケータ`MPI_COMM_WORLD`の中での自分のランクとこのコミュニケータのサイズを得ます。次はローカルプロセスの "色(color) "を決定する大切な操作です。色によって、分割後のプロセスがどのコミュニケータに属するかが決定します。そして分割を実施します。ここで注目して欲しいのは分割操作のキーとして元のランク（`world_rank`）を使っていることです。新しいコミュニケーター内のすべてのプロセスは元のコミュニケーターと同じ順序としたいので、元のランクの値を使用します。分割後に新しいコミュニケータのサイズと自分のランクを表示します。

```
WORLD RANK/SIZE: 0/16 	 ROW RANK/SIZE: 0/4
WORLD RANK/SIZE: 1/16 	 ROW RANK/SIZE: 1/4
WORLD RANK/SIZE: 2/16 	 ROW RANK/SIZE: 2/4
WORLD RANK/SIZE: 3/16 	 ROW RANK/SIZE: 3/4
WORLD RANK/SIZE: 4/16 	 ROW RANK/SIZE: 0/4
WORLD RANK/SIZE: 5/16 	 ROW RANK/SIZE: 1/4
WORLD RANK/SIZE: 6/16 	 ROW RANK/SIZE: 2/4
WORLD RANK/SIZE: 7/16 	 ROW RANK/SIZE: 3/4
WORLD RANK/SIZE: 8/16 	 ROW RANK/SIZE: 0/4
WORLD RANK/SIZE: 9/16 	 ROW RANK/SIZE: 1/4
WORLD RANK/SIZE: 10/16 	 ROW RANK/SIZE: 2/4
WORLD RANK/SIZE: 11/16 	 ROW RANK/SIZE: 3/4
WORLD RANK/SIZE: 12/16 	 ROW RANK/SIZE: 0/4
WORLD RANK/SIZE: 13/16 	 ROW RANK/SIZE: 1/4
WORLD RANK/SIZE: 14/16 	 ROW RANK/SIZE: 2/4
WORLD RANK/SIZE: 15/16 	 ROW RANK/SIZE: 3/4
```

出力順序が違っても心配しないでください。MPIプログラムで出力する場合、各プロセスはMPIジョブを起動した場所に出力を送り返さないと画面に出力されないからです。この例では見栄え良くするために並び替えています。

最後に`MPI_Comm_free`でコミュニケータを解放することを忘れないでください。これは重要なステップではないように思うかもしれませんが、メモリを使い終わったら解放するのと同じくらい重要です。MPIオブジェクトが使われなくなったら、後で再利用できるように解放しなければなりません。MPIが一度に生成できるオブジェクトの数には限りがあるので、オブジェクトを解放しておかないとMPIが割り当て可能なオブジェクトを使い果たしたときに実行時エラーになる可能性があります。

## その他のコミュニケータ作成機能 - Other communicator creation functions

`MPI_Comm_split`は最も基本的なコミュニケータ作成関数ですが、他にも多くの関数があります。`MPI_Comm_dup`はコミュニケータを複製します。複製だけを行う関数が必要か？と思われるかもしれませんがライブラリを実現するために非常に便利です。なぜなら、自分のコードとライブラリーのコードが互いに干渉しないようにしなければならないからです。ですから、アプリケーションが最初に行うべきことは`MPI_COMM_WORLD`の複製を作成することです。ライブラリ自身も`MPI_COMM_WORLD`を複製して使うべきです。

もう一つの関数は`MPI_Comm_create`です。この関数は (後述する)`MPI_Comm_create_group` とよく似ています。

```cpp
MPI_Comm_create(
	MPI_Comm comm,
	MPI_Group group,
    MPI_Comm* newcomm)
```

最も大きな違いは`MPI_Comm_create`が`comm`に含まれる全てのプロセスを集団として扱うのに対して、 (後述する)`MPI_Comm_create_group` は `group` に含まれるプロセス群だけを対象とします。これはコミュニケータのサイズが非常に大きい時に大切になります。`MPI_COMM_WORLD` のサブセットを1,000,000プロセスで実行する場合、サイズが大きくなると集団通信は非常に高価なコストとなるため、少ないプロセスで処理を実行することが重要になってくるのです。

インターコミュニケータとイントラコミュニケータの違い、その他の高度なコミュニケータ作成関数などは今回のチュートリアルでは取り上げません。しかし、コミュニケータには他にも高度な機能があります。これらは特殊なアプリケーションでのみ使用されるものです。将来のチュートリアルで取り上げるかもしれません。

## グループの概要 - Overview of groups

`MPI_Comm_split` は新しいコミュニケータを作成する最も簡単な方法ですが他にもコミュニケータを作る方法はあります。それは`MPI_Group`という新しい種類のMPIオブジェクトを使う方法です。グループについて詳しく説明する前に、コミュニケータとは何かもう少し説明します。MPI内部的にはコミュニケータを構成する2つの主要な情報、コミュニケータと他のコミュニケータを区別するコンテキスト(context)またはIDと、コミュニケータに含まれるプロセスのグループを管理しています。コンテキストは、あるコミュニケータ上の操作が他のコミュニケータの操作に影響しないようにするためのものです。このためにMPIは内部で各コミュニケータのIDを保持しています。グループとはそのコミュニケータに含まれるすべてのプロセスの集合のことです。これまで使ってきた`MPI_COMM_WORLD`とは`mpiexec`で起動されたすべてのプロセスです。他のコミュニケータはグループが異なります。上のコード例では`MPI_Comm_split`に同じ`color`にしたすべてのプロセスは同じグループになっています。

MPIは集合論(set theory)で扱われる操作をグループに適応することができます。集合論をすべて理解する必要はありませんが2つの操作の意味を知っておいてください。ここでは"集合(set)"と呼ぶ代わりに、MPIに適用される "グループ(group)"という用語を使用します。まず、和集合(union)演算は2つの集合から新しい（潜在的に）大きな集合を作ります。この新しい集合には2つの集合のすべてのメンバが含まれます(重複はありません)。次に、積集合(intersection)は、他の二つの集合から新しい（潜在的に）小さい集合を作ります。この新しい集合には、元の集合の両方に存在するメンバがすべて含まれます。これら両方の操作の例を以下に図解で示します。

![Group Operation Examples](../groups.png)

上段の例は`{0, 1, 2, 3}` と `{2, 3, 4, 5}` の和集合は `{0, 1, 2, 3, 4, 5}` となります。2つ目の例では、`{0, 1, 2, 3}` と `{2, 3, 4, 5}` の積集合は `{2, 3}` となります。

## グループの使用 - Using MPI groups

グループの仕組みの基本がわかったので実際のMPI操作でどう使うのかをみていきます。MPIでは`MPI_Comm_group`ルーチンでコミュニケータ内のプロセスのグループを簡単に取得することができます。

```cpp
MPI_Comm_group(
	MPI_Comm comm,
	MPI_Group* group)
```

コミュニケータにはコンテキスト（ID）とグループが含まれます。`MPI_Comm_group` はそのグループオブジェクトへの参照を得ます。グループオブジェクトはコミュニケータオブジェクトと同じように動作しますが、集団通信ルーチンの引数としで他のランクと通信するためには使用できません（コンテキストが付加されていないためです）。ただし、グループのランクとサイズを取得することはできます (`MPI_Group_rank` と `MPI_Group_size`)。コミュニケーターではできずにグループだけができることとは、ローカルで新しいグループを作成することです。ここで注目するのはローカル操作とリモート操作の違いに注意してください。リモート操作では他のランクと通信が発生しますが、ローカル操作では通信は発生しません。新しいコミュニケーターを作成する場合はそのアプリケーション内のすべてのプロセスで同じコンテキストとグループを決定する必要があるためリモート操作となります。しかし、グループを作成する場合は各プロセスで同じコンテキストを持つ必要がないため通信する必要はなくローカル操作となります。このため通信を気にする必要はなくなります。

グループに対する操作はとても簡単です。

```cpp
MPI_Group_union(
	MPI_Group group1,
	MPI_Group group2,
	MPI_Group* newgroup)
```

積集合もみてみましょう。

```cpp
MPI_Group_intersection(
	MPI_Group group1,
	MPI_Group group2,
	MPI_Group* newgroup)
```

どちらの演算も、操作は`group1`と`group2`に対して行われて、結果は`newgroup`に格納されます。

MPI におけるグループの使い方はたくさんあります。グループが同じかどうかを比較する、あるグループから別のグループを引く、あるグループから特定のランクを除外する、あるグループのランクを別のグループに変換する、といったようにグループを使用することができます。最近MPIに追加された関数の中で最も役に立つのは`MPI_Comm_create_group`でしょう。これは新しいコミュニケータを作成する関数ですが `MPI_Comm_split` のようにその場で計算をして構成を決めるのではなく、 `MPI_Group`を受け取り、グループと同じプロセスをすべて持つ新しいコミュニケータを作成します。

```cpp
MPI_Comm_create_group(
	MPI_Comm comm,
	MPI_Group group,
	int tag,
	MPI_Comm* newcomm)
```

## グループの使用例 - Example of using groups

グループの使い方の簡単な例です。`MPI_Group_incl`という関数を使ってグループ内の特定のランクを選択し、そのランクのみを含む新しいグループを作成します。

```cpp
MPI_Group_incl(
	MPI_Group group,
	int n,
	const int ranks[],
	MPI_Group* newgroup)
```

この関数は`group`に含まれるプロセスのうち、`ranks`に含まれるランクを持つプロセスのだけが`newgroup`に入ります。どのように機能するかを確かめるため、`MPI_COMM_WORLD`の素数のランクを含むコミュニケータを作成してみます。(訳注:素数は静的に与えています)

```cpp
// 元のコミュニケータのサイズとその中でのランクを取得
int world_rank, world_size;
MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
MPI_Comm_size(MPI_COMM_WORLD, &world_size);

// このプロセスのMPI_COMM_WORLD内でのグループを得る
MPI_Group world_group;
MPI_Comm_group(MPI_COMM_WORLD, &world_group);

int n = 7;
const int ranks[7] = {1, 2, 3, 5, 7, 11, 13};

// world_groupで素数ランクを持つプロセスだけのグループを作成する
MPI_Group prime_group;
MPI_Group_incl(world_group, 7, ranks, &prime_group);

// このグループを元にしたコミュニケータを作成する
MPI_Comm prime_comm;
MPI_Comm_create_group(MPI_COMM_WORLD, prime_group, 0, &prime_comm);

int prime_rank = -1, prime_size = -1;
// このプロセスが新しいコミュニケータに所属しない場合、MPI_COMM_NULL になります。
// MPI_Comm_rankまたはMPI_Comm_sizeを使う際に第一引数にMPI_COMM_NULL のコミュニケータを指定してはいけません
if (MPI_COMM_NULL != prime_comm) {
	MPI_Comm_rank(prime_comm, &prime_rank);
	MPI_Comm_size(prime_comm, &prime_size);
}

printf("WORLD RANK/SIZE: %d/%d \t PRIME RANK/SIZE: %d/%d\n",
	world_rank, world_size, prime_rank, prime_size);

MPI_Group_free(&world_group);
MPI_Group_free(&prime_group);
MPI_Comm_free(&prime_comm);
```

この例では、`MPI_COMM_WORLD`の素数のランクのみを選択してコミュニケータを作成します。まずは `MPI_Group_incl`でグループ`prime_group`を生成します。次に、このグループを `MPI_Comm_create_group`に渡して コミュニケータ`prime_comm`を作成します。そして`ranks` に含まれていないランクの`MPI_Comm_create_group`から返されるコミュニケータが`MPI_COMM_NULL`でないことを確認してランクやグループを確認します。

(以下は訳者の環境でn=8で実行した例です)
```
WORLD RANK/SIZE: 6/8 --- PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 0/8 --- PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 4/8 --- PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 3/8 --- PRIME RANK/SIZE: 2/5
WORLD RANK/SIZE: 1/8 --- PRIME RANK/SIZE: 0/5
WORLD RANK/SIZE: 5/8 --- PRIME RANK/SIZE: 3/5
WORLD RANK/SIZE: 2/8 --- PRIME RANK/SIZE: 1/5
WORLD RANK/SIZE: 7/8 --- PRIME RANK/SIZE: 4/5
```