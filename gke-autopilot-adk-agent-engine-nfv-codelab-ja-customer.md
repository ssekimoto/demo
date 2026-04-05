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

## 既存環境由来の運用 Helm チャートを作る
Duration: 0:20:00

このセクションでは、`existing-nfv-ops-ui` という、既存環境由来の設定を含むチャートを作成します。

このチャートは、既存の Kubernetes 環境で利用されていた設定を引き継いだ**架空の運用 Web コンソール**です。  
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
        <p>この UI は、引き継いだ運用コンソールを模したサンプルです。</p>
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

後で Agent に読ませるため、一度削除して出力を保存します。

```bash
helm uninstall "${RELEASE_NAME}" --namespace "${NAMESPACE}"
```

```bash
helm install "${RELEASE_NAME}" ./existing-nfv-ops-ui \
  --namespace "${NAMESPACE}" 2>&1 | tee "${LAB_DIR}/initial-install-error.txt" || true
```

また、Agent がマニフェスト全体を確認できるように、render 結果も保存します。

```bash
helm template "${RELEASE_NAME}" ./existing-nfv-ops-ui \
  --namespace "${NAMESPACE}" > "${LAB_DIR}/rendered-before.yaml"
```

### この Agent に持たせる役割

- Autopilot 非互換箇所を抽出する
- テレコム/NFV 顧客に説明しやすい置き換え方を選ぶ
- 完全な `values.yaml` を返す
- 完全な `templates/deployment.yaml` を返す
- 検証コマンドと変更理由を返す

### ADK プロジェクトの構成を作る

```bash
mkdir -p "${AGENT_DIR}/nfv_modernizer"
cd "${AGENT_DIR}"

cat > nfv_modernizer/__init__.py <<'EOF'
from .agent import root_agent
EOF
```

### Agent 本体を作る

```bash
cat > nfv_modernizer/agent.py <<'EOF'
import re
from typing import Any

from google.adk.agents import Agent


def analyze_autopilot_violations(text: str) -> dict[str, Any]:
    """Extract likely Autopilot-incompatible settings from chart text or error text."""
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
        recommendations.append("set hostNetwork to false and use a Service")
    if "hostPID" in findings:
        recommendations.append("set hostPID to false")
    if "hostPort" in findings:
        recommendations.append("remove hostPort and expose via a Kubernetes Service")
    if "hostPath" in findings:
        recommendations.append("replace hostPath with ConfigMap, emptyDir, or PVC")
    if "allowPrivilegeEscalation" in findings:
        recommendations.append("set allowPrivilegeEscalation to false")

    return {
        "findings": findings,
        "recommendations": recommendations,
    }


def telecom_modernization_rules() -> dict[str, Any]:
    """Return the platform rules the agent must respect."""
    return {
        "autopilot_rules": {
            "privileged": False,
            "hostNetwork": False,
            "hostPID": False,
            "allowPrivilegeEscalation": False,
            "hostPort": "not allowed",
            "hostPath": "avoid unless absolutely required; prefer ConfigMap, emptyDir, or PVC",
        },
        "preferred_replacements": {
            "node-coupled networking": "Service-based access",
            "host log mount": "application-owned paths or platform logging integration",
            "hostPort exposure": "ClusterIP Service plus port-forward or ingress path",
        },
        "customer_messaging": [
            "preserve operational intent",
            "remove host-level coupling",
            "make changes explainable to telecom platform teams",
        ],
    }


SYSTEM_INSTRUCTION = """
You are a Kubernetes modernization agent assisting a telecom platform team.

You will receive:
- the current values.yaml
- the current templates/deployment.yaml
- a Helm install or Warden error message

Your task:
1. identify every Autopilot-incompatible setting
2. rewrite the full values.yaml
3. rewrite the full templates/deployment.yaml
4. preserve the operational UI behavior where possible
5. replace host-level coupling with Kubernetes-native alternatives
6. explain the change in telecom / NFV platform terms

Output rules:
- return only one JSON object between BEGIN_REMOTE_JSON and END_REMOTE_JSON
- do not return markdown fences
- do not omit fields
- all file outputs must be full file contents, not patches

JSON schema:
{
  "values_yaml": "full file contents",
  "deployment_yaml": "full file contents",
  "change_summary": ["bullet 1", "bullet 2"],
  "customer_rationale": ["message 1", "message 2"],
  "validation_commands": ["helm lint ...", "helm template ..."]
}
"""

root_agent = Agent(
    name="nfv_modernizer",
    model="gemini-3.1-pro-preview",
    description="Modernizes an existing-environment NFV-adjacent Helm chart for GKE Autopilot.",
    instruction=SYSTEM_INSTRUCTION,
    tools=[
        analyze_autopilot_violations,
        telecom_modernization_rules,
    ],
)
EOF
```

### Vertex AI Agent Engine へデプロイするスクリプトを作る

```bash
cat > "${AGENT_DIR}/deploy_agent_engine.py" <<'EOF'
import os
import vertexai
from vertexai import agent_engines

from nfv_modernizer.agent import root_agent

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
        ],
        "staging_bucket": STAGING_BUCKET,
        "display_name": "nfv-modernizer-agent",
    },
)

print(remote_agent.resource_name)
EOF
```

