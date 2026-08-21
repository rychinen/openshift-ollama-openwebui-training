# OpenShift ハンズオン：Docker Compose から Ollama + Open WebUI を OpenShift へ

> **対象**：Docker / Docker Compose は理解していて、OpenShift は初めての受講者  
> **形式**：講師デモ中心。受講者は WSL から追操作  
> **想定時間**：標準 4〜5 時間  
> **クライアント端末**：Windows Server 2019 + WSL（Linux 版 `oc` 導入済み）  
> **OpenShift**：4.22.32  
> **GPU**：NVIDIA L4 × 1  
> **OpenShift Project**：`ollama-handson`  
> **ログインユーザー**：`kubeadmin`  
> **モデル**：`gemma4:e2b`

## この研修のゴール

受講者が、Docker Compose で動く Ollama + Open WebUI を足場として、OpenShift で必要なリソースと運用方法を理解することを目標とします。

最終的に、以下を説明・実行できる状態を目指します。

- Docker Compose の Service、Volume、Network、Port 公開を OpenShift のリソースへ対応付けられる
- GUI と CLI の両方で OpenShift クラスタへログインできる
- Project、Deployment、Pod、Service、PVC、Route、Secret を確認できる
- Ollama を GPU 付きの内部 API として展開できる
- Open WebUI を Route で外部公開できる
- PVC の `Pending` と `Bound` を LVM の `WaitForFirstConsumer` と関連付けて説明できるå
- Events、Logs、Endpoints、GUI を使って基本障害を切り分けできる
- Pod を削除しても Deployment が再作成し、PVC のデータが残ることを確認できる

## この教材の前提（重要）

| 項目 | 本ラボでの扱い |
|---|---|
| Web Console のパースペクティブ | **Administrator パースペクティブのみ**を使用します。Developer パースペクティブは表示されない前提で記載しています |
| ログインユーザー | `kubeadmin`（Token 方式、`oc login -u kubeadmin` の両方を紹介） |
| Docker 側 Ollama のホストポート | **11435**（OpenShift の `oc port-forward` と競合させないため） |
| OpenShift 側 Ollama の API 確認 | `oc port-forward svc/ollama 11434:11434` → `http://localhost:11434` |
| 作業端末の名前解決 | 内部 DNS が無いため、**Windows の hosts ファイル**へ Route Host と Ingress Router IP を登録 |
| Ollama の外部公開 | 行いません。Route は Open WebUI のみ |
| StorageClass | default の LVM StorageClass（`WaitForFirstConsumer`） |

## 全体アーキテクチャ

### Docker Compose 側（WSL / CPU 実行）

```text
ブラウザ
  |
  | http://localhost:3000
  v
Open WebUI コンテナ
  |
  | http://ollama:11434
  v
Ollama コンテナ
  |
  +-- named volume: ollama-data

WSL
  |
  | http://localhost:11435   (ports: "11435:11434")
  v
Ollama コンテナ
```

### OpenShift 側（GPU 実行）

```text
ブラウザ
  |
  | HTTPS（hosts ファイルで名前解決）
  v
Route/open-webui  （edge TLS）
  |
  v
Service/open-webui:8080
  |
  v
Pod/open-webui  --- PVC/webui-data
  |
  | http://ollama:11434
  v
Service/ollama:11434
  |
  v
Pod/ollama + NVIDIA L4  --- PVC/ollama-data
```

### Docker と OpenShift の対応

| Docker Compose | OpenShift | 本ラボでの例 |
|---|---|---|
| Compose Service | Deployment が管理する Pod | `ollama`、`open-webui` |
| コンテナ | Pod 内コンテナ | Ollama / Open WebUI |
| `ports:` | Service + Route | `open-webui` |
| Docker network の DNS | Service DNS | `http://ollama:11434` |
| named volume | PVC + `volumeMount` | `ollama-data`、`webui-data` |
| `docker compose exec` | `oc exec` / GUI Terminal | `ollama list` |
| `docker compose logs` | `oc logs` / GUI Logs | Pod Logs |
| Compose の再作成 | Deployment の自己修復 | Pod 削除後の再作成 |
| 環境変数 | 環境変数 + Secret | `WEBUI_SECRET_KEY` |

## 章構成

| 章 | 内容 | 目安 |
|---|---|---:|
| 第0章 | ログインと OpenShift GUI の概要 | 20〜30 分 |
| 第1章 | Docker Compose で Ollama + Open WebUI を確認する | 25〜35 分 |
| 第2章 | OpenShift Project の作成と基本操作 | 20〜30 分 |
| 第3章 | 永続ストレージ（PVC）と LVM StorageClass | 25〜35 分 |
| 第4章 | Ollama を OpenShift にデプロイする（GPU） | 35〜45 分 |
| 第5章 | Gemma 4 モデル取得と Ollama API 確認 | 25〜35 分 |
| 第6章 | Open WebUI のデプロイと Ollama への内部接続 | 35〜45 分 |
| 第7章 | Route による外部公開と E2E 確認（hosts 設定含む） | 25〜35 分 |
| 第8章 | 可観測性と基本的な障害対応（selector 障害演習） | 35〜45 分 |
| 第9章 | 自己修復とデータ永続化の確認 | 20〜30 分 |
| 第10章 | 後片付けとラボの振り返り | 15〜20 分 |
| 付録 | CLI / GUI / トラブルシューティング / 最終チェック | 参照用 |

## リポジトリ構成

```text
training/openshift-ollama-openwebui/
  ├─ README.md
  └─ full-training-final.md    ← 本ドキュメント（正本）

training/manifests/
  ├─ 01-pvc.yaml
  ├─ 02-ollama.yaml
  └─ 03-open-webui.yaml

exercises/
  └─ 01-service-wrong-selector.yaml

training/docker/
  └─ compose.yaml
```

> 各環境で使用する Project 名、StorageClass 名、Ingress Router IP、Route Host、コンテナイメージの tag、GPU リソース名は、実環境の値に置き換えてください。

***

***

# 第0章：ログインと OpenShift GUI の概要

> 所要時間：20〜30 分  
> 前提：Windows Server 2019 上で WSL が利用可能であり、WSL 内に Linux 版 `oc` が導入済みであること。

## 目的

OpenShift Console と WSL 上の `oc` の両方に `kubeadmin` でログインし、GUI と CLI を使い分ける準備をします。

また、まだ Project を作成しない状態で Console を一巡し、以後の演習で作成する Deployment、Pod、PVC、Service、Route がどこに表示されるかを把握します。

## 期待される結果

この章の完了時点で、以下を満たします。

- ブラウザから `kubeadmin` で OpenShift Console にログインできる
- Console の **Copy login command** の場所を確認できる
- WSL で `oc login` を実行できる
- `oc whoami` が `kubeadmin` を返す
- `oc version` で Client Version と Server Version を確認できる
- Administrator パースペクティブを標準画面として使用できる
- Project セレクター、Workloads、Storage、Networking の画面位置を把握している

## 講師向け導入説明

最初に、以下を伝えます。

> OpenShift は GUI だけでも操作できますが、再現可能な操作、YAML の適用、正確な状態確認、障害解析では CLI が重要です。  
> この研修では、CLI でリソースを作り、GUI でその状態とつながりを確認する流れを繰り返します。

> 現時点ではハンズオン用 Project はまだ作りません。画面が空であることを確認し、後から作ったリソースがどこへ現れるかを見ていきます。

## 0-1. OpenShift Console へログイン

ブラウザで OpenShift Console URL を開きます。

ログイン画面では、以下を使用します。

```text
ユーザー名: kubeadmin
パスワード: 講師が当日配布する値
```

### GUI で確認すること

ログイン後、画面右上にログインユーザーとして `kubeadmin` が表示されていることを確認します。

### 注意

- `kubeadmin` のパスワードは教材、Git、画面キャプチャへ保存しません
- パスワードをチャット、共有画面、コマンド履歴へ不用意に表示しません
- Console URL、Token、認証情報は当日配布の情報として扱います

## 0-2. Copy login command を確認

Console 右上のユーザーメニューから、以下を実行します。

```text
kubeadmin
  ↓
Copy login command
```

必要に応じて、認証画面で `kubeadmin` とパスワードを再入力します。

認証後に、次のような Token を含むコマンドが表示されます。

```bash
oc login --token=<TOKEN> --server=<API_SERVER>
```

このコマンドをコピーします。

### 講師が説明すること

- `oc` は OpenShift CLI です
- `oc login` は API Server へ認証する操作です
- Token は CLI から API を操作するための認証情報です
- Token はパスワードと同様に扱い、Git や教材には書きません

## 0-3. WSL から CLI ログイン

WSL を開き、Copy login command で取得したコマンドを貼り付けます。

```bash
oc login --token=<TOKEN> --server=<API_SERVER>
```

ログイン後、以下を実行します。

```bash
oc whoami
oc whoami --show-server
oc version
```

### コマンドの意味

| コマンド | 目的 | 期待結果 |
|---|---|---|
| `oc whoami` | 現在のログインユーザーを確認する | `kubeadmin` |
| `oc whoami --show-server` | 接続先 API Server を確認する | OpenShift API URL |
| `oc version` | CLI とクラスタのバージョンを確認する | Client / Server Version |

### 期待出力の例

```text
$ oc whoami
kubeadmin
```

```text
$ oc whoami --show-server
https://api.<cluster-domain>:6443
```

### この時点での確認

```bash
oc status
```

`oc status` では、現在の Project や基本的な接続状態を確認できます。

## 0-4. GUI を一巡する

この時点では、演習用 Project はまだ作成しません。GUI の配置と用途を短く紹介します。

### この研修で使うパースペクティブ

この研修では **Administrator パースペクティブ**を標準として使用します。

> 補足：環境によっては Developer パースペクティブや Topology が有効な場合もあります。ただし本研修では、クラスタ設定に左右されないよう、操作・確認手順をすべて Administrator パースペクティブに統一します。

### Administrator パースペクティブ

左上のパースペクティブ切替が表示される場合は、**Administrator** を選択します。

#### Project セレクター

画面上部の Project セレクターを確認します。

後続の章で `ollama-handson` Project を作成し、ここから選択します。

```text
Project
  └─ ollama-handson
```

#### Workloads

```text
Administrator
  └─ Workloads
      ├─ Deployments
      └─ Pods
```

後続の章で確認する内容です。

- Ollama Deployment
- Open WebUI Deployment
- Ollama Pod
- Open WebUI Pod
- Pod の Status
- Pod の Events
- Pod の Logs
- Pod の YAML
- Pod Terminal

#### Storage

```text
Administrator
  └─ Storage
      └─ PersistentVolumeClaims
```

後続の章で確認する内容です。

- `ollama-data`
- `webui-data`
- `Pending`
- `Bound`
- 要求容量
- StorageClass
- Events

#### Networking

```text
Administrator
  └─ Networking
      ├─ Services
      └─ Routes
```

後続の章で確認する内容です。

- `Service/ollama`
- `Service/open-webui`
- `Route/open-webui`
- Service selector
- Endpoints
- Route Host
- edge TLS termination

## CLI と GUI の使い分け

| 操作・確認 | 主に使う手段 | 理由 |
|---|---|---|
| リソース作成・削除 | CLI | 手順として再現できる |
| YAML の適用 | CLI | Git 管理するマニフェストを使える |
| 正確な一覧・絞り込み | CLI | `oc get`、label selector、watch が使える |
| Events・詳細調査 | CLI + GUI | `describe` と画面表示を対応付けられる |
| リソース間の関係 | GUI | Workloads / Networking / Storage の各画面で対応付けられる |
| Logs / Terminal | CLI を基本、GUI を補助 | 両方の操作を学べる |
| 初学者への構成説明 | GUI | Resource 間の関係を見せやすい |

## 章の確認チェック

### CLI

- [ ] WSL 内で `oc login` を実行した
- [ ] `oc whoami` が `kubeadmin` を返した
- [ ] `oc whoami --show-server` で API Server を確認した
- [ ] `oc version` を実行した

### GUI

- [ ] `kubeadmin` で Console にログインした
- [ ] Copy login command の場所を確認した
- [ ] Administrator パースペクティブを選択した
- [ ] Workloads、Storage、Networking の画面を確認した
- [ ] Project セレクターの位置を確認した

### 理解

- [ ] GUI と CLI を併用する理由を説明できる
- [ ] Token を Git や教材に保存してはいけない理由を説明できる
- [ ] Project を作成する前は、今回のアプリケーションが GUI に表示されないことを理解した

***

この第0章を基準に、次は **第1章「Docker Compose の比較環境」** を同じ粒度で作成していく想定です。
***

## 補足：`oc login -u kubeadmin` でログインする

Copy login command で取得する Token 方式に加えて、ユーザー名を明示してパスワードを対話入力する方式も利用できます。

```bash
oc login -u kubeadmin https://<API_SERVER>:6443
```

実行すると、パスワードの入力を求められます。

```text
Authentication required for https://<API_SERVER>:6443 (openshift)
Username: kubeadmin
Password:
Login successful.
```

| 要素 | 意味 |
|---|---|
| `oc login` | OpenShift クラスタへログインする |
| `-u kubeadmin` | ログインユーザーを指定する |
| `https://<API_SERVER>:6443` | API Server の URL |

注意点です。

- API Server は TLS を使うため、URL は `http://` ではなく `https://` を使います
- 自己署名証明書などで警告が出る場合、研修環境では講師の指示に従います
- パスワードはコマンド履歴に残さないため、`-p` オプションではなく対話入力を使います
- ログイン後は `oc whoami` で `kubeadmin` を確認します

```bash
oc whoami
oc whoami --show-server
```

> Token 方式は有効期限があるため再取得が必要になります。ユーザー名・パスワード方式は、研修中に何度もログインし直す場合に便利です。

***

# 第1章：Docker Compose で Ollama + Open WebUI を確認する

> 所要時間：25〜35 分  
> 前提：WSL 内で Docker Engine / Docker Compose が利用できること  
> GPU：**使用しない**。この章は CPU 実行の Docker 環境を、後続の OpenShift GPU 環境と比較するためのものです。

## 目的

WSL 上の Docker Compose で Ollama と Open WebUI を起動し、Docker の Service 名、コンテナ間通信、named volume、ポート公開を確認します。

後続の OpenShift 演習で扱う Deployment、Service、PVC、Route の役割を、Docker Compose の構成と対応付けられるようにします。

## 期待される結果

この章の完了時点で、以下を満たします。

- Docker Compose で `ollama` と `open-webui` の二つのコンテナを起動できる
- ブラウザで Open WebUI を開ける
- Open WebUI コンテナが `http://ollama:11434` で Ollama に接続できる
- Docker Compose の Service 名 `ollama` が、コンテナ間通信の DNS 名になることを確認できる
- `ollama-data` と `webui-data` の named volume を確認できる
- Docker の構成を、後続の OpenShift の Deployment、Service、PVC、Route と対応付けられる

Docker Compose では、同じ Compose プロジェクトの各 Service が既定ネットワークへ接続され、他のコンテナから Service 名で名前解決されます。そのため Open WebUI からは `ollama:11434` を接続先として使えます。

***

## 1-1. Docker 環境を確認する

まず、WSL 上で Docker が使えることを確認します。

```bash
docker version
docker compose version
docker info
```

### コマンドの意味

| コマンド | 目的 | 期待結果 |
|---|---|---|
| `docker version` | Docker Client / Server の動作確認 | Client と Server の情報が表示される |
| `docker compose version` | Compose plugin の確認 | Docker Compose のバージョンが表示される |
| `docker info` | Docker Engine の状態確認 | Storage Driver、コンテナ数などが表示される |

