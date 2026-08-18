# 可観測性

```bash
oc get pods -o wide
oc get deploy,svc,pvc,route
oc get events --sort-by=.lastTimestamp
oc describe pod <POD_NAME>
oc logs <POD_NAME> --previous
oc get endpoints
```

Pending、ImagePullBackOff、CrashLoopBackOff、Service Endpoints、Route → Service → Pod を順に調査します。