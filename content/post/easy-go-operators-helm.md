---
title: "Operatorの開発を身近にする技術 - Helm App Operator Kit"
date: 2018-11-07T10:14:03+09:00
draft: false
tags: ["helm", "operator", "kubernetes", "openshift"]
categories: ["Technology"]
comment: true
toc: true
---

{{< figure src="https://coreos.com/assets/images/operators/operator_logo_framework_color.svg" >}}

[Operator Framework](https://coreos.com/operators/)ではOperatorをGo言語で作ることになっていますが、まだまだGoはハードルが高い人もいるし、  
そもそも状態を常時管理する必要が無いようなステートレスなコンテナにいちいちイチからOperatorを作るのは面倒です。

Operator Framework自体がOperatorを簡単に実装できるようにするフレームワークではありますが、そういった要件を考慮してさらに開発を簡単にするラッパーが開発されています。

* [Helm App Operator Kit](https://github.com/operator-framework/helm-app-operator-kit)
* [Ansible Operator](https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/user-guide.md)

前者は、既存のhelmチャートをOperatorに変換するもの、後者はAnsibleでOperatorを作るものです。

この記事では、`Helm App Operator Kit`を紹介します。

## Helm App Operator Kit

helmはKubernetesの世界で既に広く使われているパッケージマネージャーであり、多くのhelmチャートが作られています。

この資産、helmチャートからOperatorを作ることができれば便利そうですね。

helmチャートをOperator化することで、

* CustomResourceによる宣言的なインストール、アップデート、設定変更が可能
* 上記によりhelmコマンドやTillerをインストールすることなくkubectlでインストールやアップデートが完結する
* [Operator Lifecyle Manager](https://github.com/operator-framework/operator-lifecycle-manager)による管理対象にできる

といったメリットがあります。これを実現するのが[Helm App Operator Kit](https://github.com/operator-framework/helm-app-operator-kit)です。

公式ベージにTomcatのhelmチャートをOperator化する例があるので試してみたいと思います。

### 前提

* ローカルマシンにDockerがインストールされていること
* QuayやDockerHubに自分で管理しているリポジトリがあること
* kubectlコマンドで操作できるKubernetes環境があること

### Helm App Operator Kitのダウンロード

git cloneしてくるだけです。

```bash
$ git clone https://github.com/operator-framework/helm-app-operator-kit
```

### HelmチャートからOperatorのコンテナイメージをビルドする

`Helm App Operator Kit`では、build-argsにhelmチャートのtarballをURLで指定してdocker buildすることで、helmチャートをコンテナイメージに変換します。

```bash
$ cd helm-app-operator-kit/examples/tomcat-operator

$ export IMAGE=quay.io/<namespace>/tomcat-operator:v0.0.1

# Operatorコンテナイメージをhelmチャートtomcat-0.1.0.tgzからビルド
$ docker build \
  --build-arg HELM_CHART=https://storage.googleapis.com/kubernetes-charts/tomcat-0.1.0.tgz \
  --build-arg API_VERSION=apache.org/v1alpha1 \
  --build-arg KIND=Tomcat \
  -t $IMAGE ../../

# コンテナレジストリにログイン
$ docker login -u="xxx" -p="xxx" quay.io
$ docker push $IMAGE
```

> なお、Tomcatのhelmチャートは何故かHostPortを空けようとするのでOpenShift環境だと権限の関係でPodがデプロイされません。tarをダウンロードして、変更することで修正できます。
> 以下に修正したものをアップロードしましたのでご使用ください。
> https://s3-ap-northeast-1.amazonaws.com/literalice-helm-charts/tomcat-0.1.0.tgz

これで、Operatorのコンテナイメージが作成されコンテナレジストリにPushされました。

### Operatorをインストールする

さて、今回作成したTomcat Operatorは、以下のようなCRが作成、変更、削除されたときにTomcatを指定通りにインストールしたりアンインストールします。

スキーマは、`docker build`時に指定したKINDなどのパラメーターと、helmチャートの`values.yaml`に従い決定されます。

```yaml
apiVersion: apache.org/v1alpha1
kind: Tomcat
metadata:
  name: example-app
spec:
  replicaCount: 2
```

このCRをapplyすれば、TomcatがKubernetes上にプロビジョニングされるというわけですね。

このCRを作成するためには、まずCRDを作成しなければなりません。

サンプルのディレクトリには既に用意されているので、以下コマンドを適用するだけです。

```bash
$ kubectl apply -f crd.yaml
```

```bash
$ kubectl apply -n <operator-namespace> -f rbac.yaml
```

次に、Operatorをインストールします。テンプレートが用意されているので、プレースホルダを先ほどビルドしたOperatorコンテナイメージに置換して使用します。

```bash
$ sed "s|REPLACE_IMAGE|$IMAGE|" operator.yaml.template > operator.yaml
$ kubectl apply -n <operator-namespace> -f operator.yaml
$ kubectl get pods -n playground 
NAME                               READY     STATUS    RESTARTS   AGE
tomcat-operator-79c876b9fc-bc4rv   1/1       Running   0          43s
```

### OperatorでTomcatをインストールする

これで、Tomcatをインストールする準備ができました。

以下のようにCRを適用すると、OperatorがTomcatをインストールしてくれるはずです。

```bash
$ kubectl apply -n <operator-namespace> -f cr.yaml
$ kubectl get pods -n <operator-namespace>
NAME                                                            READY     STATUS    RESTARTS   AGE
example-app-dqbff6a9lkwr0us934kfcb9m8-tomcat-69797999bb-8c977   2/2       Running   0          4m
example-app-dqbff6a9lkwr0us934kfcb9m8-tomcat-69797999bb-zkz8r   2/2       Running   0          4m
tomcat-operator-5f895b8db6-c96wp                                1/1       Running   0          4m
```

### OperatorでTomcatのPodを変更する

このTomcat PodはOperatorで管理されているので、対応するCRをeditすることで設定を変更できます。

```bash
$ kubectl get tomcat -n <operator-namespace>
NAME          AGE
example-app   5h

$ kubectl edit tomcat example-app -n <operator-namespace>
```

```yaml
apiVersion: apache.org/v1alpha1
kind: Tomcat
# ...
spec:
  replicaCount: 2 # ここを1に変更する
# ...
```

Tomcat Podのインスタンス数を確認してみます。

```bash
$ kubectl get deploy -n <operator-namespace>
NAME                                           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
example-app-dqbff6a9lkwr0us934kfcb9m8-tomcat   1         1         1            1           5h
tomcat-operator                                1         1         1            1           5h
```

このように、Podが減少していることが分かります。

Tomcatをアンインストールするには、対応するCRを削除します。

```bash
$ kubectl delete tomcat example-app -n <operator-namespace>

$ kubectl get pods -n <operator-namespace> -w
NAME                                                            READY     STATUS        RESTARTS   AGE
example-app-dqbff6a9lkwr0us934kfcb9m8-tomcat-69797999bb-8c977   2/2       Terminating   0          17m
tomcat-operator-5f895b8db6-c96wp                                1/1       Running       0          17m

$ kubectl get pods -n <operator-namespace>
NAME                               READY     STATUS    RESTARTS   AGE
tomcat-operator-5f895b8db6-c96wp   1/1       Running   0          18m
```

`kubectl delete tomcat` 。すごく直観的で分かりやすいと思いませんか？

### Operator Lifecycle Manager(OLM)でTomcat Operatorを管理する

OLMをKubernetesにインストールしている場合は、Tomcat OperatorをOLMで管理することもできます。

```bash
# インストールしたOperatorを削除する
kubectl delete -f operator.yaml -n <operator-namespace>
kubectl delete -f rbac.yaml -n <operator-namespace>

# CSV(ClusterServiceVersion)をインストールする 
kubectl apply -n <operator-namespace> -f csv.yaml

# OperatorがOLMによってデプロイされることを確認
$ kubectl get pods -n <operator-namespace>
NAME                               READY     STATUS    RESTARTS   AGE
tomcat-operator-5577cd4bb7-hjf8m   1/1       Running   0          2m
```

さて、 OpenShiftの場合、v3.11からOLMの管理画面が付いているので、以下のように画面で動作を確認できます。

{{< figure src="/images/tomcat-operator-olm.png" >}}

せっかくなので、管理画面からTomatをインストールしてみます。

上記画面で、「Create Tomcat Server」を押下します。

作成については、CRのYAMLを画面上でapplyできるだけですね。 `kubectl apply -f cr.yaml`に相当する操作です。

{{< figure src="/images/tomcat-create-cr-olm.png" >}}

画面上でtomcatインスタンスの一覧が確認できます。

{{< figure src="/images/tomcat-operator-instances.png" >}}

コマンドラインからも、Tomcatが起動していることが確認できます。

```bash
$ kubectl get pods -n <operator-namespace>
example-e1iz7hyn42auym80tr7vw4kmo-tomcat-796b9485d7-rmj95   2/2       Running   0          8m
tomcat-operator-5577cd4bb7-hjf8m                            1/1       Running   0          19m
```

## まとめ

helmチャートがあれば、`Helm App Operator Kit`でOperatorを生成できます。

Operatorによるサービスのインストール、設定は、エンドユーザーにとって`kubectl`で操作が完結する、複雑な設定が隠蔽されシンプルなCRで設定できるなどのメリットがあります。

もし、今helmチャートで管理しているサービスがあるなら、Operator化を検討してはいかがでしょうか。
