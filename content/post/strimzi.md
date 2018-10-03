---
title: "Kafka operator (strimzi)でKubernetes上にNoOpsなメッセージングシステムを実現する"
date: 2018-10-02T16:01:23+08:00
lastmod: 2018-10-03T16:01:23+08:00
draft: false
tags: ["kafka", "openshift", "kubernetes"]
categories: ["Technology"]
comment: true
toc: true
---

{{< figure src="/images/strimzi_logo.png" >}}

## KafkaとKubernetes

メッセージングシステムは昔からいろいろな用途で使われてきましたが、近年のモダンなサービス開発という流れでもその重要性は大きくなっています。   
マイクロサービス間の非同期な連携とか、イベントソーシングなアプリのイベントストア、分散システムのログ集約などなど様々ですね。

Kubernetes上でマイクロサービスを作成する場合も、kubernetesクラスタ上にKafkaをインストールしてサクッと使ってみたいものです。

> {{< figure src="https://cdn-images-1.medium.com/max/1600/1*s0jqmRff9avwTrfQPHfeeg.png" >}}
> https://medium.com/@ulymarins/an-introduction-to-apache-kafka-and-microservices-communication-bf0a0966d63

しかしKafkaは、Kafkaのプロセスだけでなくzookeeperのクラスタも構築して保守してやる必要があったり、そもそもステートフルだったりで、kubernetes上でデプロイ、保守するのはとても敷居が高いものでした。

