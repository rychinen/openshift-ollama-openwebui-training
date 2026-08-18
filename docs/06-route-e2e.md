# Route と E2E

```bash
oc create route edge --service=open-webui --port=http
oc get route open-webui
```

ブラウザで Route を開き、初回管理者を作成して `gemma4:e2b` の推論を確認します。Ollama は外部公開しません。