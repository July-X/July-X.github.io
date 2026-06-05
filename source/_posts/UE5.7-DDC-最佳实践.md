---
title: "UE5.7 DDC 最佳实践：别再让 Cook 吃掉你的下午"
date: 2026-06-05 23:30:00
tags:
  - Unreal Engine
  - DDC
  - Zen Server
  - 构建系统
  - Cook
categories:
  - 开发日志
top_img: false
---

<!-- more -->

"改了一个材质的 Roughness 参数，Cook 跑了 45 分钟。"

这不是段子。这是我上个月在朋友工作室亲眼看到的——一个 12 人的独立团队，每次改点东西就要等大半个小时 Cook。检查之后发现：DDC 根本没配。所有资产，每次 Cook，全部重新算一遍。

他们花的钱买 build 机器，不如把 DDC 配好。

---

## DDC 到底是什么？

DDC（Derived Data Cache，派生数据缓存）是 UE5 对"已经算过的东西不再算第二遍"的实现。当你 Cook 一张贴图、编译一个 Shader、或者导入一个 Mesh 时，引擎会把处理后的结果缓存在 DDC 里。下次用到同样的输入，直接从缓存读，跳过计算。

DDC 的缓存键（Cache Key）由这些因素决定：

```
DDC Key = Hash(Asset 源码) + Engine 版本 + 平台 + 编译配置 + DDC 版本号
```

任何一个变了，Key 就不同，等于 Cache Miss，要重算。所以：

- 升级引擎 → 全量 DDC Miss
- 改了资产 → 只 Miss 这一个
- 换了平台（Win64 → Linux） → 全量 Miss
- 改了 Shader 模板 → 所有用到它的材质 Miss

---

## UE 5.7 的 DDC 格局：Zen 一统天下

UE5 的 DDC 后端经历了几轮变迁：

```
UE 4.x               UE 5.0~5.3              UE 5.4~5.6              UE 5.7
────────────────────────────────────────────────────────────────────────
Shared DDC (NAS) ────┼──────────────────────┼──────────────────────┼──→ 传统方案
Cloud DDC (S3)  ──────┤──────────────────────┤                       → 已弃用
Zen Server ───────────┼──────────────────────┼──────────────────────┼──→ 主力方案
                     (内部实验)            (对外可用)            (成熟稳定)
```

到了 UE 5.7，Zen Server 已经是事实标准。Epic 把优化精力全部放在 Zen 上，Shared DDC（NAS 共享目录）虽然还能用，但官方文档的第一推荐就是 Zen。

**为什么 Zen 比其他方案好？**

| | Shared DDC (NAS) | Cloud DDC (S3) | Zen Server |
|---|---|---|---|
| 延迟 | 低（局域网内） | 高（HTTP 往返） | 极低（gRPC 流式） |
| 并发 | SMB 锁竞争 | 好 | 优秀（无锁设计） |
| 去重 | 无 | 按 blob | 内容寻址，自动去重 |
| 运维 | 简单 | 需要云服务 | 一个二进制，一行命令启动 |
| UE 5.7 推荐度 | 兼容方案 | 已弃用 | ⭐⭐⭐⭐⭐ |

---

## Zen Server：十分钟搭起来

UE 5.7 自带 Zen Server 的二进制和 Docker 镜像。最简单的部署：

**方案一：裸机启动**

```powershell
# 从 UE 5.7 引擎目录找到 ZenServer.exe
# 默认端口 8558，数据存 D:\ZenData
D:\UE_5.7\Engine\Binaries\Win64\ZenServer.exe `
    --port 8558 `
    --data-dir D:\ZenData `
    --gc-interval 24h
```

**方案二：Docker Compose（推荐 CI 环境）**

```yaml
# docker-compose.yml
version: '3.8'
services:
  zen-server:
    image: ghcr.io/epicgames/zen-server:5.7
    ports:
      - "8558:8558"
    volumes:
      - zen_data:/data
      - zen_cold:/cold-storage  # 本地冷存储，NAS 大容量盘
    environment:
      ZEN_PORT: "8558"
      ZEN_DATA_PATH: "/data"
      ZEN_COLD_STORAGE: "/cold-storage"  # 本地 NAS 路径，归档 90 天以上冷数据
    restart: unless-stopped

volumes:
  zen_data:
  zen_cold:
```

启动后验证：

```bash
# gRPC 健康检查
grpcurl -plaintext zen-server.internal:8558 list
```

---

## DDC Graph：缓存分层策略

UE 5.7 的 DDC Graph 在 `DefaultEngine.ini` 中配置。最佳实践是**三层缓存**：

```ini
; Engine/Config/DefaultEngine.ini

; ═══════════════════════════════════════════
; DDC Graph — UE 5.7 推荐三层架构
; ═══════════════════════════════════════════

[DerivedDataBackendGraph]

; ── 第一层：Local Zen（每台机器独立）
; 开发机：本地磁盘超快读写，热数据
;   特性：50GB 上限，LRU 淘汰，写入直通共享层
Local=(Type=Zen, Host=localhost, Port=8558, 
       MaxSizeGB=50, 
       WritePolicy="Through")

; ── 第二层：Shared Zen（团队共享）
; 局域网 Zen Server，所有机器和 CI Agent 共用
;   特性：无容量上限，内容去重，读未命中时查询冷存储
Shared=(Type=Zen, Host=zen-team.internal, Port=8558,
        WritePolicy="Through",
        ColdStorageFallback=true)

; ── 第三层：本地冷存储（Zen Server 本地 NAS，自动管理）
; 长期归档，90 天未访问自动清理
;   特性：配置在 Zen Server 侧，客户端无感知
;   数据全程不出机房，内网安全可控

; ═══════════════════════════════════════════
; Root 节点从本地查起，一级级 fallback
; ═══════════════════════════════════════════
Root=(Type=Hierarchical, Heuristics=Manual, 
      Inner=Local, Inner=Shared)
```

三层配合的效果：

```
查询路径（读）：
  Local Zen (50GB, NVMe, ~0.1ms)
    → Miss → Shared Zen (无限, LAN, ~2ms)
      → Miss → 本地冷存储 (NAS, ~5ms)
        → Miss → 重新计算 ← 成本高，要避免

写入路径（写）：
  重新计算 → 写入 Local Zen (write-through)
    → 同步写入 Shared Zen (write-through)
      → Zen Server 异步归档到本地冷存储
```

---

## 冷启动问题：最容易被忽略的成本

DDC 最大的坑不是配置，是**冷启动**。

场景：团队新来一个成员，`git clone` 项目，打开 Editor，点了 Play。

此时他的 Local Zen 是空的。所有 Shader 编译、所有资产加载，全部需要重新计算。Editor 启动 30 分钟是常事。

**解决方案：种子 DDC**

UE 5.7 提供了 `UnrealZen` 工具来做 DDC 预热：

```bash
# 在 CI 构建机（已有热 DDC）上，导出种子包
UnrealZen export                          \
    --server zen-team.internal:8558       \
    --namespace MyProject                 \
    --output /mnt/nas/ddc-seed/MyProject.zen

# 在新机器上导入
UnrealZen import                          \
    --server localhost:8558               \
    --input /mnt/nas/ddc-seed/MyProject.zen
```

**CI 流水线里的种子策略**：

```
每周一凌晨 CI 触发：
  1. Full Cook 一次（此时 DDC 最热）
  2. 导出 Zen 种子包 → 上传到 NAS
  3. 归档上月种子包（保留最近 4 周）

新 Agent / 新开发者启动：
  1. 下载最新种子包
  2. 导入到 Local Zen
  3. Editor 启动 → 95% DDC 命中率
```

---

## CI 环境的 DDC 最佳实践

**反模式：每个 CI Agent 跑自己的 Local DDC**

```
❌ Agent #1 ── Local DDC ──┐
❌ Agent #2 ── Local DDC ──┤ 各自独立，互相不共享
❌ Agent #3 ── Local DDC ──┘  命中率极低
```

**正确做法：所有 Agent 指向同一个 Shared Zen**

```
✅ Agent #1 ──┐
✅ Agent #2 ──┼── Shared Zen Server ──→ 本地冷存储 NAS
✅ Agent #3 ──┘
```

CI 环境下的额外配置：

```ini
; CI Agent 的 DefaultEngine.ini 覆盖
; （通过 BuildGraph 或命令行参数注入）

[DerivedDataBackendGraph]
; CI 环境不需要 Local Zen——直连 Shared
Local=(Type=Zen, Host=localhost, Port=8558,
       MaxSizeGB=10,          # CI 机器磁盘有限
       WritePolicy="Through")

