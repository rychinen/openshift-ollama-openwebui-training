# OpenShift ハンズオン完全版：Ollama + Open WebUI

> 講師デモ用スクロール閲覧版。OpenShift 4.22.32、NVIDIA L4 x 1、Windows Server 2019 + WSL。

## 確定値

| 項目 | 値 |
|---|---|
| Project | `ollama-handson` / `OllamaHansOn` |
| Storage | default LVM StorageClass、RWO |
| PVC | Ollama 30Gi、WebUI 10Gi |
| Ollama | `ollama/ollama:latest`、`gemma4:e2b`、GPU 1枚 |
| Open WebUI | `ghcr.io/open-webui/open-webui:main` |

> `latest` / `main` は講師デモ用。反復実施・運用は固定 tag または digest を使用する。

## アーキテクチャ

```text
Browser -> Route/open-webui -> Service/open-webui -> Pod/open-webui
                                                    |
                                                    v http://ollama:11434
                                              Service/ollama -> Pod/ollama (L4 x 1)
                                                                    |
                                                                    v
                                                            PVC/ollama-data
Pod/open-webui -> PVC/webui-data
```

## 1. Project

```bash
oc new-project ollama-handson --display-name="OllamaHansOn" --description="OpenShift hands-on: Ollama + Open WebUI"
oc project ollama-handson
oc get all
oc get pvc,secret,route
oc get events --sort-by=.lastTimestamp
```

## 2. PVC

LVM StorageClass の `WaitForFirstConsumer` により PVC 作成直後の `Pending` は正常。Pod のスケジュール時に PV が provisioning され `Bound` になる。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 30Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webui-data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
```

```bash
oc apply -f manifests/01-pvc.yaml
oc get pvc -w
```

| 操作 | ollama-data | webui-data |
|---|---|---|
| PVC 作成後 | Pending | Pending |
| Ollama Pod 作成後 | Bound | Pending |
| Open WebUI Pod 作成後 | Bound | Bound |

## 3. Ollama

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
spec:
  replicas: 1
  selector:
    matchLabels: {app: ollama}
  template:
    metadata:
      labels: {app: ollama}
    spec:
      containers:
      - name: ollama
        image: ollama/ollama:latest
        ports:
        - name: http
          containerPort: 11434
        volumeMounts:
        - name: ollama-data
          mountPath: /root/.ollama
        resources:
          requests: {cpu: "1", memory: 8Gi, nvidia.com/gpu: "1"}
          limits: {cpu: "2", memory: 12Gi, nvidia.com/gpu: "1"}
      volumes:
      - name: ollama-data
        persistentVolumeClaim: {claimName: ollama-data}
---
apiVersion: v1
kind: Service
metadata:
  name: ollama
spec:
  selector: {app: ollama}
  ports:
  - name: http
    port: 11434
    targetPort: http
```

```bash
oc apply -f manifests/02-ollama.yaml
oc rollout status deployment/ollama
oc get deploy,pod,svc,pvc
oc get endpoints ollama
oc logs deployment/ollama
oc exec -it deploy/ollama -- ollama pull gemma4:e2b
oc exec -it deploy/ollama -- ollama list
```

## 4. API

```bash
oc port-forward svc/ollama 11434:11434
curl -s http://localhost:11434/api/tags | jq .
curl -s http://localhost:11434/api/generate -H 'Content-Type: application/json' -d '{"model":"gemma4:e2b","prompt":"OpenShiftを一文で説明してください。","stream":false}' | jq .
```

## 5. Open WebUI

```bash
oc create secret generic open-webui-secret --from-literal=WEBUI_SECRET_KEY="$(openssl rand -base64 32)"
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: open-webui
spec:
  replicas: 1
  selector:
    matchLabels: {app: open-webui}
  template:
    metadata:
      labels: {app: open-webui}
    spec:
      containers:
      - name: open-webui
        image: ghcr.io/open-webui/open-webui:main
        ports:
        - name: http
          containerPort: 8080
        env:
        - name: OLLAMA_BASE_URL
          value: http://ollama:11434
        - name: WEBUI_SECRET_KEY
          valueFrom:
            secretKeyRef: {name: open-webui-secret, key: WEBUI_SECRET_KEY}
        volumeMounts:
        - name: webui-data
          mountPath: /app/backend/data
      volumes:
      - name: webui-data
        persistentVolumeClaim: {claimName: webui-data}
---
apiVersion: v1
kind: Service
metadata:
  name: open-webui
spec:
  selector: {app: open-webui}
  ports:
  - name: http
    port: 8080
    targetPort: http
```

```bash
oc apply -f manifests/03-open-webui.yaml
oc rollout status deployment/open-webui
oc exec -it deploy/open-webui -- sh -c 'wget -qO- http://ollama:11434/api/tags || true'
```

`localhost` は Open WebUI Pod 自身。Ollama 接続は `http://ollama:11434`。

## 6. Route

```bash
oc create route edge --service=open-webui --port=http
ROUTE_HOST=$(oc get route open-webui -o jsonpath='{.spec.host}')
echo "https://${ROUTE_HOST}"
```

ブラウザで初回管理者を作成し、`gemma4:e2b` の推論を確認する。Ollama は外部公開しない。

## 7. 可観測性

```bash
oc get pods -o wide
oc get deploy,svc,pvc,route
oc get events --sort-by=.lastTimestamp
oc describe pod <POD_NAME>
oc logs <POD_NAME> --previous
oc get endpoints
```

| 症状 | 初動 |
|---|---|
| Pending | describe pod、GPU / CPU / memory、PVC、Events |
| ImagePullBackOff | image、tag、Events、レジストリ到達性 |
| CrashLoopBackOff | logs --previous、権限、環境変数、mountPath |
| WebUI にモデルがない | OLLAMA_BASE_URL、Service、Endpoints |
| Route 接続失敗 | Route → Service → Endpoints → Pod |

## 8. 自己修復とクリーンアップ

```bash
oc delete pod <OLLAMA_POD_NAME>
oc get pods -l app=ollama -w
oc exec -it deploy/ollama -- ollama list
```

Pod は再作成され、PVC 上のモデルは残る。

```bash
oc project ollama-handson
oc delete route open-webui --ignore-not-found
oc delete deployment open-webui ollama --ignore-not-found
oc delete service open-webui ollama --ignore-not-found
oc delete secret open-webui-secret --ignore-not-found
oc delete pvc webui-data ollama-data --ignore-not-found
```

Project ごと削除する場合は `oc delete project ollama-handson`。

## 付録：WSL

WSL 内で Linux 用 `oc`、`curl`、`jq`、`openssl` を使用し、Windows の `oc.exe` と混在させない。

```powershell
wsl --status
wsl -l -v
wsl
```

```bash
sudo apt-get update && sudo apt-get install -y curl jq openssl
oc version --client
```