### 講師が説明すること

> OpenShift の演習はクラスタ上で実行しますが、この章では WSL 内の Docker を使います。  
> Docker 側は CPU 実行であり、OpenShift 側では L4 GPU を要求します。  
> 同じアプリケーション構成を別のプラットフォームで動かし、必要なリソースの違いを比較します。

## 1-2. Docker Compose ファイルを読む

作業用ディレクトリを作成します。

```bash
mkdir -p ~/ollama-docker-demo
cd ~/ollama-docker-demo
```

以下を `compose.yaml` として保存します。

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    ports:
      - "11435:11434"
    volumes:
      - ollama-data:/root/.ollama

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "3000:8080"
    environment:
      OLLAMA_BASE_URL: http://ollama:11434
    volumes:
      - webui-data:/app/backend/data
    depends_on:
      - ollama

volumes:
  ollama-data:
  webui-data:
```

> 講師デモでは `latest` と `main` を使用します。反復実施、受講者配布、運用環境では、検証済みの固定タグまたは image digest に置き換えます。

### YAML の読み方

| Docker Compose の要素 | この構成での意味 | OpenShift 側の対応 |
|---|---|---|
| `services.ollama` | Ollama コンテナの定義 | `Deployment/ollama` |
| `services.open-webui` | Open WebUI コンテナの定義 | `Deployment/open-webui` |
| `ports` | ホストからコンテナへのポート公開 | Service / Route |
| `OLLAMA_BASE_URL` | Open WebUI の Ollama 接続先 | 同じ環境変数を利用 |
| `ollama-data` | Ollama モデルを保存する named volume | `PVC/ollama-data` |
| `webui-data` | WebUI データを保存する named volume | `PVC/webui-data` |
| `depends_on` | 起動順序の補助 | OpenShift では直接の対応なし |

### 特に重要な説明

```yaml
OLLAMA_BASE_URL: http://ollama:11434
```

この `ollama` はコンテナ名やホスト名を手作業で指定したものではなく、Docker Compose の **Service 名**です。

Compose は既定ネットワークを作成し、各 Service 名を内部 DNS へ登録します。Open WebUI コンテナは `ollama` を名前解決し、Ollama コンテナへ接続します。

この考え方は、OpenShift の以下の構成と対応します。

```text
Docker Compose:
  http://ollama:11434
       |
       +-- Compose Service 名

OpenShift:
  http://ollama:11434
       |
       +-- Service/ollama の DNS 名
```

***

## 1-3. コンテナを起動する

以下を実行します。

```bash
docker compose up -d
docker compose ps
```

### コマンドの意味

| コマンド | 目的 | 期待結果 |
|---|---|---|
| `docker compose up -d` | コンテナをバックグラウンドで起動する | Ollama と Open WebUI が作成・起動される |
| `docker compose ps` | Compose 管理下のコンテナを確認する | 二つの Service が表示される |

### 期待出力のイメージ

```text
NAME                         SERVICE      STATUS
ollama-docker-demo-ollama-1  ollama       running
ollama-docker-demo-open-webui-1
                             open-webui   running
```

コンテナの起動直後は Open WebUI が初期化中の場合があります。`running` にならない、またはブラウザで開けない場合は、ログを確認します。

```bash
docker compose logs ollama
docker compose logs open-webui
```

ログを追跡する場合は、以下を使います。

```bash
docker compose logs -f ollama
docker compose logs -f open-webui
```

`-f` は新しいログを継続して表示するオプションです。

***

## 1-4. Docker の named volume を確認する

以下を実行します。

```bash
docker volume ls
```

Compose プロジェクト名が付いた、以下に相当する volume が表示されます。

```text
ollama-docker-demo_ollama-data
ollama-docker-demo_webui-data
```

詳細を確認します。

```bash
docker volume inspect ollama-docker-demo_ollama-data
docker volume inspect ollama-docker-demo_webui-data
```

### 講師が説明すること

> Docker の named volume は、コンテナを削除・再作成してもデータを残すために使います。  
> OpenShift では、同じ目的を PVC で実現します。

```text
Docker:
  named volume
    ├─ ollama-data
    └─ webui-data

OpenShift:
  PersistentVolumeClaim
    ├─ ollama-data
    └─ webui-data
```

***

## 1-5. Ollama のモデルを取得する

まず Ollama コンテナ内でモデルを取得します。

```bash
docker compose exec ollama ollama pull gemma4:e2b
```

取得済みモデルを確認します。

```bash
docker compose exec ollama ollama list
```

### コマンドの意味

| コマンド | 目的 |
|---|---|
| `docker compose exec` | Compose の Service に対応するコンテナ内でコマンドを実行する |
| `ollama pull gemma4:e2b` | モデルを取得する |
| `ollama list` | コンテナ内で利用可能なモデルを表示する |

### 講師が説明すること

> Docker 側では GPU を使わず CPU で実行します。  
> `gemma4:e2b` のモデル取得や推論は、OpenShift の L4 GPU 環境より時間がかかる可能性があります。  
> この章の目的は性能評価ではなく、Docker と OpenShift の構成比較です。

モデルは次の named volume に保存されます。

```text
ollama-data
  └─ /root/.ollama
```

OpenShift 側でも同じコンテナ内パスへ PVC をマウントします。

```text
PVC/ollama-data
  └─ /root/.ollama
```

***

## 1-6. Ollama API を Docker 側で確認する

Compose ファイルでは、Ollama API のコンテナポート 11434 を、WSL ホストの **11435** へ公開しています。

```yaml
ports:
  - "11435:11434"
```

この表記は `ホスト側ポート:コンテナ側ポート` です。

> **ホスト側を 11435 にする理由**  
> 後続の OpenShift 章では、`oc port-forward svc/ollama 11434:11434` を使って WSL の `localhost:11434` から OpenShift 側の Ollama を確認します。Docker 側がホストの 11434 を占有していると、この port-forward がポート競合で失敗します。  
> Docker 側のホストポートを 11435 にしておくことで、Docker と OpenShift を**同時に起動したまま**比較できます。

| 用途 | 接続先 |
|---|---|
| Docker Ollama を WSL から確認 | `http://localhost:11435` |
| Docker Open WebUI から Docker Ollama | `http://ollama:11434` |
| OpenShift Ollama を WSL から確認 | `oc port-forward` 後の `http://localhost:11434` |
| OpenShift Open WebUI から OpenShift Ollama | `http://ollama:11434` |

重要なのは、Docker Compose 内部の接続先 `OLLAMA_BASE_URL: http://ollama:11434` は変更しないことです。これは Compose ネットワーク内の通信であり、コンテナ側ポート 11434 を使います。

そのため WSL から以下を実行できます。

```bash
curl -s http://localhost:11435/api/tags | jq .
```

モデル一覧に `gemma4:e2b` が表示されることを確認します。

推論 API を確認します。

```bash
curl -s http://localhost:11435/api/generate \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "gemma4:e2b",
    "prompt": "Docker Composeを一文で説明してください。",
    "stream": false
  }' | jq .
```

### 講師が説明すること

> Docker 側では、ホスト側の `localhost:11435` へ接続しています。  
> これは Docker の `ports` で公開した経路です。  
> OpenShift 側では外部公開ではなく、`oc port-forward` を使って一時的に同じ確認を行います。

```text
Docker:
  WSL localhost:11435
      |
      +-- ports: "11435:11434"
      |
      +-- Ollama コンテナ

OpenShift:
  WSL localhost:11434
      |
      +-- oc port-forward
      |
      +-- Service/ollama
      |
      +-- Ollama Pod
```

***

## 1-7. Open WebUI をブラウザで確認する

ブラウザで以下を開きます。

```text
http://localhost:3000
```

初回は Open WebUI の管理者ユーザー作成画面が表示されます。

### 確認すること

- Open WebUI のログインまたは初回登録画面が開く
- モデル一覧に `gemma4:e2b` が表示される
- モデルを選択できる
- 短いプロンプトに応答する

### 期待結果

Open WebUI が Ollama に接続できていれば、モデル一覧に `gemma4:e2b` が表示されます。

Open WebUI の Docker 構成でも、永続データを `/app/backend/data` へ保存する volume mount が使われます。

***

## 1-8. Docker 側の内部通信を確認する

Open WebUI コンテナから Ollama Service 名を使って API を確認します。

```bash
docker compose exec open-webui sh -c \
  'wget -qO- http://ollama:11434/api/tags || true'
```

### このコマンドで確認していること

```text
Open WebUI コンテナ
  |
  | http://ollama:11434
  v
Docker Compose の内部 DNS
  |
  v
Ollama コンテナ
```

ここで `localhost` を使わない点が重要です。

```text
Open WebUI コンテナ内の localhost
  |
  +-- Open WebUI コンテナ自身

ollama
  |
  +-- Docker Compose 内の Ollama Service
```

これは OpenShift でも同じです。

```text
Open WebUI Pod 内の localhost
  |
  +-- Open WebUI Pod 自身

http://ollama:11434
  |
  +-- Service/ollama
```

***

## Docker と OpenShift の対比まとめ

| 確認項目 | Docker Compose | OpenShift |
|---|---|---|
| アプリ起動 | `docker compose up -d` | `oc apply -f` |
| 状態確認 | `docker compose ps` | `oc get pod` |
| ログ確認 | `docker compose logs` | `oc logs` |
| コンテナ内実行 | `docker compose exec` | `oc exec` |
| コンテナ間通信 | `http://ollama:11434` | `http://ollama:11434` |
| ホストからの API 確認 | `http://localhost:11435` | `oc port-forward` 後の `http://localhost:11434` |
| データ保存 | named volume | PVC |
| 外部公開 | `ports` | Service + Route |
| 再作成 | `docker compose up -d` | Deployment が自動再作成 |
| GPU | この章では未使用 | L4 を 1 枚要求 |

***

## 章の確認チェック

### Docker 操作

- [ ] `docker compose up -d` を実行した
- [ ] `docker compose ps` で二つの Service を確認した
- [ ] `docker compose logs` でログを確認した
- [ ] `docker compose exec ollama -- ollama list` を実行した
- [ ] `docker volume ls` で named volume を確認した

### 接続確認

- [ ] WSL の `localhost:11435` から Ollama API を確認した
- [ ] ブラウザで `http://localhost:3000` を開いた
- [ ] Open WebUI から `gemma4:e2b` を確認した
- [ ] Open WebUI コンテナから `http://ollama:11434` へ接続できた

### 理解

- [ ] Docker Compose の Service 名が内部 DNS 名になることを説明できる
- [ ] Docker の named volume と OpenShift PVC の対応を説明できる
- [ ] コンテナ内の `localhost` がそのコンテナ自身を指すことを説明できる
- [ ] Docker のポート公開と OpenShift Route の役割の違いを説明できる

***

# 第2章：OpenShift Project の作成と基本操作

> 所要時間：20〜30 分  
> 前提：`kubeadmin` で GUI と `oc` にログイン済みであること  
> この章から、OpenShift 上で実際にリソースを作成します。

## 目的

ハンズオン専用の OpenShift Project を作成し、以後の Ollama、Open WebUI、PVC、Service、Route を一つの作業・権限分離単位へまとめます。

Project を作成する前と後で、CLI と GUI の見え方がどう変わるかを確認します。

## 期待される結果

この章の完了時点で、以下を満たします。

- `ollama-handson` Project を作成できる
- Console 上では表示名 `OllamaHansOn` を確認できる
- CLI の現在 Project が `ollama-handson` になる
- `oc get all`、`oc get pvc,secret,route` が空に近い状態であることを確認できる
- GUI の Project セレクターから `ollama-handson` を選択できる
- Workloads に今回のワークロードがまだないことを確認できる

OpenShift Project は Kubernetes Namespace を拡張した概念で、ユーザーがリソースを分離・管理する単位です。

***

## 2-1. Project を作成する

以下を WSL 上で実行します。

```bash
oc new-project ollama-handson \
  --display-name="OllamaHansOn" \
  --description="OpenShift hands-on: Ollama + Open WebUI"
```

### コマンドの意味

| 要素 | 意味 |
|---|---|
| `oc new-project` | OpenShift Project を作成する |
| `ollama-handson` | CLI、YAML、リソース URL で利用する内部名 |
| `--display-name` | Console に表示する人間向けの名称 |
| `--description` | Project の説明 |

`oc new-project` は、Project 名に加えて表示名と説明を指定して作成できます。

### 講師が説明すること

> `ollama-handson` は内部名です。YAML の namespace や CLI 操作で使うため、小文字とハイフンを使います。  
> `OllamaHansOn` は Console で見やすくするための表示名です。

> 今後作る Deployment、Pod、PVC、Service、Route、Secret はすべて `ollama-handson` Project に作成します。  
> 他の Project のリソースとは、名前が同じでも分離されます。

***

## 2-2. 現在の Project を確認する

Project 作成後、以下を実行します。

```bash
oc project
```

### 期待出力

```text
Using project "ollama-handson" on server "https://api.<cluster-domain>:6443".
```

この表示により、以後の `oc apply` や `oc get` が `ollama-handson` Project を対象にすることを確認します。

現在の Project 名だけを取得したい場合は、以下も使えます。

```bash
oc project -q
```

期待結果です。

```text
ollama-handson
```

***

## 2-3. 空の Project を CLI で観察する

以下を実行します。

```bash
oc get all
oc get pvc
oc get secret
oc get route
oc get events --sort-by=.lastTimestamp
```

### なぜ複数の `oc get` を使うのか

`oc get all` は便利ですが、すべてのリソース種別を表示するわけではありません。

| コマンド | 主な確認対象 |
|---|---|
| `oc get all` | Pod、Deployment、Service、ReplicaSet など |
| `oc get pvc` | PersistentVolumeClaim |
| `oc get secret` | Secret |
| `oc get route` | Route |
| `oc get events` | 作成・配置・起動に関するイベント |

### 期待結果

新規作成直後は、今回のアプリケーションに関するリソースが表示されません。

```text
No resources found in ollama-handson namespace.
```

または、最小限の自動作成リソースのみが見える場合があります。

### 講師が説明すること

> 今は空の Project です。  
> 次の章から PVC、Ollama、Open WebUI を作成するたびに、CLI と GUI の両方で「何が増えたか」を確認します。

***

## 2-4. GUI で Project を選択する

OpenShift Console に戻ります。

### GUI 操作

1. 画面上部の **Project セレクター**を開く
2. `ollama-handson` を選択する
3. 表示名が有効な Console 構成では、`OllamaHansOn` も確認する
4. Project の切替後、画面上部で対象 Project が変わったことを確認する

Console では Project セレクターから既存 Project の選択や、新規 Project の作成を行えます。

***

## 2-5. GUI で確認する場所

この研修では Administrator パースペクティブを標準として使用します。

```text
Administrator
  ├─ Workloads
  │   ├─ Deployments
  │   ├─ Pods
  │   └─ Secrets
  ├─ Storage
  │   └─ PersistentVolumeClaims
  └─ Networking
      ├─ Services
      └─ Routes
```

Project 作成直後は、以下のようにまだ何もない状態です。

```text
Administrator
  └─ Workloads
      ├─ Deployments：まだ何もない
      └─ Pods：まだ何もない
```

> 補足：環境によっては Developer パースペクティブや Topology が有効な場合があります。ただし本研修では、操作・確認手順を Administrator パースペクティブに統一します。

### 講師が説明すること

> Console の見た目やパースペクティブの表示は、OpenShift のバージョンや管理者設定によって異なる場合があります。  
> この研修で重要なのは、同じリソースを CLI と GUI の両方で確認できることです。

***