; 可以关闭 EditorUsage 相关缓存，省空间
[ShaderCompiler]
; CI 只编 Shipping Shader，不编 Editor/Development
r.Shaders.Optimize=1
```

**Cook 时指定 DDC 参数**：

```bash
# CI Cook 命令 — 利用 Shared Zen 加速
UE5Editor-Cmd.exe MyProject.uproject     \
    -run=Cook                             \
    -TargetPlatform=Win64                 \
    -DDC=Shared                           \  # 优先用 Shared Zen
    -NoDDCWorker                          \  # CI 不需本地 DDC worker
    -Unattended
```

---

## 监控与调试：DDC 不是黑盒

UE 5.7 提供了丰富的 DDC 可观测性工具。

**运行时统计**：

```bash
# 启动 Editor 时记录 DDC 统计
UE5Editor.exe MyProject.uproject -DDC-ShowStats

# 输出示例：
# DDC Stats:
#   Local Zen:        23,451 hits,    142 misses  (99.4% hit rate)
#   Shared Zen:          138 hits,      4 misses  (  .   )
#   Cold Storage:         2 hits,      2 misses  (  .   )
#   Total compute:         2 new derivations
#   Total time saved: ~47 minutes
```

**Zen Server 自身监控**：

Zen Server 暴露 Prometheus 端点：

```yaml
# Prometheus scrape config
scrape_configs:
  - job_name: 'zen-server'
    scrape_interval: 30s
    static_configs:
      - targets: ['zen-team.internal:9100']  # Zen metrics port
```

关键指标：

| 指标 | 含义 | 告警阈值 |
|------|------|----------|
| `zen_cache_hit_rate` | 缓存命中率 | < 80% |
| `zen_cache_size_bytes` | 缓存占用 | > 磁盘 80% |
| `zen_request_latency_p99` | P99 延迟 | > 10ms |
| `zen_cold_storage_fallback_rate` | 回退到本地冷存储的比例 | > 5% 说明热数据被淘汰 |

**常见问题诊断**：

```bash
# 1. 命中率低：检查 key 是否频繁变化
UE5Editor.exe -DDC-ShowKeys  # 打印每次查询的 DDC Key

# 2. 延迟高：检查网络
ping zen-team.internal        # 应 < 2ms（局域网）
# 如果 > 5ms，考虑在每台机器上加大 Local Zen

# 3. 磁盘爆满：检查 Zen 垃圾回收
ZenServer.exe --gc-now        # 手动触发 GC
```

---

## 五个常见错误

**错误一：Zen Server 放云上**

> "我们把 Zen Server 部署在阿里云上，整个团队 VPN 连过去。"

结果：gRPC 延迟 30ms+，每次 DDC 查询比本地计算还慢。而且项目资产通过公网传输，安全风险不可控。**Zen 必须部署在本地机房，和 Editor/CI Agent 在同一局域网内，延迟 < 2ms。所有数据不出机房。**

**错误二：不设 Local Zen 上限**

本地磁盘 1TB，Zen 用掉了 900GB——都是三个月前的资产，早就不用了。`MaxSizeGB` 是强制性的，不是可选的。

**错误三：不同分支共用同一个 Namespace**

`main` 分支和 `feature-x` 分支如果共用同一个 Zen namespace，DDC 会互相污染。正确做法：

```ini
; 每个分支独立的 DDC namespace
; main 分支
[DerivedDataBackendGraph]
Shared=(Type=Zen, Namespace="MyProject-main")

; feature 分支（通过命令行覆盖）
UE5Editor.exe -DDC-SharedNamespace="MyProject-feature-x"
```

**错误四：CI 和开发者共用同一个 Local Zen**

CI Agent 的 Local Zen 会写入大量临时数据，和开发者共用会挤掉有效缓存。CI Agent 应该用独立的 Zen 实例，或者直接只用 Shared Zen。

**错误五：忽视 DDC 版本号**

引擎升级后，旧的 DDC 数据不会被自动清理——它会一直占着磁盘空间，但再也不会被命中。升级引擎后清理旧的 Local Zen 数据：

```bash
# 引擎升级后清理旧 DDC
rm -rf ~/Library/Application\ Support/Epic/UnrealEngine/Common/DerivedDataCache/*
```

---

**更新记录**

- 2026-06-05：初稿发布。

---

> 一个 12 核的 CPU 跑 Cook 任务，80% 的时间在等 IO。你把 CPU 换成 32 核，等待时间还是 80%——只不过多了 20 个空闲的核而已。DDC 要解决的不是"算得快"，而是"不用算"。你的 DDC 命中率是多少？
