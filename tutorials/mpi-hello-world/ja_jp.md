---
layout: post
title: 初めてのMPIプログラム - MPI Hello World
author: Wes Kendall
categories: Beginner MPI
tags: 
redirect_from: '/mpi-hello-world/'
---

このレッスンでは基本的なMPI Hello Worldアプリケーションの作成方法とMPIプログラムの実行方法を説明します。MPIの初期化と複数のプロセスにわたるMPIジョブの実行の基本について説明します。このレッスンは、MPICH2(1.4)のインストールで動作することを確認しています。MPICH2をインストールしていない場合は[installing MPICH2 lesson]({{ site.baseurl }}/tutorials/installing-mpich2/)を参照してください。


> **Note** : このサイトのコードはすべて [GitHub]({{ site.github.repo }})にあります。このチュートリアルのコードは[tutorials/mpi-hello-world/code]({{ site.github.code }}/tutorials/mpi-hello-world/code)にあります。

## Hello, World!: Hello world code examples
それでは[mpi_hello_world.c]({{ site.github.code }}/tutorials/mpi-hello-world/code/mpi_hello_world.c)にあるコードを読んでいきます。


```cpp
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
    // Initialize the MPI environment
    // MPI環境の初期化
    MPI_Init(NULL, NULL);

    // Get the number of processes
    // プロセスの数を得る
    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    // Get the rank of the process
    // このコミュニケータ内の自分のランクを得る
    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

    // Get the name of the processor
    // このプロセッサーの名前を得る
    char processor_name[MPI_MAX_PROCESSOR_NAME];
    int name_len;
    MPI_Get_processor_name(processor_name, &name_len);

    // Print off a hello world message
    // Hello, World!を表示する。
    printf("Hello world from processor %s, rank %d out of %d processors\n",
           processor_name, world_rank, world_size);

    // Finalize the MPI environment.
    // MPI環境をクローズする
    MPI_Finalize();
}
```

MPIプログラムではMPIヘッダのインクルードが必要です(`#include <mpi.h>`)。次にMPI環境を初期化する必要があります。



```cpp
MPI_Init(
    int* argc,
    char*** argv)
```

`MPI_Init`: MPIのグローバル変数と内部変数が初期化・設定されます。このプログラムでは、全てのプロセスのそれぞれにランクを割り当て、それら全てを含むコミュニケータを作成します。`MPI_Init`は特に設定する引数がありません。追加のパラメータは将来の実装で必要になった場合に備えて予約されています。

`MPI_Init`の後には２つの関数を呼び出します。これらはほとんど全てのMPIプログラムで呼び出されるコードです。

```cpp
MPI_Comm_size(
    MPI_Comm communicator,
    int* size)
```

`MPI_Comm_size`はコミュニケータのサイズを返します。サンプルで引数に与えている`MPI_COMM_WORLD`(MPIにより初期化される変数)はジョブ内のすべてのプロセスを含むのでこの呼び出しはジョブに要求したプロセスの数を返します。

```cpp
MPI_Comm_rank(
    MPI_Comm communicator,
    int* rank)
```

`MPI_Comm_rank`:コミュニケータ内におけるそのプロセスのランクを返します。コミュニケータ内の各プロセスには0から順にランクが割り当てられます。ランクは主にメッセージの送受信のために使用されます。

次の関数は実際のコードではあまり使われることはないでしょう。

```cpp
MPI_Get_processor_name(
    char* name,
    int* name_length)
```

`MPI_Get_processor_name`はプロセッサの名前を取得します。最後に呼ばれる関数も見ていきましょう。

```cpp
MPI_Finalize()
```

`MPI_Finalize`はMPI環境のクリーンアップです。この関数の後はMPI関数を使うことはできません。

## アプリケーションの実行: Running the MPI hello world application
gitからコードをcloneしてコードのフォルダを見てみましょう。その中にはMakefileがあります。

```
>>> git clone {{ site.github.repo }}
>>> cd mpitutorial/tutorials/mpi-hello-world/code
>>> cat makefile
EXECS=mpi_hello_world
MPICC?=mpicc

all: ${EXECS}

mpi_hello_world: mpi_hello_world.c
    ${MPICC} -o mpi_hello_world mpi_hello_world.c

clean:
    rm ${EXECS}
```

このmakefileはMPICC環境変数が設定されているならばそれを用います。MPICH2をローカルディレクトリにインストールした場合は、MPICC環境変数を適切なmpiccバイナリパスを指すように設定してください。mpiccは必要なライブラリやインクルードを行ってくれるgccのラッパーです。

