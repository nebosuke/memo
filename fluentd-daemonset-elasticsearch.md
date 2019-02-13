# KubernetesのDaemonSetとしてfluentd(公式イメージ)をデプロイしてElasticSearchにすべてのログを転送する

## 巷にあふれる情報からの要点
- image のタグは、debian のものを指定しないとうまく動かない。alpine のものを選んではダメ
- FLUENT_UID に 0 を指定しないと、ノードに記録されているログファイルが開けない

