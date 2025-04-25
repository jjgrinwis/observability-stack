# Fluent Bit and Open Telemetry Collector setup

This little repo will create two ArgoCD applications via Helm:

1. Fluent Bit
2. Opentelemetry Collector

So your k8s should be up and running, including Traefik and ArgoCD!

This version of the repo will create some IngressRoute and some Middleware for basic auth including and will add some sealed secrets.
It is Helm-based, so all the configuration can be found in values.yaml for the different ArgoCD apps. For secrets, we're using sealed secrets, which will be added as ENV vars in the pods.
