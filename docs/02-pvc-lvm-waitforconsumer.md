# LVM PVC と WaitForFirstConsumer

LVM StorageClass では PVC 作成直後の Pending は正常です。PVC を使う Pod のスケジュール時に PV が provisioning され Bound になります。

| 操作 | ollama-data | webui-data |
|---|---|---|
| PVC 作成後 | Pending | Pending |
| Ollama Pod 作成後 | Bound | Pending |
| Open WebUI Pod 作成後 | Bound | Bound |

```bash
oc get pod,pvc -o wide
oc describe pvc ollama-data
oc get events --sort-by=.lastTimestamp
```