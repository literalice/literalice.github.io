---
title: "minishift addonでIstioを簡単に試す"
date: 2018-10-05T14:01:05+09:00
draft: false
tags: ["istio", "kubernetes", "minishift", "openshift"]
categories: ["Technology"]
comment: true
toc: true
---

## Istio Operator

このOperator全盛時代、IstioもOperatorが開発されています。

https://github.com/Maistra/istio-operator

Istioは、十数個のPodとCRD、RBAC設定などからなる大規模なミドルウェアであり、インストールやアップデート、保守作業も面倒なので、Operatorに保守してもらえれば嬉しいですね。

OpenShiftのIstioが先月9月にTech Previewとなりましたが、ここでもこのOperatorでインストールすることになっています。

https://docs.openshift.com/container-platform/3.10/servicemesh-install/servicemesh-install.html

## minishift addon

{{< figure src="/images/minishift-logo.png" >}}

`minishift`は、OpenShift版の`minikube`とも言うべきツールで、ローカルでOpenShiftを簡単に立ち上げることができます。

https://github.com/minishift/minishift

`minishift`の特長の一つに、addonという仕組みでどんどん機能を追加していくことができる点が挙げられます。

例えば、以下のようにaddonを適用することで、 Eclipse Cheをローカルで立ち上げたクラスタにインストールできます。

```bash
minishift addons apply che
```

さらに、このaddonは後から自分で追加することも可能です。

```bash
minishift addons install <addonのディレクトリ>/
```

上記のように、addonのあるディレクトリを指定して`addons install`コマンドを実行するだけでaddonを追加できます。

