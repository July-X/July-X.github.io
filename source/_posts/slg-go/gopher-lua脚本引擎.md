---
title: "gopher-lua：脚本引擎的集成、并发和热更新"
date: 2025-06-04 14:30:00
tags:
  - 游戏后端
  - Go
  - Lua
  - 热更新
categories:
  - 开发日志
top_img: false
---

<!-- more -->

`slg-go` 用 `gopher-lua` 做了两件事：战力计算和等级计算。目前还没有实现运行时热更新——改脚本还是得重启服务。这篇文章记录我们怎么集成 gopher-lua、踩了哪些坑、以及热更新要做的话该怎么设计。

## 为什么选 Lua？

| 语言 | 嵌入难度 | 性能 | 热更新潜力 | 游戏行业生态 |
|------|----------|------|------------|-------------|
| Lua | 低（gopher-lua） | 中 | 原生支持 | 标准 |
| JavaScript | 中（goja） | 中高 | 需额外机制 | 非主流 |
| Python | 高（嵌入 CPython） | 低 | 受 GIL 限制 | 非主流 |

Lua 天生为嵌入设计，`gopher-lua` 是纯 Go 实现，不依赖 CGO，交叉编译无障碍。`go.mod` 里直接 `go get github.com/yuin/gopher-lua` 就行。

## 实际的集成方式

项目的 Lua 集成集中在两个计算器：`player_power_lua.go` 和 `player_level_lua.go`。结构一样，以战力计算器为例：

```go
// internal/logic/app/player_power_lua.go
type LuaPlayerPowerCalculator struct {
    scriptFile string
    loadOnce   sync.Once      // 脚本只加载一次
    loadErr    error
    script     string         // 脚本内容字符串
    statePool  sync.Pool      // 复用 LState 实例
}
```

初始化时用 `sync.Once` 加载脚本内容到内存，后续不再读文件：

```go
func NewLuaPlayerPowerCalculator(scriptDir string) *LuaPlayerPowerCalculator {
    calc := &LuaPlayerPowerCalculator{
        scriptFile: filepath.Join(scriptDir, "power.lua"),
    }
    calc.statePool = sync.Pool{
        New: func() interface{} {
            return lua.NewState()
        },
    }
    return calc
}

func (c *LuaPlayerPowerCalculator) ensureLoaded() error {
    c.loadOnce.Do(func() {
        data, err := os.ReadFile(c.scriptFile)
        if err != nil {
            c.loadErr = err
            return
        }
        c.script = string(data)
    })
    return c.loadErr
}
```

## LState 不是线程安全的——用 sync.Pool 解决

这是踩的第一个坑。`gopher-lua` 的 `LState` **不是并发安全的**，一个 `LState` 同一时刻只能被一个 goroutine 使用。

最开始想过加 `sync.RWMutex`，`RLock` 让多个请求同时读。但 `LState` 不支持并发读——10 个 goroutine 同时 `RLock` 成功，然后同时操作同一个 `LState`，直接 panic。

最终方案是 `sync.Pool`：

```go
func (c *LuaPlayerPowerCalculator) acquireState() *lua.LState {
    return c.statePool.Get().(*lua.LState)
}

func (c *LuaPlayerPowerCalculator) releaseState(L *lua.LState) {
    c.statePool.Put(L)
}
```

每次调用从池里取一个 LState，用完放回去。多个请求并发时各自用各自的 LState，互不干扰。Pool 为空时 `New` 函数自动创建新实例。

## 计算流程

```go
func (c *LuaPlayerPowerCalculator) CalcPlayerPower(...) (int64, error) {
    if err := c.ensureLoaded(); err != nil {
        return 0, err
    }

    L := c.acquireState()
    defer c.releaseState(L)

    // 用 DoString 而不是 DoFile——脚本已经在内存里了
    if err := L.DoString(c.script); err != nil {
        return 0, err
    }

    fn := L.GetGlobal("calculate_power")
    // ... 推参数、调用、取返回值
}
```

