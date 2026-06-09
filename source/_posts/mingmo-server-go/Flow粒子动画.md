---
title: "Flow 粒子动画：UDP 消息流的可视化实现"
date: 2026-06-08 20:00:00
tags:
  - Go
  - Electron
  - 可视化
  - SSE
  - AntV X6
categories:
  - mingmo-server-go
top_img: false
---

<!-- more -->

游戏联机服务器的 UDP 流量监控一直是个麻烦事。日志里密密麻麻的 ingress/forward/drop 事件，运维人员需要在海量数据中肉眼定位异常包。我们给 `mingmo-server-go` 的 UDP Proxy 加了一个实时可视化面板，把消息流变成了拓扑图上的粒子动画。

这篇文章讲的是这个可视化系统的实现，重点放在 Flow 粒子动画的设计和踩坑。

## 问题：为什么需要消息流可视化

UDP Proxy 转发层每秒处理数万个包。传统的日志排查方式有两个痛点：

1. **延迟高**：从出问题到发现问题，中间隔着日志采集、落盘、查询，至少几分钟
2. **上下文丢失**：单条日志只能看到"收到了包"，看不到"这个包从哪来、到哪去、有没有丢"

我们需要一个实时的、有拓扑关系的可视化工具。

## 方案选型：三种实现路径

我们在 X6 重写时评估了三种 Flow 粒子的实现方案。

### 方案 A：Edge + strokeDasharray 动画

X6 的边支持 `strokeDasharray` 和 `strokeDashoffset` 动画。创建一条虚线边，通过动画让虚线"流动"，视觉上就像粒子在移动。

```javascript
const edge = graph.addEdge({
  id: `flow-${Date.now()}`,
  source: [from.x, from.y],
  target: [to.x, to.y],
  attrs: {
    line: {
      stroke: color,
      strokeWidth: width,
      strokeDasharray: Math.max(4, Math.round(len / 14)),
    },
  },
});

edge.animate(
  { "attrs/line/strokeDashoffset": -len },
  { duration: 1200, iterations: Infinity, easing: "linear" },
);
```

**优点**：零自定义 NodeView，纯 X6 API。代码量小，维护成本低。

**缺点**：只能做"虚线流动"效果，做不出 trail 拖尾和 glow 阴影。

### 方案 B：自定义 NodeView

继承 X6 的 `NodeView`，每帧用 `requestAnimationFrame` 在 bezier 路径上计算当前位置，自己画 trail。

**优点**：视觉效果可以做到 1:1 还原 Canvas2D 原版。

**缺点**：自己接管动画循环，X6 的自动渲染不参与。边管理复杂，和 X6 体系割裂。

### 方案 C：SVG overlay 层

X6 画静态拓扑，额外叠一层 SVG 在画布上面画 flow 粒子，手动算坐标转换。

**优点**：完全控制粒子样式。

**缺点**：双层渲染，坐标转换麻烦，容易出现对不齐的问题。

**最终选择了方案 A**。视觉差异是 flow 粒子从"带 trail 的彗星"变成"无 trail 的亮圆点"，但颜色、位置、时序都不变，业务上可以接受。

## 核心实现：flow 类型映射

UDP 流量事件有多种类型，每种类型的粒子动画逻辑不同：

| 事件类型 | 粒子方向 | 颜色 | 说明 |
|---------|---------|------|------|
| ingress | 玩家 → Proxy Port | 蓝色 | 上行数据包 |
| forward | Proxy Port → 玩家 | 绿色 | 下行转发 |
| drop | 玩家位置 | 红色 | 丢包标记 |
| handshake | 玩家 → Proxy Port | 金色 | 握手包 |
| send | 双向 | 黄色 | 控制触发（合成） |
| join/leave/bind | Room 上方 | 各色 | 房间事件气泡 |

`enqueueFlow` 是入口函数，根据 `flow.kind` 分发到不同的绘制逻辑：

