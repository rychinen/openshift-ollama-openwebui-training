# 付録：運用・トラブルシューティング早見表

本付録は、OpenShift 上で Ollama と Open WebUI を利用するハンズオンの最終版に添付する参照資料です。通常の手順は本文を優先し、障害時や復習時には本付録を使用してください。

## 付録A：CLI コマンド早見表

### 状態確認

```bash
oc project
oc status
oc get deploy,pod,svc,route,pvc,secret
oc get pods -o wide
oc get events --sort-by=.lastTimestamp
```

| 目的 | コマンド |
|---|---|
| Pod の状態を確認 | `oc get pods` |
| Pod の IP と配置ノードを確認 | `oc get pods -o wide` |
| Deployment の Ready 数を確認 | `oc get deployment` |
| PVC の接続状態を確認 | `oc get pvc` |
| Route Host を確認 | `oc get route` |
| Service の転送先を確認 | `oc get endpoints` |
| Project 内の Event を時系列で確認 | `oc get events --sort-by=.lastTimestamp` |

### 詳細調査

```bash
oc describe pod <POD_NAME>
oc describe deployment <DEPLOYMENT_NAME>
oc describe svc <SERVICE_NAME>
oc describe route <ROUTE_NAME>
oc describe pvc <PVC_NAME>
oc logs <POD_NAME>
oc logs <POD_NAME> --previous
```

- `oc describe`：設定内容と Events を確認します。
- `oc logs`：現在実行中コンテナのアプリケーションログを確認します。
- `oc logs --previous`：再起動前コンテナのログを確認します。CrashLoopBackOff の調査時に使用します。

### Ollama と Open WebUI

```bash
oc exec -it deploy/ollama -- ollama list
oc exec -it deploy/ollama -- ollama pull gemma4:e2b
oc exec -it deploy/open-webui -- sh -c \
  'wget -qO- http://ollama:11434/api/tags || true'
oc logs deployment/ollama
oc logs deployment/open-webui
```

### Route と名前解決

```bash
oc get route open-webui
oc get route open-webui -o jsonpath='{.spec.host}'
oc describe route open-webui
```

Windows のブラウザで Route にアクセスする場合、内部 DNS を利用できない本ラボ環境では、Windows 側の hosts ファイルに Route Host 名と Ingress Router の IP アドレスを登録します。

```text
C:\Windows\System32\drivers\etc\hosts

<INGRESS_ROUTER_IP>  <ROUTE_HOST>
```

> ブラウザが Windows 上で動く場合、WSL の `/etc/hosts` ではなく Windows の hosts ファイルを編集します。hosts ファイルは管理者権限で編集し、不要になったエントリーはラボ終了後に削除します。

### 自己修復と再作成

```bash
oc delete pod <OLLAMA_POD_NAME>
oc get pods -l app=ollama -w
oc get endpoints ollama
oc exec -it deploy/ollama -- ollama list
```

Pod の削除後、Deployment が新しい Pod を作成します。Pod 名と Pod IP は変わる可能性がありますが、Deployment、Service、PVC は維持されます。

---

## 付録B：OpenShift Web Console 早見表

操作は **Administrator** パースペクティブで行います。

| 確認対象 | Web Console の場所 | 主な確認項目 |
|---|---|---|
| Pod | Workloads → Pods | Status、Ready、Events、Logs、YAML、Terminal |
| Deployment | Workloads → Deployments | Desired / Current / Available Replicas、Pod template |
| PVC | Storage → PersistentVolumeClaims | Status、容量、StorageClass、Volume |
| Service | Networking → Services | Selector、Port、Endpoints |
| Route | Networking → Routes | Host、Service、Target Port、TLS |
| Event | Workloads → Pods → 対象 Pod → Events | Warning、直近の失敗、再起動理由 |

### GUI による基本調査順序

```text
1. Workloads → Pods で対象 Pod の Status と Ready を確認
2. 対象 Pod の Events を確認
3. 対象 Pod の Logs を確認
4. Deployment の Replica 数を確認
5. PVC が Bound か確認
6. Service の selector と Endpoints を確認
7. Route の Host、Service、TLS を確認
```

---

## 付録C：トラブルシューティング早見表

