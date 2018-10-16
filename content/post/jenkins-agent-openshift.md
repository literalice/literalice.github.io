---
title: "OpenShiftのJenkinsでカスタムしたコンテナをagentとして使用する"
date: 2018-10-16T23:00:00+09:00
draft: false
categories: ["Technology"]
tags: ["jenkins", "openshift"]
comment: true
---

OpenShiftに組み込まれたJenkinsは、パイプラインビルドを開始したときにコンテナをOpenShift上に起動し、その上でパイプラインを実行します。

このようなJenkinsからビルドするために起動、破棄されるコンテナをAgentコンテナとかSlaveコンテナとか呼ぶようです。

このAgentとして使われるコンテナイメージは簡単に追加、変更できるのですが、方法がいまいち分かりにくいというか目立たない気がするので紹介させてください。

ここでは、`hugo`を使って静的なサイトをOpenShiftのJenkinsでビルドする想定で、`hugo`がインストールされた以下のコンテナイメージをJenkinsのagentとして使用します。

https://github.com/literalice/agent-hugo/blob/master/Dockerfile

```docker
# We are basing our builder image on openshift base-centos7 image
FROM openshift/jenkins-slave-base-centos7

# Inform users who's the maintainer of this builder image
MAINTAINER Masatoshi Hayashi <literalice@monochromeroad.com>

# Install the required software
ADD daftaupe-hugo-epel-7.repo /etc/yum.repos.d/

RUN yum -y install hugo && \
    yum -y clean all

RUN hugo version

USER 1001
```

## ImageStream

OpenShiftは、内部で使用するイメージの管理を `ImageStream` というAPIオブジェクトで行っています。

[OpenShiftのImageStream](https://nekop.hatenablog.com/entry/2017/12/17/152353)

また、OpenShiftとJenkinsの設定には、以下の二つのプラグインが使われます。

* [kubernetes-plugin](https://github.com/jenkinsci/kubernetes-plugin)
  * KubernetesのPodをJenkinsのagentとして起動できるようにする
* [openshift-jenkins-sync-plugin](https://github.com/openshift/jenkins-sync-plugin)
  * OpenShiftのビルドとJenkinsのジョブとビルドを同期する。OpenShift上でビルドが作られて実行されたときにJenkins上でジョブが作られてビルドされる、など

Jenkinsのagentに関して言えば、`openshift-jenkins-sync-plugin`のおかげで以下のことが実現されています。

1. OpenShift上で`role=jenkins-slave`というラベルが付けられた`ImageStream`が作成されると、
2. `kubernetes-plugin`で、この`ImageStream`のコンテナイメージが使われるように設定される

つまり、OpenShiftでagentとして使いたいイメージを指す`ImageStream`オブジェクトを作って`role=jenkins-slave`というラベルを付ければ、JenkinsのAgentとして使えるようになるということですね。

どうでもいいんですが、Jenkinsのネーミングガイドラインで`slave`を`agent`にする、というのをどこかで聞いた気がするんですが、現時点ではネーミングが混ざってて混乱しますね。

## 使用したいコンテナイメージを参照するImageStreamを作成する

では、さっそくJenkinsから使うagentイメージを追加してみましょう。

OpenShift内でビルドしてもいいのですが、面倒なので既存のイメージからこれから`ImageStream`を作りましょう。

[quay.io/literalice/agent-hugo](https://quay.io/repository/literalice/agent-hugo)にイメージを挙げているのでこれを使うことにします。

```bash
$ oc project myproject # プロジェクトがない場合は作成する oc new-project myproject
$ oc import-image agent-hugo --from=quay.io/literalice/agent-hugo --confirm

The import completed successfully.

Name:			agent-hugo
Namespace:		playground
Created:		Less than a second ago
Labels:			<none>
Annotations:		openshift.io/image.dockerRepositoryCheck=2018-10-09T09:21:24Z
Docker Pull Spec:	docker-registry.default.svc:5000/playground/agent-hugo
...
```

これで、`quay.io/literalice/agent-hugo`を示す`ImageStream`ができました。

## ImageStreamにラベルを付ける

このイメージがJenkinsのagentとして使われるよう、ラベル付けします。

```bash
$ oc label is agent-hugo "role=jenkins-slave"
imagestream.image.openshift.io/agent-hugo labeled
```

## コンテナイメージをagentにしてJenkinsビルドする

上記で準備した`ImageStream`を使ってJenkinsビルドを行います。

```yaml
kind: "BuildConfig"
apiVersion: "v1"
metadata:
  name: "hugo-sample-pipeline"
spec:
  strategy:
    type: JenkinsPipeline
    jenkinsPipelineStrategy:
      jenkinsfile: |-
        pipeline {
          agent {
            node {
              label 'agent-hugo' 
            }
          }
          stages {
            stage('version') {
              steps {
                script {
                  sh "hugo version"
                }
              }
            }
          }
        }
```

`pipeline.agent.node.label`に、使用する`ImageStream`名を指定します。

上記を、`buildconfig.yml`という名前で保存して適用します。

```bash
$ oc apply -f buildconfig.yml
$ oc start-build hugo-sample-pipeline
```

ここで、作成されるPodを確認してみてください。

```bash
$ oc get pods -w
NAME               READY     STATUS    RESTARTS   AGE
agent-hugo-dpp9p   1/1       Running   0          4m
jenkins-1-5spgx    1/1       Running   0          15m
```

Jenkinsのコンテナとagentである`agent-hugo`のコンテナ、両方が立ち上がっていることが分かります。

## 結果確認

では、以下コマンドでJenkisのURLを確認し、アクセスしてビルド結果を確認してみましょう。

```bash
$ oc get route jenkins --template='{{ .spec.host }}'
$ open http://$(oc get route jenkins --template='{{ .spec.host }}')
```

{{< figure src="/images/jenkins-hugo-pipeline.png" >}}

無事、ビルドできているようです。

## まとめ

OpenShiftのJenkinsで利用するagentのコンテナは簡単に追加できます。

個人的には、agentを追加してビルド環境を追加するよりも、OpenShiftの`s2i`に投げるほうがキャッシュの仕組みなどカプセル化度が高く好きですが、  
Jenkinsのビルド環境もお手軽に追加できると嬉しいケースもありそうですので紹介してみました。