addonはコミュニティで開発、サポートされている[リポジトリ](https://github.com/minishift/minishift-addons)があるので、 `git clone`して試してみてください。

```bash
git clone https://github.com/minishift/minishift-addons.git

cd minishift-addons/
minishift addons install add-ons/fabric8
```

## istioをminishiftでインストール

Istioもaddonとしてこのコミュニティリポジトリにホストされています。

https://github.com/minishift/minishift-addons/tree/master/add-ons/istio

このaddonは、minishiftに立ち上げたクラスタに[istio-operator](https://github.com/Maistra/istio-operator)を使ってistioをセットアップするものです。

本記事では、このaddonでIstioを試してみる方法を紹介します。

といっても、上記のようにaddonをapplyするだけです。
ただし、前述のようにistioはPodを多数起動する関係上、メモリを多く使うので、少しデフォルトより積んだ状態でクラスタを起動しておきます。

```bash
minishift start --memory=6GB

#..

Login to server ...
Creating initial project "myproject" ...
Server Information ...
OpenShift server started.

The server is accessible via web console at:
    https://192.168.64.68:8443

You are logged in as:
    User:     developer
    Password: <any value>

To login as administrator:
    oc login -u system:admin
```

### dynamic-admission-controllers addonの適用

起動が終わったら、 istio addonを適用します。

ただし、Podデプロイ時にistioのsidecarコンテナを自動で挿入するためには、Dynamic Admission Controllerを有効化する必要があります。

https://docs.openshift.com/container-platform/3.10/servicemesh-install/servicemesh-install.html#updating-master-configuration

今回はistioctlコマンドでsidecarを自分で挿入するのではなく、便利な自動挿入を使ってみたいのでこのDynamic Admission Controllerも有効化します。

```bash
git clone https://github.com/minishift/minishift-addons.git

cd minishift-addons/
minishift addons install add-ons/dynamic-admission-controllers/
# Addon 'dynamic-admission-controllers' installed

minishift addons apply dynamic-admission-controllers

# -- Applying addon 'dynamic-admission-controllers':
# Enable required admission configs...
# Restart kube-api...
# Dynamic admission controllers add-on successfully applied
```

これで、必要なDynamic Admission Controllerが有効化されました。

### istio addonの適用

次に、istio addonを適用します。

```bash
minishift addons install add-ons/istio/
# Addon 'istio' installed

minishift addons apply istio
# -- Applying addon 'istio':
```

これで、クラスタ上にistio Operatorがインストールされ、そのOperatorによりistioのインストール、必要なCRD、権限の設定が行われ、istioが使用できるようになります。

istioのpodがデプロイされているか確認してみましょう。

```bash
oc get pods -w -n istio-system --as system:admin

NAME                                          READY     STATUS              RESTARTS   AGE
openshift-ansible-istio-installer-job-rm6hn   0/1       ContainerCreating   0          15s
```

istioをインストールするjobが実行され、以下のようにpodがデプロイされることが分かります。

```bash
NAME                                          READY     STATUS      RESTARTS   AGE
elasticsearch-0                               1/1       Running     0          4h
grafana-65db6b47c9-bj2gc                      1/1       Running     0          4h
istio-citadel-84fb7985bf-qsrvs                1/1       Running     0          4h
istio-egressgateway-86f49899c9-fscgb          1/1       Running     0          4h
istio-galley-655c4f9ccd-n66rx                 1/1       Running     0          4h
istio-ingressgateway-8695db5498-5x9gx         1/1       Running     0          4h
istio-pilot-b969499c4-bvcbr                   2/2       Running     0          4h
istio-policy-5455899b66-mzxsb                 2/2       Running     0          4h
istio-sidecar-injector-8975849b4-ghsqz        1/1       Running     0          4h
istio-statsd-prom-bridge-7f44bb5ddb-kbldf     1/1       Running     0          4h
istio-telemetry-584c9ff7f5-7nksl              2/2       Running     0          4h
jaeger-agent-5zjdv                            1/1       Running     0          4h
jaeger-collector-7764fc77b6-p4pln             1/1       Running     0          4h
jaeger-query-5c7fb9878d-z7nf2                 1/1       Running     1          4h
kiali-5dd65695f7-f5hsd                        1/1       Running     0          4h
openshift-ansible-istio-installer-job-rm6hn   0/1       Completed   0          4h
prometheus-84bd4b9796-njpzv                   1/1       Running     0          4h
```

### istioの動作確認

istioがインストールされ、sidecarコンテナがデプロイされることを確認します。

istioのsidecarコンテナが適切にデプロイされるには、sidecarコンテナの動作に必要な権限が割り当てられている必要があります。

そのために、以下設定を行ってください。

```bash
# sidecarコンテナの起動処理で必要な権限を追加
oc adm policy add-scc-to-user privileged -z default -n myproject
```

以下をデプロイして、sidecarコンテナが適切に追加されていることを確認してください。

```yaml
# sample-istio.yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: sleep
      annotations:
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: sleep
        image: tutum/curl
        command: ["/bin/sleep","infinity"]
        imagePullPolicy: IfNotPresent
```

```bash
oc apply -f sample-istio.yml
oc get pods -w

NAME                     READY     STATUS        RESTARTS   AGE
sleep-f845bcb9d-z889d    0/2       Init:0/1      0          3h
sleep-f845bcb9d-z889d   0/2       PodInitializing   0         3h
sleep-f845bcb9d-z889d   2/2       Running   0         3h
```

上記のように、2つのコンテナがデプロイされていれば適切にistioがインストールされています。

## まとめ

この記事では、istioが簡単に試すことができる、minishiftのaddonを紹介しました。

このほか、istioによるservice meshの管理画面であるkialiもインストールされるので、色々といじってみると楽しいと思います。

```bash
minishift openshift service kiali
|--------------|-------|----------|-------------------------------------------------|--------|
|  NAMESPACE   | NAME  | NODEPORT |                    ROUTE-URL                    | WEIGHT |
|--------------|-------|----------|-------------------------------------------------|--------|
| istio-system | kiali |          | https://kiali-istio-system.192.168.64.68.nip.io |        |
|--------------|-------|----------|-------------------------------------------------|--------|

open `minishift openshift service -u kiali` # username/pass : admin/admin
```

{{< figure src="/images/kiali-screenshot-20181005.png" >}}