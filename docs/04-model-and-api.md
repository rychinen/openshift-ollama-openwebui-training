# モデル取得と API

```bash
oc exec -it deploy/ollama -- ollama pull gemma4:e2b
oc exec -it deploy/ollama -- ollama list
oc port-forward svc/ollama 11434:11434
```

別端末で `curl http://localhost:11434/api/tags` を実行します。port-forward は講師端末の一時確認用です。