| 症状 | 最初に確認するもの | 主な確認コマンド | 代表的な原因 |
|---|---|---|---|
| Pod が Pending | Events、PVC、Node、GPU | `oc describe pod <POD_NAME>` | GPU、CPU、memory、不足。PVC 未接続。Node 制約 |
| PVC が Pending | PVC Events、Pod の状態 | `oc describe pvc <PVC_NAME>` | WaitForFirstConsumer、Node 配置未確定、LVM 容量不足 |
| ImagePullBackOff | Pod Events | `oc describe pod <POD_NAME>` | image 名・tag の誤り、レジストリへの到達性 |
| CrashLoopBackOff | 前回ログと Events | `oc logs <POD_NAME> --previous` | 起動設定、環境変数、権限、volumeMount の誤り |
| Ollama のモデルがない | Ollama PVC、`ollama list` | `oc exec -it deploy/ollama -- ollama list` | モデル未 pull、PVC 未マウント、別の保存先を使用 |
| Open WebUI にモデルが表示されない | Ollama Service と Endpoints | `oc get endpoints ollama` | selector 不一致、`OLLAMA_BASE_URL` の誤り、Ollama 未起動 |
| Route に接続できない | hosts、Route、Service、Endpoints、Pod | `oc get route,svc,endpoints,pod` | hosts 未登録、Route Host の誤り、Service に転送先がない |
| Open WebUI が起動しない | Pod Events、Logs、PVC、Secret | `oc describe pod <POD_NAME>`、`oc logs <POD_NAME>` | PVC、Secret、環境変数、リソース不足 |
| 推論が遅い・失敗する | Ollama Logs、GPU 割り当て、モデル | `oc logs deployment/ollama` | GPU 未割り当て、GPU memory 不足、モデル問題 |

### Route 障害時の確認経路

```text
Windows hosts ファイル
  ↓
Route/open-webui
  ↓
Service/open-webui
  ↓
Endpoints/open-webui
  ↓
Open WebUI Pod
```

### Open WebUI から Ollama への内部通信経路

```text
Open WebUI Pod
  ↓  http://ollama:11434
Service/ollama
  ↓  selector: app=ollama
Endpoints/ollama
  ↓
Ollama Pod
```

Service selector と Pod label が一致しない場合、Endpoints は空になります。Pod が Running でも Service 経由の通信はできません。

---

## 付録D：最終確認チェックリスト

### アプリケーション

- [ ] Ollama Pod が `Running`、`Ready 1/1` である
- [ ] Open WebUI Pod が `Running`、`Ready 1/1` である
- [ ] `ollama-data` PVC が `Bound` である
- [ ] `webui-data` PVC が `Bound` である
- [ ] `oc exec -it deploy/ollama -- ollama list` で `gemma4:e2b` を確認できる
- [ ] Open WebUI で `gemma4:e2b` を選択して推論できる

### ネットワーク

- [ ] `Service/ollama` の selector が Ollama Pod の label と一致する
- [ ] `Endpoints/ollama` に Pod IP と port `11434` が表示される
- [ ] `Service/open-webui` の Endpoints に Pod IP と port `8080` が表示される
- [ ] `Route/open-webui` の Host、Service、Target Port が正しい
- [ ] 作業端末の Windows hosts ファイルに Route Host と Ingress Router IP が登録されている

### 障害対応

- [ ] `oc get`、`oc describe`、`oc logs`、`oc get events` を使い分けられる
- [ ] selector 不一致で Endpoints が空になることを説明できる
- [ ] Route 障害を hosts → Route → Service → Endpoints → Pod の順で調査できる
- [ ] Pod 削除後に Deployment が新しい Pod を作成することを確認した
- [ ] Pod 再作成後も PVC 上の Ollama モデルが残ることを確認した

### ラボ終了時

- [ ] 必要な manifest を Git に保存した
- [ ] ラボ専用 Project を削除する場合、必要なデータ退避を完了した
- [ ] Project を削除する場合、正しい Project 名を確認した
- [ ] 不要になった Windows hosts のエントリーを削除した

---

## 参照する manifest

```text
manifests/
  00-namespace.yaml
  01-storage.yaml
  02-ollama.yaml
  03-open-webui.yaml
  04-route.yaml

exercises/
  01-service-wrong-selector.yaml
```

各環境で使用する Project 名、StorageClass 名、Ingress Router IP、Route Host、コンテナイメージの tag、GPU リソース名は、実環境の値に置き換えてください。