## 2-6. 今後表示されるリソースを確認する

この時点では空ですが、今後使う GUI の場所を確認します。

| 後続で作るリソース | GUI の確認場所 | 作成する章 |
|---|---|---|
| PVC | Administrator → Storage → PersistentVolumeClaims | 第3章 |
| Ollama Deployment / Pod | Administrator → Workloads | 第4章 |
| Open WebUI Deployment / Pod | Administrator → Workloads | 第6章 |
| Ollama Service | Administrator → Networking → Services | 第4章 |
| Open WebUI Service | Administrator → Networking → Services | 第6章 |
| Open WebUI Route | Administrator → Networking → Routes | 第7章 |
| Secret | Administrator → Workloads → Secrets | 第6章 |
| 構成全体の状態 | Administrator → Workloads / Networking / Storage | 第4章以降 |

***

## 章の確認チェック

### CLI

- [ ] `oc new-project` で `ollama-handson` を作成した
- [ ] `oc project` で現在の Project を確認した
- [ ] `oc project -q` が `ollama-handson` を返した
- [ ] `oc get all` を実行した
- [ ] `oc get pvc,secret,route` を個別に確認した
- [ ] `oc get events --sort-by=.lastTimestamp` を実行した

### GUI

- [ ] Project セレクターから `ollama-handson` を選択した
- [ ] Workloads でリソースがまだないことを確認した
- [ ] Storage、Networking の画面位置を確認した
- [ ] 後続のリソースがどの画面に現れるかを理解した

### 理解

- [ ] Project と Kubernetes Namespace の関係を説明できる
- [ ] Project がリソース・権限分離の単位であることを説明できる
- [ ] `oc get all` だけでは PVC、Secret、Route を確認できない理由を説明できる
- [ ] GUI の表示が環境設定により異なる場合があることを理解した

***

# 第3章：永続ストレージ（PVC）と LVM StorageClass

> 所要時間：25〜35 分  
> 前提：`ollama-handson` Project を作成・選択済みであること  
> この章では、Ollama のモデルと Open WebUI の状態データを保持する PVC を作成します。

## 目的

Pod を削除・再作成しても、Ollama のモデルデータと Open WebUI の設定・ユーザー情報を失わないように、永続ストレージを作成します。

また、default の LVM StorageClass が使う `WaitForFirstConsumer` により、PVC が作成直後に `Pending` となり、Pod 作成後に `Bound` へ変わる流れを理解します。

## 期待される結果

この章の完了時点では、以下の状態が**正常**です。

```text
ollama-data   Pending
webui-data    Pending
```

まだ Ollama Pod も Open WebUI Pod も作成していないため、PVC は `Pending` のままです。

次章で Ollama Pod を作成すると、以下へ変化します。

```text
ollama-data   Bound
webui-data    Pending
```

さらに Open WebUI Pod を作成すると、最終的に以下となります。

```text
ollama-data   Bound
webui-data    Bound
```

LVM StorageClass では `WaitForFirstConsumer` が使われ、PVC を利用する Pod が配置されるまで PVC は `Pending` のままになります。

***

## 3-1. 永続ストレージが必要な理由

Docker Compose 側では、以下の named volume を使いました。

```text
ollama-data
  └─ Ollama モデル保存先

webui-data
  └─ Open WebUI の設定・ユーザー・会話履歴の保存先
```

OpenShift 側では、これを PersistentVolumeClaim、略して PVC に置き換えます。

```text
Docker Compose                     OpenShift
--------------                     ---------
named volume                       PersistentVolumeClaim
ollama-data                        PVC/ollama-data
webui-data                         PVC/webui-data
```

### 講師が説明すること

> Pod のファイルシステムは一時的です。  
> Pod を削除したり、障害で再作成されたりすると、Pod 内だけに保存したデータは失われる可能性があります。

> モデルやユーザー設定のように残すべきデータは、Pod 内ではなく PVC に保存します。

***

## 3-2. PVC YAML を読む

以下を `manifests/01-pvc.yaml` として使用します。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webui-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## YAML の要素

| YAML 要素 | 意味 | この研修での値 |
|---|---|---|
| `kind` | 作成するリソース種別 | `PersistentVolumeClaim` |
| `metadata.name` | PVC のリソース名 | `ollama-data`、`webui-data` |
| `accessModes` | ストレージへのアクセス方式 | `ReadWriteOnce` |
| `resources.requests.storage` | 要求する容量 | Ollama 30Gi、WebUI 10Gi |

### `ReadWriteOnce` とは

`ReadWriteOnce`、略して RWO は、ボリュームを単一ノードから read/write でマウントできるアクセスモードです。

今回の構成では、Ollama と Open WebUI をそれぞれ 1 Pod で動かすため、RWO を使用します。

### なぜ `storageClassName` を書かないのか

このクラスタでは、LVM StorageClass が default に設定されています。

そのため YAML に以下を書かなくても、default StorageClass が自動選択されます。

```yaml
storageClassName: <DEFAULT_STORAGE_CLASS>
```

講師が説明するポイントです。

> StorageClass を明示しない場合、クラスタの default StorageClass が利用されます。  
> この環境では LVM StorageClass が default のため、PVC YAML は簡潔に保ちます。

***

## 3-3. PVC を作成する

以下を実行します。

```bash
oc apply -f manifests/01-pvc.yaml
```

### コマンドの意味

| コマンド要素 | 意味 |
|---|---|
| `oc apply` | YAML に記述された期待状態をクラスタに適用する |
| `-f` | YAML ファイルを指定する |
| `manifests/01-pvc.yaml` | Ollama 用と Open WebUI 用の PVC 定義 |

作成結果を確認します。

```bash
oc get pvc
```

### 期待出力

```text
NAME          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ollama-data   Pending                                      <default>      10s
webui-data    Pending                                      <default>      10s
```

ここで `Pending` が表示されても、まだ問題ではありません。

***

## 3-4. WaitForFirstConsumer を理解する

LVM StorageClass は `WaitForFirstConsumer` を使います。

これは、PVC を作成した時点では即座にストレージを割り当てず、その PVC を利用する Pod がスケジュールされる時点でストレージを provisioning する動作です。

### 状態遷移

```text
PVC を作成
  |
  v
Pending
  |
  | Ollama Pod が ollama-data を参照してスケジュールされる
  v
Bound
```

Ollama は L4 GPU を要求します。

```text
Ollama Pod
  |
  +-- GPU を利用可能なノードへ配置する必要がある
  |
  +-- そのノードで LVM ストレージを確保する必要がある
```

そのため、Pod の配置先が決まる前に PVC を `Bound` にしてしまうのではなく、Pod を利用する時点まで待ちます。

### PVC ごとの期待状態

| タイミング | `ollama-data` | `webui-data` |
|---|---|---|
| この章の PVC 作成直後 | `Pending` | `Pending` |
| 次章の Ollama Pod 作成後 | `Bound` | `Pending` |
| Open WebUI Pod 作成後 | `Bound` | `Bound` |

***

## 3-5. PVC の詳細を CLI で確認する

以下を実行します。

```bash
oc describe pvc ollama-data
```

続けて、Open WebUI 用 PVC を確認します。

```bash
oc describe pvc webui-data
```

### `oc describe` で確認する項目

| 項目 | 確認内容 |
|---|---|
| Name | `ollama-data` または `webui-data` |
| Status | 現時点では `Pending` |
| Capacity | `Pending` 中は未表示の場合がある |
| Access Modes | `RWO` |
| StorageClass | default LVM StorageClass |
| Events | first consumer を待つイベント |

### 講師が説明すること

> `oc get` は一覧を確認するコマンドです。  
> `oc describe` は、リソースの詳細と Events を確認するコマンドです。

> 問題が起きたときは、まず `oc describe` と Events を見る習慣を付けます。

***

## 3-6. GUI で PVC を確認する

Administrator パースペクティブで以下を開きます。

```text
Administrator
  └─ Storage
      └─ PersistentVolumeClaims
```

### GUI で確認すること

1. Project が `ollama-handson` であることを確認する
2. `ollama-data` が表示されることを確認する
3. `webui-data` が表示されることを確認する
4. 二つの Status が `Pending` であることを確認する
5. `ollama-data` を開く
6. Requested Storage が `30Gi` であることを確認する
7. Access Mode が `ReadWriteOnce` または `RWO` であることを確認する
8. `webui-data` を開く
9. Requested Storage が `10Gi` であることを確認する
10. Events または Details に first consumer 待機に相当する内容があることを確認する

### GUI で講師が伝えること

> `Pending` は常に障害を意味するわけではありません。  
> 今回の LVM StorageClass では、Pod がまだ存在しないため、PVC は意図的に `Pending` のままです。

> 次章で Ollama Pod を作成すると、GPU を利用するノードへの配置に合わせて `ollama-data` が `Bound` へ変化します。

***

## 3-7. `Pending` を調査すべきケース

この章では `Pending` は正常です。ただし、**PVC を利用する Pod を作成した後も** `Pending` が続く場合は調査が必要です。

以下を実行します。

```bash
oc get pod,pvc -o wide
oc describe pod -l app=ollama
oc describe pvc ollama-data
oc get events --sort-by=.lastTimestamp
```

### 主な確認観点

| 確認対象 | 確認する内容 |
|---|---|
| Ollama Pod | GPU、CPU、メモリ要求を満たせるか |
| Pod Events | スケジューリング失敗、node selector、taint など |
| PVC Events | provisioning 失敗、LVM 容量、StorageClass |
| ノード | GPU と LVM を利用できるノードがあるか |

LVM ローカルストレージで PVC が Pod 作成後も `Pending` になる代表的な要因には、計算資源不足、StorageClass または node selector の不一致、利用可能 PV 不足、対象ノードが `Not Ready` であることなどがあります。

***

## Docker Compose と OpenShift の対比

| 項目 | Docker Compose | OpenShift |
|---|---|---|
| 永続ストレージ | named volume | PVC |
| Ollama データ | `ollama-data` | `PVC/ollama-data` |
| WebUI データ | `webui-data` | `PVC/webui-data` |
| 保存先 | `/root/.ollama` | `/root/.ollama` |
| Pod / コンテナ再作成後 | volume にデータが残る | PVC にデータが残る |
| 作成直後の状態 | volume は即利用可能 | LVM PVC は `Pending` の場合がある |
| 割当タイミング | Docker がローカル管理 | Pod スケジュール時に provisioning |

***

## 章の確認チェック

### CLI

- [ ] `oc apply -f manifests/01-pvc.yaml` を実行した
- [ ] `oc get pvc` で二つの PVC を確認した
- [ ] `oc describe pvc ollama-data` を実行した
- [ ] `oc describe pvc webui-data` を実行した
- [ ] PVC 作成直後の Status が `Pending` であることを確認した

### GUI

- [ ] Administrator → Storage → PersistentVolumeClaims を開いた
- [ ] `ollama-data` と `webui-data` を確認した
- [ ] 要求容量 30Gi / 10Gi を確認した
- [ ] Access Mode が RWO であることを確認した
- [ ] `Pending` がこの時点では正常であることを確認した
- [ ] PVC の Events を確認した

### 理解

- [ ] Docker named volume と OpenShift PVC の対応を説明できる
- [ ] `ReadWriteOnce` の意味を説明できる
- [ ] `WaitForFirstConsumer` の意味を説明できる
- [ ] PVC 作成直後の `Pending` が、この環境では正常であることを説明できる
- [ ] `Bound` へ変わる契機が Pod のスケジューリングであることを説明できる

***

# 第4章：Ollama を OpenShift にデプロイする

> 所要時間：35〜45 分  
> 前提：`ollama-handson` Project が選択済みであり、`ollama-data` PVC が `Pending` で作成済みであること  
> GPU：NVIDIA L4 × 1 を使用します。

## 目的

Ollama を OpenShift 上の GPU 対応 Pod として起動し、後続の Open WebUI がクラスタ内部から利用できる API Service を作成します。

また、Deployment、Pod、Service、PVC、label、selector、Endpoints の関係を確認します。

## 期待される結果

この章の完了時点で、以下を満たします。

- `Deployment/ollama` が作成されている
- Ollama Pod が GPU を 1 枚要求している
- Ollama Pod が `Running`、`Ready 1/1` になる
- `PVC/ollama-data` が `Pending` から `Bound` へ変わる
- `Service/ollama` が作成される
- `Service/ollama` の Endpoints に Ollama Pod が登録される
- Ollama API の待受ポートとして 11434 が定義される

Ollama の API は通常 `http://localhost:11434/api` で提供されます。この研修では、Pod 内の 11434 を Service 経由で Open WebUI に公開します。

***

## 4-1. 作成するリソースを確認する

この章で作成するものです。

```text
Deployment/ollama
  |
  +-- Pod/ollama-<ReplicaSet>-<Suffix>
  |     |
  |     +-- NVIDIA L4 GPU × 1
  |     |
  |     +-- PVC/ollama-data
  |
  +-- Service/ollama:11434
```

### 講師が説明すること

> Deployment は「Ollama Pod を常に一つ稼働させる」という期待状態を持つリソースです。  
> Pod が削除されても、Deployment が新しい Pod を作成します。

> Service は Pod の IP アドレスへ直接接続しないための安定した接続先です。  
> Open WebUI は `http://ollama:11434` を使い、Ollama Pod ではなく Service へ接続します。

***

## 4-2. Ollama YAML を読む

以下を `manifests/02-ollama.yaml` として使用します。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  labels:
    app: ollama
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
        - name: ollama
          # 講師デモ用。反復実施・運用では固定 tag または digest を使用する。
          image: ollama/ollama:latest
          ports:
            - name: http
              containerPort: 11434
          volumeMounts:
            - name: ollama-data
              mountPath: /root/.ollama
          resources:
            requests:
              cpu: "1"
              memory: 8Gi
              nvidia.com/gpu: "1"
            limits:
              cpu: "2"
              memory: 12Gi
              nvidia.com/gpu: "1"
      volumes:
        - name: ollama-data
          persistentVolumeClaim:
            claimName: ollama-data
---
apiVersion: v1
kind: Service
metadata:
  name: ollama
spec:
  selector:
    app: ollama
  ports:
    - name: http
      protocol: TCP
      port: 11434
      targetPort: http
```

> `ollama/ollama:latest` は講師デモ用です。将来同じ YAML を使っても別のイメージが取得される可能性があるため、研修を固定化する場合は検証済み tag または digest に置き換えます。

***

## 4-3. Deployment の重要な要素

### `replicas: 1`

```yaml
spec:
  replicas: 1
```

Ollama Pod を常に一つ稼働させることを意味します。

```text
期待状態:
  Ollama Pod が 1 個稼働している

Pod が削除された場合:
  Deployment が新しい Pod を作成する
```

### `selector.matchLabels` と Pod label

```yaml
selector:
  matchLabels:
    app: ollama
```

```yaml
template:
  metadata:
    labels:
      app: ollama
```

この二つは一致している必要があります。

```text
Deployment
  |
  | selector: app=ollama
  v
Pod
  |
  +-- label: app=ollama
```

一致しない場合、Deployment は管理対象 Pod を正しく判別できません。

### GPU 要求

```yaml
resources:
  requests:
    nvidia.com/gpu: "1"
  limits:
    nvidia.com/gpu: "1"
```

この設定により、Ollama Pod は NVIDIA GPU を一枚要求します。

今回の環境では、NVIDIA L4 を一枚使います。

### CPU とメモリ

```yaml
requests:
  cpu: "1"
  memory: 8Gi
limits:
  cpu: "2"
  memory: 12Gi
