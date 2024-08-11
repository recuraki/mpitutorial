---
layout: post
title: Amazon EC2 MPIクラスタを起動する - Launching an Amazon EC2 MPI Cluster
author: Wes Kendall
categories: Beginner MPI
tags:
redirect_from: '/launching-an-amazon-ec2-mpi-cluster/'
---

[前回のレッスン]({{ site.baseurl }}/tutorials/installing-mpich2/)ではMPICH2を1台のマシンにインストールしする方法を説明しました。ただし、MPIを学習しプログラムを実行するために十分なリソースが1台のマシンで提供できるとは限りません。クラスタに簡単にアクセスできる初心者はそういないでしょう。最高のMPIチュートリアルのためには、このサイトのチュートリアルコードはもちろん、自分の並列コードを実行できる環境が必要なので、仮想MPIクラスタをセットアップする方法を説明します。

## Amazon EC2で始めよう
このレッスンではAmazonのElastic Compute Cloud (EC2)を使ったクラスタの説明をします。Amazon EC2を始めるには、[Amazon Web Services (AWS)](http://aws.amazon.com/)にアクセスし、"Sign Up "ボタンを押します。サービスを利用するには支払い情報を入力する必要があり、利用したサービスに応じて課金されます。

> **Note** - AWSにサインアップする前に、必ず[EC2の価格設定](http://aws.amazon.com/ec2/pricing/)を読んでなにをしようとしているのかを理解してください。この記事を書いている時点では、AWSは一部のマシンサイズに無料時間枠を提供しており、1時間あたり2アメリカセントという低価格のマシンも提供されています。

AWSにサインアップしたら、[EC2スタートガイド](http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html?r=1874)を読んでください。さまざまなインスタンスの起動、アクセス方法、終了の方法などを知っておかなければなりません。

一方で、MPIクラスタを作成しアクセスするためにAmazonのEC2インフラストラクチャを完全に理解する必要はありません。EC2の基本がわかったら次のステップに進んでください。

## StarCluster のインストール
仮想クラスタを作成するために使うツールは[MITのStarCluster toolkit](http://star.mit.edu/cluster/)です。StarClusterは、EC2上にクラスタを構築してアクセス可能とする一連のプロセスを自動化したツールセットです。StarClusterはクラスタの作成・起動だけでなく、OpenMPIや並列アプリケーションのためのソフトウェアも一括でインストールしてくれます。

StarClusterツールキットをローカルマシンにインストールするには(Linux/MacOSXでは)、次のように入力します：

```
$ sudo easy_install StarCluster
```

Windowsを使っている場合は[Windows installation instructions](http://star.mit.edu/cluster/docs/latest/installation.html#installing-on-windows)を参照してください。

## StarClusterの設定 - Configuring StarCluster
インストールが終わったら次のように実行してみてください。

```
$ starcluster help
```

この時点ではStarClusterは設定されていないため、以下のように出力されるでしょう（ディレクトリが私のとは異なることに注意してください）。

```
StarCluster - (http://web.mit.edu/starcluster) (v. 0.93.3)
Software Tools for Academics and Researchers (STAR)
Please submit bug reports to starcluster@mit.edu

!!! ERROR - config file /Users/wesleykendall/.starcluster/config does not exist

Options:
--------
[1] Show the StarCluster config template
[2] Write config template to /Users/wesleykendall/.starcluster/config
[q] Quit

Please enter your selection:
```

2を入力しましょう。StarClusterはホームディレクトリ内の `~/.starcluster/config`にデフォルトの構成ファイルを生成します。

次はAWSアカウントからAWSアクセスキー、シークレットアクセスキー、12桁のユーザーIDを取得しましょう。この情報は、[Amazon Web Services](http://aws.amazon.com/)にアクセスし、右上にある "My Account/Console "をクリックし、"My Security Credentials "をクリックすることで確認できます。

![Amazon EC2 Security Credentials Access](../security_creds.png)

"Access Credentials"セクションの中に"Access Key ID "フィールドと"Secret Access Key"フィールドがあります。ページの下部には"Account Identifiers"セクションがあり、"AWS Account ID"フィールがあります。

デフォルトの設定ファイル(`~/.starcluster/config`)をテキストエディタで開き`[aws info]`の行を探し、適切なフィールドにAWSの情報を入力してください。

```
[aws info]
AWS_ACCESS_KEY_ID = # Your Access Key ID here
AWS_SECRET_ACCESS_KEY = # Your Secret Access Key here
AWS_USER_ID = # Your 12-digit AWS Account ID here (no hyphens)
```

これらの情報を入力したら設定ファイルを保存してsshの公開/秘密鍵ペアを作成します。この鍵はAmazonにアップロードされ、クラスタにログインする際の認証に使用します。StarClusterで公開/秘密鍵ペアを生成しましょう。

```
$ starcluster createkey mykey -o ~/.ssh/mykey.rsa
```
これにより、`~/.ssh/mykey.rsa`に「mykey」キーが作成され、AWSアカウントにもキーが作成されます。Amazonの認証情報を正しく入力した場合は、次のような出力になるでしょう。

```
>>> Successfully created keypair: mykey
>>> fingerprint: ...
>>> contents:
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

設定ファイルを再度開き、`[key mykey]`エントリがあることを確認してください。このエントリがない場合は、設定に次の内容を追加します。

```
[key mykey]
KEY_LOCATION = ~/.ssh/mykey.rsa
```

そしてクラスタパラメータを設定します。デフォルトの設定では、`[cluster smallcluster]`に "smallcluster "というクラスタの設定が書かれています。デフォルトで以下のパラメータが設定されているはずでしょう。

```
[cluster smallcluster]
KEYNAME = mykey
CLUSTER_SIZE = 2
CLUSTER_USER = sgeadmin
CLUSTER_SHELL = bash
NODE_IMAGE_ID = ami-899d49e0
NODE_INSTANCE_TYPE = m1.small
```

このフィールドをみていきます。２つ以外のノードでクラスタを開始したい場合は、`CLUSTER_SIZE`オプションを変更してください。別のキー(例の"mykey"以外) を定義している場合は、`KEYNAME` フィールドに適切なキーを追加します。クラスタを実行すると、ネットワークファイルシステム (NFS) にマウントされたホームディレクトリを持つ `CLUSTER_USER` ユーザ名が自動的に生成されます。`NODE_IMAGE_ID` はクラスタソフトウェアのイメージIDです。最後のパラメータ `NODE_INSTANCE_TYPE` は各ノードのサイズを決定します。利用可能なインスタンスタイプとその属性のリストについては、[ここ](http://aws.amazon.com/ec2/instance-types/)を参照してください。

クラスタの実行コストを決定するには、ノード数にインスタンス・タイプの時間単価を掛ければよいです。この記事を書いている時点ではm1.smallインスタンスは1時間あたり6.5アメリカセントです。費用は時間に対して課金されます。クラスタを30分間稼働させた場合、1時間分の料金が請求されます。

> **Note** - t1.microインスタンスは最も安価ですが、私の場合StarClusterでクラスタを起動すると、うまく起動できません。

### Starclusterのmpich2プラグインを有効にする。
最後に、Starclusterを起動する前に、[以下の手順](http://star.mit.edu/cluster/docs/0.93.3/plugins/mpich2.html)に従って、Starcluster用の`mpich2`プラグインを有効にしてください。

## クラスタの起動、アクセス、停止
設定が終わったら次のように入力して"mpicluster"クラスタを起動します。デフォルトの構成では、デフォルトのクラスタタイプとして"smallcluster"が使用されます：

```
starcluster start mpicluster
```

このプロセスは構成によっては少し時間がかかります。コマンドの完了後StarClusterはクラスタへのアクセス、停止、および再起動に使用できるコマンドを出力します。

次のコマンドでクラスタのマネージャノードにSSHでログインできます。

```
starcluster ssh manager mpicluster
```

クラスタにログインすると、カレントディレクトリは`/root`となります。コードのコンパイルは`/home/ubuntu`または`/home/sgeadmin`に移動してから行ってください。このディレクトリはNFSマウントされており、クラスタ内のすべてのノードから共有されています。

次にGitHubレポジトリからこのMPIチュートリアルのコードをチェックアウトしてください。このサイトのすべてのレッスンで使用されているコードにアクセスできます。

```
git clone git://github.com/mpitutorial/mpitutorial.git
```

クラスタへのアクセスに慣れてきたらログアウトしてクラスタを停止ましょう。

```
starcluster stop mpicluster
```

クラスターは次のように入力して再度起動できます。

```
starcluster start -x mpicluster
```

クラスターを完全に終了するには、次のように入力します。

```
starcluster terminate mpicluster
```

"stop"と"terminate"の違いはなんでしょう？stopしたクラスタはまだAmazonのElastic Block Store（EBS）上にイメージが残っています。Amazon EBSの料金は保存されている量に対して課金されるため、当面クラスタを使用しない場合はクラスタのterminateをお勧めします。しっかりと[Amazon EC2 Pricing](http://aws.amazon.com/ec2/pricing/)について理解してください。

## MPI　クラスタの準備はできましたか？ - Ready to run MPI programs on your cluster?
ついに自分のMPIクラスタを手に入れたので、プログラムを実行しましょう。最初に[MPI hello world アプリケーション]({{ site.baseurl }}/tutorialss/mpi-hello-world/)のコンパイルと実行方法をチェックしてください。ローカルクラスタを構築して同じことを試したい場合は、[running an MPI cluster within a LAN]({{ site.baseurl }}/tutorialss/running-an-mpi cluster-within-a-lan)チュートリアルを参照してください。全てのレッスンは、[MPIチュートリアル]({{ site.baseurl }}/tutorials/)をチェックしてください。もしレッスンで何か問題があれば、以下にコメントを残してください。