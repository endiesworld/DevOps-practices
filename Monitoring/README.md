## Extract Default Monitoring Stack using Helm
```bash
helm show values prometheus-community/kube-prometheus-stack > Monitoring/prometheus-default-values.yaml
```

Get the actual Grafana password (the correct way)

Since your release name is prometheus-1 and namespace is monitoring, run:
```bash
kubectl -n monitoring get secret | grep grafana

# You should see something like prometheus-1-grafana.
# Now decode the password:

kubectl -n monitoring get secret prometheus-1-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Login with: username: admin and the password printed above.