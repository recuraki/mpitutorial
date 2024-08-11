---
layout: post
title: LAN上でのMPIクラスタ構築 - Running an MPI Cluster within a LAN
author: Dwaraka Nath
categories: Beginner MPI
tags: MPI, Cluster, LAN
redirect_from: '/running-an-mpi-cluster-within-a-lan'
---

先のレッスンではMPIプログラムを[1台のマシン]({{ site.baseurl }}/tutorialss/mpi-hello-world/)で実行し、CPUに1つ以上のコアがある利点を生かしてコードを並列処理することを見てきました。それでは同じことを1台のコンピュータではなく、LANに接続されたノードで実行できるようにしましょう。物事を単純にするために、このレッスンでは2台のコンピュータを考えます。ノードをもっと増やすのは同じようにできます。

他のチュートリアルと同様にLinuxマシンを使用することを前提としています。以下のチュートリアルはUbuntuでテストしましたが、他のディストリビューションでも同じはずです。また、あなたの２台のマシンを**管理者(manager)**とし、もう1台を**作業者(worker)**と呼ぶとしましょう。

## 前提

すべてのマシンにMPICH2をインストールしていない場合は、[この手順]({{ site.baseurl }}/tutorials/installing-mpich2/)を実行してください。

## Step 1: `hosts`の準備 - Configure your ```hosts``` file

`hosts`を用意しましょう。これはマシンのオペレーティングシステムがホスト名をIPアドレスにマッピングするために使用します。以下に例を示します。

```bash

$ cat /etc/hosts

127.0.0.1       localhost
172.50.88.34    worker
```
これは`Manager`の`hosts`で、`worker`は、あなたが計算を行いたい他のマシン名です。同様に、Workerでは`manager`も同じようにします。

## Step 2: ユーザの作成 - Create a new user

既存のユーザーアカウントでクラスタを操作することもできます。しかし、設定をシンプルにするために新しいユーザーアカウントを作成することをお勧めします。全てのホストでユーザー`mpiuser`を作成しましょう。

```bash
$ sudo adduser mpiuser
```
プロンプトに従ってください。`adduser`ではなく`useradd`コマンドを使ってしまうとホームディレクトリが作成されないので使用しないでください。

## Step 3: SSHの設定 - Setting up SSH

