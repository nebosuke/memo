# KubernetesのDaemonSetとしてfluentd(公式イメージ)をデプロイしてElasticSearchにすべてのログを転送する

このメモは2019年2月時点での情報で作成しました。

[fluentd-daemonset-elasticsearch.yaml](fluentd-daemonset-elasticsearch.yaml)

## 巷にあふれる情報からの要点
- image のタグは、debian のものを指定しないとうまく動かない。alpine のものを選んではダメ
- FLUENT_UID に 0 を指定しないと、ノードに記録されているログファイルが開けない

## 変更すべき箇所
- FLUENT_ELASTICSEARCH_HOST

## デプロイ方法
```
kubectl apply -f fluentd-daemonset-elasticsearch.yaml
```

## 動作を確認
```
$ kubectl get daemonset
NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   3         3         3       3            3           <none>          19h

$ kubectl get po
NAME                                  READY   STATUS    RESTARTS   AGE
fluentd-elasticsearch-jlfl2           1/1     Running   0          19h
fluentd-elasticsearch-svv49           1/1     Running   0          19h
fluentd-elasticsearch-wlsfk           1/1     Running   0          19h
```

ノードの数だけfluentdのpodが立ち上がっている。
