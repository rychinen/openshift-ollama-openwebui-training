# 事前準備

```bash
oc login --token=<TOKEN> --server=<API_SERVER>
oc whoami
oc version
oc new-project ollama-handson --display-name="OllamaHansOn" --description="OpenShift hands-on: Ollama + Open WebUI"
oc project ollama-handson
oc get all
oc get pvc,secret,route
oc get events --sort-by=.lastTimestamp
```

Console では Topology、Pods、PVC、Services、Routes を確認します。