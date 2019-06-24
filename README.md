

```
kubectl create secret generic thanos-storage-config --from-file=thanos.yaml=thanos-storage-config.yaml -n default
helm install  --name tst-prom stable/prometheus-operator -f prometheus-operator-thanos-values.yaml  --tiller-namespace=default
kubectl create secret tls -n default thanos-ingress-secret --key ./tst-client.key --cert ./tst-client.cert
kubectl create secret generic -n default thanos-ca-secret --from-file=ca.crt=./cacerts.cer
kubectl apply -f thanos-ingress-rules.yaml -n default
```

