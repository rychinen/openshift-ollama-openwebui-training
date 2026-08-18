# OpenShift ハンズオン詳細版：Ollama + Open WebUI

既存資料を置き換えない、コマンドの目的・期待結果・確認ポイントを加えたスクロール閲覧版です。

## 全体像

Ollama は LLM 推論 API、Open WebUI はブラウザ画面です。Open WebUI だけを Route で外部公開し、Ollama は Project 内の Service として使います。モデルと設定は PVC に保存します。

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

### 目的

ハンズオンのリソースを他のアプリケーションから分離します。Project は Namespace を基盤にしたリソース・権限分離単位です。

```bash
oc new-project ollama-handson --display-name="OllamaHansOn" --description="OpenShift hands-on: Ollama + Open WebUI"
```

- `oc new-project` は Project を作成します。
- `ollama-handson` は YAML と CLI で使用する内部名です。
- 表示名は Console 上の人間向け名称です。

```bash
oc project ollama-handson
oc get all
oc get pvc,secret,route
oc get events --sort-by=.lastTimestamp
```

新規 Project では `oc get all` が空でも正常です。PVC、Secret、Route は `oc get all` に含まれないため個別に取得します。

## 2. PVC

### 目的

Pod の削除・再作成後も、Ollama モデルと Open WebUI のデータを残します。

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

- PVC はアプリケーションが要求する永続ストレージです。
- StorageClass は未指定で、default LVM StorageClass を使います。
- RWO は通常単一ノードから read/write するモードです。

```bash
oc apply -f manifests/01-pvc.yaml
oc get pvc
oc describe pvc ollama-data
```

LVM StorageClass の `WaitForFirstConsumer` により、作成直後の `Pending` は正常です。Pod がスケジュールされた時点で PV が provisioning されます。

| 操作 | ollama-data | webui-data |
|---|---|---|
| PVC 作成後 | Pending | Pending |
| Ollama Pod 作成後 | Bound | Pending |
| Open WebUI Pod 作成後 | Bound | Bound |

## 3. Ollama

### 目的

Ollama を GPU 付き Pod として起動し、Open WebUI が利用する内部 API を Service で提供します。

```bash
oc apply -f manifests/02-ollama.yaml
oc rollout status deployment/ollama
oc get deploy,pod,svc,pvc
oc get endpoints ollama
oc logs deployment/ollama
```

- `oc apply` は YAML の期待状態をクラスタに反映します。
- `rollout status` は Deployment が Ready になるまで待ちます。
- `endpoints` は Service が実際に転送できる Pod を表示します。
- Ollama は `nvidia.com/gpu: 1` で L4 一枚を要求します。
- Service selector、Deployment selector、Pod labels は `app: ollama` で一致する必要があります。

## 4. モデルと API

```bash
oc exec -it deploy/ollama -- ollama pull gemma4:e2b
oc exec -it deploy/ollama -- ollama list
```

- `oc exec` は稼働 Pod 内でコマンドを実行します。
- `ollama pull` はモデルを PVC に保存します。
- `ollama list` は取得済みモデルを確認します。

ターミナル A：

```bash
oc port-forward svc/ollama 11434:11434
```

これは WSL の `localhost:11434` を Service の 11434 へ一時転送します。ターミナル B：

```bash
curl -s http://localhost:11434/api/tags | jq .
curl -s http://localhost:11434/api/generate -H 'Content-Type: application/json' -d '{"model":"gemma4:e2b","prompt":"OpenShiftを一文で説明してください。","stream":false}' | jq .
```

`/api/tags` はモデル一覧、`/api/generate` は推論結果を返します。

## 5. Open WebUI

### 目的

Open WebUI を配置し、内部 Service の Ollama へ接続します。

```bash
oc create secret generic open-webui-secret --from-literal=WEBUI_SECRET_KEY="$(openssl rand -base64 32)"
oc apply -f manifests/03-open-webui.yaml
oc rollout status deployment/open-webui
oc get pods,svc,pvc
oc get endpoints open-webui
oc logs deployment/open-webui
```

- Secret の実値は Git に保存しません。
- `OLLAMA_BASE_URL=http://ollama:11434` は Service DNS を指定します。
- Pod 内の `localhost` は Open WebUI 自身であり Ollama ではありません。
- `webui-data` PVC は WebUI Pod の作成後に Bound になります。

```bash
oc exec -it deploy/open-webui -- sh -c 'wget -qO- http://ollama:11434/api/tags || true'
```

この結果でモデル一覧が返れば、Open WebUI Pod から Ollama Service への内部通信が成功しています。

## 6. Route と E2E

```bash
oc create route edge --service=open-webui --port=http
oc get route open-webui
oc describe route open-webui
ROUTE_HOST=$(oc get route open-webui -o jsonpath='{.spec.host}')
echo "https://${ROUTE_HOST}"
```

- edge Route は OpenShift Router で TLS を終端します。
- Route は Open WebUI だけに作成します。Ollama は外部公開しません。
- ブラウザで URL を開き、初回管理者を作り、`gemma4:e2b` を選択して推論します。

## 7. 可観測性と障害対応

```bash
oc get pods -o wide
oc get deploy,svc,pvc,route
oc get events --sort-by=.lastTimestamp
oc describe pod <POD_NAME>
oc logs <POD_NAME>
oc logs <POD_NAME> --previous
oc get endpoints
```

| 症状 | まず何を見るか | 主な原因 |
|---|---|---|
| Pending | describe pod、Events、PVC | GPU / CPU / memory、PVC、ノード制約 |
| ImagePullBackOff | Pod Events | image、tag、レジストリ到達性 |
| CrashLoopBackOff | logs --previous | 権限、環境変数、mountPath |
| WebUI にモデルがない | Ollama Endpoints、OLLAMA_BASE_URL | selector、Service、通信 |
| Route で接続不可 | Route → Service → Endpoints → Pod | port / selector 不一致、Pod 未 Ready |

障害演習：

```bash
oc apply -f exercises/01-service-wrong-selector.yaml
oc get endpoints ollama
oc describe svc ollama
oc apply -f manifests/02-ollama.yaml
```

誤った selector では Endpoints が空になります。最後の apply は正常 Service へ復旧します。

## 8. 自己修復と永続化

```bash
oc get pods -l app=ollama
oc delete pod <OLLAMA_POD_NAME>
oc get pods -l app=ollama -w
oc exec -it deploy/ollama -- ollama list
```

- Pod 名と IP は変わります。
- Deployment は `replicas: 1` を維持するため新しい Pod を作ります。
- Service 名は変わりません。
- PVC 上の `gemma4:e2b` は残るため再 pull は不要です。

Open WebUI Pod も削除して、Route 再アクセス後にデータが残ることを確認します。

```bash
oc delete pod <OPEN_WEBUI_POD_NAME>
oc rollout status deployment/open-webui
```

## 付録：WSL とクリーンアップ

WSL 内で Linux 用 `oc`、`curl`、`jq`、`openssl` を使い、Windows の `oc.exe` と混在させません。

```bash
sudo apt-get update && sudo apt-get install -y curl jq openssl
oc version --client
```

クリーンアップでは、PVC 削除がモデル・WebUI データを消すことに注意します。

```bash
oc project ollama-handson
oc delete route open-webui --ignore-not-found
oc delete deployment open-webui ollama --ignore-not-found
oc delete service open-webui ollama --ignore-not-found
oc delete secret open-webui-secret --ignore-not-found
oc delete pvc webui-data ollama-data --ignore-not-found
```

Project 全体を消す場合は `oc delete project ollama-handson` を実行します。
