---
title: "Deployment の API Version が古いと revisionHistoryLimit のデフォルト値が 2147483647 に設定される"
date: 2019-12-17T23:45:16+09:00
draft: false
aliases:
  - /posts/deployment-revisionhistorylimit-default/
tags: [Kubernetes]
categories: [Kubernetes]
---

小ネタです。

<!--more-->

ある日 `kubectl get replicaset` とかを叩いた際にやたら履歴の数が多いことに気づきました。

Deployment に紐づく ReplicaSet の履歴の保持数を設定するためのフィールドとして `revisionHistoryLimit` というものが存在します。
デフォルト値は確か 10 だったはずと公式ドキュメントを確認しに行きましたが、そこには

> You can set `.spec.revisionHistoryLimit` field in a Deployment to specify how many old ReplicaSets for this Deployment you want to retain. The rest will be garbage-collected in the background. By default, it is 10.

https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#clean-up-policy

と記載されています。

でも実際に `kubectl get deployment foo -o yaml` をしてみると実際に設定されていたのは `revisionHistoryLimit: 2147483647` でした。
うーん多い。こういうときの便利コマンドとして `kubectl explain` というコマンドが存在します。

`kubectl explain deployment.spec.revisionHistoryLimit` のような感じで各 Object のフィールドのドキュメントを表示することができます。

```text
$ kubectl explain deployment.spec.revisionHistoryLimit
KIND:     Deployment
VERSION:  extensions/v1beta1

FIELD:    revisionHistoryLimit <integer>

DESCRIPTION:
     The number of old ReplicaSets to retain to allow rollback. This is a
     pointer to distinguish between explicit zero and not specified. This is set
     to the max value of int32 (i.e. 2147483647) by default, which means
     "retaining all old RelicaSets".
```

> This is set to the max value of int32 (i.e. 2147483647) by default, which means "retaining all old RelicaSets".

`2147483647` が設定されているのは正しそうですが、公式ドキュメントの記載とは食い違います。
ここで注目してほしいのが `VERSION:  extensions/v1beta1` です。古い。

```text
$ kubectl explain -h
List the fields for supported resources

 This command describes the fields associated with each supported API resource. Fields are identified via a simple
JSONPath identifier:

  <type>.<fieldName>[.<fieldName>]

 Add the --recursive flag to display all of the fields at once without descriptions. Information about each field is
retrieved from the server in OpenAPI format.

Use "kubectl api-resources" for a complete list of supported resources.

Examples:
  # Get the documentation of the resource and its fields
  kubectl explain pods

  # Get the documentation of a specific field of a resource
  kubectl explain pods.spec.containers

Options:
      --api-version='': Get different explanations for particular API version
      --recursive=false: Print the fields of fields (Currently only 1 level deep)

Usage:
  kubectl explain RESOURCE [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).
```

Help を見ると `--api-version` というオプションで指定ができそうです。
`apps/v1` の方を explain してみます。

```text
$ kubectl explain deployment.spec.revisionHistoryLimit --api-version apps/v1
KIND:     Deployment
VERSION:  apps/v1

FIELD:    revisionHistoryLimit <integer>

DESCRIPTION:
     The number of old ReplicaSets to retain to allow rollback. This is a
     pointer to distinguish between explicit zero and not specified. Defaults to
     10.
```

はい、完全に理解しました。
バージョンは追従しましょう。
Deployment の API Version が `apps/v1beta2` になった際にデフォルトが 10 になったみたいです。

`kubectl explain` は意外と便利なのでおすすめです。バージョンは追従しましょう (大事なことなので二回言いました)