```

| 項目 | 意味 |
|---|---|
| `requests` | Pod を配置するために必要な最低リソース |
| `limits` | コンテナが使用できる上限リソース |
| `cpu: "1"` | 1 CPU コア相当を要求 |
| `memory: 8Gi` | 8GiB のメモリを要求 |
| `nvidia.com/gpu: "1"` | GPU を一枚要求 |

### PVC のマウント

```yaml
volumeMounts:
  - name: ollama-data
    mountPath: /root/.ollama
```

```yaml
volumes:
  - name: ollama-data
    persistentVolumeClaim:
      claimName: ollama-data
```

この二つの設定により、`PVC/ollama-data` が Ollama コンテナ内の `/root/.ollama` へマウントされます。

```text
PVC/ollama-data
  |
  v
Ollama Pod
  |
  v
/root/.ollama
```

後続の章で `ollama pull gemma4:e2b` を実行すると、モデルはこの PVC 上へ保存されます。

***

## 4-4. Service の重要な要素

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ollama
spec:
  selector:
    app: ollama
  ports:
    - name: http
      port: 11434
      targetPort: http
```

### Service の役割

Service は、label selector に一致する Pod を探し、安定した接続先を提供します。

```text
Open WebUI Pod
  |
  | http://ollama:11434
  v
Service/ollama
  |
  | selector: app=ollama
  v
Ollama Pod
```

Service は selector を使って Pod を探し、該当する Pod を Endpoints として登録します。

### `port` と `targetPort`

| 項目 | 値 | 意味 |
|---|---:|---|
| `port` | 11434 | Service が提供するポート |
| `targetPort` | `http` | Pod 内コンテナへの転送先 |
| `containerPort` | 11434 | Ollama コンテナが待ち受けるポート |

```text
Service/ollama:11434
  |
  v
targetPort: http
  |
  v
containerPort: 11434
```

### `containerPort` だけでは公開されない

以下の設定は、コンテナが使うポート情報です。

```yaml
containerPort: 11434
```

これだけでは、他 Pod や外部から接続できません。

```text
containerPort
  └─ コンテナ内部の待受情報

Service
  └─ クラスタ内部からの接続先

Route
  └─ クラスタ外部からの HTTP/HTTPS 接続先
```

この研修では Ollama に Route を作らず、Service による内部公開のみとします。

***

## 4-5. Ollama を適用する

以下を実行します。

```bash
oc apply -f manifests/02-ollama.yaml
```

作成状況を確認します。

```bash
oc rollout status deployment/ollama
```

### コマンドの意味

| コマンド | 目的 |
|---|---|
| `oc apply -f` | Deployment と Service を作成・更新する |
| `oc rollout status deployment/ollama` | Ollama Deployment が Ready になるまで待つ |

### 期待結果

```text
deployment "ollama" successfully rolled out
```

Pod が GPU ノードへ配置され、PVC の provisioning が完了するまで少し時間がかかることがあります。

***

## 4-6. CLI で状態を確認する

以下を順番に実行します。

```bash
oc get deployment
oc get pods
oc get pod -l app=ollama -o wide
oc get svc
oc get pvc
oc get endpoints ollama
```

### 確認する内容

| コマンド | 確認ポイント |
|---|---|
| `oc get deployment` | Ollama Deployment の Ready 数が `1/1` |
| `oc get pods` | Ollama Pod が `Running` |
| `oc get pod -l app=ollama -o wide` | Pod 名、Pod IP、配置ノード |
| `oc get svc` | `Service/ollama` が存在する |
| `oc get pvc` | `ollama-data` が `Bound` になった |
| `oc get endpoints ollama` | Ollama Pod の IP:11434 が表示される |

### 期待出力のイメージ

```text
$ oc get deployment
NAME     READY   UP-TO-DATE   AVAILABLE
ollama   1/1     1            1
```

```text
$ oc get pvc
NAME          STATUS
ollama-data   Bound
webui-data    Pending
```

```text
$ oc get endpoints ollama
NAME     ENDPOINTS          AGE
ollama   10.x.x.x:11434     1m
```

### 講師が説明すること

> `ollama-data` が `Bound` へ変わったのは、Ollama Pod が PVC を利用するためです。  
> 一方、`webui-data` はまだ Open WebUI Pod が存在しないため、`Pending` のままです。

> Endpoints に Ollama Pod の IP と 11434 が見えていれば、Service は Pod を発見できています。

***

## 4-7. Ollama のログを確認する

以下を実行します。

```bash
oc logs deployment/ollama
```

直近のログだけ確認する場合は以下を使います。

```bash
oc logs deployment/ollama --tail=100
```

### 講師が説明すること

> `oc logs deployment/ollama` は、Deployment が管理する Pod のログを確認する便利な書き方です。  
> Pod 名を毎回コピーしなくても、Deployment 名を使ってログを確認できます。

Pod が再起動を繰り返している場合は、前回のコンテナのログを確認します。

```bash
oc logs deployment/ollama --previous
```

***

## 4-8. GUI で Ollama を確認する

Administrator パースペクティブで確認します。

### Deployment の確認

```text
Administrator
  └─ Workloads
      └─ Deployments
          └─ ollama
```

確認する内容です。

- Deployment 名が `ollama`
- Desired Replicas が 1
- Current Replicas が 1
- Available Replicas が 1

### Pod の確認

```text
Administrator
  └─ Workloads
      └─ Pods
          └─ ollama-<ReplicaSet>-<Suffix>
```

確認する内容です。

- Status が `Running`
- Ready が `1/1`
- Pod の Node 名
- Pod の Events
- Pod の Logs
- Pod YAML
- Terminal

### PVC の確認

```text
Administrator
  └─ Storage
      └─ PersistentVolumeClaims
          └─ ollama-data
```

確認する内容です。

- Status が `Bound`
- Requested Storage が 30Gi
- Access Mode が RWO
- PVC が削除されず存在している

### Service の確認

```text
Administrator
  └─ Networking
      └─ Services
          └─ ollama
```

確認する内容です。

- Service 名が `ollama`
- Port が 11434
- selector が `app=ollama`
- Endpoints が Ollama Pod を指している

### GUI で講師が伝えること

> GUI の Pod Events では、GPU、PVC、スケジューリング、イメージ pull に関する問題を確認できます。

> Service の selector と Pod の label が一致しない場合、Endpoints が空になり、Open WebUI は Ollama へ接続できません。

***

## 4-9. よくある状態と初動

| 症状 | 最初に確認するコマンド | 主な原因 |
|---|---|---|
| Pod が `Pending` | `oc describe pod -l app=ollama` | GPU、CPU、メモリ、PVC、ノード制約 |
| `ImagePullBackOff` | `oc describe pod -l app=ollama` | image 名、tag、レジストリ接続 |
| `CrashLoopBackOff` | `oc logs deployment/ollama --previous` | 起動設定、権限、PVC mountPath |
| PVC が `Pending` のまま | `oc describe pvc ollama-data` | Pod 未配置、LVM provisioning、容量 |
| Endpoints が空 | `oc get pod -l app=ollama --show-labels` | Service selector と Pod label の不一致 |

***

## 章の確認チェック

### CLI

- [ ] `oc apply -f manifests/02-ollama.yaml` を実行した
- [ ] `oc rollout status deployment/ollama` が成功した
- [ ] `oc get pod -l app=ollama -o wide` を実行した
- [ ] `oc get pvc` で `ollama-data` が `Bound` になったことを確認した
- [ ] `oc get endpoints ollama` で Endpoints を確認した
- [ ] `oc logs deployment/ollama` を実行した

### GUI

- [ ] Administrator → Workloads → Deployments で Ollama を確認した
- [ ] Ollama Pod が `Running`、`Ready 1/1` であることを確認した
- [ ] Pod の Logs と Events を確認した
- [ ] `ollama-data` PVC が `Bound` であることを確認した
- [ ] Service `ollama` の selector と Endpoints を確認した

### 理解

- [ ] Deployment が Pod の期待状態を維持することを説明できる
- [ ] GPU 要求 `nvidia.com/gpu: 1` の意味を説明できる
- [ ] `volumeMounts` と `volumes` の役割を説明できる
- [ ] Service、selector、Pod label、Endpoints の関係を説明できる
- [ ] Ollama を外部公開せず、内部 Service にする理由を説明できる

***

# 第5章：Gemma 4 モデル取得と Ollama API 確認

> 所要時間：25〜35 分  
> 前提：`Deployment/ollama` が Ready であり、`PVC/ollama-data` が `Bound` であること  
> モデル：`gemma4:e2b`

## 目的

Ollama Pod 内で `gemma4:e2b` を取得し、モデルが PVC に保存されることを確認します。

さらに、以下の二つの方法で Ollama API を確認します。

- Pod 内で `ollama` CLI を実行する
- WSL から `oc port-forward` と `curl` を使う

## 期待される結果

この章の完了時点で、以下を満たします。

- `ollama pull gemma4:e2b` が完了する
- `ollama list` に `gemma4:e2b` が表示される
- `GET /api/tags` で利用可能なモデル一覧を取得できる
- `POST /api/generate` で推論結果を取得できる
- Pod を再作成しても、モデルが `ollama-data` PVC 上に残る設計であることを理解する
- `oc port-forward` が開発・検証用の一時的経路だと説明できる

Ollama CLI では `ollama pull <モデル名>` でモデルを取得し、`ollama list` で取得済みモデルを表示します。 Ollama API の `/api/generate` は通常ストリーミング応答ですが、`"stream": false` を指定すると単一の JSON 応答として取得できます。

***

## 5-1. モデル保存先を確認する

前章の Ollama Deployment では、以下の設定を行いました。

```yaml
volumeMounts:
  - name: ollama-data
    mountPath: /root/.ollama
```

```yaml
volumes:
  - name: ollama-data
    persistentVolumeClaim:
      claimName: ollama-data
```

そのため、Ollama が取得するモデルは以下の関係で保存されます。

```text
Ollama コンテナ
  |
  +-- /root/.ollama
        |
        v
PVC/ollama-data
```

### 講師が説明すること

> モデルは Pod の一時的なファイルシステムではなく、PVC に保存します。  
> Ollama Pod が削除・再作成されても、同じ PVC を再度マウントするため、モデルを再ダウンロードせずに利用できます。

***

## 5-2. Ollama Pod 内でモデルを取得する

以下を実行します。

```bash
oc exec -it deploy/ollama -- ollama pull gemma4:e2b
```

### コマンドの意味

| コマンド要素 | 意味 |
|---|---|
| `oc exec` | 実行中の Pod 内でコマンドを実行する |
| `-it` | 対話形式で入出力を接続する |
| `deploy/ollama` | Ollama Deployment が管理する Pod を対象にする |
| `--` | `oc` の引数と Pod 内で実行するコマンドを区切る |
| `ollama pull gemma4:e2b` | Gemma 4 の指定モデルを取得する |

### 期待結果

モデルの取得進捗が表示されます。

```text
pulling manifest
pulling <layer>
verifying sha256 digest
writing manifest
success
```

モデルのサイズ、ネットワーク帯域、コンテナレジストリやモデル配布元への到達性により、完了まで時間がかかります。

### 講師が説明すること

> CPU の Docker Compose 側でも同じモデルを取得できますが、OpenShift 側の L4 GPU は主に推論処理を高速化するために使います。  
> モデル取得の時間は GPU 性能ではなく、モデルサイズ、ストレージ、ネットワーク帯域の影響を大きく受けます。

***

## 5-3. 取得済みモデルを確認する

以下を実行します。

```bash
oc exec -it deploy/ollama -- ollama list
```

### 期待出力のイメージ

```text
NAME          ID            SIZE      MODIFIED
gemma4:e2b    <MODEL_ID>    <SIZE>    <TIME>
```

表示される項目を確認します。

| 項目 | 意味 |
|---|---|
| `NAME` | Ollama で指定するモデル名 |
| `ID` | モデルの識別子 |
| `SIZE` | 保存されているモデル容量 |
| `MODIFIED` | モデルを取得または更新した時刻 |

`ollama list` は、Ollama がローカルに保持しているモデルを確認するためのコマンドです。

***

## 5-4. GUI Terminal で同じ操作を確認する

Administrator パースペクティブで以下を開きます。

```text
Administrator
  └─ Workloads
      └─ Pods
          └─ ollama-<ReplicaSet>-<Suffix>
              └─ Terminal
```

Terminal を開き、以下を実行します。

```bash
ollama list
```

必要であれば、以下も実行します。

```bash
ollama ps
```

### GUI で確認すること

- `gemma4:e2b` が表示される
- Pod が `Running`、`Ready 1/1` である
- Pod の Logs にエラーがない
- `ollama-data` PVC が `Bound` のままである

### 講師が説明すること

> GUI Terminal と `oc exec` はどちらも Pod 内でコマンドを実行する手段です。  
> 手順を自動化・再現する場合は CLI を使い、画面で簡単に確認する場合は GUI Terminal を使えます。

***

## 5-5. Service を port-forward する

Ollama は外部公開しない方針です。そのため、講師端末から API を直接確認する場合は `oc port-forward` を使用します。

WSL のターミナル A で以下を実行します。

```bash
oc port-forward svc/ollama 11434:11434
```

> 第1章の Docker 側は、ホストポートを `11435` に変更しています。そのため WSL の `11434` は空いており、この port-forward と Docker 環境を同時に利用できます。

### コマンドの意味

| 要素 | 意味 |
|---|---|
| `oc port-forward` | ローカル端末とクラスタ内リソースを一時的につなぐ |
| `svc/ollama` | 転送先を Ollama Service に指定する |
| 左側の `11434` | WSL ローカルの待受ポート |
| 右側の `11434` | Service のポート |

```text
WSL
  |
  | http://localhost:11434
  v
oc port-forward
  |
  v
Service/ollama:11434
  |
  v
Ollama Pod:11434
```

### 期待結果

以下のような待受メッセージが表示されます。

```text
Forwarding from 127.0.0.1:11434 -> 11434
Forwarding from [::1]:11434 -> 11434
```

このターミナルは開いたままにします。

### 講師が説明すること

> `oc port-forward` は開発・確認のための一時的な経路です。  
> アプリケーション間通信や本番の外部公開には使いません。

> Open WebUI は `oc port-forward` を使わず、クラスタ内部の `Service/ollama` へ直接接続します。

***

## 5-6. モデル一覧 API を確認する

別の WSL ターミナル B を開き、以下を実行します。

```bash
curl -s http://localhost:11434/api/tags | jq .
```

### コマンドの意味

| 要素 | 意味 |
|---|---|
| `curl` | HTTP API を呼び出す |
| `-s` | 進捗表示を抑制する |
| `/api/tags` | Ollama にあるモデル一覧を取得する API |
| `jq .` | JSON を読みやすく整形する |

### 期待結果

JSON の中に `gemma4:e2b` が含まれます。

```json
{
  "models": [
    {
      "name": "gemma4:e2b"
    }
  ]
}
```

### 講師が説明すること

> `ollama list` は Pod 内の CLI 操作です。  
> `/api/tags` は HTTP API を通じた確認です。  
> Open WebUI は内部的に、このような Ollama API を利用してモデル一覧や推論結果を取得します。

***

## 5-7. 推論 API を確認する

以下を実行します。

```bash
curl -s http://localhost:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "gemma4:e2b",
    "prompt": "OpenShiftを一文で説明してください。",
    "stream": false
  }' | jq .
```

### リクエスト本文の意味

| JSON 項目 | 意味 |
|---|---|
| `model` | 使用する Ollama モデル名 |
| `prompt` | モデルへ送る入力文 |
| `stream` | 応答をストリーミングするか |
| `false` | 応答を一つの JSON として受け取る |

