# 自己修復と永続化

```bash
oc delete pod <OLLAMA_POD_NAME>
oc get pods -l app=ollama -w
oc exec -it deploy/ollama -- ollama list
```

Deployment が Pod を再作成し、PVC 上のモデルは残ります。Open WebUI Pod も削除して状態保持を確認します。