```
>>> export MPICC=/home/kendall/bin/mpicc
>>> make
/home/kendall/bin/mpicc -o mpi_hello_world mpi_hello_world.c
```

プログラムのコンパイルが終わり実行の準備が整いました！複数のノードのクラスターでMPIプログラムを実行する場合はホストファイルをセットアップする必要があることに注意してください。単一のマシンでMPIを実行するだけの場合は次の情報は無視してください。

host_fileには、MPIジョブが実行されるすべてのコンピューターのhostnameが含まれています。実行を容易にするために、これらのホストにSSHアクセスできることを確認する必要があります。また、SSHのパスワードプロンプトを回避するために[setup an authorized keys file](http://www.eng.cam.ac.uk/help/jpmg/ssh/authorized_keys_howto.html)を設定する必要があります。たとえばhost_fileの例を示します。

```
>>> cat host_file
cetus1
cetus2
cetus3
cetus4
```
gitレポジトリに含まれるrun.pyで複数のホストを用いるのにはMPI_HOSTSという環境変数を設定する必要があります。これを設定してMPIジョブが起動されると、実行コマンドラインにホストファイルが自動的に含められます。hostファイルが必要ない場合は環境変数は設定不要です。また、MPIのローカルインストールで特定のmpirunバイナリを指したい場合はMPIRUN環境変数を設定する必要があります。

これが完了したら、レポジトリに含まれているrun.pyを使用できます。このスクリプトは*tutorials*ディレクトリに保存されており、すべてのチュートリアルの任意のプログラムを実行できます(実行前に実行可能ファイルのビルドも試行します)。mpitutorialフォルダから次の操作を試してください。


```
>>> export MPIRUN=/home/kendall/bin/mpirun
>>> export MPI_HOSTS=host_file
>>> cd tutorials
>>> ./run.py mpi_hello_world
/home/kendall/bin/mpirun -n 4 -f host_file ./mpi_hello_world
Hello world from processor cetus2, rank 1 out of 4 processors
Hello world from processor cetus1, rank 0 out of 4 processors
Hello world from processor cetus4, rank 3 out of 4 processors
Hello world from processor cetus3, rank 2 out of 4 processors
```

想定通りMPIプログラムはホストファイル内のすべてのホストで動作しました。各プロセスは一意の(ユニークな)ランクを持ち、プロセス名とともに出力されました。出力例からわかるように、実行順序は同期がしていないためプロセスの出力はランダムになっています。

スクリプトはmpirunを呼び出していることに注目してください。これは、MPIがジョブを起動するために使用するプログラムです。このプログラムはプロセスをホストファイル内のホストで生成し、実際のプログラムは各プロセスで実行されます。このスクリプトではMPIプロセスの数を4に設定するために`-n`フラグを自動的に提供します。実行スクリプトを変更して、より多くのプロセスを起動してみてください。ただし、誤ってシステムをクラッシュさせないようにしてくださいね。:-)

さて「私のコンピュータはマルチコアなので他のノードよりも先にあるノードのコアを使いたいのだが」と思っていますか？これは簡単に制御できます。ホストファイルを変更し、ホスト名の後にコロンとプロセッサあたりのコア数を入力すれば良いです。たとえば、各ホストの2つのコアを使いたいとしましょう。

```
>>> cat host_file
cetus1:2
cetus2:2
cetus3:2
cetus4:2
```

実行スクリプトを再度実行するとMPIジョブによって2つのホストのみで4つのプロセスが生成されました！(訳注:hostsは8個のプロセスを処理できることを示しますが、要求されたのは4個のプロセスのみなので2つのホストだけが使われています)

```
>>> ./run.py mpi_hello_world
/home/kendall/bin/mpirun -n 4 -f host_file ./mpi_hello_world
Hello world from processor cetus1, rank 0 out of 4 processors
Hello world from processor cetus2, rank 2 out of 4 processors
Hello world from processor cetus2, rank 3 out of 4 processors
Hello world from processor cetus1, rank 1 out of 4 processors
```

## 次に: Up next
MPIプログラムの実行方法について基本的な理解ができたので、次は基本的なポイントツーポイント通信ルーチンを学習します。次のレッスンは[MPI の基本的な送信ルーチンと受信ルーチン: basic sending and receiving routines in MPI]({{ site.baseurl }}/tutorials/mpi-send-and-receive/)です。
[MPI tutorials]({{ site.baseurl }}/tutorials/)をMPIレッスンの完全なリファレンスとして活用してください。

困っていますか?混乱していますか?お気軽に下記にコメントを残してください。私や他の読者がお役に立てるかもしれません。

