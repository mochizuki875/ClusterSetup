# ArgoCD(helmfile)
以下Helm Chartをベースに`helmfile`を作成。

Argo CD Chart
https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd

インストール
```bash
$ helmfile apply
```

初期PW取得
```bash
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d ; echo
```