# llm-d Optimized Baseline + TNG：命令与文件映射笔记

> 基于 `/root/repos/llm-d/guides/optimized-baseline/README.md` 和相关 chart/values/kustomize 文件整理。
> 用于把 `learning/deployment-llm-d-tng-k8s.md` 中的通用 YAML 替换为 optimized-baseline 实际使用的镜像和配置。

---

## 1. istiod 与 TNG 的关系

**结论：istiod 安装步骤和 TNG 没有直接关系。**

在 Istio Gateway 部署流程中（`docs/resources/gateway/istio.md`）：

| 步骤 | 命令/操作 | 目的 | 与 TNG 关系 |
|---|---|---|---|
| Step 1 | 安装 Gateway API + Gateway API Inference Extension CRDs | 提供 `Gateway`、`HTTPRoute`、`InferencePool` 等 CRD | 无关 |
| Step 2 | `istioctl install` 部署 istiod | Istio 控制面，把 Gateway 资源编程为 Envoy 配置 | **无关** |
| Step 3 | `kubectl apply -k guides/recipes/gateway/istio` | 创建 `Gateway` 资源，Istio 据此生成数据面代理 | TNG 段 1 的后端就是这个 Gateway |
| Step 4 | 部署 optimized-baseline 的 model server 和 router | 搭建 EPP、vLLM、HTTPRoute | TNG 段 2/3 在此处插入 |

TNG 是独立的隧道进程，它只关心：
- 和谁建立隧道（对端 IP/端口）
- 用什么协议（RATS-TLS / OHTTP）
- 远程证明角色（attest / verify）

TNG 不读取 `Gateway`、`HTTPRoute`、`InferencePool` 等资源，也不依赖 Istio 控制面。istiod 只是让 Istio Gateway 能够正常工作；如果 TNG 段 1 的后端不是 Istio Gateway 而是其他入口，istiod 这一步完全可以跳过。

---

## 2. Optimized Baseline 配置命令 → 文件映射

### 2.1 前置条件

```bash
export GAIE_VERSION=v1.5.0
export ROUTER_CHART_VERSION=v0
export GUIDE_NAME="optimized-baseline"
export NAMESPACE=llm-d-optimized-baseline
export REPO_ROOT=$(realpath $(git rev-parse --show-toplevel))
```

| 命令 | 对应文件/资源 | 说明 |
|---|---|---|
| `kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/${GAIE_VERSION}/v1-manifests.yaml` | 远程 CRD manifest | 安装 Gateway API Inference Extension |
| `kubectl create namespace ${NAMESPACE}` | — | 创建部署命名空间 |
| `kubectl create secret generic llm-d-hf-token ...` | — | HuggingFace token，用于拉取模型 |

### 2.2 Gateway Mode 下 Istio 部署

文档：`docs/resources/gateway/istio.md`

```bash
ISTIO_VERSION=1.29.2
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} sh -
export PATH="$PWD/istio-${ISTIO_VERSION}/bin:$PATH"
istioctl install -y \
  --set values.pilot.env.ENABLE_GATEWAY_API_INFERENCE_EXTENSION=true
```

创建 Gateway：

```bash
export LLM_D_VERSION=main
kubectl apply -k "https://github.com/llm-d/llm-d/guides/recipes/gateway/istio?ref=${LLM_D_VERSION}" -n ${NAMESPACE}
```

本地对应文件：

| 文件 | 作用 |
|---|---|
| `guides/recipes/gateway/istio/kustomization.yaml` | Kustomize 入口，引用 `../base` 和 `configmap.yaml` |
| `guides/recipes/gateway/istio/gateway.yaml` | patch `Gateway`，设置 `gatewayClassName: istio` 和 `infrastructure.parametersRef` |
| `guides/recipes/gateway/istio/configmap.yaml` | 配置 Gateway 的 deployment/service 参数（`istio-proxy` resources、ClusterIP） |
| `guides/recipes/gateway/base/gateway.yaml` | 基础 `Gateway` 定义，监听 80 端口 |
| `guides/recipes/gateway/istio/telemetry.yaml` | 调试访问日志，默认被注释 |

### 2.3 Router 部署（Gateway Mode）

```bash
export PROVIDER_NAME=istio
helm install ${GUIDE_NAME} \
    oci://ghcr.io/llm-d/charts/llm-d-router-gateway-dev  \
    -f ${REPO_ROOT}/guides/recipes/router/base.values.yaml \
    -f ${REPO_ROOT}/guides/${GUIDE_NAME}/router/${GUIDE_NAME}.values.yaml \
    --set provider.name=${PROVIDER_NAME} \
    --set httpRoute.create=true \
    --set httpRoute.inferenceGatewayName=llm-d-inference-gateway \
    -n ${NAMESPACE} --version ${ROUTER_CHART_VERSION}
```

本地对应文件：

| 文件 | 作用 |
|---|---|
| `guides/recipes/router/base.values.yaml` | Router 基础配置：EPP 镜像、Envoy sidecar 参数、tracing 等 |
| `guides/optimized-baseline/router/optimized-baseline.values.yaml` | 本场景覆盖：extraServicePorts、pluginsConfig、modelServers matchLabels |
| `oci://ghcr.io/llm-d/charts/llm-d-router-gateway-dev` | Helm chart，包含 EPP Deployment、Envoy sidecar、Service、HTTPRoute、InferencePool |

