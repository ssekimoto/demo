id: gke-autopilot-adk-agent-engine-nfv-modernization-ja
summary: ADK と Vertex AI Agent Engine を使って、既存環境の Helm チャートを GKE Autopilot 向けにモダナイズする
categories: gke,ai,devops,telecom,nfv,kubernetes
environments: Web
status: Draft
feedback link: https://github.com/ssekimoto/tbd
authors: Shintaro Sekimoto

# ADK と Vertex AI Agent Engine を使って、既存環境 Helm チャートを GKE Autopilot 向けにモダナイズする

## 概要
Duration: 0:10:00

### このラボで作るもの

このラボでは、通信キャリアのコアネットワーク部門を支援するプラットフォームエンジニアの立場で作業します。

対象は、**NFV 隣接の運用系コンポーネント**として配布されている、既存環境の Helm チャートです。  
このチャートは、オンプレミス前提の権限が強い Kubernetes 環境を前提に作られており、次のようなパターンを含んでいます。

- `privileged: true`
- `hostNetwork: true`
- ノードローカルな情報参照を前提にした `hostPath`
- `hostPort` 前提の公開方法
- 広範な特権昇格設定

こうした設計は、たとえば次のようなコンポーネントで見かけられます。

- 古い監視サイドカー
- パケットコア周辺の運用補助サービス
- ベンダー提供のトラブルシュート UI
- 広いノードアクセスを前提にした Day-2 運用ユーティリティ

### このラボで行うこと

1. GKE Autopilot クラスタを作成する
2. 既存環境由来の設定を含む Helm チャートをデプロイし、拒否されることを確認する
3. **ADK ベースのモダナイゼーション Agent** を作る
4. **ADK Web UI** から Agent に診断・修正・検証を実行させる
5. 修正済みチャートを GKE Autopilot へ正常デプロイする
6. 同じ Agent を **Vertex AI Agent Engine** にデプロイする

### このラボで学べること

このラボを終えると、次の点を説明できるようになります。

- なぜ GKE Autopilot が強いデフォルトガードレールを必要とするプラットフォームチームに向いているのか
- ADK を使って、繰り返し可能なプラットフォームエンジニアリング作業を Agent 化する方法
- ローカルで作った Agent を Agent Engine でマネージド実行基盤へ持ち上げる流れ

プラットフォームチームは、次のような特徴を持つワークロードを引き継ぐことがよくあります。

- 権限が緩いクラスタ向けに書かれたマニフェスト
- ホストアクセスを前提にしたベンダー YAML
- root 相当権限を前提にした Day-2 運用スクリプト
- 品質が揃っていない Helm テンプレート
- 環境差分が大きく、継続的な検証が難しいオーバーレイ

このため、移行は遅く、レビューコストが高く、失敗しやすくなります。

このタイプのハンズオンでは、**GKE Autopilot は guarded cluster をすばやく用意しやすい** ことを学ぶことが可能です。
- 最初に node group、worker 設定、autoscaler、IaC テンプレートの説明に多くの時間を割かなくてよい
- ノード運用、スケーリング、多くのデフォルトセキュリティ前提が最初からマネージドである
- そのため、**クラスタ plumbing の説明ではなくモダナイゼーションの本題**に時間を使いやすい

## ラボ構成
Duration: 0:20:00

このラボでは次の構成を使います。

- オペレーター環境としての **Cloud Shell**
- デプロイ先としての **GKE Autopilot**
- NFV 隣接の運用補助コンポーネントを表現する **既存環境由来 Helm chart**
- ファイル操作と限定的なシェル実行ができる **ADK Agent**
- インタラクティブに試せる **ADK Web UI**
- 同じ Agent のマネージド実行先としての **Vertex AI Agent Engine**

### 全体フロー

1. Autopilot クラスタを作成する
2. 既存環境由来 Helm チャートのインストールを試す
3. チャートが拒否されることを確認する
4. 次のツールを持つ Agent を作る
   - chart ファイルを読む
   - Autopilot 非互換箇所を診断する
   - manifest を書き換える
   - 検証コマンドを実行する