```javascript
function enqueueFlow(flow) {
  if (!graph || !flow) return;
  const style = cfg.FLOW_STYLES[flow.kind] || cfg.FLOW_STYLES.ingress;

  // 计数（除 send 外）
  if (flow.kind !== "send") {
    if (Number.isInteger(flow.slot)) bumpPortActivity(flow.slot, flow.kind);
    if (flow.roomId) bumpRoomActivity(flow.roomId, flow.kind);
  }

  if (flow.kind === "send") {
    // send 是控制触发器，但前端要画出对应的 ingress + forward 粒子
    // 让用户能直观看到"消息从 sender 进 proxy，再被 proxy 广播给同房间其他人"的完整链路
    const eps = resolveEndpoints(flow);
    // ... 绘制上行和下行粒子
    return;
  }
  // ... 其他类型的处理
}
```

## 时序控制：BURST 和 STAGGER

单个粒子太单调。我们用 BURST（5个粒子）+ STAGGER（180ms 间隔）来模拟"一串数据包"的感觉：

```javascript
const BURST = 5;      // 每次事件触发的粒子数
const STAGGER = 180;  // 粒子错开的启动时间（ms）
const T_INBOUND = 1200;  // 上行传输时间
const T_WAIT = 350;      // Proxy 处理延迟
const T_OUTBOUND = 1000; // 下行传输时间
```

上行粒子先出发，等待 T_WAIT 后下行粒子才开始。这模拟了真实的数据流时序：包到了 Proxy，处理一下，再转发出去。

```javascript
// 上行：sender → proxy port（蓝色 ingress 粒子，BURST 个）
for (let i = 0; i < cfg.BURST; i++) {
  setTimeout(() => {
    spawnFlowEdge({
      from: eps.sender,
      to: eps.target,
      color: cfg.FLOW_STYLES.ingress.color,
      width: 2.4,
      duration: cfg.T_INBOUND,
    });
  }, i * cfg.STAGGER);
}

// 下行：proxy → 同 room 其他玩家（绿色 forward 粒子）
// 等上行 + proxy 处理完成后再画下行
const startAfter = cfg.T_INBOUND + cfg.T_WAIT;
others.forEach((p, i) => {
  setTimeout(() => {
    // ... 绘制下行粒子
  }, startAfter + i * 100);
});
```

## 活动衰减：滑动窗口算法

端口和房间的活动计数需要"慢慢衰减"，而不是瞬间归零。我们用指数衰减实现：

```javascript
function decayActivity() {
  const decay = 0.96;
  for (const p of portActivity) {
    if (p) { p.rx *= decay; p.fwd *= decay; p.drop *= decay; }
  }
  for (const a of MM.store.roomActivity.values()) {
    a.rx *= decay; a.fwd *= decay;
  }
}
```

每 200ms 执行一次衰减。0.96 的衰减系数意味着：如果一个端口在某一秒有 100 次 ingress，10 秒后这个计数会降到 66（100 × 0.96^50）。

这个衰减直接驱动端口颜色变化——活动越多，端口颜色越亮。

## 后端：EventHub 手写 JSON 序列化

后端的 `EventHub` 负责把 FlowEvent 广播给所有 SSE 订阅者。为了压榨性能，我们手写了 JSON 序列化，避免 `json.Marshal` 的反射开销：

```go
func (ev FlowEvent) appendJSON(dst []byte) []byte {
  dst = append(dst, '{')
  first := true
  writeStr := func(key, val string) {
    if val == "" { return }
    if !first { dst = append(dst, ',') }
    first = false
    dst = append(dst, '"')
    dst = append(dst, key...)
    dst = append(dst, '"', ':', '"')
    // 转义特殊字符
    for i := 0; i < len(val); i++ {
      switch val[i] {
      case '"':  dst = append(dst, '\\', '"')
      case '\\': dst = append(dst, '\\', '\\')
      case '\n': dst = append(dst, '\\', 'n')
      default:   dst = append(dst, val[i])
      }
    }
    dst = append(dst, '"')
  }
  // ... 其他字段
  writeStr("type", ev.Type)
  writeStr("uid", ev.UID)
  // ...
  return dst
}
```

