```bash
kubectl delete pods --field-selector status.phase=Pending -A
```

```bash
kubectl delete pods --field-selector status.phase=Succeeded -A
```

```bash
kubectl delete pods --field-selector status.phase=Failed -A
kubectl delete pods --field-selector status.phase=Unknown -A
```