5. ブラウザ UI から Agent を操作する
6. 修正済み chart をインストールする
7. 同じ Agent を Agent Engine へデプロイする

## 事前準備
Duration: 0:20:00

必要なものは次のとおりです。

- 課金が有効な Google Cloud プロジェクト
- API 有効化とリソース作成ができる権限
- Cloud Shell
- Kubernetes と Helm の基本知識

### 環境変数を設定する

Cloud Shell を開いて、次を実行します。

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION="us-central1"
export CLUSTER_NAME="nfv-modernization-lab"
export RELEASE_NAME="existing-ops-ui"
export NAMESPACE="nfv-modernization"
export AGENT_DIR="${HOME}/nfv_modernizer_agent"
export LAB_DIR="${HOME}/nfv-modernization-lab"
export STAGING_BUCKET="gs://${PROJECT_ID}-agent-engine-staging"

gcloud config set project "${PROJECT_ID}"
gcloud config set compute/region "${REGION}"
mkdir -p "${LAB_DIR}"
cd "${LAB_DIR}"
```

### 必要な API を有効化する

```bash
gcloud services enable \
  container.googleapis.com \
  aiplatform.googleapis.com \
  storage.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com
```

### Cloud Shell にツールを用意する

Cloud Shell には多くのツールが最初から入っていますが、再現性のために Helm と Python パッケージをそろえます。

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install --upgrade "google-cloud-aiplatform[agent_engines,adk]>=1.112" pyyaml
```

### Agent Engine 用の staging bucket を作る

```bash
gcloud storage buckets create "${STAGING_BUCKET}" \
  --location="${REGION}" \
  --uniform-bucket-level-access || true
```

## GKE Autopilot クラスタを作成する
Duration: 0:15:00

クラスタを作成します。

```bash
gcloud container clusters create-auto "${CLUSTER_NAME}" \
  --region "${REGION}"
```

認証情報を取得します。

```bash
gcloud container clusters get-credentials "${CLUSTER_NAME}" \
  --region "${REGION}"
```

namespace を作成します。

```bash
kubectl create namespace "${NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
```

クラスタを確認します。

```bash
kubectl get nodes
kubectl get ns
```

## 既存環境由来のテレコム運用 Helm チャートを作る
Duration: 0:20:00

このセクションでは、`existing-nfv-ops-ui` という、既存環境由来の設定を含むチャートを作成します。

このチャートは、既存の Kubernetes 環境で利用されていた設定を引き継いだ**架空のテレコム運用 Web コンソール**です。  
NFV 隣接のツールで見かけやすい、いくつかの古い前提を含めています。

### chart ディレクトリを作る

```bash
cd "${LAB_DIR}"
mkdir -p existing-nfv-ops-ui/templates
```

### `Chart.yaml` を作る

```bash
cat > existing-nfv-ops-ui/Chart.yaml <<'EOF'
apiVersion: v2
name: existing-nfv-ops-ui
description: Existing-environment telecom NFV operations UI with inherited security assumptions
type: application
version: 0.1.0
appVersion: "1.0.0"
EOF
```

### `values.yaml` を作る

```bash
cat > existing-nfv-ops-ui/values.yaml <<'EOF'
replicaCount: 1

image:
  repository: nginx
  tag: "1.27-alpine"
  pullPolicy: IfNotPresent

inheritedSettings:
  privileged: true
  hostNetwork: true
  hostPID: true
  allowPrivilegeEscalation: true
  readOnlyRootFilesystem: false
  hostPathEnabled: true
  hostPortEnabled: true
  hostLogPath: /var/log
  hostPort: 8080

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

env:
  NFV_ENVIRONMENT: "preprod"
  CORE_DOMAIN: "packet-core"
  OPERATION_MODE: "existing-env-node-inspection"
EOF
```

### `templates/configmap.yaml` を作る

