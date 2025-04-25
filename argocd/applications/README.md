Just add these argocd apps in argocd namespace via:

```
kubectl apply -f otel-collector-app.yaml -n argocd
kubectl apply -f fluentbit-app.yaml -n argocd
```

It will add two helm managed apps with values.yaml coming from repo.
