# Open WebUI

```bash
oc create secret generic open-webui-secret --from-literal=WEBUI_SECRET_KEY="$(openssl rand -base64 32)"
oc apply -f manifests/03-open-webui.yaml
oc rollout status deployment/open-webui
oc exec -it deploy/open-webui -- sh -c 'wget -qO- http://ollama:11434/api/tags || true'
```

Open WebUI は `http://ollama:11434` で内部 Service に接続します。