```bash
cat > existing-nfv-ops-ui/templates/configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: existing-nfv-ops-ui-config
data:
  default.conf: |
    server {
      listen 8080;
      server_name _;
      location / {
        root /usr/share/nginx/html;
        index index.html;
      }
      location /healthz {
        return 200 "ok\n";
      }
    }
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="utf-8" />
      <title>既存環境由来 NFV Ops UI</title>
      <style>
        body {
          font-family: Arial, sans-serif;
          margin: 40px;
          background: #0f172a;
          color: #e2e8f0;
        }
        h1 { color: #38bdf8; }
        .card {
          border: 1px solid #334155;
          padding: 24px;
          border-radius: 12px;
          background: #111827;
        }
        code {
          background: #1e293b;
          padding: 2px 6px;
          border-radius: 6px;
        }
      </style>
    </head>
    <body>
      <div class="card">
        <h1>既存環境由来 NFV Operations UI</h1>
        <p>この UI は、引き継いだテレコム運用コンソールを模したサンプルです。</p>
        <ul>
          <li>想定モード: <code>existing-env-node-inspection</code></li>
          <li>元の前提: ホストアクセスが許されている</li>
          <li>ターゲット基盤: GKE Autopilot</li>
        </ul>
      </div>
    </body>
    </html>
EOF
```

### `templates/deployment.yaml` を作る

```bash
cat > existing-nfv-ops-ui/templates/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: existing-nfv-ops-ui
  labels:
    app: existing-nfv-ops-ui
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: existing-nfv-ops-ui
  template:
    metadata:
      labels:
        app: existing-nfv-ops-ui
    spec:
      hostNetwork: {{ .Values.inheritedSettings.hostNetwork }}
      hostPID: {{ .Values.inheritedSettings.hostPID }}
      containers:
        - name: web
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            - name: NFV_ENVIRONMENT
              value: "{{ .Values.env.NFV_ENVIRONMENT }}"
            - name: CORE_DOMAIN
              value: "{{ .Values.env.CORE_DOMAIN }}"
            - name: OPERATION_MODE
              value: "{{ .Values.env.OPERATION_MODE }}"
          command:
            - /bin/sh
            - -c
            - |
              cp /config/default.conf /etc/nginx/conf.d/default.conf
              cp /config/index.html /usr/share/nginx/html/index.html
              nginx -g 'daemon off;'
          ports:
            - name: http
              containerPort: 8080
{{- if .Values.inheritedSettings.hostPortEnabled }}
              hostPort: {{ .Values.inheritedSettings.hostPort }}
{{- end }}
          securityContext:
            privileged: {{ .Values.inheritedSettings.privileged }}
            allowPrivilegeEscalation: {{ .Values.inheritedSettings.allowPrivilegeEscalation }}
            readOnlyRootFilesystem: {{ .Values.inheritedSettings.readOnlyRootFilesystem }}
          resources:
{{ toYaml .Values.resources | indent 12 }}
          volumeMounts:
            - name: config
              mountPath: /config
{{- if .Values.inheritedSettings.hostPathEnabled }}
            - name: host-logs
              mountPath: /node-logs
{{- end }}
      volumes:
        - name: config
          configMap:
            name: existing-nfv-ops-ui-config
{{- if .Values.inheritedSettings.hostPathEnabled }}
        - name: host-logs
          hostPath:
            path: {{ .Values.inheritedSettings.hostLogPath }}
            type: Directory
{{- end }}
EOF
```

### `templates/service.yaml` を作る

```bash
cat > existing-nfv-ops-ui/templates/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: existing-nfv-ops-ui
  labels:
    app: existing-nfv-ops-ui
spec:
  selector:
    app: existing-nfv-ops-ui
  type: {{ .Values.service.type }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
EOF
```

### chart を確認する

```bash
tree existing-nfv-ops-ui
helm template "${RELEASE_NAME}" ./existing-nfv-ops-ui | sed -n '1,220p'
```

## Autopilot に拒否されることを確認する
Duration: 0:10:00

それでは、この既存環境由来 chart をそのまま Autopilot クラスタへデプロイしてみます。

