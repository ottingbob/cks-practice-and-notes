### Examples

How to create a pod with a command:
```bash

$ k run pod --image busybox --command -o yaml --dry-run=client > pod.yaml -- sh -c 'sleep 1d'

```