注意是 `DoString` 不是 `DoFile`。脚本内容在 `sync.Once` 时已经读到内存，每次调用直接执行字符串，省掉了重复的文件 IO。

## 优先用 Rust，Lua 做兜底

项目的战力计算有个有趣的架构——优先用 Rust（通过 CGO 调用），Rust 不可用时 fallback 到 Lua：

```go
// internal/logic/app/game_app.go
if a.rustPowerCalc != nil {
    calculated, err = a.rustPowerCalc.CalcPlayerPower(...)
} else if a.powerCalc != nil {
    calculated, err = a.powerCalc.CalcPlayerPower(...)
    logger.Warn("calc player power by lua failed:", ...)
}
```

Rust 版本性能更好（gopher-lua 约是 C Lua 的 1/10~1/5），Lua 版本是兜底方案。这种双引擎设计和 game-bass 的 ELO 计算类似——Go 内置兜底，脚本引擎做灵活扩展。

## 热更新：还没做，但可以聊聊怎么设计

目前改脚本需要重启服务。对于战力/等级计算这种低频场景，重启是可以接受的。但如果未来要把更多逻辑放到 Lua（比如战斗公式、建筑规则），就需要热更新了。

### 热更新面临的问题

**问题 1：LState 的并发安全**

热更新的瞬间可能有请求正在用旧 LState。直接 Close 会崩。需要一个机制让旧实例"优雅退役"——等正在执行的请求跑完再销毁。

设计思路：VM 池 + 引用计数。

```go
type LuaPool struct {
    vms    []*lua.LState
    refCnt atomic.Int64
    closed atomic.Bool
}
```

每次 acquire 时 refCnt+1，release 时 refCnt-1。Reload 时 atomic.Swap 切换到新池，旧池标记 closed，等 refCnt 归零后 Close 所有 VM。

**问题 2：脚本里的全局状态**

Lua 脚本里的模块级变量（`local counter = 0`）热更新后会重置。如果某个逻辑依赖这个计数器，更新后会出 Bug。

解决方案：Lua 只放无状态的计算逻辑，有状态的数据留在 Go 端管理。

```
Lua 负责的（无状态，安全热更新）：
  - calculate_power(stats) → 数字
  - building_upgrade_time(level, type) → 数字

Go 负责的（有状态，不热更新）：
  - 玩家数据、成就计数
  - 战斗中的临时状态
```

**问题 3：脚本语法错误**

热更新时新脚本如果有语法错误，直接加载会炸。需要先在临时 VM 里验证语法，通过后再替换。

```go
func validateScript(script string) error {
    testVM := lua.NewState()
    defer testVM.Close()
    return testVM.DoString(script)
}
```

### 如果要做，设计大概是这样

```
1. /api/admin/reload 接口接收新脚本
2. 临时 VM 验证语法
3. 创建新 VM 池，加载新脚本
4. atomic.Swap 切换到新池
5. 旧池标记退役，等引用归零后销毁
6. 新请求用新公式，正在跑的请求用旧公式跑完
```

目前还没做，因为战力/等级计算的改动频率不高，重启成本可接受。但如果将来把战斗公式也放 Lua，这就是必须解决的问题。

## 性能考量

`gopher-lua` 是纯 Go 实现的 Lua 5.1 解释器，性能约为 C Lua 的 1/10~1/5。对战力/等级计算来说足够——每秒最多几十次调用，单次计算几十行代码，瓶颈在数据库 IO 不在脚本计算。

如果未来性能不够：
1. 热点函数用 Go/Rust 实现，Lua 只做参数调整
2. 缓存计算结果
3. 升级到 LuaJIT（需要 CGO，和交叉编译冲突）

> 目前 Lua 在项目里是"锦上添花"的角色——Rust 做主力，Lua 做兜底。这个定位决定了热更新暂时不是刚需。但随着脚本化的逻辑越来越多，热更新迟早要做。