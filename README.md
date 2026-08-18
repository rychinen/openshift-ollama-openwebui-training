# OpenShift ハンズオン：Ollama + Open WebUI

講師デモ用。OpenShift 4.22.32、NVIDIA L4 x 1、Windows Server 2019 + WSL を前提とします。

## 実行

```bash
oc new-project ollama-handson --display-name="OllamaHansOn" --description="OpenShift hands-on: Ollama + Open WebUI"
oc apply -f manifests/01-pvc.yaml
oc apply -f manifests/02-ollama.yaml
oc rollout status deployment/ollama
oc exec -it deploy/ollama -- ollama pull gemma4:e2b
oc create secret generic open-webui-secret --from-literal=WEBUI_SECRET_KEY="$(openssl rand -base64 32)"
oc apply -f manifests/03-open-webui.yaml
oc rollout status deployment/open-webui
oc create route edge --service=open-webui --port=http
```

LVM StorageClass の WaitForFirstConsumer により、PVC 作成直後の Pending は正常です。Ollama Pod 作成後に `ollama-data`、Open WebUI Pod 作成後に `webui-data` が Bound になります。

`latest` / `main` は講師デモ用です。反復実施・運用では固定タグまたは image digest を使用してください。

## クリーンアップ

```bash
oc delete project ollama-handson
oc get project ollama-handson -w
```