`/api/generate` はテキスト生成 API です。ストリーミングを無効化すると、`curl` と `jq` で一回の JSON 応答として扱いやすくなります。

### 期待結果

以下のような JSON 応答を取得します。

```json
{
  "model": "gemma4:e2b",
  "response": "OpenShiftは、コンテナ化されたアプリケーションを…",
  "done": true
}
```

`response` に回答文が含まれ、`done` が `true` であれば生成完了です。

***

## 5-8. API と Pod Logs を関連付ける

推論 API を実行した後、ターミナル A とは別に以下を実行します。

```bash
oc logs deployment/ollama --tail=100
```

または GUI で以下を開きます。

```text
Administrator
  └─ Workloads
      └─ Pods
          └─ Ollama Pod
              └─ Logs
```

### 講師が説明すること

> API リクエストを実行したら、アプリケーションログも確認します。  
> 利用者から「応答が遅い」「エラーになった」と言われたとき、API の応答だけでなく、Pod の Logs や Events を確認します。

***

## Docker Compose と OpenShift の対比

| 操作 | Docker Compose | OpenShift |
|---|---|---|
| モデル取得 | `docker compose exec ollama -- ollama pull gemma4:e2b` | `oc exec -it deploy/ollama -- ollama pull gemma4:e2b` |
| モデル一覧 | `docker compose exec ollama -- ollama list` | `oc exec -it deploy/ollama -- ollama list` |
| API 確認 | `http://localhost:11434` | `oc port-forward` 後の `http://localhost:11434` |
| コンテナ内確認 | `docker compose exec` | `oc exec` / GUI Terminal |
| ログ確認 | `docker compose logs ollama` | `oc logs deployment/ollama` |
| データ保存 | Docker named volume | PVC |

***

## 章の確認チェック

### CLI

- [ ] `oc exec -it deploy/ollama -- ollama pull gemma4:e2b` を実行した
- [ ] `oc exec -it deploy/ollama -- ollama list` を実行した
- [ ] `oc port-forward svc/ollama 11434:11434` を実行した
- [ ] `/api/tags` でモデル一覧を確認した
- [ ] `/api/generate` で推論結果を確認した
- [ ] `oc logs deployment/ollama --tail=100` を確認した

### GUI

- [ ] Ollama Pod の Terminal を開いた
- [ ] GUI Terminal で `ollama list` を実行した
- [ ] Ollama Pod の Logs を確認した
- [ ] `ollama-data` PVC が `Bound` のままであることを確認した

### 理解

- [ ] `ollama pull` と `ollama list` の役割を説明できる
- [ ] モデルが PVC に保存される理由を説明できる
- [ ] `oc port-forward` が一時的な確認経路であることを説明できる
- [ ] `/api/tags` と `/api/generate` の役割を説明できる
- [ ] `stream: false` を指定する理由を説明できる

***

# 第6章：Open WebUI のデプロイと Ollama への内部接続

> 所要時間：35〜45 分  
> 前提：Ollama Pod が `Running` / `Ready 1/1` であり、`Service/ollama` の Endpoints が存在すること  
> この章では、Open WebUI を内部 Service 経由で Ollama に接続します。

## 目的

Open WebUI を OpenShift 上へデプロイし、Ollama Service を使って内部通信させます。

この章では、以下を扱います。

- Open WebUI 用の PVC
- `WEBUI_SECRET_KEY` の Secret 管理
- `OLLAMA_BASE_URL`
- Pod 内の `localhost` の意味
- Open WebUI Service
- Open WebUI Pod から Ollama Pod への通信確認

## 期待される結果

この章の完了時点で、以下を満たします。

- `Secret/open-webui-secret` が作成されている
- `Deployment/open-webui` が作成されている
- Open WebUI Pod が `Running` / `Ready 1/1` である
- `PVC/webui-data` が `Pending` から `Bound` になる
- `Service/open-webui` が作成される
- Open WebUI Pod から `http://ollama:11434/api/tags` を取得できる
- `OLLAMA_BASE_URL` が `http://ollama:11434` に設定されている

***

## 6-1. Open WebUI の責務を確認する

ここまでに作成済みの Ollama は、モデルを保持し、推論 API を提供する内部サーバーです。

Open WebUI は、利用者がブラウザから会話やモデル選択を行うための UI です。

```text
利用者
  |
  | ブラウザ
  v
Open WebUI
  |
  | Ollama API
  v
Ollama
  |
  | モデルデータ
  v
PVC/ollama-data
```

Open WebUI 自身も、ユーザー、設定、会話履歴などの状態データを持つため、別の PVC を使います。

```text
Open WebUI Pod
  |
  v
PVC/webui-data
```

### Docker Compose との対比

| Docker Compose | OpenShift |
|---|---|
| `open-webui` Service | `Deployment/open-webui` |
| `webui-data` named volume | `PVC/webui-data` |
| `OLLAMA_BASE_URL=http://ollama:11434` | 同じ環境変数を使用 |
| `ports: "3000:8080"` | `Service/open-webui` + Route |
| Docker Compose network | OpenShift Service DNS |

***

## 6-2. Secret を作成する

Open WebUI の `WEBUI_SECRET_KEY` は、Secret として作成します。

以下を実行します。

```bash
oc create secret generic open-webui-secret \
  --from-literal=WEBUI_SECRET_KEY="$(openssl rand -base64 32)"
```

### コマンドの意味

| 要素 | 意味 |
|---|---|
| `oc create secret generic` | 汎用 Secret を作成する |
| `open-webui-secret` | Secret のリソース名 |
| `--from-literal` | 文字列を Secret のキー・値として登録する |
| `WEBUI_SECRET_KEY` | Open WebUI が参照する Secret のキー |
| `openssl rand -base64 32` | ランダムな値を生成する |

Secret が存在することを確認します。

```bash
oc get secret open-webui-secret
oc describe secret open-webui-secret
```

### 期待結果

```text
NAME                TYPE     DATA   AGE
open-webui-secret   Opaque   1      10s
```

`oc describe secret` では、Secret のキー名と件数を確認します。値そのものは出力しません。

### 講師が説明すること

> Secret は、パスワード、Token、API Key、暗号鍵などを扱うための OpenShift リソースです。

> Secret の実値は Git の YAML、画面共有、スクリーンショット、チャット履歴へ残しません。

> この研修では、Secret を作成し、Deployment の環境変数へ注入します。

***

### 補足：`WEBUI_SECRET_KEY` は何のためにあるか

`WEBUI_SECRET_KEY` は、Ollama への接続情報ではありません。

Open WebUI 自身のログインセッションを保護するための秘密鍵であり、主にログイン Token（JWT）の署名や、OAuth 連携時の機密情報の暗号化に使われます。

| 用途 | `WEBUI_SECRET_KEY` の役割 |
|---|---|
| ログインセッション | JWT の署名・検証 |
| ログイン維持 | Pod 再起動後も既存セッションを正しく検証する |
| OAuth / OIDC | OAuth セッションデータや Token の暗号化 |
| 複数 Pod 構成 | すべての Open WebUI Pod で同じ JWT を検証できる |

Pod の再作成時にこの値が変わると、利用者がログアウトしたり、外部連携の認証情報を再登録する必要が出たりする可能性があります。

このため、この研修ではランダムな値を OpenShift Secret に保存し、Open WebUI Pod へ環境変数として渡します。重要なのは、値そのものを YAML や Git に書かないことです。

> 講師の一言説明  
> `WEBUI_SECRET_KEY` は、Open WebUI のログイン状態を安全かつ継続的に扱うための鍵です。Pod が再作成されても同じ鍵を使えるよう、OpenShift Secret に保存します。

***

## 6-3. Open WebUI YAML を読む

以下を `manifests/03-open-webui.yaml` として使用します。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: open-webui
  labels:
    app: open-webui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: open-webui
  template:
    metadata:
      labels:
        app: open-webui
    spec:
      containers:
        - name: open-webui
          # 講師デモ用。反復実施・運用では固定 tag または digest を使用する。
          image: ghcr.io/open-webui/open-webui:main
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: OLLAMA_BASE_URL
              value: http://ollama:11434
            - name: WEBUI_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: open-webui-secret
                  key: WEBUI_SECRET_KEY
          volumeMounts:
            - name: webui-data
              mountPath: /app/backend/data
          resources:
            requests:
              cpu: 250m
              memory: 1Gi
            limits:
              cpu: "1"
              memory: 2Gi
      volumes:
        - name: webui-data
          persistentVolumeClaim:
            claimName: webui-data
---
apiVersion: v1
kind: Service
metadata:
  name: open-webui
spec:
  selector:
    app: open-webui
  ports:
    - name: http
      port: 8080
      targetPort: http
```

***

## 6-4. `OLLAMA_BASE_URL` を理解する

以下の設定が最も重要です。

```yaml
- name: OLLAMA_BASE_URL
  value: http://ollama:11434
```

### 接続経路

```text
Open WebUI Pod
  |
  | http://ollama:11434
  v
Service/ollama
  |
  | selector: app=ollama
  v
Ollama Pod
```

ここで使う `ollama` は、Ollama Pod 名ではなく `Service/ollama` の名前です。

### `localhost` を使わない理由

Open WebUI Pod 内の `localhost` は、Open WebUI Pod 自身です。

```text
Open WebUI Pod の localhost
  |
  +-- Open WebUI Pod 自身
```

以下のように指定すると、Open WebUI は自分自身の 11434 ポートへ接続しようとします。

```text
http://localhost:11434
```

Open WebUI Pod 内には Ollama がいないため、接続できません。

正しい接続先です。

```text
http://ollama:11434
```

Docker Compose 側も同じ考え方です。

```text
Docker Compose 内部:
  http://ollama:11434

OpenShift 内部:
  http://ollama:11434
```

***

## 6-5. Secret を環境変数へ注入する

以下の設定を確認します。

```yaml
- name: WEBUI_SECRET_KEY
  valueFrom:
    secretKeyRef:
      name: open-webui-secret
      key: WEBUI_SECRET_KEY
```

### 要素の意味

| 項目 | 意味 |
|---|---|
| `valueFrom` | 値を直接書かず、別リソースから取得する |
| `secretKeyRef` | Secret を参照する |
| `name` | Secret 名 |
| `key` | Secret 内のキー名 |

```text
Secret/open-webui-secret
  |
  +-- WEBUI_SECRET_KEY
         |
         v
Open WebUI Pod の環境変数
  |
  +-- WEBUI_SECRET_KEY
```

### 講師が説明すること

> Deployment YAML に Secret の実値を書かず、Secret を参照します。

> Git に保存するのは「どの Secret を参照するか」だけです。Secret の値そのものはクラスタ内で作成します。

***

## 6-6. Open WebUI 用 PVC を確認する

Open WebUI は以下へ状態データを書き込みます。

```yaml
volumeMounts:
  - name: webui-data
    mountPath: /app/backend/data
```

対応する PVC です。

```yaml
volumes:
  - name: webui-data
    persistentVolumeClaim:
      claimName: webui-data
```

```text
PVC/webui-data
  |
  v
Open WebUI Pod
  |
  v
/app/backend/data
```

この PVC は、前章で作成済みですが、まだ Open WebUI Pod が存在しないため `Pending` のままです。

Open WebUI Pod が作成されると、LVM の `WaitForFirstConsumer` により `Bound` になります。

***

## 6-7. Open WebUI を適用する

以下を実行します。

```bash
oc apply -f manifests/03-open-webui.yaml
```

Deployment の rollout を待ちます。

```bash
oc rollout status deployment/open-webui
```

### 期待結果

```text
deployment "open-webui" successfully rolled out
```

***

## 6-8. CLI で状態を確認する

以下を実行します。

```bash
oc get deployment
oc get pods
oc get pod -l app=open-webui -o wide
oc get svc
oc get pvc
oc get endpoints open-webui
oc logs deployment/open-webui
```

### 期待される状態

```text
$ oc get deployment
NAME        READY   UP-TO-DATE   AVAILABLE
ollama      1/1     1            1
open-webui  1/1     1            1
```

```text
$ oc get pvc
NAME          STATUS
ollama-data   Bound
webui-data    Bound
```

```text
$ oc get endpoints open-webui
NAME        ENDPOINTS         AGE
open-webui  10.x.x.x:8080     1m
```

### 確認するポイント

| 確認対象 | 期待状態 |
|---|---|
| Open WebUI Deployment | `READY 1/1` |
| Open WebUI Pod | `Running`、`Ready 1/1` |
| `webui-data` PVC | `Bound` |
| Open WebUI Service | 存在する |
| Open WebUI Endpoints | Pod IP と 8080 が表示される |
| Open WebUI Logs | 起動エラーがない |

***

## 6-9. Open WebUI Pod から Ollama API を確認する

Open WebUI Pod 内から、Ollama Service へ接続します。

```bash
oc exec -it deploy/open-webui -- sh -c \
  'wget -qO- http://ollama:11434/api/tags || true'
```

### コマンドの意味

| 要素 | 意味 |
|---|---|
| `oc exec` | Open WebUI Pod 内でコマンドを実行する |
| `deploy/open-webui` | Open WebUI Deployment が管理する Pod を対象にする |
| `wget -qO-` | HTTP 応答本文を標準出力へ表示する |
| `http://ollama:11434/api/tags` | Ollama Service のモデル一覧 API |
| `|| true` | `wget` が存在しない場合などでも、演習のシェルを即終了させないための補助 |

### 期待結果

以下のような JSON が返ります。

```json
{
  "models": [
    {
      "name": "gemma4:e2b"
    }
  ]
}
```

これが返れば、以下の経路が成功しています。

```text
Open WebUI Pod
  |
  v
Service/ollama
  |
  v
Ollama Pod
  |
  v
/api/tags
```

***

## 6-10. GUI で Open WebUI を確認する

すべて Administrator パースペクティブで確認します。

### Secret の確認

```text
Administrator
  └─ Workloads
      └─ Secrets
          └─ open-webui-secret
```

確認することです。

- `open-webui-secret` が存在する
- Type が `Opaque`
- Data が 1 件ある
- Secret の実値は表示・共有しない

### Deployment の確認

```text
Administrator
  └─ Workloads
      └─ Deployments
          └─ open-webui
```

確認することです。

- Desired Replicas が 1
- Current Replicas が 1
- Available Replicas が 1

### Pod の確認

```text
Administrator
  └─ Workloads
      └─ Pods
          └─ open-webui-<ReplicaSet>-<Suffix>
```

確認することです。

- Status が `Running`
- Ready が `1/1`
- Logs に起動エラーがない
- Events に PVC、イメージ pull、権限エラーがない
- YAML または環境変数表示で `OLLAMA_BASE_URL` を確認する
- `WEBUI_SECRET_KEY` の実値は表示・共有しない

### PVC の確認

```text
Administrator
  └─ Storage
      └─ PersistentVolumeClaims
          └─ webui-data
```

確認することです。

- Status が `Bound`
- Requested Storage が 10Gi
- Access Mode が RWO

### Service の確認

```text
Administrator
  └─ Networking
      └─ Services
          └─ open-webui
```

確認することです。

- selector が `app=open-webui`
- Service Port が 8080
- targetPort が `http`
- Endpoints が Open WebUI Pod を指している

***

## 6-11. よくある状態と初動