`optimized-baseline.values.yaml` 关键内容：

```yaml
router:
  extraServicePorts:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8081   # Envoy 监听端口

  epp:
    pluginsConfigFile: "optimized-baseline-plugins.yaml"
    # 插件：queue-scorer, kv-cache-utilization-scorer, prefix-cache-scorer, no-hit-lru-scorer

  modelServers:
    matchLabels:
      llm-d.ai/guide: "optimized-baseline"
```

### 2.4 Model Server 部署

默认 GPU/vLLM/base：

```bash
export INFRA_PROVIDER=base
export MODEL_SERVER=vllm
kubectl apply -n ${NAMESPACE} -k ${REPO_ROOT}/guides/${GUIDE_NAME}/modelserver/gpu/${MODEL_SERVER}/${INFRA_PROVIDER}/
```

本地对应文件：

| 文件 | 作用 |
|---|---|
| `guides/optimized-baseline/modelserver/gpu/vllm/base/kustomization.yaml` | Kustomize 入口，引用 `recipes/modelserver/base/single-host/default`，设置镜像和标签 |
| `guides/optimized-baseline/modelserver/gpu/vllm/base/patch-vllm.yaml` | 覆盖 vLLM Deployment 参数 |

`patch-vllm.yaml` 关键内容：

```yaml
spec:
  replicas: 1
  template:
    spec:
      runtimeClassName: nvidia
      containers:
        - name: modelserver
          command: ["vllm", "serve"]
          args:
            - "/data/models/Qwen/Qwen3.5-9B/"
            - "--tensor-parallel-size=1"
            - "--gpu-memory-utilization=0.9"
          resources:
            limits:
              cpu: '4'
              memory: 32Gi
              nvidia.com/gpu: 1
```

这是用户提到的**降低硬件需求的修改点**：官方默认是 `Qwen/Qwen3-32B` + 8 replicas + TP=2 + 16 GPUs，本机测试改为 `Qwen3.5-9B` + 1 replica + TP=1 + 1 GPU。

### 2.5 验证命令

```bash
# Gateway Mode 获取入口 IP
export IP=$(kubectl get gateway llm-d-inference-gateway -n ${NAMESPACE} -o jsonpath='{.status.addresses[0].value}')

# 发送测试请求
kubectl run curl-debug --rm -it \
    --image=cfmanteiga/alpine-bash-curl-jq \
    --namespace="$NAMESPACE" \
    --env="IP=$IP" \
    -- /bin/bash

curl -X POST http://${IP}/v1/completions \
    -H 'Content-Type: application/json' \
    -d '{
        "model": "Qwen/Qwen3.5-9B",
        "prompt": "How are you today?"
    }' | jq
```

---

## 3. TNG 插入点与这些资源的对应关系

基于 `learning/deployment-llm-d-tng-k8s.md` 中的三段设计，把占位符替换为 optimized-baseline 真实资源：

| TNG 段 | TNG 角色 | 对端真实资源 | 备注 |
|---|---|---|---|
| 段 1 | Ingress (mapping) | 监听 LoadBalancer 8443 | 用户访问入口 |
| 段 1 | Egress (netfilter) | `llm-d-inference-gateway` Pod 内 istio-proxy 的 80 端口 | 解密后交给 Gateway |
| 段 2 | Ingress (netfilter) | llm-d-router-gateway chart 中的 Envoy sidecar | 捕获发往 EPP 的流量 |
| 段 2 | Egress (netfilter) | EPP Pod 内容器 | 解密后交给 EPP |
| 段 3 | Ingress (netfilter) | llm-d-router-gateway chart 中的 Envoy sidecar | 捕获发往 vLLM Service 的流量 |
| 段 3 | Egress (netfilter) | vLLM Pod 内 modelserver 的 8000 端口 | 解密后交给 vLLM |

需要替换的占位符：
- `<your-llm-d-envoy-image>` → 无需替换，Envoy 由 Helm chart 自动注入 sidecar
- `<your-llm-d-epp-image>` → `ghcr.io/llm-d/llm-d-router-endpoint-picker-dev:main`
- `<your-vllm-image>` → `vllm/vllm-openai:v0.19.1`
- `epp-service` → 由 Helm chart 创建的 EPP Service 名（通常是 `${GUIDE_NAME}-epp`）
- `vllm-service` → 由 Kustomize 创建的 vLLM Service 名（带 `optimized-baseline-nvidia-gpu-vllm-` 前缀）

---

## 4. 下一步建议

1. 在 `learning/deployment-llm-d-tng-k8s.md` 基础上，把占位符替换为 optimized-baseline 真实值。
2. 决定 TNG 以 sidecar 形式注入到哪些 Pod：
   - Gateway Pod（Istio 生成）中注入 Egress 较复杂，因为 Gateway Pod 由 Istio 控制面管理；更简单的做法是把 TNG Ingress 独立部署在 Gateway 前面。
   - Envoy + EPP 的 Pod 由 Helm chart 生成，需要修改 chart values 或 post-render 注入 TNG sidecar。
   - vLLM Pod 由 Kustomize 生成，可以直接在 `patch-vllm.yaml` 中增加 TNG Egress sidecar。