```bash
helm install "${RELEASE_NAME}" ./existing-nfv-ops-ui \
  --namespace "${NAMESPACE}"
```

Autopilot 互換の制約に違反しているため、admission レイヤーで失敗するはずです。

後で Agent に読ませるため、出力を保存します。

```bash
helm install "${RELEASE_NAME}" ./existing-nfv-ops-ui \
  --namespace "${NAMESPACE}" 2>&1 | tee "${LAB_DIR}/initial-install-error.txt" || true
```

また、Agent がマニフェスト全体を確認できるように、render 結果も保存します。

```bash
helm template "${RELEASE_NAME}" ./existing-nfv-ops-ui \
  --namespace "${NAMESPACE}" > "${LAB_DIR}/rendered-before.yaml"
```

## ADK でモダナイゼーション Agent を作る
Duration: 0:35:00

このセクションでは、ADK を使ってローカルの Agent プロジェクトを作成します。

この Agent には次のようなツールを持たせます。

- ファイルを読む
- ファイルを書く
- diff を見る
- 制限付きシェルコマンドを実行する
- chart を検証する
- install / upgrade の準備を行う

### ADK プロジェクトの構成を作る

```bash
mkdir -p "${AGENT_DIR}/nfv_modernizer"
cd "${AGENT_DIR}"

cat > nfv_modernizer/__init__.py <<'EOF'
from . import agent
EOF
```

### Agent 本体を作る

```bash
cat > nfv_modernizer/agent.py <<'EOF'
import difflib
import os
import re
import subprocess
from pathlib import Path

from google.adk.agents import Agent

BASE_DIR = Path(os.environ.get("LAB_DIR", str(Path.home() / "nfv-modernization-lab"))).resolve()
CHART_DIR = BASE_DIR / "existing-nfv-ops-ui"
ALLOWED_PREFIXES = [
    "helm template",
    "helm lint",
    "helm upgrade",
    "helm install",
    "helm uninstall",
    "kubectl get",
    "kubectl apply",
    "kubectl describe",
    "cat ",
    "sed ",
    "grep ",
    "python3 ",
]

def _safe_path(path: str) -> Path:
    p = (BASE_DIR / path).resolve() if not path.startswith("/") else Path(path).resolve()
    if BASE_DIR not in p.parents and p != BASE_DIR:
        raise ValueError(f"path not allowed: {path}")
    return p

def read_text_file(path: str) -> dict:
    """Read a text file from the lab directory."""
    try:
        p = _safe_path(path)
        return {"status": "success", "path": str(p), "content": p.read_text()}
    except Exception as e:
        return {"status": "error", "error": str(e)}

def write_text_file(path: str, content: str) -> dict:
    """Write a text file under the lab directory."""
    try:
        p = _safe_path(path)
        p.parent.mkdir(parents=True, exist_ok=True)
        p.write_text(content)
        return {"status": "success", "path": str(p), "bytes": len(content.encode())}
    except Exception as e:
        return {"status": "error", "error": str(e)}

def unified_diff(path_a: str, path_b: str) -> dict:
    """Show a unified diff between two text files."""
    try:
        a = _safe_path(path_a).read_text().splitlines(keepends=True)
        b = _safe_path(path_b).read_text().splitlines(keepends=True)
        diff = "".join(
            difflib.unified_diff(
                a,
                b,
                fromfile=path_a,
                tofile=path_b,
            )
        )
        return {"status": "success", "diff": diff}
    except Exception as e:
        return {"status": "error", "error": str(e)}

def run_command(command: str) -> dict:
    """Run a restricted shell command needed for chart validation and deployment."""
    try:
        if not any(command.startswith(prefix) for prefix in ALLOWED_PREFIXES):
            return {
                "status": "error",
                "error": f"command not allowed: {command}",
                "allowed_prefixes": ALLOWED_PREFIXES,
            }

        proc = subprocess.run(
            command,
            shell=True,
            cwd=str(BASE_DIR),
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            text=True,
            timeout=180,
        )
        return {
            "status": "success",
            "returncode": proc.returncode,
            "output": proc.stdout,
        }
    except Exception as e:
        return {"status": "error", "error": str(e)}

def analyze_autopilot_violations(text: str) -> dict:
    """Extract likely Autopilot-incompatible fields from error text or rendered manifests."""
    findings = []
    patterns = {
        "privileged": r"privileged",
        "hostNetwork": r"hostNetwork",
        "hostPID": r"hostPID",
        "hostPort": r"hostPort",
        "hostPath": r"hostPath",
        "allowPrivilegeEscalation": r"allowPrivilegeEscalation",
    }
    lowered = text.lower()
    for name, pattern in patterns.items():
        if re.search(pattern.lower(), lowered):
            findings.append(name)

    recommendations = []
    if "privileged" in findings:
        recommendations.append("set privileged to false")
    if "hostNetwork" in findings:
        recommendations.append("set hostNetwork to false and use Service networking")
    if "hostPID" in findings:
        recommendations.append("set hostPID to false")
    if "hostPort" in findings:
        recommendations.append("remove hostPort and use a Kubernetes Service")
    if "hostPath" in findings:
        recommendations.append("remove hostPath and use ConfigMap, emptyDir, or PVC")
    if "allowPrivilegeEscalation" in findings:
        recommendations.append("set allowPrivilegeEscalation to false")

    return {
        "status": "success",
        "findings": findings,
        "recommendations": recommendations,
    }

def validate_chart() -> dict:
    """Run a validation sequence for the chart."""
    checks = []

    cmds = [
        'helm lint ./existing-nfv-ops-ui',
        'helm template existing-ops-ui ./existing-nfv-ops-ui --namespace nfv-modernization',
        'kubectl apply --dry-run=server -n nfv-modernization -f rendered-after.yaml',
    ]

    for cmd in cmds:
        if cmd.endswith("rendered-after.yaml"):
            rendered = subprocess.run(
                'helm template existing-ops-ui ./existing-nfv-ops-ui --namespace nfv-modernization > rendered-after.yaml',
                shell=True,
                cwd=str(BASE_DIR),
                stdout=subprocess.PIPE,
                stderr=subprocess.STDOUT,
                text=True,
                timeout=180,
            )
            checks.append(
                {
                    "command": "helm template existing-ops-ui ./existing-nfv-ops-ui --namespace nfv-modernization > rendered-after.yaml",
                    "returncode": rendered.returncode,
                    "output": rendered.stdout,
                }
            )

        proc = subprocess.run(
            cmd,
            shell=True,
            cwd=str(BASE_DIR),
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            text=True,
            timeout=180,
        )
        checks.append({"command": cmd, "returncode": proc.returncode, "output": proc.stdout})

    success = all(item["returncode"] == 0 for item in checks)
    return {"status": "success", "success": success, "checks": checks}

SYSTEM_INSTRUCTION = """
You are a Kubernetes modernization agent helping a telecom platform team.

Your job:
1. inspect existing-environment Helm charts under the lab directory
2. identify settings incompatible with GKE Autopilot
3. rewrite full files, not partial snippets
4. preserve application intent where possible
5. replace node-coupled behavior with Kubernetes-native alternatives

Modernization rules:
- privileged must be false
- hostNetwork must be false
- hostPID must be false
- remove hostPort usage
- remove hostPath usage
- set allowPrivilegeEscalation to false
- prefer Service-based access over host networking
- keep the resulting workload easy to explain to an NFV customer
- preserve the operational UI behavior
- always validate after changes
- when changing a file, write the whole file content

Workflow:
- read values.yaml and deployment.yaml first
- analyze the Helm install error if present
- propose the exact file rewrites
- write the changes
- run validation
- if validation passes, suggest a helm upgrade/install command
"""

root_agent = Agent(
    name="nfv_modernizer",
    model="gemini-2.5-flash",
    description="Modernizes an existing-environment NFV-adjacent Helm chart for GKE Autopilot.",
    instruction=SYSTEM_INSTRUCTION,
    tools=[
        read_text_file,
        write_text_file,
        unified_diff,
        run_command,
        analyze_autopilot_violations,
        validate_chart,
    ],
)
EOF
```

