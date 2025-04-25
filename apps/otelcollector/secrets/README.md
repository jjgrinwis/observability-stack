# Sealed Secrets

First create a secret.yaml with API key.

```
$ kubectl create secret generic splunk-api-key \
--from-literal=API_KEY=<token>\
--namespace=observability \
--dry-run=client -o yaml > secret.yaml
```

Next seal it
```
$ kubeseal --controller-name sealed-secrets --controller-namespace sealed-secrets --format=yaml < secret.yaml > signalfx-api-secret-sealed.yaml
```
After you commit your changes, ArgoCD will pick them up and update your sealed secret:
