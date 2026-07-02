# K8s 部署指南：llm-d Optimized Baseline + 三段 TNG

> 目标：在 Kubernetes 集群中部署 llm-d optimized baseline，并在三处关键链路插入 TNG 隧道。
> - 段 1：外部用户 → Istio Gateway
> - 段 2：llm-d Envoy → EPP
> - 段 3：llm-d Envoy → vLLM

---

## 1. 前置条件

| 组件 | 版本/要求 |
|---|---|
| Kubernetes | v1.25+，cgroup v2 推荐 |
| Container Runtime | containerd，已配置 `ctr -n k8s.io` 可访问 |
| Istio | 已安装 Ingress Gateway（可选，用于南北向入口） |
| TNG 镜像 | `ghcr.io/inclavare-containers/tng:2.6.0` 已拉取到 `ctr -n k8s.io` |
| 权限 | TNG netfilter 模式需要 `NET_ADMIN` / `privileged` |

拉取镜像到 containerd（node 上执行）：

```bash
ctr -n k8s.io images pull ghcr.io/inclavare-containers/tng:2.6.0
```

---

## 2. 总体架构

```mermaid
graph LR
    subgraph 集群外部
        U[用户 curl]
    end

    subgraph 段1：用户 → Istio Gateway
        I1[TNG Ingress<br/>mapping模式<br/>LoadBalancer Service]
        I1 -->|RATS-TLS 隧道| E1
        E1[TNG Egress<br/>netfilter模式]
        E1 --> IG[Istio Gateway]
    end

    subgraph 段2：Envoy → EPP
        IG --> EN[llm-d Envoy]
        EN -->|访问 epp:9002<br/>被 netfilter 捕获| I2
        I2[TNG Ingress<br/>netfilter模式]
        I2 -->|RATS-TLS 隧道| E2
        E2[TNG Egress<br/>netfilter模式]
        E2 --> EP[EPP Pod]
    end

    subgraph 段3：Envoy → vLLM
        EN -->|访问 vllm:8000<br/>被 netfilter 捕获| I3
        I3[TNG Ingress<br/>netfilter模式]
        I3 -->|RATS-TLS 隧道| E3
        E3[TNG Egress<br/>netfilter模式]
        E3 --> VLLM[vLLM Pod]
    end
```

设计要点：
- **段 1** 使用 `mapping` 模式，因为外部用户无法被透明拦截，必须暴露明确的 TNG 入口。
- **段 2 / 段 3** 使用 `netfilter` 模式，对 Envoy、EPP、vLLM 透明，不修改它们的监听端口或访问地址。
- 所有 TNG 容器以 sidecar 或独立 Deployment 方式运行，共享 network namespace。

---

## 3. 命名空间与基础配置

以下示例使用命名空间 `llm-d`。请先创建：

```bash
kubectl create namespace llm-d
```

---

## 4. ConfigMap：TNG 各段配置

### 4.1 段 1：外部入口 Ingress

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tng-segment-1-ingress
  namespace: llm-d
data:
  config.json: |
    {
      "add_ingress": [
        {
          "mapping": {
            "in": { "host": "0.0.0.0", "port": 8443 },
            "out": { "host": "istio-gateway", "port": 8080 }
          },
          "rats_tls": {
            "cert": { "source": "self_signed" }
          },
          "verify": {
            "as_addr": "http://attestation-service:8080/",
            "policy_ids": ["default"]
          }
        }
      ]
    }
```

### 4.2 段 1：Istio Gateway 侧 Egress

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tng-segment-1-egress
  namespace: llm-d
data:
  config.json: |
    {
      "add_egress": [
        {
          "netfilter": {
            "capture_dst": [{ "port": 8080 }],
            "capture_local_traffic": false,
            "listen_port": 40000,
            "so_mark": 565
          },
          "rats_tls": {
            "cert": { "source": "self_signed" }
          },
          "attest": {
            "aa_addr": "unix:///run/confidential-containers/attestation-agent/attestation-agent.sock"
          }
        }
      ]
    }
```

### 4.3 段 2：Envoy 侧 Ingress

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tng-segment-2-ingress
  namespace: llm-d
data:
  config.json: |
    {
      "add_ingress": [
        {
          "netfilter": {
            "capture_dst": [
              { "host": "epp-service", "port": 9002 }
            ],
            "listen_port": 50002,
            "so_mark": 566
          },
          "rats_tls": {
            "cert": { "source": "self_signed" }
          },
          "verify": {
            "as_addr": "http://attestation-service:8080/",
            "policy_ids": ["default"]
          }
        }
      ]
    }
```

### 4.4 段 2：EPP 侧 Egress

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tng-segment-2-egress
  namespace: llm-d
data:
  config.json: |
    {
      "add_egress": [
        {
          "netfilter": {
            "capture_dst": [{ "port": 9002 }],
            "capture_local_traffic": false,
            "listen_port": 40002,
            "so_mark": 566
          },
          "rats_tls": {
            "cert": { "source": "self_signed" }
          },
          "attest": {
            "aa_addr": "unix:///run/confidential-containers/attestation-agent/attestation-agent.sock"
          }
        }
      ]
    }
```

### 4.5 段 3：Envoy 侧 Ingress

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tng-segment-3-ingress
  namespace: llm-d
data:
  config.json: |
    {
      "add_ingress": [
        {
          "netfilter": {
            "capture_dst": [
              { "host": "vllm-service", "port": 8000 }
            ],
            "listen_port": 50003,
            "so_mark": 567
          },
          "rats_tls": {
            "cert": { "source": "self_signed" }
          },
          "verify": {
            "as_addr": "http://attestation-service:8080/",
            "policy_ids": ["default"]
          }
        }
      ]
    }
```

### 4.6 段 3：vLLM 侧 Egress

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tng-segment-3-egress
  namespace: llm-d
data:
  config.json: |
    {
      "add_egress": [
        {
          "netfilter": {
            "capture_dst": [{ "port": 8000 }],
            "capture_local_traffic": false,
            "listen_port": 40003,
            "so_mark": 567
          },
          "rats_tls": {
            "cert": { "source": "self_signed" }
          },
          "attest": {
            "aa_addr": "unix:///run/confidential-containers/attestation-agent/attestation-agent.sock"
          }
        }
      ]
    }
```

---

## 5. Deployment 清单

### 5.1 段 1：TNG Ingress 独立入口

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tng-segment-1-ingress
  namespace: llm-d
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tng-segment-1-ingress
  template:
    metadata:
      labels:
        app: tng-segment-1-ingress
    spec:
      containers:
        - name: tng
          image: ghcr.io/inclavare-containers/tng:2.6.0
          command: ["tng"]
          args: ["launch", "--config-file", "/etc/tng/config.json"]
          securityContext:
            privileged: true
            capabilities:
              add: ["NET_ADMIN"]
          ports:
            - containerPort: 8443
          volumeMounts:
            - name: config
              mountPath: /etc/tng
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
      volumes:
        - name: config
          configMap:
            name: tng-segment-1-ingress
---
apiVersion: v1
kind: Service
metadata:
  name: tng-segment-1-ingress
  namespace: llm-d
spec:
  type: LoadBalancer
  selector:
    app: tng-segment-1-ingress
  ports:
    - port: 8443
      targetPort: 8443
```

### 5.2 Istio Gateway + TNG Egress sidecar

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istio-gateway
  namespace: llm-d
spec:
  replicas: 2
  selector:
    matchLabels:
      app: istio-gateway
  template:
    metadata:
      labels:
        app: istio-gateway
    spec:
      containers:
        - name: istio-proxy
          image: istio/proxyv2:1.24.0
          # ... Istio Gateway 标准配置 ...
          ports:
            - containerPort: 8080
        - name: tng-egress
          image: ghcr.io/inclavare-containers/tng:2.6.0
          command: ["tng"]
          args: ["launch", "--config-file", "/etc/tng/config.json"]
          securityContext:
            privileged: true
            capabilities:
              add: ["NET_ADMIN"]
          volumeMounts:
            - name: tng-config
              mountPath: /etc/tng
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
      volumes:
        - name: tng-config
          configMap:
            name: tng-segment-1-egress
```

### 5.3 llm-d Envoy + TNG Ingress sidecar（段 2 + 段 3）

一个 TNG 进程同时配置两个 Ingress：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tng-envoy-ingress
  namespace: llm-d
data:
  config.json: |
    {
      "add_ingress": [
        {
          "netfilter": {
            "capture_dst": [{ "host": "epp-service", "port": 9002 }],
            "listen_port": 50002,
            "so_mark": 566
          },
          "rats_tls": { "cert": { "source": "self_signed" } },
          "verify": {
            "as_addr": "http://attestation-service:8080/",
            "policy_ids": ["default"]
          }
        },
        {
          "netfilter": {
            "capture_dst": [{ "host": "vllm-service", "port": 8000 }],
            "listen_port": 50003,
            "so_mark": 567
          },
          "rats_tls": { "cert": { "source": "self_signed" } },
          "verify": {
            "as_addr": "http://attestation-service:8080/",
            "policy_ids": ["default"]
          }
        }
      ]
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-d-envoy
  namespace: llm-d
spec:
  replicas: 2
  selector:
    matchLabels:
      app: llm-d-envoy
  template:
    metadata:
      labels:
        app: llm-d-envoy
    spec:
      containers:
        - name: envoy
          image: <your-llm-d-envoy-image>
          # ... llm-d Envoy 配置 ...
        - name: tng-ingress
          image: ghcr.io/inclavare-containers/tng:2.6.0
          command: ["tng"]
          args: ["launch", "--config-file", "/etc/tng/config.json"]
          securityContext:
            privileged: true
            capabilities:
              add: ["NET_ADMIN"]
          volumeMounts:
            - name: tng-config
              mountPath: /etc/tng
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
      volumes:
        - name: tng-config
          configMap:
            name: tng-envoy-ingress
```

### 5.4 EPP + TNG Egress sidecar

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: epp
  namespace: llm-d
spec:
  replicas: 2
  selector:
    matchLabels:
      app: epp
  template:
    metadata:
      labels:
        app: epp
    spec:
      containers:
        - name: epp
          image: <your-llm-d-epp-image>
          ports:
            - containerPort: 9002
        - name: tng-egress
          image: ghcr.io/inclavare-containers/tng:2.6.0
          command: ["tng"]
          args: ["launch", "--config-file", "/etc/tng/config.json"]
          securityContext:
            privileged: true
            capabilities:
              add: ["NET_ADMIN"]
          volumeMounts:
            - name: tng-config
              mountPath: /etc/tng
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
      volumes:
        - name: tng-config
          configMap:
            name: tng-segment-2-egress
---
apiVersion: v1
kind: Service
metadata:
  name: epp-service
  namespace: llm-d
spec:
  selector:
    app: epp
  ports:
    - port: 9002
      targetPort: 9002
```

### 5.5 vLLM + TNG Egress sidecar

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm
  namespace: llm-d
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vllm
  template:
    metadata:
      labels:
        app: vllm
    spec:
      containers:
        - name: vllm
          image: <your-vllm-image>
          ports:
            - containerPort: 8000
        - name: tng-egress
          image: ghcr.io/inclavare-containers/tng:2.6.0
          command: ["tng"]
          args: ["launch", "--config-file", "/etc/tng/config.json"]
          securityContext:
            privileged: true
            capabilities:
              add: ["NET_ADMIN"]
          volumeMounts:
            - name: tng-config
              mountPath: /etc/tng
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
      volumes:
        - name: tng-config
          configMap:
            name: tng-segment-3-egress
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-service
  namespace: llm-d
spec:
  selector:
    app: vllm
  ports:
    - port: 8000
      targetPort: 8000
```

---

## 6. 应用所有资源

```bash
kubectl apply -f tng-configmaps.yaml
kubectl apply -f tng-deployments.yaml
kubectl apply -f llm-d-components.yaml
```

---

## 7. 验证步骤

### 7.1 查看 Pod 状态

```bash
kubectl get pods -n llm-d
```

### 7.2 查看 TNG 日志

```bash
kubectl logs -n llm-d deployment/llm-d-envoy -c tng-ingress -f
kubectl logs -n llm-d deployment/vllm -c tng-egress -f
```

### 7.3 测试段 1 入口

```bash
export TNG_INGRESS=$(kubectl get svc tng-segment-1-ingress -n llm-d -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -k https://$TNG_INGRESS:8443/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"test","messages":[{"role":"user","content":"hello"}]}'
```

### 7.4 检查控制面健康

如果配置了 `control_interface`：

```bash
kubectl port-forward -n llm-d deployment/vllm 8081:8081
curl http://127.0.0.1:8081/livez
curl http://127.0.0.1:8081/readyz
```

---

## 8. 关键注意事项

| 问题 | 说明 |
|---|---|
| `privileged: true` | netfilter 模式操作 iptables，必须开启 |
| `so_mark` | 每段使用不同 mark 值，避免回环 |
| `capture_local_traffic` | Envoy 到同集群 Service 的流量通常不是 local traffic，设为 `false` |
| Attestation Agent | Egress sidecar 需要挂载 AA socket 或运行 AA sidecar |
| Istio mTLS | 如果 Istio 已开启 mTLS，TNG 隧道与 Istio mTLS 是嵌套关系，注意端口冲突 |
| readinessProbe | vLLM 的 probe 指向 8000 会经过 TNG Egress，确保 TNG 先启动 |

---

## 9. 故障排查

| 现象 | 排查方向 |
|---|---|
| TNG 启动失败 | 检查配置 JSON 语法；确认 `attest`/`verify` 与传输协议配对 |
| 连接超时 | 检查 `capture_dst` 是否匹配实际访问地址；确认 `so_mark` 无冲突 |
| 远程证明失败 | 确认 AA socket 可达；AS 地址和 `policy_ids` 正确 |
| 流量未进入隧道 | 在 sidecar 容器内执行 `iptables -t nat -L -n -v` 查看规则 |
| Envoy 返回 503 | 确认 EPP 和 vLLM 健康；TNG Egress 已就绪 |