### Agent が chart を見つけられるように環境変数を export する

```bash
export LAB_DIR="${LAB_DIR}"
```

### diff 用のベースラインを保存する

```bash
cp existing-nfv-ops-ui/values.yaml values-before.yaml
cp existing-nfv-ops-ui/templates/deployment.yaml deployment-before.yaml
```

## ADK Web UI を使う
Duration: 0:20:00

Agent プロジェクトの親ディレクトリへ移動します。

```bash
cd "${AGENT_DIR}"
source "${LAB_DIR}/.venv/bin/activate"
adk web
```

Cloud Shell の Web Preview から、公開されたポートを開きます。

ADK Web UI 上で、`nfv_modernizer` Agent を選択します。

### Agent へのプロンプト

次のプロンプトを使います。

```text
You are assisting a telecom platform team that inherited an NFV operations Helm chart from an existing environment.

Please modernize the chart under the lab directory so it can run on GKE Autopilot.

Requirements:
- rewrite full files, not fragments
- preserve the UI behavior
- remove host-level coupling
- use Kubernetes-native networking
- validate with helm lint, helm template, and kubectl apply --dry-run=server
- explain every change in telecom platform terms
```

### 期待される Agent の動き

良い結果であれば、少なくとも次の多くを満たします。

- `privileged` を無効化する
- `hostNetwork` を無効化する
- `hostPID` を無効化する
- `allowPrivilegeEscalation` を無効化する
- `hostPath` volume を削除する
- `hostPort` を削除する
- ネットワーク公開抽象として Service を維持する
- ConfigMap から NGINX を構成する挙動を維持する
- resource requests / limits を保持する
- 検証コマンドを生成または実行する

### 更新後のファイルを確認する

Cloud Shell 側で次を実行します。

```bash
cp existing-nfv-ops-ui/values.yaml values-after.yaml
cp existing-nfv-ops-ui/templates/deployment.yaml deployment-after.yaml

diff -u values-before.yaml values-after.yaml || true
diff -u deployment-before.yaml deployment-after.yaml || true
```

### 参考実装を使いたい場合

Agent がうまく進まない場合は、以下の完成版を使って続行できます。

#### 参考 `values.yaml`

```bash
cat > existing-nfv-ops-ui/values.yaml <<'EOF'
replicaCount: 1

image:
  repository: nginx
  tag: "1.27-alpine"
  pullPolicy: IfNotPresent

inheritedSettings:
  privileged: false
  hostNetwork: false
  hostPID: false
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: false
  hostPathEnabled: false
  hostPortEnabled: false
  hostLogPath: /var/log
  hostPort: 8080

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

env:
  NFV_ENVIRONMENT: "preprod"
  CORE_DOMAIN: "packet-core"
  OPERATION_MODE: "kubernetes-service-mode"
EOF
```

#### 参考 `templates/deployment.yaml`

```bash
cat > existing-nfv-ops-ui/templates/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: existing-nfv-ops-ui
  labels:
    app: existing-nfv-ops-ui
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: existing-nfv-ops-ui
  template:
    metadata:
      labels:
        app: existing-nfv-ops-ui
    spec:
      containers:
        - name: web
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            - name: NFV_ENVIRONMENT
              value: "{{ .Values.env.NFV_ENVIRONMENT }}"
            - name: CORE_DOMAIN
              value: "{{ .Values.env.CORE_DOMAIN }}"
            - name: OPERATION_MODE
              value: "{{ .Values.env.OPERATION_MODE }}"
          command:
            - /bin/sh
            - -c
            - |
              cp /config/default.conf /etc/nginx/conf.d/default.conf
              cp /config/index.html /usr/share/nginx/html/index.html
              nginx -g 'daemon off;'
          ports:
            - name: http
              containerPort: 8080
          securityContext:
            privileged: {{ .Values.inheritedSettings.privileged }}
            allowPrivilegeEscalation: {{ .Values.inheritedSettings.allowPrivilegeEscalation }}
            readOnlyRootFilesystem: {{ .Values.inheritedSettings.readOnlyRootFilesystem }}
          resources:
{{ toYaml .Values.resources | indent 12 }}
          volumeMounts:
            - name: config
              mountPath: /config
      volumes:
        - name: config
          configMap:
            name: existing-nfv-ops-ui-config
EOF
```

