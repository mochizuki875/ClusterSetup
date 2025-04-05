kube-prometheus-stack向けに`--skip-diff-on-install`をつける

```bash
$ helmfile apply --skip-diff-on-install
```

https://fand.jp/technologies/how-to-install-prometheus-operator-with-helm/#%E8%A3%9C%E8%B6%B3%E4%BA%8B%E9%A0%85



monitoring nodeの設定
```bash
$ kubectl label no k8s-cluster1-monitoring01 dedicated=monitoring 
$ kubectl taint node k8s-cluster1-monitoring01 dedicated=monitoring:NoSchedule
```


calicoメトリクス取得用設定
```bash
$ kubectl patch felixconfiguration default --type merge --patch '{"spec":{"prometheusMetricsEnabled": true}}'
$ kubectl patch installation default --type=merge -p '{"spec": {"typhaMetricsPort":9093}}'
```

https://docs.tigera.io/calico/latest/operations/monitor/monitor-component-metrics