| 症状 | 最初に確認すること | 主な原因 |
|---|---|---|
| Open WebUI Pod が `Pending` | `oc describe pod -l app=open-webui` | PVC、CPU / memory、ノード制約 |
| `ImagePullBackOff` | Pod Events | image 名、`main`、GHCR 到達性 |
| `CrashLoopBackOff` | `oc logs deployment/open-webui --previous` | Secret、権限、PVC mountPath |
| `webui-data` が `Pending` のまま | `oc describe pvc webui-data` | Pod 未配置、LVM provisioning |
| Ollama モデルが見えない | `OLLAMA_BASE_URL`、Ollama Endpoints | URL 誤り、Service selector、内部通信 |
| Open WebUI Service の Endpoints が空 | Pod label / Service selector | `app=open-webui` の不一致 |

***

## 章の確認チェック

### CLI

- [ ] `open-webui-secret` を作成した
- [ ] `oc apply -f manifests/03-open-webui.yaml` を実行した
- [ ] `oc rollout status deployment/open-webui` が成功した
- [ ] `oc get pvc` で `webui-data` が `Bound` になったことを確認した
- [ ] `oc get endpoints open-webui` を確認した
- [ ] Open WebUI Pod から `http://ollama:11434/api/tags` を取得した

### GUI

- [ ] Administrator → Workloads → Secrets で Secret の存在を確認した
- [ ] Open WebUI Deployment と Pod を確認した
- [ ] Open WebUI Pod の Logs と Events を確認した
- [ ] `webui-data` PVC が `Bound` であることを確認した
- [ ] Open WebUI Service の selector と Endpoints を確認した

### 理解

- [ ] Secret を使う理由を説明できる
- [ ] `secretKeyRef` の役割を説明できる
- [ ] Pod 内の `localhost` が Open WebUI 自身を指すことを説明できる
- [ ] `OLLAMA_BASE_URL=http://ollama:11434` を使う理由を説明できる
- [ ] Open WebUI と Ollama のデータを別 PVC にする理由を説明できる

***

# 第7章：Route による Open WebUI の外部公開と E2E 確認

> 所要時間：25〜35 分  
> 前提：Open WebUI Pod が `Running` / `Ready 1/1` であり、`Service/open-webui` の Endpoints が存在すること  
> この章では、Open WebUI のみを HTTPS で外部公開し、ブラウザから Ollama 推論を実行します。

## 目的

OpenShift Route を作成し、クラスタ外部のブラウザから Open WebUI へアクセスできるようにします。

その後、Open WebUI、Ollama Service、Ollama Pod、Gemma 4 モデルまでの一連の経路が動作することを確認します。

## 期待される結果

この章の完了時点で、以下を満たします。

- `Route/open-webui` が作成されている
- Route に Host 名が自動生成されている
- Route の TLS termination が `edge` である
- ブラウザで Open WebUI を開ける
- 初回管理者ユーザーを作成できる
- `gemma4:e2b` を選択できる
- ブラウザから送ったプロンプトに応答できる
- Open WebUI と Ollama のログでリクエスト処理を確認できる

***

## 7-1. なぜ Open WebUI だけを公開するのか

今回の公開方針です。

```text
外部公開する:
  Route/open-webui

外部公開しない:
  Service/ollama
```

```text
利用者ブラウザ
  |
  | HTTPS
  v
Route/open-webui
  |
  v
Service/open-webui
  |
  v
Open WebUI Pod
  |
  | クラスタ内部通信
  v
Service/ollama
  |
  v
Ollama Pod
```

### 講師が説明すること

> Open WebUI は利用者向けの認証・画面を提供するアプリケーションです。  
> Ollama は推論 API であり、今回の構成では Open WebUI からだけ使える内部 Service として扱います。

> Ollama を直接外部公開すると、API を誰が利用できるか、推論リソースをどう保護するか、認証をどう行うかを別途考える必要があります。  
> この研修では、公開窓口を Open WebUI のみに限定します。

***

## 7-1-2. Route 到達性の事前準備（hosts ファイル）

このラボの作業端末では、OpenShift の `apps` ドメインを名前解決する内部 DNS を利用できません。

そのため、Open WebUI Route へブラウザからアクセスする前に、作業端末の hosts ファイルへ **Route Host 名**と **Ingress Router のアクセス先 IP アドレス**を登録しておきます。

ブラウザは Windows Server 2019 上で使うため、WSL の `/etc/hosts` ではなく、Windows 側の hosts ファイルを編集します。

```text
C:\Windows\System32\drivers\etc\hosts
```

### 記載例

Route Host が以下だったとします。

```text
open-webui-ollama-handson.apps.<CLUSTER_DOMAIN>
```

Ingress Router のアクセス先 IP が `192.0.2.50` の場合、hosts ファイルへ次の行を追記します。

```text
192.0.2.50  open-webui-ollama-handson.apps.<CLUSTER_DOMAIN>
```

> IP アドレスと Host 名は、実環境の値に置き換えます。

### 名前解決の経路

```text
通常環境:
ブラウザ
  |
  v
DNS
  |
  v
Ingress Router IP
  |
  v
Route/open-webui

本ラボ:
ブラウザ
  |
  v
Windows hosts ファイル
  |
  v
Ingress Router IP
  |
  v
Route/open-webui
```

### 注意点

- ブラウザが Windows 上で動くため、登録先は **Windows の hosts ファイル**です
- WSL の `/etc/hosts` へ書いても、Windows ブラウザの名前解決には使われません
- hosts 編集時は、エディタを管理者として起動します
- hosts ファイルを `hosts.txt` として保存しないようにします
- Route Host を事前に決める場合は、Route YAML で `spec.host` を固定するか、Route 作成後に Host 名を取得して hosts へ追記します
- ラボ終了後、不要になった hosts エントリーは削除します（第10章）

### 講師が説明すること

> OpenShift Route は Host 名でアクセスします。通常は DNS が Route Host 名を Ingress Router の IP アドレスへ解決します。  
> このラボでは作業端末から内部 DNS を利用できないため、Windows の hosts ファイルで同じ名前解決を一時的に実現します。  
> Route URL をブラウザで開けないときは、Route、Service、Pod だけでなく、**作業端末の名前解決**も確認対象になります。

***

## 7-2. Route を作成する

以下を実行します。

```bash
oc create route edge --service=open-webui --port=http
```

### コマンドの意味

| 要素 | 意味 |
|---|---|
| `oc create route` | OpenShift Route を作成する |
| `edge` | edge TLS termination を使う |
| `--service=open-webui` | 転送先 Service を指定する |
| `--port=http` | Service の名前付きポート `http` を指定する |

Service の YAML では、以下のポート名を定義しています。

```yaml
ports:
  - name: http
    port: 8080
    targetPort: http
```

そのため Route の `--port=http` は、`Service/open-webui` の `http` ポートを参照します。

### 期待出力

```text
route.route.openshift.io/open-webui created
```

***

## 7-3. edge TLS termination を理解する

今回使う Route は edge termination です。

```text
ブラウザ
  |
  | HTTPS
  v
OpenShift Ingress Controller / Router
  |
  | HTTP
  v
Service/open-webui
  |
  v
Open WebUI Pod
```

edge Route では、TLS は OpenShift の Ingress Controller / Router で終端され、その後の Service・Pod への通信は HTTP で転送されます。

### この研修で edge Route を使う理由

| 選択肢 | この研修での扱い |
|---|---|
| edge | 使用する |
| passthrough | 使用しない |
| reencrypt | 使用しない |

| termination | 特徴 |
|---|---|
| edge | Router が TLS を終端し、バックエンドへ HTTP 転送 |
| passthrough | TLS を Pod まで通す |
| reencrypt | Router で一度 TLS を終端し、バックエンドへ再暗号化して転送 |

### 証明書について

この研修では Route にカスタム証明書を設定しません。

```text
Route:
  custom certificate: 指定しない

Ingress Controller:
  default certificate: 使用する
```

edge Route で証明書を指定しない場合、Ingress Controller の default certificate が使用されます。

### 講師が説明すること

> ブラウザで証明書警告が出る場合は、クラスタの default certificate をクライアント端末が信頼していない可能性があります。  
> 研修用環境では、講師が事前にブラウザから Route へ到達できることを確認しておきます。

***

## 7-4. Route の状態を CLI で確認する

以下を実行します。

```bash
oc get route
```

### 期待出力のイメージ

```text
NAME        HOST/PORT                                        PATH   SERVICES    PORT   TERMINATION
open-webui  open-webui-ollama-handson.apps.<cluster-domain>         open-webui  http   edge
```

Route の詳細を確認します。

```bash
oc describe route open-webui
```

### 確認する項目

| 項目 | 期待値 |
|---|---|
| Name | `open-webui` |
| Host | 自動生成された Host 名 |
| Service | `open-webui` |
| Port | `http` |
| TLS Termination | `edge` |

Host 名だけを変数へ保存します。

```bash
ROUTE_HOST=$(oc get route open-webui -o jsonpath='{.spec.host}')
```

表示します。

```bash
echo "https://${ROUTE_HOST}"
```

### コマンドの意味

| コマンド要素 | 意味 |
|---|---|
| `-o jsonpath=...` | リソースの JSON から指定項目だけを取得する |
| `.spec.host` | Route に設定された Host 名 |
| `ROUTE_HOST=...` | WSL シェル変数へ保存する |
| `echo` | ブラウザで開く HTTPS URL を表示する |

***

## 7-5. GUI で Route を確認する

Administrator パースペクティブで以下を開きます。

```text
Administrator
  └─ Networking
      └─ Routes
          └─ open-webui
```

### GUI で確認すること

1. Project が `ollama-handson` であることを確認する
2. `Route/open-webui` が表示されることを確認する
3. Host 名を確認する
4. Service が `open-webui` であることを確認する
5. Target Port が `http` であることを確認する
6. TLS termination が `edge` であることを確認する
7. Route の URL または Host を開く

### 講師が説明すること

> Route は外部からの HTTP / HTTPS リクエストを、OpenShift 内部の Service へ転送します。  
> Service はクラスタ内部の接続先、Route はクラスタ外部からの入口です。

```text
Service:
  クラスタ内部の安定した接続先

Route:
  クラスタ外部から HTTP / HTTPS で入る入口
```

***

## 7-6. ブラウザで Open WebUI を開く

WSL 上で表示した URL、または GUI の Route URL をブラウザで開きます。

```text
https://open-webui-ollama-handson.apps.<cluster-domain>
```

### 初回アクセス時に行うこと

1. Open WebUI の初回登録画面を開く
2. 管理者ユーザーを作成する
3. ログインする
4. モデル選択画面を開く
5. `gemma4:e2b` が表示されることを確認する

### 期待結果

```text
Open WebUI
  |
  +-- モデル一覧
        |
        +-- gemma4:e2b
```

モデルが表示されない場合は、Open WebUI から Ollama Service への通信を確認します。

```bash
oc exec -it deploy/open-webui -- sh -c \
  'wget -qO- http://ollama:11434/api/tags || true'
```

***

## 7-7. ブラウザから推論を実行する

Open WebUI で `gemma4:e2b` を選択し、以下のプロンプトを送信します。

```text
OpenShift を一文で説明してください。
```

### 期待結果

Open WebUI に Gemma 4 からの応答が表示されます。

```text
ブラウザ
  |
  v
Route/open-webui
  |
  v
Open WebUI Pod
  |
  v
Service/ollama
  |
  v
Ollama Pod + gemma4:e2b
  |
  v
Open WebUI に応答表示
```

### 講師が説明すること

> この一回のチャット送信で、Route、Service、Open WebUI Pod、Ollama Service、Ollama Pod、GPU、PVC 上のモデルまでを通る End-to-End の動作確認ができています。

***

## 7-8. Logs で E2E の処理を確認する

ブラウザから推論を送信した後、Open WebUI と Ollama のログを確認します。

```bash
oc logs deployment/open-webui --tail=100
oc logs deployment/ollama --tail=100
```

### GUI で確認する場合

```text
Administrator
  └─ Workloads
      └─ Pods
          ├─ open-webui-<Suffix>
          │   └─ Logs
          └─ ollama-<Suffix>
              └─ Logs
```

### 確認すること

| 対象 | 見る内容 |
|---|---|
| Open WebUI Logs | ブラウザからのリクエスト、Ollama API 呼び出し、接続エラー |
| Ollama Logs | モデル推論、エラー、処理状況 |
| Route | Host、Service、Port、TLS |
| Service | Endpoints が存在すること |
| Pod | `Running` / `Ready` 状態 |

***

## 7-9. Route 接続障害の調査

ブラウザで Open WebUI を開けない場合は、以下の順番で確認します。

```text
Route
  ↓
Service
  ↓
Endpoints
  ↓
Open WebUI Pod
```

### CLI の確認手順

```bash
oc get route open-webui
oc describe route open-webui

oc get svc open-webui
oc get endpoints open-webui

oc get pod -l app=open-webui
oc describe pod -l app=open-webui
oc logs deployment/open-webui
```

### GUI の確認手順

1. Administrator → Networking → Routes → `open-webui`
2. Route の Host、Service、Port、TLS を確認する
3. Administrator → Networking → Services → `open-webui`
4. selector と Endpoints を確認する
5. Administrator → Workloads → Pods → Open WebUI Pod
6. Status、Events、Logs を確認する

### 主な症状

| 症状 | 最初に確認すること |
|---|---|
| Route URL が開けない | Host、ブラウザ到達性、証明書、Ingress |
| 503 などが表示される | Service Endpoints、Open WebUI Pod の Ready |
| Endpoints が空 | Service selector と Pod label |
| Open WebUI 画面は開くが推論できない | Ollama Service、`OLLAMA_BASE_URL`、Ollama Endpoints |
| 証明書警告が出る | default certificate、クライアントの信頼設定 |

***

## Docker Compose と OpenShift の対比

| Docker Compose | OpenShift |
|---|---|
| `ports: "3000:8080"` | Service + Route |
| `http://localhost:3000` | `https://<Route Host>` |
| WSL ホストのポート公開 | Ingress Controller 経由の公開 |
| Docker network 内部通信 | OpenShift Service 経由の内部通信 |
| `http://ollama:11434` | `http://ollama:11434` |
| Docker Ollama API は `localhost:11435` | OpenShift API 確認は `oc port-forward` で `localhost:11434` |

***

## 章の確認チェック

### CLI

- [ ] `oc create route edge --service=open-webui --port=http` を実行した
- [ ] `oc get route open-webui` を確認した
- [ ] `oc describe route open-webui` を確認した
- [ ] `ROUTE_HOST` から HTTPS URL を取得した
- [ ] Open WebUI と Ollama の Logs を確認した

### GUI

- [ ] Administrator → Networking → Routes を開いた
- [ ] Route の Host、Service、Target Port、TLS termination を確認した
- [ ] Route URL をブラウザで開いた
- [ ] 初回管理者を作成した
- [ ] `gemma4:e2b` を選択して推論した

### 理解

- [ ] Service と Route の役割の違いを説明できる
- [ ] edge TLS termination の意味を説明できる
- [ ] Ollama に Route を作らない理由を説明できる
- [ ] ブラウザの推論が通る経路を説明できる
- [ ] Route 障害時に Route → Service → Endpoints → Pod の順で調査できる

***

# 第8章：可観測性と基本的な障害対応

> 所要時間：35〜45 分  
> 前提：Ollama、Open WebUI、Service、PVC、Route を作成済みであり、ブラウザから Open WebUI にアクセスできること  
> この章では、正常な状態を確認する方法と、意図的な障害の調査・復旧を学びます。

## 目的

OpenShift 上で問題が起きたときに、推測だけで対応せず、リソースの状態、Events、Logs、Service、Endpoints、Route を順序立てて確認できるようにします。

また、Service selector の不一致を意図的に発生させ、Open WebUI から Ollama へ接続できなくなる原因と復旧方法を体験します。

## 期待される結果

