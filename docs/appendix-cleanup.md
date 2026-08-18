# クリーンアップ

PVC を削除するとモデルと WebUI データが失われます。

```bash
oc project ollama-handson
oc delete route open-webui --ignore-not-found
oc delete deployment open-webui ollama --ignore-not-found
oc delete service open-webui ollama --ignore-not-found
oc delete secret open-webui-secret --ignore-not-found
oc delete pvc webui-data ollama-data --ignore-not-found
```

Project ごと削除する場合：`oc delete project ollama-handson`