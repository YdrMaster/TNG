# TNG 业务原理与配置学习路线

> 目标：不深入 Rust 代码，重点理解 TNG 是什么、为什么这样设计、如何配置使用。
> 学习过程中会在本目录持续输出图文并茂的解读文档。

---

## 阶段一：建立整体认知（1-2 天）

### 学习目标
- 理解 TNG 在机密计算生态中的定位
- 清楚 Ingress / Egress 两个角色的分工
- 了解 RATS-TLS 和 OHTTP 两种传输协议的区别

### 学习资料
| 文件 | 重点 |
|---|---|
| `README.md` / `README_zh.md` | 项目定位、核心能力、快速开始 |
| `docs/architecture.md` / `architecture_zh.md` | Ingress/Egress 架构、RA 角色、协议对比 |
| `docs/scenarios/` 目录 | 典型部署拓扑 |

### 输出物
- 本阶段解读文档：`learning/step-01-architecture-overview.md`
- TNG 数据流图：`learning/images/step-01-tunnel-model.png`
- RATS-TLS vs OHTTP 对比图：`learning/images/step-01-protocol-comparison.png`
- （扩展）安全威胁模型：`learning/security-threat-model.md`

---

## 阶段二：远程证明（Remote Attestation）基础（2-3 天）

### 学习目标
- 理解为什么需要远程证明
- 掌握 RATS 模型中的关键角色：Attester / Verifier / Relying Party
- 理解 TNG 中的证据（Evidence）和令牌（Token）流转

### 学习资料
| 文件 | 重点 |
|---|---|
| `docs/architecture.md` 中 RA 相关章节 | TNG 里谁生成证据、谁验证证据 |
| `docs/peer_shared.md` / `peer_shared_zh.md` | peer_shared 模式下的密钥协商 |
| `rats-cert/README.md` | rats-cert 在证据/证书/验证中的职责 |

### 输出物
- 本阶段解读文档：`learning/step-02-remote-attestation.md`
- RATS 角色关系图：Mermaid `graph LR`
- Background Check / Passport 模式对比图：Mermaid `graph LR`

---

## 阶段三：传输协议深度理解（2-3 天）

### 3.1 RATS-TLS
- 在标准 TLS 1.3 握手之上做了什么扩展
- Evidence 是怎么嵌入证书的
- 客户端（Ingress）如何验证服务端（Egress）的身份

### 3.2 OHTTP
- Oblivious HTTP 的基本思想：Gateway / Relay / Target 三分离
- TNG 中 OHTTP 的密钥管理三种模式：`self_generated`、`file`、`peer_shared`
- OHTTP 与 RATS-TLS 的关系

### 学习资料
| 文件 | 重点 |
|---|---|
| `docs/architecture.md` | RATS-TLS 和 OHTTP 的架构图 |
| `docs/peer_shared.md` / `peer_shared_zh.md` | peer_shared 的集群发现、密钥同步 |
| `docs/scenarios/` 中 ohttp 相关示例 | OHTTP 实际部署长什么样 |

### 输出物
- 本阶段解读文档：`learning/step-03-transport-protocols.md`
- RATS-TLS 握手流程图：`learning/images/step-03-rats-tls-handshake.png`
- OHTTP 消息流程图：`learning/images/step-03-ohttp-flow.png`

---

## 阶段四：配置方法实战（3-5 天）

### 学习目标
- 能读懂并独立编写 TNG 的 JSON 配置
- 理解每个配置字段的含义和相互依赖
- 掌握不同场景下的配置组合

### 学习资料
| 文件 | 重点 |
|---|---|
| `docs/configuration.md` / `configuration_zh.md` | 所有配置字段的权威参考 |
| `docs/scenarios/` 下所有示例 | 真实场景的配置模板 |
| `dist/config.json` | 默认配置文件 |
| `tng-python/README.md` | SDK 层如何封装配置 |

### 输出物
- 本阶段解读文档：`learning/step-04-configuration.md`
- 至少 3 个可运行的配置示例：`learning/configs/`

---

## 阶段五：部署与运维视角（2-3 天）

### 学习目标
- 理解 TNG 如何接入真实基础设施
- 掌握 systemd、Docker、RPM 三种部署形态
- 理解可观测性配置（metrics / traces / logs）

### 学习资料
| 文件 | 重点 |
|---|---|
| `dist/trusted-network-gateway.service` | systemd 启动方式 |
| `Dockerfile` | 容器镜像构建依赖和启动命令 |
| `trusted-network-gateway.spec` | RPM 包安装内容 |
| `docs/configuration.md` 中 observability 部分 | metric / trace / log 配置 |
| `Makefile` | 常用构建/测试/打包命令 |

### 输出物
- 本阶段解读文档：`learning/step-05-deployment.md`
- 部署方式对比图：`learning/images/step-05-deployment-comparison.png`

---

## 阶段六：安全与排错（2 天）

### 学习目标
- 理解 TNG 的信任根和威胁模型
- 知道 `no_ra` 为什么不能用于生产
- 掌握常见问题的排查思路

### 学习资料
| 文件 | 重点 |
|---|---|
| `AGENTS.md` 中 Security Considerations | 安全底线 |
| `docs/developer.md` / `developer_zh.md` | 测试环境搭建、常见坑 |
| `docs/version_compatibility.md` | 版本兼容性注意事项 |

### 输出物
- 本阶段解读文档：`learning/step-06-security-troubleshooting.md`
- 安全 checklist：`learning/step-06-security-checklist.md`

---

## 推荐学习顺序

| 天数 | 主题 | 主要输出物 |
|---|---|---|
| 1-2 | 阶段一：整体认知 | `step-01-architecture-overview.md` |
| 3-5 | 阶段二：远程证明基础 | `step-02-remote-attestation.md` |
| 6-8 | 阶段三：传输协议 | `step-03-transport-protocols.md` |
| 9-13 | 阶段四：配置实战 | `step-04-configuration.md` |
| 14-15 | 阶段五：部署运维 | `step-05-deployment.md` |
| 16+ | 阶段六：安全排错 | `step-06-security-troubleshooting.md` |

---

## 贯穿全程的三个问题

每读一个文档，都尝试回答：

1. **数据从哪来？**（客户端、上游、Attestation Service？）
2. **信任根是什么？**（证书、Evidence、Token、HPKE 公钥？）
3. **配置哪个字段控制它？**

这三个问题串起来，就能从业务原理自然过渡到配置方法。
