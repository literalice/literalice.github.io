---
title: "OpenShiftのJenkinsを好きなプラグインをインストールした状態でプロビジョニングする"
date: 2018-10-06T22:22:23+09:00
draft: false
categories: ["Technology"]
tags: ["openshift"]
comment: true
---

OpenShiftに機能の一部として同梱されているJenkinsですが、デプロイ時に環境変数を指定することで好きなJenkinsプラグインを追加することができます。

https://docs.openshift.com/container-platform/3.10/using_images/other_images/jenkins.html#jenkins-environment-variables

指定するのは、以下の二つの環境変数です。

* `OVERRIDE_PV_PLUGINS_WITH_IMAGE_PLUGINS`
  * Jenkinsプラグインが保存されているPVへ、Jenkinsのコンテナからプラグインを上書きコピーするかどうか。
    `false`(デフォルト)の場合、Jenkinsが初回に起動したときしかプラグインがコピーされない。
  * 例： `true`
* `INSTALL_PLUGINS`
  * `OVERRIDE_PV_PLUGINS_WITH_IMAGE_PLUGINS`が有効な場合に、追加でインストールするプラグインをカンマ区切りで指定する
  * 例： `ssh-agent:1.17,slack`

この環境変数を指定するには、JenkinsのDeployment Config設定で、

{{< figure src="/images/jenkins-deploymentconfig-edit.png" >}}

以下のように環境変数を追加します。

{{< figure src="/images/jenkins-deploymentconfig-edit-env.png" >}}

そうすると、起動したJenkinsで指定したプラグインが設定されているはずです。

{{< figure src="/images/jenkins-plugin-installed.png" >}}

Templateなどを用意してやれば、必要なプラグインがインストールされた状態でJenkinsを使ってもらうことができますね。
