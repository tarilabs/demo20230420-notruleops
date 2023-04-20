
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
- cannot define variable with dollar prefix `events.$pvc << ...`
- the following in the doc, does not run (fail yaml schema validation, had to use either `msg` or `var`) https://ansible-rulebook.readthedocs.io/en/stable/conditions.html#:~:text=action%3A%0A%20%20debug%3A%0A%20%20%20%20first%3A%20%22%7B%7B%20events.first%20%7D%7D%22
- can't express `from` tried with `event.spec.volumes is selectattr("persistentVolumeClaim.claimName", "==", events.pvc.metadata.name)` but not sure event the dot in the property accessor (first param)