そこを解決するのが[strimzi](http://strimzi.io/)というOSSです。

strimziは、`operator`という仕組みでkubernetes上のKafkaクラスタの管理を大幅に省力化します。

`operator`に関しては、 https://pocketstudio.net/2016/11/10/introducing-operators-translate-jp/ あたりが参考になります。  
その実体は、 **Kubernetesにデプロイされたコンテナ** です。  
KubernetesのAPIでイベントを監視して、コンテナをデプロイしたり設定したりバックアップしたりといった保守作業を行うコンテナのことを`operator`と呼びます。

strimziでは、以下3つの`operator`を提供しています。

* [Cluster Operator](http://strimzi.io/docs/0.7.0/#cluster-operator-str)  
`Kafka`というCustom Resourceの状態を監視して、zookeeperやKafkaのデプロイ、設定、保守を行う  
{{< figure src="/images/strimzi_cluster_operator.png" title="Strimzi Cluster Operator" >}}
* [Topic Operator](http://strimzi.io/docs/0.7.0/#assembly-getting-started-topic-operator-str)  
KafkaTopicというCustom Resourceの状態を監視して、Cluster Operatorが保守しているKafka上にtopicを作成したり削除したり設定変更したりする
* [User Operator](http://strimzi.io/docs/0.7.0/#assembly-getting-started-user-operator-str)  
KafkaUserというCustom Resourceの状態を監視して、Kafkaのユーザーを管理する

つまり、strimziを使うと、以下のようなyamlを、

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
    listeners:
      plain: {}
      tls: {}
    readinessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 5
    livenessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 5
# ...
```
`kubectl -f kafka-cluster.yml`というように適用するだけでkubernetes上にKafkaをプロビジョニングできるということです。便利ですね。

そういうわけで、さっそくローカルのk8s環境で試してみましょう。

## minishiftにKafkaをoperatorでインストールする

strimziがサポートしているKubernetes環境は、Kubernetes 1.9以上またはOpenShift3.9以上とのことです。  
http://strimzi.io/docs/0.7.0/#getting-started-str

今回は、minishiftでローカルにOpenShift環境を立ち上げて試してみます。

minishiftのインストールはこちらを参照してください。  
https://docs.okd.io/latest/minishift/getting-started/installing.html

インストールしたら、以下コマンドでローカルにOpenShiftを立ち上げます。  
何となくメモリたくさん使いそうなので、6GB割り当てるようにしました。

```sh
$ minishift start --memory=6GB

#...

$ oc login -u system:admin
```

これ以降の手順ではclusterスコープのCRDを設定したりする関係上、`cluster-admin`権限が必要なので、とりあえず`system:admin`でログインしておきます。

### strimziのダウンロード

strimziをGitHubからダウンロードします。

https://github.com/strimzi/strimzi-kafka-operator/releases

### strimziのCluster Operatorをインストールする

Kafkaのクラスタをインストール、設定してくれる、strimziの`Cluster Operator`をインストールします。

```bash
$ oc new-project kafka-op # Cluster Operatorをインストールするproject(namespace)を作成する
$ cd strimzi-0.7/ # ダウンロードしたstrimziを解凍した先に移動
# Cluster Operatorのデプロイ設定に記載されているプロジェクト名を変更する
$ sed -i '' 's/namespace: .*/namespace: kafka-op/' examples/install/cluster-operator/*RoleBinding*.yaml
# Cluster Operatorのデプロイ
$ oc apply -f examples/install/cluster-operator -n kafka-op
$ oc apply -f examples/templates/cluster-operator -n kafka-op
```

### Cluster OperatorでKafkaのクラスタをインストールする

Cluster Operatorがインストールされれば、それを使ってKafkaのクラスタをインストールできます。

Cluster Operatorは、project(namespace)内のイベントを監視しており、ここに`Kafka`(CRD)を作成することでKafkaのクラスタをインストール、設定してくれます。

`Kafka`(CRD)は、yamlの例が「examples/kafka」に格納されています。

examples/kafka/kafka-ephemeral.yaml
: 永続ストレージを割り当てない揮発性のKafkaクラスタ(検証用)

examples/kafka/kafka-persistent.yaml
: 永続ストレージを割り当てたKafkaクラスタ

今回は、検証用にephemeralなほうをインストールします。`Kafka`CRDは以下のような形式です。

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
    listeners:
      plain: {}
      tls: {}
    readinessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 5
    livenessProbe:
      initialDelaySeconds: 15
      timeoutSeconds: 5
```

なんとなく、どんなクラスタができるか想像できますね。このCRDをKubernetes上に作成します。

```sh
$ oc apply -f examples/kafka/kafka-ephemeral.yaml
$ oc get kafka

NAME           AGE
my-cluster     14s
```

上記のようにkafkaリソースが作成されたら、Cluster Operatorがkafkaリソースの定義に従ってKafkaのクラスタをセットアップしてくれるので、それを確認します。

```sh
$ oc get pods -w

NAME                                          READY     STATUS    RESTARTS   AGE
my-cluster-entity-operator-54758cf6f6-nvcqm   3/3       Running   0          2m
my-cluster-kafka-0                            2/2       Running   1          3m
my-cluster-kafka-1                            2/2       Running   1          3m
my-cluster-kafka-2                            2/2       Running   1          3m
my-cluster-zookeeper-0                        2/2       Running   0          4m
my-cluster-zookeeper-1                        2/2       Running   1          4m
my-cluster-zookeeper-2                        2/2       Running   0          4m
strimzi-cluster-operator-7d8898b9b9-d6sqd     1/1       Running   0          46m
```

このように、Kafkaとzookeeperがインストールされていることを確認できます。

## Topic Operatorで、KafkaクラスタにTopicを作成する

strimziは、Kafkaクラスタそのものを保守するCluster Operator以外にも、Kafkaクラスタにtopicやuserを作ってくれるOperatorも設定されます。

Topic Operatorは、上記手順でインストールされた「my-cluster-entity-operator-xxxx」(Entity Operator)にあるので、すでに使える状態になっています。

さっそくTopic OperatorでKafkaクラスタ上にtopicを作成してみます。「examples/topic/kafka-topic.yaml」に例がありますが、以下のようなKafkaTopicというCRDを作成することでTopic OperatorがKafkaクラスタ上にtopicを作成してくれます。

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaTopic
metadata:
  name: my-topic
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 1
  replicas: 1
  config:
    retention.ms: 7200000
    segment.bytes: 1073741824
```

これを、以下のようにKubernetesクラスタに適用します。

```sh
$ oc apply -f examples/topic/kafka-topic.yaml
```

これで、Kafkaクラスタにtopicが作成されます。

## Kafkaクラスタにメッセージを投げてみる

さて、Kafkaクラスタがインストールされたので、このKafkaにメッセージをPublish/Subscribeしてみます。

strimziには、サンプルのクライアントも付いているので、簡単にKafkaクラスタを検証できます。

### メッセージをSubscribeするクライアントを実行する
以下でメッセージをSubscribeしてみます。

```sh
$ oc run kafka-consumer -ti --image=strimzi/kafka:0.7.0 --restart=Never \-- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```

上記コマンドを実行すると、Kubernetes上でサンプルクライアントが実行され、Kafkaのトピック「my-topic」をSubscribeする状態になります。

メッセージをPublishするクライアントを実行する
ターミナルをもう一つ開いて、以下コマンドを実行します。

```sh
$ oc run kafka-producer -ti --image=strimzi/kafka:0.7.0 --restart=Never \-- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
```

プロンプトが開くので、適当に文字列を入力してみてください。

```sh
If you don't see a command prompt, try pressing enter.
>test
>a aaa
```

すると、kafka-consumerの上で入力したメッセージが表示されるのを確認できるはずです。

```sh
If you don't see a command prompt, try pressing enter.
>test
>a aaa
```

まとめ
さて、Kubernetes上にKafkaをインストールし、メッセージングプラットフォームを構築してみました。

実際には、JavaやRubyのクライアントからこのKafkaのトピックをPub/Subして非同期通信、マイクロサービスを実現したり、IoTなイベントを受けたり、ログ集約基盤を作ったりするわけです。

そういうわけで、次はJavaアプリからこのKafkaを使ってみます。