この章の完了時点で、以下を満たします。

- `oc get`、`oc describe`、`oc logs`、`oc get events` の使い分けを説明できる
- Pod 障害時に Logs と Events を確認できる
- Service selector と Pod label の関係を説明できる
- Endpoints が Service の転送先 Pod を示すことを説明できる
- selector 不一致で Endpoints が空になることを確認できる
- 正常な manifest を再適用して、Service 接続を復旧できる
- Route → Service → Endpoints → Pod の経路で障害を調査できる

Pod の障害調査では、`oc describe pod` の Events と、現在・前回コンテナの `oc logs` を確認することが基本です。

***

## 8-1. 正常な状態を基準として確認する

障害を調査する前に、正常な状態を確認します。

```bash
oc get deploy,pod,svc,pvc,route
```

### 期待状態

```text
Deployment:
  ollama       READY 1/1
  open-webui   READY 1/1

Pod:
  ollama-<Suffix>       Running   1/1
  open-webui-<Suffix>   Running   1/1

PVC:
  ollama-data   Bound
  webui-data    Bound

Service:
  ollama
  open-webui

Route:
  open-webui
```

続けて Endpoints を確認します。

```bash
oc get endpoints
```

### 期待状態

```text
NAME        ENDPOINTS
ollama      <OLLAMA_POD_IP>:11434
open-webui  <WEBUI_POD_IP>:8080
```

### 講師が説明すること

> Service は selector に一致する Pod を探し、EndPoints として登録します。  
> Endpoints が空なら、Service は転送先を持たないため、アプリケーション通信はできません。

Service の selector は、一致する label を持つ Pod へトラフィックを転送するための条件です。

***

## 8-2. 基本調査コマンドを確認する

```bash
oc get pods -o wide
oc get deploy,svc,pvc,route
oc get events --sort-by=.lastTimestamp
oc describe pod <POD_NAME>
oc logs <POD_NAME>
oc logs <POD_NAME> --previous
oc get endpoints
```

## コマンドの使い分け

| コマンド | 目的 | 主に見るもの |
|---|---|---|
| `oc get` | 一覧と大まかな状態 | Running、Pending、Bound、Ready |
| `oc get -o wide` | ノード・Pod IP も確認 | 配置先、Pod IP |
| `oc describe` | 詳細情報と Events | scheduling、image pull、mount、再起動理由 |
| `oc logs` | アプリケーションログ | 起動エラー、接続エラー、推論ログ |
| `oc logs --previous` | 前回終了したコンテナのログ | CrashLoopBackOff の直前のエラー |
| `oc get events` | Project 全体の時系列イベント | Warning、作成、配置、失敗 |
| `oc get endpoints` | Service の転送先 | Pod IP、ポート、空かどうか |

### `oc get events` の使い方

```bash
oc get events --sort-by=.lastTimestamp
```

新しい Event が下に表示されます。

Warning に絞る場合は、以下も使えます。

```bash
oc get events --sort-by=.lastTimestamp | grep Warning
```

***

## 8-3. GUI での基本調査順序

Administrator パースペクティブを使います。

```text
1. Workloads → Pods
2. 対象 Pod の Events
3. 対象 Pod の Logs
4. Workloads → Deployments
5. Storage → PersistentVolumeClaims
6. Networking → Services
7. Networking → Routes
```

### Pod で確認すること

```text
Administrator
  └─ Workloads
      └─ Pods
          └─ 対象 Pod
              ├─ Details
              ├─ Events
              ├─ Logs
              ├─ YAML
              └─ Terminal
```

| GUI 項目 | 何を確認するか |
|---|---|
| Status | `Running`、`Pending`、`CrashLoopBackOff` など |
| Ready | `1/1` か、`0/1` か |
| Events | pull、scheduling、PVC、権限などの失敗 |
| Logs | アプリケーションのエラー |
| YAML | image、環境変数、volumeMount、resources |
| Terminal | Pod 内から API や設定を確認する |

***

## 8-4. 症状別の初動

| 症状 | 最初に見る場所 | 主な原因 |
|---|---|---|
| Pod が `Pending` | `oc describe pod`、Events、PVC | GPU / CPU / memory 不足、PVC、ノード制約 |
| PVC が `Pending` | `oc describe pvc`、Pod Events | WaitForFirstConsumer、LVM 容量、Pod 未配置 |
| `ImagePullBackOff` | Pod Events | image 名、tag、レジストリ到達性 |
| `CrashLoopBackOff` | `oc logs --previous`、Events | 起動設定、権限、環境変数、mountPath |
| Open WebUI にモデルが出ない | Ollama Endpoints、`OLLAMA_BASE_URL` | Service 名、selector、内部通信 |
| Route に接続できない | Route → Service → Endpoints → Pod | hosts、Route、Service、Pod Ready |
| 推論が失敗する | Open WebUI Logs、Ollama Logs | Ollama API、GPU、モデル、接続 |

***

# 障害演習：Service selector 不一致

## 8-5. 演習の目的

Service selector が Pod label と一致しない場合、Service は転送先 Pod を発見できず、Endpoints が空になることを確認します。

この状態では Open WebUI から Ollama へ接続できなくなります。

## 期待される結果

障害注入後は、以下の状態になります。

```text
Ollama Pod:
  Running
  Ready 1/1

Service/ollama:
  存在する

Endpoints/ollama:
  空

Open WebUI:
  Ollama API へ接続できない
```

***

## 8-6. 正常な状態を記録する

障害を入れる前に、正常な状態を確認します。

```bash
oc get pod -l app=ollama --show-labels
oc get svc ollama
oc get endpoints ollama
```

### 期待結果

```text
Ollama Pod label:
  app=ollama
```

```text
Service selector:
  app=ollama
```

```text
Endpoints:
  <OLLAMA_POD_IP>:11434
```

この三つが一致している状態が正常です。

```text
Service selector
  app=ollama
       |
       v
Pod label
  app=ollama
       |
       v
Endpoints
  <POD_IP>:11434
```

***

## 8-7. 意図的な障害を適用する

以下のファイルを使います。

`exercises/01-service-wrong-selector.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ollama
spec:
  selector:
    app: ollama-incorrect
  ports:
    - name: http
      port: 11434
      targetPort: http
```

適用します。

```bash
oc apply -f exercises/01-service-wrong-selector.yaml
```

### 障害として何が起きるか

```text
Service selector:
  app=ollama-incorrect

Ollama Pod label:
  app=ollama

結果:
  selector が一致しない
  ↓
  Service が Pod を見つけられない
  ↓
  Endpoints が空になる
  ↓
  Open WebUI が Ollama API へ接続できない
```

***

## 8-8. CLI で障害を調査する

まず Endpoints を確認します。

```bash
oc get endpoints ollama
```

### 期待結果

```text
NAME     ENDPOINTS   AGE
ollama   <none>      10s
```

Service の詳細を確認します。

```bash
oc describe svc ollama
```

Pod の label を確認します。

```bash
oc get pod -l app=ollama --show-labels
```

Service の YAML を確認します。

```bash
oc get svc ollama -o yaml
```

### 講師が説明すること

> Pod 自体は Running です。  
> しかし Service selector が Pod label と一致しないため、Service は転送先を持てません。

> Pod が Running であっても、Service Endpoints が空なら、アプリケーション間通信は失敗します。

***

## 8-9. GUI で障害を調査する

Administrator パースペクティブで確認します。

### Ollama Pod の確認

```text
Administrator
  └─ Workloads
      └─ Pods
          └─ Ollama Pod
```

確認することです。

- Pod が `Running`
- Ready が `1/1`
- Pod label が `app=ollama`
- Pod の Events に異常がない

### Ollama Service の確認

```text
Administrator
  └─ Networking
      └─ Services
          └─ ollama
```

確認することです。

- selector が `app=ollama-incorrect`
- Endpoints が空
- Service 自体は存在するが、転送先がない

### Open WebUI の確認

```text
Administrator
  └─ Workloads
      └─ Pods
          └─ Open WebUI Pod
              └─ Logs
```

確認することです。

- Ollama API への接続エラー
- モデル一覧取得失敗
- Open WebUI 画面でモデルが表示されない、または推論できない

***

## 8-10. Open WebUI Pod から接続を確認する

以下を実行します。

```bash
oc exec -it deploy/open-webui -- sh -c \
  'wget -qO- http://ollama:11434/api/tags || true'
```

### 期待結果

障害中は、モデル一覧 JSON が返りません。

### 講師が説明すること

> Open WebUI の設定は正しくても、`Service/ollama` の Endpoints が空であれば、接続先 Pod が存在しないため通信できません。

***

## 8-11. 正常な Service へ復旧する

以下を実行します。

```bash
oc apply -f manifests/02-ollama.yaml
```

このファイルには、正しい `Service/ollama` 定義が含まれています。

```yaml
selector:
  app: ollama
```

復旧を確認します。

```bash
oc get endpoints ollama
oc get svc ollama
oc exec -it deploy/open-webui -- sh -c \
  'wget -qO- http://ollama:11434/api/tags || true'
```

### 期待結果

```text
Endpoints:
  <OLLAMA_POD_IP>:11434
```

Open WebUI Pod からモデル一覧 JSON が取得できます。

ブラウザで Open WebUI を再読み込みし、`gemma4:e2b` が利用できることを確認します。

***

## 8-12. Route 障害の確認順序

Route で Open WebUI に接続できない場合は、以下の順番で確認します。

```text
作業端末の hosts
  ↓
Route
  ↓
Service/open-webui
  ↓
Endpoints/open-webui
  ↓
Open WebUI Pod
```

### CLI

```bash
oc get route open-webui
oc describe route open-webui

oc get svc open-webui
oc get endpoints open-webui

oc get pod -l app=open-webui
oc describe pod -l app=open-webui
oc logs deployment/open-webui
```

### hosts の確認

ブラウザが Windows Server 2019 上で動くため、確認対象は Windows の hosts ファイルです。

```text
C:\Windows\System32\drivers\etc\hosts
```

Route Host 名が Ingress Router の IP アドレスへ対応付けられていることを確認します。

```text
<INGRESS_ROUTER_IP>  <ROUTE_HOST>
```

***

## 章の確認チェック

### CLI

- [ ] `oc get deploy,pod,svc,pvc,route` を実行した
- [ ] `oc get events --sort-by=.lastTimestamp` を実行した
- [ ] `oc describe pod` を実行した
- [ ] `oc logs --previous` の用途を確認した
- [ ] `oc get endpoints` を確認した
- [ ] selector 不一致の障害 manifest を適用した
- [ ] Endpoints が空になることを確認した
- [ ] 正常 manifest の再適用で復旧した

### GUI

- [ ] Pod の Status、Events、Logs を確認した
- [ ] PVC の Status を確認した
- [ ] Service selector と Endpoints を確認した
- [ ] Route の Service、Port、Host、TLS を確認した
- [ ] 障害中に Ollama Pod が正常でも Service が通信できないことを確認した

### 理解

- [ ] `oc get`、`oc describe`、`oc logs`、Events の役割を説明できる
- [ ] `oc logs --previous` を使う場面を説明できる
- [ ] Service selector と Pod label の関係を説明できる
- [ ] Endpoints が空である意味を説明できる
- [ ] Route 障害を hosts → Route → Service → Endpoints → Pod の順に調査できる

***

# 第9章：自己修復とデータ永続化の確認

> 所要時間：20〜30 分  
> 前提：Ollama、Open WebUI、PVC、Service、Route が正常に動作し、ブラウザから `gemma4:e2b` の推論ができること  
> この章では、Pod を意図的に削除し、Deployment の自己修復と PVC の永続化を確認します。

## 目的

OpenShift では Pod を永続的なサーバーとして扱わず、一時的な実行単位として扱います。

この章では Ollama Pod と Open WebUI Pod を削除し、以下を確認します。

- Deployment が新しい Pod を自動作成する
- Pod 名と Pod IP は変わる
- Service 名は変わらない
- Ollama モデルが PVC に残る
- Open WebUI の状態データが PVC に残る
- Route を再作成しなくても Open WebUI へ再アクセスできる

OpenShift は、Deployment の変更や再作成時に既存 Pod を終了させ、新しい Pod を作成する仕組みを持ちます。

## 期待される結果

Ollama Pod を削除した後、以下の状態になります。

```text
削除前:
  ollama-aaaaa-bbbbb
  Pod IP: 10.x.x.x

削除後:
  ollama-ccccc-ddddd
  Pod IP: 10.y.y.y
```

一方、以下は変わりません。

```text
Deployment/ollama
Service/ollama
PVC/ollama-data
```

新しい Ollama Pod が Ready になった後、以下が成功します。

```bash
oc exec -it deploy/ollama -- ollama list
```

出力に `gemma4:e2b` が残っていれば、モデルデータは PVC に保存されていたことが確認できます。

***

## 9-1. 削除前の状態を記録する

まず、Ollama の現在の状態を確認します。

```bash
oc get pod -l app=ollama -o wide
oc get svc ollama
oc get pvc ollama-data
oc get endpoints ollama
```

### 確認する項目

| リソース | 確認する内容 |
|---|---|
| Ollama Pod | Pod 名、Status、Pod IP、Node |
| Deployment | `READY 1/1` |
| Service | `ollama` が存在する |
| PVC | `ollama-data` が `Bound` |
| Endpoints | Ollama Pod IP と 11434 |

Pod 名を変数へ保存しておくと、削除前後の比較に使えます。

```bash
OLLAMA_POD=$(oc get pod -l app=ollama -o jsonpath='{.items.metadata.name}')
echo "${OLLAMA_POD}"
```

### 講師が説明すること

> 今から Pod を削除しますが、Deployment は削除しません。  
> Deployment が `replicas: 1` を維持するため、新しい Pod が自動的に作成されます。

> そのため、Pod は「消えてもよい」一時的な実行単位として扱えます。

***

## 9-2. Ollama Pod を削除する

以下を実行します。

```bash
oc delete pod "${OLLAMA_POD}"
```

または、Pod 名を直接指定します。

```bash
oc delete pod <OLLAMA_POD_NAME>
```

### 期待出力

```text
pod "<OLLAMA_POD_NAME>" deleted
```

この時点では Pod が削除されますが、Deployment は残っています。

***

## 9-3. 新しい Ollama Pod の作成を監視する

以下を実行します。

```bash
oc get pods -l app=ollama -w
```

### コマンドの意味

| 要素 | 意味 |
|---|---|
| `oc get pods` | Pod の状態を表示する |
| `-l app=ollama` | Ollama Pod のみに絞る |
| `-w` | 状態変化を継続的に監視する |

### 期待する状態遷移

```text
NAME                      READY   STATUS
ollama-aaaaa-bbbbb        1/1     Terminating
ollama-ccccc-ddddd        0/1     Pending
ollama-ccccc-ddddd        0/1     ContainerCreating
ollama-ccccc-ddddd        1/1     Running
```

`Ctrl+C` で監視を終了します。

### 講師が説明すること

> 新しい Pod は、GPU、PVC、コンテナイメージ、アプリケーション初期化の準備を行うため、すぐに `Running` にならない場合があります。

> `Pending`、`ContainerCreating`、`Running` の状態変化を確認することで、OpenShift が期待状態を復元していることを観察できます。

***

## 9-4. 削除前後の Pod を比較する

新しい Pod が Ready になった後、以下を実行します。

```bash
oc get pod -l app=ollama -o wide
```

### 比較する項目

| 項目 | 期待結果 |
|---|---|
| Pod 名 | 変わる |
| Pod IP | 変わる可能性が高い |
| Node | 同じ場合も異なる場合もある |
| Deployment 名 | 変わらない |
| Service 名 | 変わらない |
| PVC 名 | 変わらない |