## モダナイズ済み chart を検証してデプロイする
Duration: 0:10:00

lint と render を実行します。

```bash
cd "${LAB_DIR}"
helm lint ./existing-nfv-ops-ui
helm template "${RELEASE_NAME}" ./existing-nfv-ops-ui \
  --namespace "${NAMESPACE}" > rendered-after.yaml
```

server-side validation を実行します。

```bash
kubectl apply --dry-run=server -n "${NAMESPACE}" -f rendered-after.yaml
```

chart をインストールします。

```bash
helm upgrade --install "${RELEASE_NAME}" ./existing-nfv-ops-ui \
  --namespace "${NAMESPACE}"
```

rollout を確認します。

```bash
kubectl get pods -n "${NAMESPACE}" -w
```

Service を確認します。

```bash
kubectl get svc -n "${NAMESPACE}"
kubectl port-forward -n "${NAMESPACE}" svc/existing-nfv-ops-ui 8080:80
```

Cloud Shell の Web Preview から転送ポートを開き、UI が表示されることを確認します。

## 同じ Agent を Vertex AI Agent Engine にデプロイする
Duration: 0:20:00

ここでは、ローカルで使った Agent をそのまま Agent Engine にデプロイします。

### デプロイスクリプトを作る

```bash
cat > "${AGENT_DIR}/nfv_modernizer/deploy_agent_engine.py" <<'EOF'
import os
import vertexai
from vertexai import agent_engines

from agent import root_agent

PROJECT_ID = os.environ["PROJECT_ID"]
REGION = os.environ["REGION"]
STAGING_BUCKET = os.environ["STAGING_BUCKET"]

client = vertexai.Client(project=PROJECT_ID, location=REGION)

app = agent_engines.AdkApp(agent=root_agent)

remote_agent = client.agent_engines.create(
    agent=app,
    config={
        "requirements": [
            "google-cloud-aiplatform[agent_engines,adk]>=1.112",
            "pyyaml",
            "pydantic",
            "cloudpickle",
        ],
        "staging_bucket": STAGING_BUCKET,
    },
)

print("REMOTE_AGENT_RESOURCE_NAME")
print(remote_agent.resource_name)
EOF
```

### デプロイを実行する

```bash
cd "${AGENT_DIR}/nfv_modernizer"
source "${LAB_DIR}/.venv/bin/activate"

export PROJECT_ID="${PROJECT_ID}"
export REGION="${REGION}"
export STAGING_BUCKET="${STAGING_BUCKET}"

python deploy_agent_engine.py
```

### 任意: リモート Agent をコードから試す

```bash
cat > "${AGENT_DIR}/nfv_modernizer/test_remote_agent.py" <<'EOF'
import asyncio
import os
import vertexai

RESOURCE_NAME = os.environ["REMOTE_AGENT_RESOURCE_NAME"]
PROJECT_ID = os.environ["PROJECT_ID"]
REGION = os.environ["REGION"]

client = vertexai.Client(project=PROJECT_ID, location=REGION)
remote_agent = client.agent_engines.get(RESOURCE_NAME)

async def main():
    async for event in remote_agent.async_stream_query(
        user_id="lab-user",
        message="Summarize what changed in the Helm chart and why it is safer for GKE Autopilot.",
    ):
        print(event)

asyncio.run(main())
EOF
```

前のステップで表示された resource name を設定して、テストします。

```bash
export REMOTE_AGENT_RESOURCE_NAME="PASTE_THE_RESOURCE_NAME_HERE"
python test_remote_agent.py
```

Google Cloud Console から、デプロイされた Agent を確認しても構いません。
