# Ollama GPU デプロイ

```bash
oc apply -f manifests/02-ollama.yaml
oc rollout status deployment/ollama
oc get deploy,pod,svc,pvc
oc get endpoints ollama
oc logs deployment/ollama
```

`nvidia.com/gpu: 1` で L4 を一枚要求します。Service selector と Pod labels は `app: ollama` で一致させます。