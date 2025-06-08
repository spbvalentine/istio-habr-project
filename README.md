# Istio habr example project

## Render template
```
helm template . --output-dir output
```

## Install helm chart
```
helm install <release-name> <chart-name> -f values.yaml
```