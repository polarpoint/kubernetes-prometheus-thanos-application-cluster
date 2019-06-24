

Install Prometheus Operator in each app cluster

```


kubectl create secret generic thanos-storage-config --from-file=thanos.yaml=thanos-storage-config.yaml -n default
```

Ensure the name below uses the env (dev in this example) format matches the domain in the ingress rules.

```
helm install  --name dev-prom stable/prometheus-operator -f prometheus-operator-thanos-values.yaml  --tiller-namespace=default
```

```
kubectl create secret tls -n default thanos-ingress-secret --key ./tst-client.key --cert ./tst-client.cert
kubectl create secret generic -n default thanos-ca-secret --from-file=ca.crt=./cacerts.cer
```

Edit the thanos-ingress-rules to ensure it matches the CNAME setup
```
  - host: prom.dev-polarpoint.local 
```

```
kubectl apply -f thanos-ingress-rules.yaml -n default
```
