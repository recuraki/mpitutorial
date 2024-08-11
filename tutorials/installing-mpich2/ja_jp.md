---
layout: post
title: MPIをシングルマシンにインストールする - Installing MPICH2 on a Single Machine
author: Wes Kendall
categories: Beginner MPI
tags:
redirect_from: '/installing-mpich2/'
---

MPIとは標準仕様を指すものであるのでMPIの実装は複数存在します。このレッスンでは主な実装の１つであるMPICHを使用します。あなたが望むなら他の実装を使用しても良いですが、このレッスンではMPICHインストール手順を紹介します。また、チュートリアル全体で提供されるスクリプトとコードは、最新バージョンのMPICHでの実行だけを確認しています。

MPICHは米国のアルゴンヌ国立研究所が主に開発したメジャーなMPI実装です。MPICHを選択した理由は、私がこのインターフェイスに精通しておりアルゴンヌ国立研究所と縁が深いためです。広く使用されている実装である[OpenMPI](https://www.open-mpi.org/)もぜひ調べてみてください。

## MPICHのインストール - Installing MPICH
[ここ](https://www.mpich.org/)からMPICHの最新バージョンを入手できます。このチュートリアルでは3.3-2(2019年11月13日リリース)を利用します。tar.gzファイルをダウンロードし以下のように伸長します。

```
>>> tar -xzf mpich-3-3.2.tar.gz
>>> cd mpich-3-3.2
```

`./configure`でmakeの準備をします。この際にマシンの特権がなくユーザディレクトリにインストールしたい場合は`./configure --prefix=/installation/directory/path`としてインストールディレクトリを指定できます。Fortranへの対応が不要である場合は`./configure --disable-fortran`とします。利用可能なオプションを全て表示するには`./configure --help`とします。

```
>>> ./configure
Configuring MPICH version 3.3.2
Running on system: Linux localhost.localdomain 5.8.18-100.fc31.x86_64 #1 SMP Mon Nov 2 20:32:55 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
checking build system type... x86_64-unknown-linux-gnu
```

configureが終了して*"Configuration completed."*と表示さたら`make; sudo make install`を使用して MPICH2 をビルド・インストールします。
```
>>> make; sudo make install
make
make  all-recursive

```

成功したなら`mpiexec --version`でインストールした情報を出力できます。

```
>>> mpiexec --version
HYDRA build details:
    Version:                                 3.3.2
    Release Date:                            Tue Nov 12 21:23:16 CST 2019
    CC:                              gcc    
    CXX:                             g++    
    F77:                             gfortran   
    F90:                             gfortran 
```

皆さんの環境でもビルドが無事に完了することを祈りますが、依存関係不足などで問題が起こる可能性があるでしょう。このような場合はエラーメッセージをGoogleで検索してみてください。

## 次は？
単一の環境にMPICHを構築できたのでこのサイトで次に進むための選択肢がいくつかあります。ローカルクラスタをセットアップするためのハードウェアとリソースがすでにある場合は、[running an MPI cluster in LAN]({{ site.baseurl }}/tutorials/running-an-mpi-cluster-within-a-lan)に進むことを推奨します。そのようなクラスタにアクセスできない場合や仮想 MPIクラスターの構築について詳しく知りたい場合は、[building and running your own cluster on Amazon EC2]({{ site.baseurl }}/tutorials/launching-an-amazon-ec2-mpi-cluster/)に進んでください。いずれかの方法でクラスターを構築し終わっていたり、残りのレッスンをスタンドアロンで実行する場合は[MPI hello world lesson]({{ site.baseurl }}/tutorials/mpi-hello-world/)に進んでください。Hello Worldではプログラミングの基礎と最初のMPIプログラムの実行の概要が説明されています。