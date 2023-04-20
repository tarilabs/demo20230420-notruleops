
# Installation used

```
ansible-galaxy collection install sabre1041.eda
pip install requests
```

# Demo environment

Docker up and running, then:

```sh
minikube start
```

# Relax the ResourceQuota limits Deployment PENDING

```sh
ansible-rulebook -i inventory.yml --rulebook demo1.rulebook.yml
# in the other terminal
kubectl apply -f k8s/quota-pod.yml && kubectl apply -f k8s/quota-pod-deployment.yml
kubectl delete -f k8s/quota-pod.yml && kubectl delete -f k8s/quota-pod-deployment.yml
```

# FixthePersistentVolumeClaim

```sh
ansible-rulebook -i inventory.yml --rulebook demo1.rulebook.yml
# in the other terminal
kubectl apply -f k8s/pv-claim.yml && kubectl apply -f k8s/pvc-deployment.yml
kubectl delete -f k8s/pv-claim.yml && kubectl delete -f k8s/pvc-deployment.yml
```

# Investigation "limitations" found

- No way to trigger a level-trigger update
- No "live reload" ?
- `kubectl delete -f` triggers a MODIFIED event (before a DELETED event)
- cannot define variable with dollar prefix `events.$pvc << ...`
- the following in the doc, does not run (fail yaml schema validation, had to use either `msg` or `var`) https://ansible-rulebook.readthedocs.io/en/stable/conditions.html#:~:text=action%3A%0A%20%20debug%3A%0A%20%20%20%20first%3A%20%22%7B%7B%20events.first%20%7D%7D%22
- See below about `from`.<br/>
Can't express `from`; for instance I need the equivalent of:

```yaml
# DRL version
  - given: Pod
    as: $pod
    having:
    - status.phase == "Pending"
  - given: Volume
    having:
    - persistentVolumeClaim!.claimName == $pvc.metadata.name
    from: $pod.spec.volumes
```

tried with 

```yaml
# rulebook version
          - |
            events.pod << (
              (event.type == "ADDED" or event.type == "MODIFIED")
              and
              event.resource.kind == "Pod"
              and
              event.resource.status.phase == "Pending"
              and
              event.spec.volumes is selectattr("persistentVolumeClaim.claimName", "==", events.pvc.metadata.name)
            )
```

but not working (`Expected end of text, found '<'  (at char 11), (line:1, col:12)`).
Also, not sure event the dot in the property accessor (first param I need access the `claimName` inside of `persistentVolumeClaim` composite attribute of `volumes` in the `Pod`).

- No rule analysis wrong.<br/>
Defining a condition such as:
```yaml
      condition:
        all:
          - |
            events.d << (
              (event.type == "ADDED" or event.type == "MODIFIED")
              and
              event.resource.kind == "Deployment"
              and
              status.conditions is defined
            )
```

hangs forever without mentioning the alleged error `ansible_rulebook.exception.InvalidIdentifierException: Invalid identifier : status.conditions Should start with event., events.,fact., facts. or vars`.

- Cannot have all the rules in 1 rulebook (having rules `Relax the ResourceQuota limits Deployment PENDING` and `Fix the PersistentVolumeClaim Pod PENDING` and all their related sabre sources, does not work)
- I want to define 2 separate grouping.<br/>
For instance
```yaml
# DRL version
- name: Relax the ResourceQuota limits Deployment PENDING
  when:
  - given: Deployment
    as: $d
  - given: DeploymentCondition
    having:
    - type == "Available"
    - status == "False"
    from: $d.status.conditions
  - given: DeploymentCondition
    having:
    - message contains "exceeded quota"
    from: $d.status.conditions
```

but in rulebook format is not grouped "for the same Object in collection"

```yaml
    - name: Relax the ResourceQuota limits Deployment PENDING
      condition:
        all:
          - |
            events.d << (
              (event.type == "ADDED" or event.type == "MODIFIED")
              and
              event.resource.kind == "Deployment"
              and
              event.resource.status.conditions is selectattr("type", "==", "Available")
              and
              event.resource.status.conditions is selectattr("status", "==", "False")
              and
              event.resource.status.conditions is selectattr("message", "search", "exceeded quota")
            )
```