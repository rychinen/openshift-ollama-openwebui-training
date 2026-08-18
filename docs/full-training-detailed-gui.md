# OpenShift ハンズオン詳細版：CLI + GUI 確認

既存資料を変更しない、CLI 操作のあとに OpenShift Web Console で何を見るかを追加したスクロール閲覧版です。

## Console の基本

Project セレクターで `ollama-handson` を選択します。

- **Developer → Topology**：アプリケーションと接続関係を確認
- **Administrator**：Pods、Deployments、PVC、Services、Routes、Events を詳細確認

CLI は正確な状態確認と再現操作、GUI は関連性・状態・ログを視覚的に確認するために使います。

## 1. Project

```bash
oc new-project ollama-handson --display-name="OllamaHansOn" --description="OpenShift hands-on: Ollama + Open WebUI"
oc project ollama-handson
oc get all
```

### GUI

1. Project セレクターで `ollama-handson` を選択
2. Developer → Topology を開く
3. ワークロードがまだないことを確認
4. Administrator → Home → Projects で Project を確認

新規 Project の Topology が空なのは正常です。

## 2. PVC

```bash
oc apply -f manifests/01-pvc.yaml
oc get pvc
oc describe pvc ollama-data
```

LVM StorageClass の `WaitForFirstConsumer` により、作成直後の Pending は正常です。

### GUI

1. Administrator → Storage → PersistentVolumeClaims
2. `ollama-data` と `webui-data` を確認
3. 作成直後は Status が Pending を確認
4. Requested Storage が 30Gi / 10Gi を確認
5. Event で first consumer 待機に相当する表示を確認

Ollama Pod 作成後は `ollama-data`、WebUI Pod 作成後は `webui-data` が Bound になります。

## 3. Ollama

```bash
oc apply -f manifests/02-ollama.yaml
oc rollout status deployment/ollama
oc get deploy,pod,svc,pvc
oc get endpoints ollama
oc logs deployment/ollama
```

### GUI

1. Developer → Topology で `ollama` を確認
2. ワークロードから Deployment、Pod、Service を確認
3. Pod が Running、Ready 1/1 を確認
4. Pod の Logs と Events を確認
5. PVC `ollama-data` が Bound を確認
6. Administrator → Networking → Services → `ollama` で selector `app=ollama` を確認

`get endpoints` は Service が転送先 Pod を発見できているかの確認です。

## 4. モデルと API

```bash
oc exec -it deploy/ollama -- ollama pull gemma4:e2b
oc exec -it deploy/ollama -- ollama list
oc port-forward svc/ollama 11434:11434
```

別 WSL ターミナル：

```bash
curl -s http://localhost:11434/api/tags | jq .
```

### GUI

1. Administrator → Workloads → Pods → Ollama Pod
2. Terminal を開き `ollama list` を実行
3. `gemma4:e2b` を確認
4. Logs で pull / 推論ログを確認
5. PVC が Bound のままであることを確認

GUI Terminal は `oc exec` と同様に Pod 内コマンドを実行する手段です。port-forward は WSL CLI で実行します。

## 5. Open WebUI

```bash
oc create secret generic open-webui-secret --from-literal=WEBUI_SECRET_KEY="$(openssl rand -base64 32)"
oc apply -f manifests/03-open-webui.yaml
oc rollout status deployment/open-webui
oc get pods,svc,pvc
oc get endpoints open-webui
```

### GUI

1. Administrator → Workloads → Secrets で `open-webui-secret` の存在を確認
2. Secret の実値は画面共有しない
3. Developer → Topology で `open-webui` を確認
4. Pod の Running / Ready、Logs、Events を確認
5. Pod YAML / Environment で `OLLAMA_BASE_URL=http://ollama:11434` を確認
6. PVC `webui-data` が Bound を確認
7. Service `open-webui` の selector と target port を確認

```bash
oc exec -it deploy/open-webui -- sh -c 'wget -qO- http://ollama:11434/api/tags || true'
```

モデル一覧が返れば内部通信は成功です。Pod 内の `localhost` は Open WebUI 自身を指します。

## 6. Route と E2E

```bash
oc create route edge --service=open-webui --port=http
oc get route open-webui
ROUTE_HOST=$(oc get route open-webui -o jsonpath='{.spec.host}')
echo "https://${ROUTE_HOST}"
```

### GUI

1. Developer → Topology → `open-webui` → Open URL
2. または Administrator → Networking → Routes → `open-webui`
3. Host、Service、Target Port、edge TLS を確認
4. ブラウザで初回管理者を作成
5. `gemma4:e2b` を選んで推論
6. Ollama / WebUI の Logs を確認

Ollama は Route を作らず内部 Service に維持します。

## 7. 障害対応

```bash
oc get pods -o wide
oc get deploy,svc,pvc,route
oc get events --sort-by=.lastTimestamp
oc describe pod <POD_NAME>
oc logs <POD_NAME> --previous
oc get endpoints
```

### GUI の調査順序

1. Administrator → Pods の Status
2. Pod の Events
3. Pod の Logs、CrashLoopBackOff なら Previous logs
4. Deployment の Replicas / Available Replicas
5. PVC Status
6. Service selector / Endpoints
7. Route の Service、Port、Host、TLS

障害演習：

```bash
oc apply -f exercises/01-service-wrong-selector.yaml
oc get endpoints ollama
```

GUI の Service `ollama` で Pod label と異なる selector、空の Endpoints を確認します。復旧：

```bash
oc apply -f manifests/02-ollama.yaml
```

## 8. 自己修復

```bash
oc delete pod <OLLAMA_POD_NAME>
oc get pods -l app=ollama -w
oc exec -it deploy/ollama -- ollama list
```

### GUI

1. Pods で旧 Pod の Terminating と新 Pod 作成を観察
2. 新 Pod の名前と IP の変化を確認
3. Deployment の Desired / Current / Available が 1 へ戻ることを確認
4. `ollama-data` PVC が Bound のままであることを確認
5. Terminal でモデルが残ることを確認
6. Open WebUI Pod も削除し、Route 再アクセス後に状態保持を確認

## 付録：クリーンアップ

```bash
oc project ollama-handson
oc delete route open-webui --ignore-not-found
oc delete deployment open-webui ollama --ignore-not-found
oc delete service open-webui ollama --ignore-not-found
oc delete secret open-webui-secret --ignore-not-found
oc delete pvc webui-data ollama-data --ignore-not-found
```

PVC 削除はモデルと WebUI 状態を削除します。Project ごと削除する場合は `oc delete project ollama-handson` を実行します。