### 講師が説明すること

> Pod 名と Pod IP は安定した接続先ではありません。  
> そのため Open WebUI は Pod IP ではなく、常に `Service/ollama` の名前で接続します。

```text
変わる可能性がある:
  Pod 名
  Pod IP
  配置ノード

変わらない:
  Service/ollama
  PVC/ollama-data
  Deployment/ollama
```

***

## 9-5. Service Endpoints が新しい Pod を指すことを確認する

以下を実行します。

```bash
oc get endpoints ollama
```

### 期待結果

新しい Ollama Pod の IP アドレスと 11434 が表示されます。

```text
NAME     ENDPOINTS
ollama   <NEW_OLLAMA_POD_IP>:11434
```

### 講師が説明すること

> Service は label selector を使い、新しい Pod を自動的に Endpoints へ登録します。

> Open WebUI は `http://ollama:11434` に接続し続けるだけで、新しい Pod の IP を知る必要はありません。

***

## 9-6. Ollama モデルが残っていることを確認する

以下を実行します。

```bash
oc exec -it deploy/ollama -- ollama list
```

### 期待結果

```text
NAME          ID            SIZE
gemma4:e2b    <MODEL_ID>    <SIZE>
```

モデルが表示されれば、Pod 削除後もモデルデータが残っていることを確認できます。

### PVC の状態も確認する

```bash
oc get pvc ollama-data
```

期待結果です。

```text
NAME          STATUS   VOLUME
ollama-data   Bound    pvc-<UUID>
```

### 講師が説明すること

> Pod は削除されましたが、PVC は削除されていません。  
> 新しい Pod が同じ PVC を `/root/.ollama` へマウントしたため、モデルを再 pull せずに利用できます。

***

## 9-7. GUI で Ollama の自己修復を確認する

Administrator パースペクティブで以下を開きます。

```text
Administrator
  └─ Workloads
      └─ Pods
```

### GUI で確認すること

1. Ollama Pod を削除前に開く
2. Actions から Pod 削除を実行する、または CLI で削除する
3. 旧 Pod が `Terminating` になることを確認する
4. 新しい Ollama Pod が作成されることを確認する
5. 新 Pod の名前が異なることを確認する
6. 新 Pod が `Running`、`Ready 1/1` になることを確認する

Deployment も確認します。

```text
Administrator
  └─ Workloads
      └─ Deployments
          └─ ollama
```

確認する項目です。

- Desired Replicas が 1
- Current Replicas が 1
- Available Replicas が 1

PVC も確認します。

```text
Administrator
  └─ Storage
      └─ PersistentVolumeClaims
          └─ ollama-data
```

確認する項目です。

- PVC が残っている
- Status が `Bound`
- 要求容量が 30Gi

***

## 9-8. Open WebUI Pod の自己修復を確認する

次に Open WebUI Pod も削除します。

削除前に確認します。

```bash
oc get pod -l app=open-webui -o wide
oc get pvc webui-data
oc get route open-webui
```

Pod 名を変数へ保存します。

```bash
WEBUI_POD=$(oc get pod -l app=open-webui -o jsonpath='{.items.metadata.name}')
echo "${WEBUI_POD}"
```

Pod を削除します。

```bash
oc delete pod "${WEBUI_POD}"
```

新しい Pod の作成を監視します。

```bash
oc get pods -l app=open-webui -w
```

新 Pod が Ready になった後、以下を確認します。

```bash
oc get pvc webui-data
oc get endpoints open-webui
oc get route open-webui
```

### 期待結果

| リソース | 期待状態 |
|---|---|
| Open WebUI Pod | 新しい Pod 名、`Running` / `Ready 1/1` |
| `webui-data` PVC | `Bound` のまま |
| `Service/open-webui` | 存在する |
| `Endpoints/open-webui` | 新 Pod の IP:8080 |
| `Route/open-webui` | 同じ Host 名で存在する |

***

## 9-9. ブラウザから状態保持を確認する

Open WebUI Route を再度ブラウザで開きます。

```bash
ROUTE_HOST=$(oc get route open-webui -o jsonpath='{.spec.host}')
echo "https://${ROUTE_HOST}"
```

### 確認すること

- Route Host 名が変わっていない
- Open WebUI のログイン画面またはログイン済み画面が開く
- モデル一覧に `gemma4:e2b` が表示される
- 必要に応じて、作成済みのユーザー・設定・会話履歴が残っている

### 講師が説明すること

> Open WebUI Pod も再作成されましたが、`webui-data` PVC を再マウントしています。  
> そのため、アプリケーションがその保存先を正しく使っていれば、状態データを引き継げます。

> `WEBUI_SECRET_KEY` を Secret で固定しているため、Pod 再作成時にログインセッションや暗号化された連携情報の扱いが不安定になることを避けられます。

***

## 9-10. Docker Compose と OpenShift の対比

| 確認項目 | Docker Compose | OpenShift |
|---|---|---|
| コンテナ停止後の再作成 | `docker compose up -d` などで実施 | Deployment が自動作成 |
| 実行単位 | コンテナ | Pod |
| 安定接続先 | Compose Service 名 | OpenShift Service 名 |
| データ保持 | named volume | PVC |
| データ確認 | `docker volume ls` | `oc get pvc` |
| 再作成後のモデル | volume が残れば再利用 | PVC が残れば再利用 |
| IP の扱い | コンテナ IP に依存しない | Pod IP に依存しない |

***

## 章の確認チェック

### CLI

- [ ] Ollama Pod の削除前後で Pod 名を比較した
- [ ] `oc get pods -l app=ollama -w` で再作成を観察した
- [ ] `oc get endpoints ollama` で新 Pod が登録されたことを確認した
- [ ] `ollama list` で `gemma4:e2b` が残っていることを確認した
- [ ] Open WebUI Pod も削除・再作成した
- [ ] Open WebUI Route が変わらず残っていることを確認した

### GUI

- [ ] Ollama Pod の Terminating と新 Pod 作成を確認した
- [ ] Ollama Deployment の replicas が最終的に 1 になったことを確認した
- [ ] `ollama-data` PVC が `Bound` のままであることを確認した
- [ ] Open WebUI Pod の再作成を確認した
- [ ] `webui-data` PVC が `Bound` のままであることを確認した

### 理解

- [ ] Pod が一時的な実行単位であることを説明できる
- [ ] Deployment が自己修復する理由を説明できる
- [ ] Pod IP を接続先として使わない理由を説明できる
- [ ] Service を使う理由を説明できる
- [ ] PVC がモデル・状態データの永続化に必要な理由を説明できる

***

# 第10章：後片付けとラボの振り返り

> 所要時間：15〜20 分  
> この章では、ラボ環境を安全に終了するため、作成した OpenShift リソースと作業端末の hosts 設定を確認・削除します。  
> **Project を削除すると、その Project 内のリソースはまとめて削除されます。削除前に必要な manifest や確認結果を保存してください。**

## 目的

この章で、以下を実施します。

- 作成したリソースを一覧で確認する
- 必要に応じて manifest を退避する
- Project 単位、または個別リソース単位で削除する
- PVC と PV の扱いを確認する
- Windows hosts ファイルの一時エントリーを削除する
- ラボ全体の学習事項を振り返る

OpenShift Project を削除すると、状態は `Active` から `Terminating` へ移行し、Project 内のコンテンツを削除した後に Project 自体が削除されます。

## 10-1. 削除前の確認

まず、現在の Project とリソースを確認します。

```bash
oc project
oc status

oc get deploy,pod,svc,route,pvc,secret
oc get events --sort-by=.lastTimestamp
```

### 確認すること

| リソース | 想定される名称 |
|---|---|
| Deployment | `ollama`、`open-webui` |
| Service | `ollama`、`open-webui` |
| Route | `open-webui` |
| PVC | `ollama-data`、`webui-data` |
| Secret | `open-webui-secret` など |
| Pod | Ollama Pod、Open WebUI Pod |

### 退避が必要なもの

削除前に、以下を手元の Git リポジトリなどへ保存します。

```text
manifests/
  01-pvc.yaml
  02-ollama.yaml
  03-open-webui.yaml

exercises/
  01-service-wrong-selector.yaml
```

実行中リソースから YAML を取得して保存する場合は、以下を実行します。

```bash
mkdir -p backup

oc get deployment ollama -o yaml > backup/ollama-deployment.yaml
oc get deployment open-webui -o yaml > backup/open-webui-deployment.yaml

oc get service ollama -o yaml > backup/ollama-service.yaml
oc get service open-webui -o yaml > backup/open-webui-service.yaml

oc get route open-webui -o yaml > backup/open-webui-route.yaml

oc get pvc ollama-data -o yaml > backup/ollama-data-pvc.yaml
oc get pvc webui-data -o yaml > backup/webui-data-pvc.yaml
```

> 実行中リソースから取得した YAML には、`status`、`resourceVersion`、`uid` など、再適用に不要な管理情報が含まれます。基本的には、ラボで作成した元の manifest を正本として Git 管理します。

***

## 10-2. 削除方法を選ぶ

ラボ環境の削除方法は、主に二つあります。

| 方法 | 適したケース | 削除対象 |
|---|---|---|
| Project を削除 | この Project がラボ専用で、すべて破棄してよい | Project 内のリソース全体 |
| 個別に削除 | Project を他用途でも使う、PVC を残したい | 指定したリソースのみ |

### 推奨

今回の Project が Open WebUI + Ollama ハンズオン専用であれば、**Project 単位で削除**します。

ただし、PVC のデータを残す必要がある場合、または共用 Project の場合は、個別削除を選択します。

***

## 10-3. Project 単位で削除する

> 以下のコマンドは、対象 Project 内の Deployment、Pod、Service、Route、PVC、Secret などを削除します。  
> 実行前に Project 名が正しいことを必ず確認してください。

現在の Project 名を確認します。

```bash
oc project
```

例です。

```text
Using project "ollama-handson" on server "https://api.<CLUSTER_DOMAIN>:6443".
```

Project を削除します。

```bash
oc delete project ollama-handson
```

削除状況を確認します。

```bash
oc get project ollama-handson
```

### 期待状態

削除途中は、以下のように表示される場合があります。

```text
NAME             STATUS
ollama-handson   Terminating
```

削除が完了すると、Project が見つからなくなります。

```bash
oc get project ollama-handson
```

```text
Error from server (NotFound): namespaces "ollama-handson" not found
```

### 講師が説明すること

> Project 削除は、ラボで作成したリソースをまとめて削除する最も簡単な方法です。

> ただし Project 内の PVC も削除対象になります。PVC 削除時に実データを保持するか削除するかは、StorageClass や PV の Reclaim Policy に依存するため、実運用ではストレージ管理者のルールを確認します。

***

## 10-4. 個別リソースを削除する

Project を残す場合は、以下の順で削除します。

まず、外部公開を止めます。

```bash
oc delete route open-webui
```

次に、アプリケーションを停止します。

```bash
oc delete deployment open-webui
oc delete deployment ollama
```

Service を削除します。

```bash
oc delete service open-webui
oc delete service ollama
```

### データを削除する場合

Ollama のモデルデータと Open WebUI の状態データも削除する場合だけ、PVC を削除します。

```bash
oc delete pvc webui-data
oc delete pvc ollama-data
```

Secret も不要であれば削除します。

```bash
oc delete secret open-webui-secret
```

最後に、残存リソースを確認します。

```bash
oc get deploy,pod,svc,route,pvc,secret
```

***

## 10-5. hosts ファイルを元に戻す

このラボでは、作業端末から内部 DNS を利用できないため、Windows の hosts ファイルに Route Host 名と Ingress Router の IP アドレスを記載しました。

ラボ終了後、不要になったエントリーを削除します。

対象ファイルです。

```text
C:\Windows\System32\drivers\etc\hosts
```

たとえば、以下のような行を追記していた場合です。

```text
<INGRESS_ROUTER_IP>  open-webui-ollama-handson.apps.<CLUSTER_DOMAIN>
```

Open WebUI Route を削除する、またはラボ環境を使用しなくなった時点で、この行を削除します。

### 注意点

- Windows 側のブラウザを使用するため、編集対象は **Windows の hosts ファイル**です
- WSL の `/etc/hosts` ではありません
- 編集時は管理者権限でエディタを起動します
- hosts ファイルを誤って `hosts.txt` として保存しないようにします
- ほかのラボや業務で使う Host 名の行を削除しないようにします

***

## 10-6. 最終確認

Project を削除した場合は、以下を実行します。

```bash
oc get project ollama-handson
```

個別削除の場合は、以下を実行します。

```bash
oc get deploy,pod,svc,route,pvc,secret
```

### Project を残した場合の期待結果

```text
No resources found in <PROJECT_NAME> namespace.
```

PVC を意図的に残した場合は、以下のようになります。

```text
NAME          STATUS   VOLUME
ollama-data   Bound    pvc-<UUID>
webui-data    Bound    pvc-<UUID>
```

この場合、次回のラボで同じ PVC を利用すれば、Ollama モデルや Open WebUI の保存済みデータを引き継げる可能性があります。

***

## ラボ全体の振り返り

今回のハンズオンで、以下の一連の流れを実施しました。

```text
1. OpenShift Project を作成する
2. PVC を作成する
3. Ollama を Deployment としてデプロイする
4. Ollama モデルを pull する
5. Open WebUI をデプロイする
6. Service を通じて Open WebUI から Ollama へ接続する
7. Route で Open WebUI を外部公開する
8. Windows hosts ファイルで Route Host の名前解決を補う
9. Logs、Events、Endpoints で障害を調査する
10. selector 不一致による通信障害を復旧する
11. Pod 削除後の自己修復と PVC 永続化を確認する
12. リソースを削除し、hosts 設定を戻す
```

## 最終チェック

### OpenShift の基本概念

- [ ] Project と Namespace の役割を説明できる
- [ ] Pod が一時的な実行単位であることを説明できる
- [ ] Deployment が望ましい Pod 数を維持する理由を説明できる
- [ ] Service が安定した内部接続先になる理由を説明できる
- [ ] Route が外部アクセスを提供する理由を説明できる
- [ ] Service selector と Pod label の関係を説明できる
- [ ] Endpoints が Service の実際の転送先を示すことを説明できる
- [ ] PVC が Pod 削除後にもデータを残す理由を説明できる

### GPU・ストレージ

- [ ] GPU リソースを request / limit する理由を説明できる
- [ ] Ollama のモデル保存先を PVC として扱う理由を説明できる
- [ ] Open WebUI の状態データを PVC として扱う理由を説明できる
- [ ] LVM Storage を用いる PVC が Node ローカルである点を理解している

### ネットワーク・運用

- [ ] Open WebUI が `http://ollama:11434` で Ollama と通信する理由を説明できる
- [ ] Pod IP を接続先として使わない理由を説明できる
- [ ] Route 障害時に hosts → Route → Service → Endpoints → Pod の順で確認できる
- [ ] 内部 DNS が利用できない作業端末では、Windows hosts ファイルに Route Host と Ingress Router IP を対応付ける理由を説明できる
- [ ] `oc get`、`oc describe`、`oc logs`、`oc get events` を使い分けられる
- [ ] Project 削除が Project 内のコンテンツ削除を伴うことを理解している

***

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
  01-pvc.yaml
  02-ollama.yaml
  03-open-webui.yaml

exercises/
  01-service-wrong-selector.yaml
```

各環境で使用する Project 名、StorageClass 名、Ingress Router IP、Route Host、コンテナイメージの tag、GPU リソース名は、実環境の値に置き換えてください。