マシン間が通信できるようにSSHの設定をします。[NFS](#step-4-setting-up-nfs)を使ってデータを共有します。
```bash
$ sudo apt­-get install openssh-server
```

まず、このホストでユーザをmpiuserに切り替えます。

```bash
$ su - mpiuser
```
すでに `ssh` サーバーがインストールされているので、他のマシンに `ssh username@hostname` でログインできるはずです。パスワードを尋ねられないようにキーを生成して他のマシンの ``authorized_keys`` のリストにコピーします。(訳注：この処理は公開鍵認証を可能にする、という処理です。パスワード認証を禁止してはいないことに注意してください)

```bash
$ ssh-keygen -t dsa
```

この例ではDSA鍵を作成しましたがRSA鍵を生成することもできます。より高いセキュリティを求めるのであれば、RSAを使っても良いですがDSAでも十分です。そして、生成した鍵を他のコンピューターに追加すしましょう。wokerノードに対してコピーします。

```bash
$ ssh-copy-id worker #ip-address may also be used
```

それぞれのWorkerマシンと自分のユーザ (localhost) に対してここまでの手順を実行してください。

これで `openssh-server` がセットアップされ、Workerと安全に通信できるようになります。すべてのマシンに一度 `ssh` を実行してください。そして`known_hosts` のリストに他のホストのフィンガープリントを追加します。このステップは`ssh`ログインのトラブルになりやすいので重要なステップです。

パスワードなしの sshができるようにしましょう。

```bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/id_dsa
```
さてパスワードのプロンプトなしで他のマシンにログインできるはずです。

```bash
$ ssh worker
```

> **Note** - すべてのワーカーマシンで共通のユーザアカウントとして `mpiuser` を作成したと仮定しています。ManagerとWorkerで異なる名前のユーザーアカウントを作成した場合は適切なユーザ名を指定してください。

## Step 4: NFSの設定 - Setting up NFS

**manager**はNFS経由でディレクトリを共有し、それを**worker**がマウントしてデータをやり取りします。

### NFS-Server

パッケージをインストールします。

```bash
$ sudo apt-get install nfs-kernel-server
```

まだ `mpiuser` にログインしていると仮定して `cloud` という名前の共有フォルダを作成します。

```bash
$ mkdir cloud
```

`cloud`を共有するために`/etc/exports`を以下のようにします。

```bash
$  cat /etc/exports
/home/mpiuser/cloud *(rw,sync,no_root_squash,no_subtree_check)
```
`*`の部分にはこのフォルダを共有したいIPアドレスを指定することができますが。今回は簡単のため`*`にしています。

* **rw**: read/writeを許可します。readのみを割り当てたいなら **ro**を設定します。
* **sync**: 変更をコミットした後にのみ、共有ディレクトリに変更を適用されます
* **no_subtree_check**: サブツリーをチェックしません。共有ディレクトリが大きなファイルシステムのサブディレクトリである場合、 nfsはその上のすべてのディレクトリのパーミッションと詳細を確認するために スキャンを実行します。サブツリー・チェックを無効にするとNFSの信頼性は向上しますがセキュリティが低下することがあります。
* **no_root_squash**: rootアカウントによるフォルダへの接続を許可します。

> この説明を作るために[Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-12-04)を参考にしました. このライセンスはCreative Commons Attribution-NonCommercial-ShareAlike 4.0 International Licenseに従います。このライセンスの詳細は [ここ](https://creativecommons.org/licenses/by-nc-sa/4.0/)です。

NFSを実行します。

```bash
$ exportfs -a
```

`/etc/exports` に変更を加えた場合はこのコマンドが必要です。

必要に応じて`nfs` サーバを再起動してください。

```bash
$ sudo service nfs-kernel-server restart
```

### NFS-worker

以下を実行します。

```bash
$ sudo apt-get install nfs-common
```

workerで同じ名前のディレクトリを作成します。

```bash
$ mkdir cloud
```

そして次のように共有ディレクトリをマウントしてください。

```bash
$ sudo mount -t nfs manager:/home/mpiuser/cloud ~/cloud
```

正常にマウントできたか確認しましょう。

```bash
$ df -h
Filesystem      		    Size  Used Avail Use% Mounted on
manager:/home/mpiuser/cloud  49G   15G   32G  32% /home/mpiuser/cloud
```

再起動のために手動で共有ディレクトリをマウントする必要がないようにするには ``/etc/fstab`` ファイルに次のようなエントリを作成してください。

```bash
$ cat /etc/fstab
#MPI CLUSTER SETUP
manager:/home/mpiuser/cloud /home/mpiuser/cloud nfs
```

## Step 5: MPIプログラムの実行 - Running MPI programs

それでは正常性の確認のため、MPICH2のインストールパッケージ `mpich2/examples/cpi` に付属しているサンプルプログラムを並列実行してみましょう。

自分のコードをコンパイルしたい場合は、コードを`mpi_sample.c` とすると、以下に示す方法でコンパイルして ``mpi_sample`` を生成できます。


```bash
$ mpicc -o mpi_sample mpi_sample.c
```

そして実行ファイルを共有ディレクトリ `cloud` にコピーしてください。もちろん、NFS 共有ディレクトリ内でコードをコンパイルしても良いです。

```bash
$ cd cloud/
$ pwd
/home/mpiuser/cloud
```

自分のマシン(manager)だけで実行するときは以下のようにします。

```bash
$ mpirun -np 2 ./cpi # No. of processes = 2
```

クラスタで実行してみましょう。

```bash
$ mpirun -np 5 -hosts worker,localhost ./cpi
#ホスト名の代わりにIPアドレスでも良いです。
```

予め用意したホストファイルを使うこともできます。

```bash
$ mpirun -np 5 --hostfile mpi_file ./cpi
```

これでmanagerが接続しているマシンでMPIプログラムが動くはずです！

## 一般的なエラーやTIPS - Common errors and tips

* 実行ファイルを実行しようとしているすべてのマシンで、MPIのバージョンが同じであることを確認してください。これを読んでいる時の推奨バージョンは [MPICH2](http://www.mpich.org/downloads/)を参照してください。
* manager の `hosts` ファイルには、 `manager` とワーカーノードのローカルネットワーク IP アドレスを記述します。ワーカには`manager`と自分自身のエントリが含まれている必要があります。

例を示します。managerには以下のようなhostsが必要になります。

```bash
$ cat /etc/hosts
127.0.0.1	localhost
#127.0.1.1	1944

#MPI CLUSTER SETUP
172.50.88.22	manager
172.50.88.56 	worker1
172.50.88.34 	worker2
172.50.88.54	worker3
172.50.88.60 	worker4
172.50.88.46	worker5
```
このとき、worker3に必要な最低限のhostsは以下の通りです。

```bash
$ cat /etc/hosts
127.0.0.1	localhost
#127.0.1.1	1947

#MPI CLUSTER SETUP
172.50.88.22	manager
172.50.88.54	worker3
```
* MPIを使用してプロセスを並列実行しようとする場合、ローカルだけ、ローカルノードとリモートノードの組み合わせでプロセスを実行することができます。リモートのみでプロセスを起動することは**できません**。

以下の呼び出しは正常です。
```bash
$ mpirun -np 10 --hosts manager ./cpi
# To run the program only on the same manager node
```

以下の呼び出しは正常です。

```bash
$ mpirun -np 10 --hosts manager,worker1,worker2 ./cpi
# To run the program on manager and worker nodes.
```

ですが次の呼び出しをmanagerから行うことはできません(worker1からなら良いのですが)。

```bash
$ mpirun -np 10 --hosts worker1 ./cpi
# Trying to run the program only on remote worker
```

## 次は？ - So, what's next?

クラスタが構築できました！わくわくしますか？ついに並列実行できるプログラムを書くための詳細を知る時が来ました。[MPI hello world lesson]({{ site.baseurl }}/tutorials/mpi-hello-world/)から始めるのが良いです。もしも、Amazon EC2インスタンスを使って同じことを再現したいのであれば、[building and running your own cluster on Amazon EC2]({{ site.baseurl }}/tutorials/launching-an-amazon-ec2-mpi-cluster/)を見てください。その他のレッスンについては、[MPIチュートリアル]({{ site.baseurl }}/tutorials/)のページを参照してください。

ローカルクラスタのセットアップで何か問題があれば、遠慮なく以下にコメントしてください。