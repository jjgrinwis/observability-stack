# ArgoCD apps

After installing argocd via terraform, create two apps.
Just add these argocd apps in argocd namespace via:

```
$ kubectl apply -f otel-collector-app.yaml -n argocd
$ kubectl apply -f fluentbit-app.yaml -n argocd
```
 
It will add two Helm managed apps with values.yaml coming from repo. You should see something like this in your ArgoCD setup:
![image](https://github.com/user-attachments/assets/ca7edaf8-ebb2-4662-91dd-5a064b9bc302)

All config can be found in values.yaml in this repo for the different apps and this inclused some sealed secrets for API keys etc.
