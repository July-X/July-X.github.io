---
title: "Boss 多阶段战斗：用两层状态机做 AI 节奏"
date: 2025-06-03 16:30:00
tags:
  - Godot
  - 游戏开发
  - 状态机
  - Boss 战
categories:
  - 游戏开发
top_img: false
---

<!-- more -->

`godot-game-one` 的 Boss 是一艘"裂隙母舰"，战斗从压制阶段打到核心过载，全程大约 42 秒。914 行的 `boss.gd` 里，核心问题只有一个：**怎么用最少的代码做出有节奏感的多阶段 AI？**

答案是两层状态机。

```mermaid
stateDiagram-v2
    [*] --> IDLE
    IDLE --> CIRCLE: 太远
    IDLE --> ATTACK: 50%权重
    IDLE --> RETREAT: 太近
    CIRCLE --> ATTACK: 计时器到期
    CIRCLE --> RETREAT: 太近
    ATTACK --> CIRCLE: 计时器到期
    ATTACK --> RETREAT: 太近
    RETREAT --> CIRCLE: 安全距离
    RETREAT --> ATTACK: 计时器到期
    IDLE --> ENRAGED: Phase3激活
    ENRAGED --> ATTACK: 复用逻辑
```

## 两层状态，各管各的

```gdscript
enum State { IDLE, CIRCLE, ATTACK, RETREAT, ENRAGED }       # 行为状态（微状态）
enum BossPhase { SUPPRESSION, RIFT, OVERLOAD }               # 战斗阶段（宏状态）
```

**State** 控制每帧的具体行为——移动、射击、撤退，生命周期 0.5~2 秒，通过 `_state_timer` 倒计时自动切换。

**BossPhase** 由血量比例驱动，是长周期的节奏控制器。它不直接控制移动或射击，而是修改全局参数：攻击频率、弹幕密度、移动速度、环绕半径。

一句话总结：**State 负责"怎么做"，BossPhase 负责"做什么"。**

## 行为状态机：match + 计时器

`_physics_process` 里每帧分发：

```gdscript
match _state:
    State.IDLE:    _tick_idle(delta)
    State.CIRCLE:  _tick_circle(delta)
    State.ATTACK:  _tick_attack(delta)
    State.RETREAT: _tick_retreat(delta)
    State.ENRAGED: _tick_attack(delta)   # 复用 ATTACK 逻辑
```

状态切换由 `_pick_state()` 决定，核心逻辑基于距离判断：

```gdscript
var dist := global_position.distance_to(_target.global_position)
if dist < BOSS_STANDOFF_MIN_DIST:
    _enter_state(State.RETREAT)        # 太近就撤
elif dist > BOSS_STANDOFF_MAX_DIST:
    _enter_state(State.CIRCLE)         # 太远就靠近
else:
    # 按权重随机选：环形移动 40%、攻击 50%、待机 10%
```

没有用面向对象的 State Pattern，没有状态转换表，就是 `match` + `if/else` + 随机权重。简单、直觉、好调试。

## 战斗阶段：血量驱动的参数切换

三个阶段各有明确的设计意图：

**Phase 1 — 压制校准（100%~60% HP）**：Boss 保持中距离环形移动，稳定射击。玩家学习规律、建立节奏。

**Phase 2 — 裂隙展开（60%~30% HP）**：Boss 进入狂暴状态，攻击频率翻倍，环绕半径缩小，裂隙弹幕展开。压力陡增。

**Phase 3 — 核心过载（30%~0% HP）**：Boss 释放补给碎片（给玩家回血机会），同时进入最后的高密度弹幕阶段。玩家在生死线附近完成击杀。

阶段切换的实现极其简洁：

```gdscript
func _update_phase() -> void:
    var hp_ratio = float(current_hp) / float(max_hp)
    if hp_ratio <= 0.3:
        _set_phase(BossPhase.OVERLOAD)
    elif hp_ratio <= 0.6:
        _set_phase(BossPhase.RIFT)
```

阶段切换的血量阈值流程：

```mermaid
graph TD
    A[Boss 受伤] --> B{计算 hp_ratio}
    B -->|> 60%| C[Phase 1: 压制校准]
    B -->|30%~60%| D[Phase 2: 裂隙展开]
    B -->|< 30%| E[Phase 3: 核心过载]
    C -->|稳定射击| F[玩家学习节奏]
    D -->|攻击频率翻倍| G[压力陡增]
    E -->|释放补给碎片| H[生死线击杀]
```

`_set_phase()` 不切换行为状态，只修改参数字典——射击间隔、移动速度、弹幕模式。行为状态机继续独立运转，但每个 tick 函数读取的参数已经变了。

## 这个设计的好处

1. **解耦**：移动逻辑和战斗节奏完全分离，改阶段参数不影响行为状态，改行为状态不影响阶段节奏。
2. **可读**：914 行的单一脚本，但因为分层清晰，定位问题很快——"Boss 攻击太慢"改阶段参数，"Boss 走位太死板"改行为状态权重。
3. **可扩展**：加新阶段只需要加一个 enum 值 + 一段 `_set_phase` 参数配置 + 一段血量阈值判断。精英怪（Elite）复用了同样的架构，但只有两个阶段（压制→裂隙），实现只需 267 行。

## 局限

所有状态都在一个脚本里，没有节点化。如果 Boss 行为复杂到 10 种以上状态，这个文件会膨胀到难以维护。当前 5 个行为状态 + 3 个战斗阶段的规模刚好在控制范围内。

> 状态机不是越多越好，也不是越抽象越好。匹配问题规模的最简方案，就是好方案。