### リモート Agent を呼び出して、修正版ファイルをローカルへ保存する helper script を作る

```bash
cat > "${AGENT_DIR}/run_remote_modernize.py" <<'EOF'
import json
import os
import re
from pathlib import Path
from typing import Any

import vertexai

PROJECT_ID = os.environ["PROJECT_ID"]
REGION = os.environ["REGION"]
RESOURCE_NAME = os.environ["REMOTE_AGENT_RESOURCE_NAME"]
LAB_DIR = Path(os.environ["LAB_DIR"])

VALUES_PATH = LAB_DIR / "existing-nfv-ops-ui" / "values.yaml"
DEPLOYMENT_PATH = LAB_DIR / "existing-nfv-ops-ui" / "templates" / "deployment.yaml"
ERROR_PATH = LAB_DIR / "initial-install-error.txt"
OUTPUT_JSON = LAB_DIR / "remote-modernization-output.json"
RAW_TEXT = LAB_DIR / "remote-modernization-raw.txt"


def collect_strings(obj: Any) -> list[str]:
    results: list[str] = []
    if obj is None:
        return results
    if isinstance(obj, str):
        results.append(obj)
        return results
    if isinstance(obj, dict):
        for value in obj.values():
            results.extend(collect_strings(value))
        return results
    if isinstance(obj, (list, tuple, set)):
        for item in obj:
            results.extend(collect_strings(item))
        return results
    for attr in ("text", "content", "parts", "message"):
        if hasattr(obj, attr):
            try:
                results.extend(collect_strings(getattr(obj, attr)))
            except Exception:
                pass
    return results


values_text = VALUES_PATH.read_text()
deployment_text = DEPLOYMENT_PATH.read_text()
error_text = ERROR_PATH.read_text() if ERROR_PATH.exists() else ""

message = f"""
You are assisting a telecom core network platform team.

Please modernize the following Helm chart so it can run on GKE Autopilot.

Return only one JSON object between BEGIN_REMOTE_JSON and END_REMOTE_JSON.
Do not use markdown fences.

Current values.yaml:
---VALUES_YAML_START---
{values_text}
---VALUES_YAML_END---

Current templates/deployment.yaml:
---DEPLOYMENT_YAML_START---
{deployment_text}
---DEPLOYMENT_YAML_END---

Observed install / Warden error:
---ERROR_START---
{error_text}
---ERROR_END---
"""

client = vertexai.Client(project=PROJECT_ID, location=REGION)
remote_agent = client.agent_engines.get(RESOURCE_NAME)

chunks: list[str] = []
for event in remote_agent.stream_query(
    user_id="lab-user",
    message=message,
):
    chunks.extend(collect_strings(event))

joined = "\n".join([chunk for chunk in chunks if chunk]).strip()
RAW_TEXT.write_text(joined)

match = re.search(r"BEGIN_REMOTE_JSON\s*(\{[\s\S]*?\})\s*END_REMOTE_JSON", joined)
if not match:
    raise RuntimeError(
        "Could not find BEGIN_REMOTE_JSON / END_REMOTE_JSON in remote response. "
        f"See {RAW_TEXT} for the raw output."
    )

payload = json.loads(match.group(1))
OUTPUT_JSON.write_text(json.dumps(payload, ensure_ascii=False, indent=2))

VALUES_PATH.write_text(payload["values_yaml"])
DEPLOYMENT_PATH.write_text(payload["deployment_yaml"])

print(f"Wrote modernization result to: {OUTPUT_JSON}")
print(f"Updated: {VALUES_PATH}")
print(f"Updated: {DEPLOYMENT_PATH}")
print("\nChange summary:")
for item in payload.get("change_summary", []):
    print(f"- {item}")

print("\nCustomer rationale:")
for item in payload.get("customer_rationale", []):
    print(f"- {item}")

print("\nSuggested validation commands:")
for item in payload.get("validation_commands", []):
    print(f"- {item}")
EOF
```

### 変更前のベースラインを保存する

```bash
cd "${LAB_DIR}"
cp existing-nfv-ops-ui/values.yaml values-before.yaml
cp existing-nfv-ops-ui/templates/deployment.yaml deployment-before.yaml
```

## Vertex AI Agent Engine に直接デプロイし、リモート Agent を呼び出す
Duration: 0:25:00

### Agent をデプロイする

```bash
cd "${AGENT_DIR}"
source "${LAB_DIR}/.venv/bin/activate"

export PROJECT_ID="${PROJECT_ID}"
export REGION="${REGION}"
export STAGING_BUCKET="${STAGING_BUCKET}"

python deploy_agent_engine.py | tee /tmp/remote_agent_resource_name.txt
```

環境変数へ resource name を設定します。

```bash
export REMOTE_AGENT_RESOURCE_NAME=$(tail -n 1 /tmp/remote_agent_resource_name.txt)
echo "${REMOTE_AGENT_RESOURCE_NAME}"
```

### リモート Agent に chart 内容を渡して修正版ファイルを生成する

```bash
export LAB_DIR="${LAB_DIR}"
python run_remote_modernize.py
```

### 修正差分を確認する

```bash
diff -u values-before.yaml existing-nfv-ops-ui/values.yaml || true
diff -u deployment-before.yaml existing-nfv-ops-ui/templates/deployment.yaml || true
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