实测比 `json.Marshal` 快约 1 微秒，少 4 次内存分配。在高 PPS 场景下，这个差距会累积。

## 采样控制：高频事件降级

生产环境的 PPS 可能达到数万，全量推送到前端会卡死。EventHub 支持按环境变量配置采样率：

```go
func (h *EventHub) Publish(ev FlowEvent) {
  // 采样：高频事件（ingress/forward）按 sampleRate 跳过
  if h.sampleRate > 0 && (ev.Type == "ingress" || ev.Type == "forward") {
    count := h.sampleCounter.Add(1)
    if count%h.sampleRate != 0 {
      return
    }
  }
  // ...
}
```

通过 `MM_PROXY_EVENT_SAMPLE_RATE=10` 可以只发送十分之一的 ingress/forward 事件。drop、handshake、debug 这些低频事件始终 100% 发送。

## SSE 双通道：events 和 metrics

前端维护两个独立的 SSE 连接：

- `/proxy/events`：推送 FlowEvent（粒子动画的数据源）
- `/proxy/metrics`：推送 PPS、Rooms、Players、Drop Rate、Memory

两个通道独立重连，互不影响。`sse.js` 里实现了去抖逻辑——如果 3 秒内连续触发 error，只执行一次重连：

```javascript
eventSource.onerror = () => {
  const now = performance.now();
  // 去抖：距上次 error 不满 RECONNECT_MS 则丢弃
  if (now - sseLastErrorAt < cfg.RECONNECT_MS) return;
  sseLastErrorAt = now;
  scheduleReconnect("events");
};
```

## 踩过的坑

### X6 3.1.7 的 port 定位问题

X6 的 `position.absolute.args` 不会自动同步到 port 元素的 `cx/cy`。我们一度以为是自己的代码问题，debug 了一下午。

最终解法：直接用 `attrs.circle.transform = "translate(x, y)"` 定位 port 元素。

### connector:"smooth" 导致动画不启动

最初用 `edge.animate` 配合 `view.selector("line")` 拿 `<line>` 元素做动画。但 `connector: "smooth"` 在 X6 3.1.7 里渲染为 `<path>`，WAAPI 永远找不到目标元素。

解法：改成 `edge.animate` 走 X6 的 attrs 系统，让 X6 自己去找正确的元素。

### send 事件的复合绘制

`send` 事件是控制触发器，前端需要画出完整的 ingress → proxy → forward 链路。我们把它拆成两部分：先画上行粒子（sender → proxy），等 T_INBOUND + T_WAIT 后再画下行粒子（proxy → 其他玩家）。这样视觉上能清楚看到"消息从哪来、到哪去"。

## 文件结构

```
frontend/proxy-visualizer/
├── index.html          # 入口，加载 X6 CDN
├── styles.css          # 节点/动画样式
├── x6-app.js           # 应用入口
└── src/
    ├── config.js       # 时序常量、颜色配置
    ├── store.js        # 全局状态
    ├── sse.js          # SSE 双通道连接
    ├── flow-events.js  # 事件分发
    ├── x6-graph.js     # X6 Graph 初始化
    ├── x6-topology.js  # 拓扑布局、节点增删
    ├── x6-flow.js      # Flow 粒子动画（核心）
    ├── x6-rooms.js     # 房间状态同步
    └── x6-ui.js        # 侧边栏、指标卡
```

## 性能数据

在本地测试环境（10 个模拟玩家，3 个房间）：

| 指标 | 值 |
|------|-----|
| FPS | 稳定 60 |
| 同时活跃 flow 边数 | 最多 ~50 |
| SSE 消息吞吐 | ~200 条/秒 |
| 前端内存增量 | < 10MB |

每个 flow 边存活 1~1.5 秒后自动清理，所以同时存在的边数有上限。即使 PPS 很高，前端也不会被撑爆。

---

下一步打算做什么？也许是给 flow 粒子加上丢包重试的视觉反馈，或者接入 Prometheus 指标做更细粒度的监控。你觉得还有什么值得加？
