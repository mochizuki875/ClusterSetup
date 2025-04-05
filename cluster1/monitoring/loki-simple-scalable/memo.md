# Loki Simple Scalable
## Object Storageの作成
Rook/CephのObject Storageを利用した。
以下の手順でRGWをShared Poolsとして作成し、loki-bucketというObjectBucketClaimおよびObjectBucketを作成する
-> Shared Poolsを使用することで複数のObjectStoreを作れるようにする
-> データ保存用に`CephBlockPool`の作成が必要
https://rook.io/docs/rook/latest-release/Storage-Configuration/Object-Storage-RGW/object-storage/#create-local-object-stores-with-shared-pools


アクセス情報を取得
->Lokiの`values.yaml`で`.loki.storage.s3`に設定する
```bash
kubectl -n monitoring get cm loki-bucket -o jsonpath='{.data.BUCKET_HOST}'
kubectl -n monitoring get cm loki-bucket -o jsonpath='{.data.BUCKET_PORT}'
kubectl -n monitoring get cm loki-bucket -o jsonpath='{.data.BUCKET_NAME}'
kubectl -n monitoring get secret loki-bucket -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 --decode
kubectl -n monitoring get secret loki-bucket -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 --decode
```

## Lokiインストール

Simple Scalable
https://grafana.com/docs/loki/latest/get-started/deployment-modes/#simple-scalable
https://github.com/grafana/loki/tree/main/production/helm/loki

-> Helmでインストールする際は`.loki.auth_enabled: false`を設定しないと認可に失敗して401エラーになるので注意

server returned HTTP status 401 Unauthorized (401): no org id 
https://github.com/grafana/loki/issues/7081



こんな感じでObject Storageにログデータが保存される。

indexデータ
```bash
[root@s3client /]# s5cmd ls s3://loki-bucket/index/loki_index_19914/
2024/07/17 05:31:49              1694  1721194069-loki-write-2-1721189793104728590.tsdb.gz
2024/07/17 05:31:52              1694  1721194072-loki-write-0-1721189796116772045.tsdb.gz
2024/07/17 05:31:57              1694  1721194077-loki-write-1-1721189792715960759.tsdb.gz
```

チャンクデータ
```bash
[root@s3client /]# s5cmd ls s3://loki-bucket/fake/
                                  DIR  1086d8c2cacd8951/
                                  DIR  1138bd56341835a5/
                                  DIR  19648b286e716390/
                                  ...
```

## promtail

- promtail
  - https://github.com/grafana/helm-charts/tree/main/charts/promtail

`.config.clients`でLokiのURLを指定する。

```yaml
config:
  clients:
  - url: http://loki-gateway/loki/api/v1/push

tolerations:
  - key: node-role.kubernetes.io/master
    operator: Exists
    effect: NoSchedule
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
  - key: dedicated
    operator: Equal
    value: monitoring
    effect: NoSchedule
```

## メトリクス監視
各Serviceでエンドポイントを公開してるのでそれに対するServiceMonitorを定義すれば良い

`kube-prometheus-stack/values.yaml`
```yaml
  additionalServiceMonitors: 
  ...
  # loki-simple-scalable
  - name: "loki-read"
    additionalLabels:
      release: monitoring
    endpoints:
    - port: http-metrics
      path: /metrics
      interval: 10s
    jobLabel: loki-read
    namespaceSelector:
      matchNames:
      - monitoring
    selector:
      matchLabels:
        app.kubernetes.io/name: loki
        app.kubernetes.io/component: read
      matchExpressions:
      - key: prometheus.io/service-monitor
        operator: NotIn
        values:
        - "false"

  - name: "loki-write"
    additionalLabels:
      release: monitoring
    endpoints:
    - port: http-metrics
      path: /metrics
      interval: 10s
    jobLabel: loki-write
    namespaceSelector:
      matchNames:
      - monitoring
    selector:
      matchLabels:
        app.kubernetes.io/name: loki
        app.kubernetes.io/component: write
      matchExpressions:
      - key: prometheus.io/service-monitor
        operator: NotIn
        values:
        - "false"

  - name: "loki-backend"
    additionalLabels:
      release: monitoring
    endpoints:
    - port: http-metrics
      path: /metrics
      interval: 10s
    jobLabel: loki-backend
    namespaceSelector:
      matchNames:
      - monitoring
    selector:
      matchLabels:
        app.kubernetes.io/name: loki
        app.kubernetes.io/component: backend
      matchExpressions:
      - key: prometheus.io/service-monitor
        operator: NotIn
        values:
        - "false"
```

ダッシュボードは以下が用意されているが、一部メトリクスのラベル値をダッシュボードに表示しているものがうまく表示されないので注意
- Loki Metrics Dashboard
  - https://grafana.com/grafana/dashboards/17781-loki-metrics-dashboard/
- How to extract label values from Prometheus metrics in Grafana
  - https://grafana.com/blog/2023/02/23/how-to-extract-label-values-from-prometheus-metrics-in-grafana/


### meta-monitoring-chart
HelmでインストールしたLokiの監視
-> Lokiの監視は今後これを使うらしい。

- meta-monitoring
  - https://grafana.com/docs/loki/latest/setup/install/helm/monitor-and-alert/with-local-monitoring/
- meta-monitoring-chart
  - https://github.com/grafana/meta-monitoring-chart
- Install this chart(Preparation for Local mode)
  - https://github.com/grafana/meta-monitoring-chart/blob/main/docs/installation.md#preparation-for-local-mode


ちょっと試してみたがminioとかインストールされてTooMuchなのと、実はデフォルトでStorageClassの定義が必要だったりとなかなか使いづらそう。

```bash
kubectl create namespace meta

kubectl create secret generic minio -n meta \
 --from-literal=USERNAME=miniouser \
 --from-literal=PASSWORD=password

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install meta-monitoring grafana/meta-monitoring -n meta -f values.yaml 

```