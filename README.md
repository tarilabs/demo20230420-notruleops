
# Installation used

```
ansible-galaxy collection install sabre1041.eda
pip install requests
```

# Demo environment

Docker

```
minikube start
```

# FixthePersistentVolumeClaim.rulebook

# Investigation "limitations" found

- No way to trigger a level-trigger update
- `kubectl delete -f` triggers a MODIFIED event (before a DELETED event)
