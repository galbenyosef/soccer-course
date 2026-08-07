# PS1/PS2 Winning Eleven 核心技术：实现手册

> 提炼自 [we2000-implementation-research.md](./we2000-implementation-research.md) 及 PS2 时代 WE 设计文档，面向 Godot 4.4 `soccer-course` 项目的可落地实现指南。

## 核心理念

### Seabass 的足球模拟哲学

制作人高塚真吾（Seabass）的核心思想：

> **「足球游戏不是让玩家完成漂亮动作，而是让玩家像真正踢球一样思考。」**

他提出：足球不是一连串个人技巧，而是**空间、节奏、攻防选择的博弈**。足球游戏的攻防逻辑本质上类似格斗游戏——读对手动作、找破绽、反击。

所以 WE 的设计目标不是：

> 「我按一个键 → 球员完成动作」

而是：

> 「我提前判断 → 球员根据情况执行动作」

老 WE 高手的总结：「WE 不是操作游戏，是判断游戏。」

这直接推导出以下工程原则：

| 原则 | PS1/PS2 WE 的做法 | 本项目的应用 |
|------|-----------|------------|
| **确定性 > 随机性** | "nothing is left to chance"（官方手册）——球轨迹、球员行为全由解析公式 + 参数驱动 | `pass_to()` 的 `intensity = sqrt(2*d*f)` 已是确定性公式，继续沿这个方向 |
| **手感 > 真实** | 弱队传球速度被刻意降慢（Seabass 原文），不是因为物理引擎自然涌现 | 调参优先级：响应 → 平衡 → 风格，最后才是"像真的" |
| **规则 > 仿真** | 护球在 PES5 之前是布尔状态无敌（"no matter which player, you were invincible"） | 状态机是关键。球的状态（carried/freeform/shot）已经是对的骨架 |
| **行为模拟 > 数值模拟** | 球员不是"属性高 → 表现好"，而是"行为库 + 参数组合不同"（详见 §5 球员个体差异） | 球员差异 = 参数差异 × 动作选择规则差异 |
| **不确定性 = 游戏深度** | 100% 准确的传球反而是坏设计——误差创造比赛的故事感（详见 §6 误差系统） | 传球/射门的结果 = 公式确定性 × 玩家属性噪声 |

**一句话方法论**：不要模拟足球的物理过程，用确定性规则 + 可控噪声来创造可阅读、可掌控、有故事的博弈体验。

---

## 1. 球物理：解析式弹道

### 原则

球的位置从来不是 `v += a*dt; p += v*dt` 迭代出来的。传球/射门在「出手瞬间」就确定了整条轨迹——因为轨迹是一道**抛物线闭式公式**。

### 本项目已有的实现

```gdscript
# ball.gd pass_to() — 已经是解析式
func pass_to(destination: Vector2, lock_duration: int = DURATION_PASS_LOCK) -> void:
    var direction := position.direction_to(destination)
    var distance := position.distance_to(destination)
    var intensity := sqrt(2 * distance * friction_ground)  # ← 闭式解
    velocity = intensity * direction
    if distance > DISTANCE_HIGH_PASS:
        height_velocity = BallState.GRAVITY * distance / (1.85 * intensity)  # ← 也是解析式
```

### 待补充：落地预测

WE2000 的 AI 能预判球会落到哪，因为它知道确定的轨迹公式。本项目需要暴露一个预测方法：

```gdscript
# ball.gd — 新增
func predict_landing_position() -> Vector2:
    if velocity == Vector2.ZERO:
        return position
    # 水平匀速 → 用时 = distance / v
    # 竖直自由落体 → 落地时刻解 height + hvel*t - 0.5*g*t² = 0
    var t_land := _solve_landing_time()
    return position + velocity * t_land
```

> AI 调用 `ball.predict_landing_position()` 来站位，而不是每帧追赶球。

### 弹跳

本项目 `BallState.move_and_bounce()` 已经是 PS1 式的"反射 + 衰减"——不需要连续碰撞检测。

```gdscript
# 本项目的做法（正确）
var collision := ball.move_and_collide(ball.velocity * delta)
if collision != null:
    ball.velocity = ball.velocity.bounce(collision.get_normal()) * ball.BOUNCINESS
```

---

## 2. 动画驱动的判定

### 原则

不做"脚的碰撞体碰球的碰撞体"这种连续物理检测。而是：**动画到第 N 帧 → 检查球是否在脚附近 → 确定：释放/控制球**。

### 核心数据结构：触球帧数据

```gdscript
# 每个球员动作定义触球窗口
class_name ContactFrameData
extends Resource

@export var hit_frame : int = 0         # 判定帧（动画的第几帧）
@export var contact_radius : float = 20.0  # 触球距离阈值
@export var launch_velocity : float = 0.0  # 释放球的初速度倍率
@export var is_cancelable : bool = true   # 此帧前可被新输入打断
```

### 动画事件驱动释放球

不靠碰撞去判断"脚踢到球了"，而是靠**动画帧事件**主动通知：

```gdscript
# player_state_shooting.gd — 关键帧回调
func on_animation_hit_frame() -> void:
    # 判定球是否在触球范围内（便宜的距离检测，非物理碰撞）
    if ball.position.distance_to(player.position) < contact_data.contact_radius:
        var shoot_dir := player.heading
        var shoot_vel := shoot_dir * state_data.shot_power
        ball.shoot(shoot_vel)
        # 球脱离，走自己的状态
    else:
        # 没踢到——球不动，自己做空挥动画
        pass
```

> **实现重点**：用 Godot 的 `AnimationPlayer` 的 `call method` track 或 `animation_finished` 信号来触发 `on_animation_hit_frame()`。

### 球接触点 = 球员位置 + 方向偏移

不做碰撞检测，直接用方向算接触点：

```gdscript
# 便宜版"脚在哪"
func get_foot_position() -> Vector2:
    return player.position + player.heading * 8.0  # 脚在角色前方8像素
```

---

## 2.5 多因子动作选择：同一按键 ≠ 同一动作

PS2 WE 与同时代足球游戏最大的区别之一：**不是「按 X 键 → 播放 X 动画」，而是「按 X 键 → 评估当前状态 → 选择最合适的动作变体」。**

### 射门的多因子计算

按射门键后，WE 不是直接播一个「射门动画」。它先评估：

| 评估维度 | 影响的动作参数 |
|----------|-------------|
| 身体朝向 vs 球门方向 | 正脚背抽射 / 外脚背 / 勉强推射 |
| 支撑脚位置 | 触球力度、击球点高度 |
| 防守压力（附近对手距离） | 正常射门 / 匆忙射门（变形动作） |
| 球速（来球速度） | 直接射 / 调整后射 |
| 球员能力（技术、射门、力量） | 动作库匹配（明星球员有专属动画参数） |
| 平衡状态 | 稳定射门 / 失去平衡捅射 |

### Godot 实现：动作选择器

```gdscript
# 新建 scenes/characters/action_selector.gd
class_name ActionSelector
extends RefCounted

# 评估结果 → 动作参数
class ShotContext:
    var body_angle_to_goal : float      # 身体与球门夹角
    var defensive_pressure : float      # 最近防守者距离
    var incoming_ball_speed : float     # 来球速度
    var is_balanced : bool              # 是否在平衡状态
    var player_technique : float        # 球员技术值

func select_shot_variant(ctx: ShotContext) -> ShotVariant:
    # 身体朝向偏移大 → 外脚背或勉强射门
    if abs(ctx.body_angle_to_goal) > deg_to_rad(45):
        if ctx.is_balanced and ctx.defensive_pressure > 0.3:
            return ShotVariant.HASTY_TOE_POKE   # 匆忙捅射
        return ShotVariant.OUTSIDE_FOOT        # 外脚背
    
    # 防守紧逼 → 匆忙出脚
    if ctx.defensive_pressure > 0.7:
        return ShotVariant.HASTY_SHOT          # 变形射门
    
    # 正常情况：正脚背
    if ctx.player_technique > 0.7:
        return ShotVariant.FULL_LACE           # 正脚背抽射
    return ShotVariant.BASIC_SHOT              # 普通射门
```

`ShotVariant` 枚举映射到不同的 `ContactFrameData` 资源（`hit_frame`、`launch_velocity` 倍率、精度偏差因子），同一个「射门」状态根据评估结果选择不同的触球参数。

### 动画匹配：模板 × 参数变化

WE 不是给每个球员录一套专属动画（PS2 存不下），而是：

> **动作模板 + 参数偏移 = 球员差异**

```
模板：正脚背抽射（hit_frame=9, launch_vel=1.0）
  ├── 普通球员：参数 = 模板默认值
  ├── C罗：    hit_frame 提前1帧（更快触球）, launch_vel × 1.15（力量+15%）
  └── 贝克汉姆：后撤步距离 × 1.3（更长助跑 = 弧线加成）
```

Godot 实现思路：

```gdscript
# player_resource.gd — 每个球员的动作修正参数
@export var shot_hit_frame_offset : int = 0      # 触球帧偏移
@export var shot_power_mult : float = 1.0         # 射门力量倍率
@export var pass_curve_mult : float = 1.0         # 弧线加成
@export var dribble_step_mult : float = 1.0       # 带球步幅倍率

# 使用时：模板 ContactFrameData + 球员参数修正
func get_personalized_contact_data(variant: ShotVariant) -> ContactFrameData:
    var template := ShotVariantData[variant]
    var data := template.duplicate()
    data.hit_frame += player_data.shot_hit_frame_offset
    data.launch_velocity *= player_data.shot_power_mult
    return data
```

> **核心**：不是「给 C 罗录 50 个动画」，而是「模板 × 参数 = 行为差异」。参数差异让 C 罗「感觉不一样」，不是因为动画多，而是因为触球时机、力量、弧线的参数组合不同。

---

## 3. AI 架构：分级 + 权重

### 原则

PS1 无力为 22 人跑完整 AI。只给**离球最近的 1-3 人**跑完整决策，其他人做廉价归位。

### 本项目的已有骨架

```gdscript
# ActorsContainer.set_on_duty_weights() — 已经在做 AI LOD
for i in range(cpu_players.size()):
    cpu_players[i].weight_on_duty_steering = 1 - ease(float(i)/10.0, 0.1)
```

### 需要补充的三层 AI

```gdscript
# ai_behavior_field.gd — AI 更新分级

const UPDATE_FULL_EVERY := 3       # 近端每3帧决策一次
const UPDATE_POSITION_EVERY := 15  # 中端每15帧
const UPDATE_IDLE_EVERY := 60      # 远端每60帧

func _process(_delta: float) -> void:
    var frame := Engine.get_process_frames()
    var dist_to_ball := player.position.distance_to(ball.position)

    if player.weight_on_duty_steering > 0.7:
        if frame % UPDATE_FULL_EVERY == 0:
            _run_full_decision_tree()       # 持球人 + 最近防守者
    elif player.weight_on_duty_steering > 0.2:
        if frame % UPDATE_POSITION_EVERY == 0:
            _move_toward_home_position()     # 中距离协防/接应
    else:
        if frame % UPDATE_IDLE_EVERY == 0:
            _idle_hold_formation()           # 远端：懒更新
```

### 归位点系统

每个球员有自己的阵型位置（spawn position），球移动时整体平移：

```gdscript
# 在 ai_behavior.gd 或 ai_behavior_field.gd 中
func get_home_position() -> Vector2:
    var base := player.spawn_position
    # 根据球的位置整体压缩/拉伸阵型
    var offset := Vector2(
        ball.position.x / PITCH_WIDTH * FORMATION_COMPRESSION_X,
        ball.position.y / PITCH_HEIGHT * FORMATION_COMPRESSION_Y
    )
    return base + offset
```

AI 主体循环：
```
每 N 帧:
  离球最近的 1-3 人 → 完整决策树（抢球、传球、射门、带球、盯人）
  中距离球员     → 向归位点插值移动（15帧一次）
  远端球员       → 只在归位点附近晃动（60帧一次），模拟"在场上"的感觉
```

---

## 3.5 无球 AI 的深层设计

WE 真正领先同期 FIFA 的地方不是手感，而是无球 AI——玩家只控制 1 人，剩下 21 人全靠 AI。Seabass 认为 AI 才是足球游戏技术的核心。

### 跑位AI：队友「理解你的意图」

按直塞键的瞬间，前锋不是在原地等球，而是：

- **观察越位线**：压线后撤，避免越位
- **反跑**：先向前跑吸引防守，再急停反跑要球
- **拉边**：中路密集时自动拉开宽度
- **回撤**：接应中场，做支点

**Godot 实现**——不是让 AI 每帧重新计算，而是用有限状态：

```gdscript
# ai_behavior_field.gd — 无球跑位状态
enum RunType {HOLD_LINE, BREAK_OFFSIDE, REVERSE_RUN, PULL_WIDE, DROP_DEEP}

func select_off_ball_run(ball_carrier_is_teammate: bool, 
                          ball_position: Vector2,
                          offside_line: float) -> RunType:
    if not ball_carrier_is_teammate:
        return RunType.HOLD_LINE
    
    var my_y := player.position.y

    # 前锋在防线附近 → 反越位跑位
    if my_y < offside_line + 30.0 and my_y > offside_line - 10.0:
        if ball_position.distance_to(player.position) < 60.0:
            return RunType.BREAK_OFFSIDE
    
    # 中路拥堵 → 拉边
    if _teammates_in_center() > 3:
        return RunType.PULL_WIDE
    
    # 离持球人远 → 回撤接应
    if ball_position.distance_to(player.position) > 100.0:
        return RunType.DROP_DEEP
    
    return RunType.HOLD_LINE
```

### 防守AI：不追球，追路线

WE 金句：「高手防守不是一直按抢球，而是控制路线。」

防守球员的判断逻辑：
1. 球路预测：球大概率经过哪些区域
2. 空间评估：哪些区域最危险（与球门的连线）
3. 危险度排序：不是追最近的，而是防最危险的

```gdscript
func get_danger_priority() -> float:
    # 计算防守者与球门之间的夹角——这个球员防住了多大的射门角度
    var angle_to_goal := player.position.angle_to(target_goal.position)
    var angle_covered := abs(ball.position.angle_to(target_goal.position) - angle_to_goal)
    # 角度越大 → 越危险 → 优先级越高
    return angle_covered / PI
```

### 阵型AI：球队像整体

4-3-3 的边锋不会永远站边，根据比赛阶段自动调整：
- 进攻时内切（向中路靠）
- 防守时回防（向底线回退）
- 反击时拉开宽度

实现方式：在归位点（home position）上叠加**阵型阶段偏移**：

```gdscript
func get_formation_phase_offset(phase: MatchPhase) -> Vector2:
    match phase:
        MatchPhase.ATTACKING:
            return Vector2(player.role_offset_attack.x, player.role_offset_attack.y)
        MatchPhase.DEFENDING:
            return Vector2(player.role_offset_defense.x, player.role_offset_defense.y)
        MatchPhase.TRANSITION:
            return lerp(Vector2.ZERO, player.role_offset_attack, transition_progress)
    return Vector2.ZERO
```

---

## 4.1 碰撞与身体系统（规则式物理）

PS2 没有物理引擎，身体对抗和触球全部用**数值规则**模拟。

### 争顶模型

```gdscript
func resolve_header(player_a: Player, player_b: Player) -> Player:
    var score_a := player_a.jump_height() + player_a.position.y
    var score_b := player_b.jump_height() + player_b.position.y
    if score_a > score_b:
        return player_a
    return player_b
    
# jump_height() 内部:
func jump_height() -> float:
    return height * 0.3 + power * 0.4 + (_is_timed_perfectly() ? 1.0 : 0.0)
```

### 身体对抗模型

```gdscript
func resolve_physical_contest(attacker: Player, defender: Player) -> ContestResult:
    # 速度差 + 平衡 + 接触角度 → 是否被撞开
    var speed_diff := attacker.velocity.length() - defender.velocity.length()
    var balance_factor := attacker.balance - defender.balance
    var angle_penalty := cos(attacker.position.angle_to(defender.position))
    
    var contest_score := speed_diff * 0.3 + balance_factor * 0.5 + angle_penalty * 0.2
    
    if contest_score > CONTEST_WIN_THRESHOLD:
        return ContestResult.ATTACKER_WINS   # 护住球或突破
    elif contest_score < -CONTEST_WIN_THRESHOLD:
        return ContestResult.DEFENDER_WINS   # 被撞开/球权转换
    else:
        return ContestResult.STALEMATE       # 互相推搡，动作变形
```

**核心**：不是等碰撞事件触发然后看物理结果，而是**主动计算数值，决定结果**。伊布背身拿球为什么强？因为 `defender.balance` 高 + `velocity` 可控 → `contest_score` 在绝大多数情况下判他赢。

---

## 5. 球员个体差异系统（Player ID）

这是 PS2 WE 让不同球员「感觉不一样」的核心技术——不是靠多录动画，而是靠**隐藏参数层 + 动作模型差异**。

### 基础能力层

```gdscript
# 已有，player_resource.gd
@export var speed : float
@export var power : float
```

### 身体模型层（新增）

```gdscript
# 扩展到 player_resource.gd
@export var height : float = 1.75          # 身高（米）
@export var balance : float = 0.5          # 平衡（0-1）
@export var body_strength : float = 0.5    # 身体强度
@export var jump : float = 0.5             # 弹跳

# 经典案例：
# 阿德里亚诺: height=1.89, balance=0.95, body_strength=0.95 → 背身无敌
# 梅西:       height=1.70, balance=0.75, body_strength=0.55 → 小但灵巧
```

### 动作模型层（关键新增——这是「行为模拟」的核心）

```gdscript
# player_resource.gd — 动作风格参数
@export var shot_preferred_foot_bias : float = 1.0     # 擅用脚偏好
@export var dribble_step_frequency : float = 1.0       # 带球触球频率
@export var pass_trajectory_curve : float = 0.0        # 传球弧线倾向
@export var shot_power_mult : float = 1.0              # 射门力量倍率
@export var free_kick_approach : FreeKickStyle          # 任意球风格
@export var feint_style : FeintStyle                    # 假动作风格

# 经典案例：
# 贝克汉姆: pass_trajectory_curve=0.9, free_kick_approach=LONG_RUNUP
# 小罗:     feint_style=WRIST_FLICK, dribble_step_frequency=1.3
# C罗:      shot_power_mult=1.15, dribble_step_frequency=0.8（大步趟球）
```

**核心公式**：

```
球员行为 = 动作模板 × 能力参数 × 动作风格参数
```

不是「数值模拟球员」，而是「**行为模拟球员**」。同样的 4-3-3 阵型，不同球员组合出来的跑位、传球节奏、对抗结果完全不同——因为参数组合不同。

---

## 6. 误差系统

这是老 WE 比赛有「故事感」的核心机制。很多人不知道：WE **不追求 100% 准确**。

### 原则

真实足球不是数学公式。梅西的传球也偶尔会偏。这个「偏差」是刻意设计的游戏系统，不是 Bug。

### 精度 = 球员能力 + 防守压力 → 误差半径

```gdscript
# utils/pass_accuracy.gd — 传球精度计算
func calculate_pass_error(player: Player, 
                           target: Vector2, 
                           pressure: float) -> float:
    var base_error := PASS_ERROR_MAX  # 最大误差半径（像素）
    
    # 球员能力降低误差
    var technique_factor := clamp(player.technique / 100.0, 0.3, 1.0)
    base_error *= (1.0 - technique_factor)
    
    # 防守压力增大误差
    base_error *= (1.0 + pressure * 1.5)
    
    # 梅西(technique=95): 误差 ~5px，几乎不偏
    # 普通后卫(technique=50): 误差 ~25px，可能传偏
    return base_error
```

### 射门同理

```gdscript
func calculate_shot_deviation(player: Player, pressure: float) -> float:
    var deviation := SHOT_DEVIATION_BASE / (player.shooting / 100.0)
    deviation *= (1.0 + pressure * 1.2)
    # 结果：C罗射门偏差小，普通球员偏差大
    return deviation
```

射门速度 = `power × 技术 × 接触角 × (1 + 偏差噪声)`

### 实现要点

- 误差使用**种子随机数**而非纯随机——可复现，但不可预测
- 误差大小与球员能力成反比
- 防守压力是误差放大器
- 误差不仅影响射门/传球，也影响触球（第一次触球的距离）

**效果**：同样的直塞，同一个球员，在不同防守压力下结果不同——这就是「比赛的故事感」。

---

## 7. 节奏模拟：强队弱队用不同节奏踢球

这是 WE 区别于早期 FIFA（所有队一样快）的重要差异化系统。

### 弱队不能像巴萨一样连续短传

不是因为玩家操作差，而是**系统层面的参数限制**：

```gdscript
# 球队节奏参数（可从 squads.json 扩展）
class TeamRhythm:
    var pass_speed_mult : float      # 传球速度倍率
    var dribble_control_range : float # 带球控制范围
    var transition_speed : float     # 攻防转换速度
    var pressing_intensity : float   # 逼抢强度

# 西班牙/巴萨:  pass_speed_mult=1.0, dribble_control=0.9
# 弱队:         pass_speed_mult=0.75, dribble_control=1.2（球离脚更远）
```

### 实现：球性控制参数

```gdscript
# Player — 球性控制影响触球
var ball_control := 0.5  # 0-1

func receive_ball(incoming_vel: Vector2) -> void:
    # 球性好的球员第一次触球近，差的球员球弹开
    var control_distance := lerp(20.0, 3.0, ball_control)
    # 球停下来的位置 = 球员位置 ± 控制偏差
    ball.position = position + incoming_vel.normalized() * control_distance
```

**设计原则**：不是「物理上不可能的传球被系统拦截」，而是「物理上可能的传球，因为球员能力差而传不到位」。玩家感知到的是"我的队控不住球"，而不是"系统在搞我"。

---

## 8. 参数化动作与工程优化

PS2 内存极小，不可能给 22 个球员各存一套动画。WE 的做法：

### 动作模板 + 参数变化 = 球员差异

```
所有球员共享一套动作模板（正脚背、外脚背、脚尖、头球…）
              ↓
每场比赛加载每个球员的参数偏移表
              ↓
运行时：模板 animation + offset(hit_frame, speed, angle)
              ↓
玩家感知：C罗的射门「感觉不一样」
```

### Godot 实现

```gdscript
# 动作模板资源
@export var shot_templates : Dictionary[int, ShotTemplate] = {
    ShotVariant.FULL_LACE:      ShotTemplate.new(9, 1.0, 0.0),   # hit_frame, power, curve
    ShotVariant.OUTSIDE_FOOT:   ShotTemplate.new(11, 0.85, 0.3),
    ShotVariant.HASTY_SHOT:     ShotTemplate.new(5, 0.6, 0.1),
    ShotVariant.TOE_POKE:       ShotTemplate.new(4, 0.5, 0.0),
}

# 运行时叠加球员修正
func get_effective_shot_params(variant: ShotVariant, player: Player) -> ShotTemplate:
    var tmpl := shot_templates[variant].duplicate()
    # 球员专属偏移
    tmpl.hit_frame += player.shot_hit_frame_offset
    tmpl.power *= player.shot_power_mult
    return tmpl
```

### SE 的设计优先级（PS2 时代）

Seabass 在 PS2 性能极限下的取舍：

| 优先级 | 系统 | 为什么 |
|--------|------|--------|
| 1 | AI（22 人决策） | 足球游戏的核心不是画面是决策 |
| 2 | 球物理（轨迹） | 球的飞行和弹跳是玩法基础 |
| 3 | 身体对抗 | 决定球权归属的核心判定 |
| 4 | 动画 | 关键动作流畅即可 |
| 5（牺牲） | 草皮细节 | 不看 |
| 6（牺牲） | 观众 | 用 billboard |
| 7（牺牲） | 球场边缘细节 | 不看 |

**对本项目的启示**：你的优先级也应该是 AI > 球物理 > 判定规则 > 手感系统 > 动画 > 细节。永远不要为了「更好看」牺牲「更好判定」。

---

## 9. 手感系统

这是 WE 战胜同期物理引擎游戏的核心秘密。

### 9.1 输入缓冲

**问题**：玩家在动画中按传球，动画结束后"没反应"。

**解法**：按键不直接触发动作，而是先写入缓冲队列。

```gdscript
# 新建 utils/input_buffer.gd
class_name InputBuffer
extends RefCounted

const BUFFER_WINDOW_MS := 200  # 缓冲窗口
var buffer : Dictionary = {}     # action_name → timestamp_ms

func press(action: String) -> void:
    buffer[action] = Time.get_ticks_msec()

func consume(action: String) -> bool:
    if buffer.has(action):
        if Time.get_ticks_msec() - buffer[action] < BUFFER_WINDOW_MS:
            buffer.erase(action)
            return true
        buffer.erase(action)
    return false
```

```gdscript
# player_state_*.gd — 在可接受输入的帧检查缓冲
func _process(_delta: float) -> void:
    if _is_in_accept_window():
        if InputBuffer.consume("shoot"):
            transition_state(Player.State.PREPPING_SHOT)
            return
        if InputBuffer.consume("pass"):
            transition_state(Player.State.PASSING)
            return

# player.gd — 输入写入缓冲，不直接触发
func _input(event: InputEvent) -> void:
    if Input.is_action_just_pressed("p1_shoot"):
        InputBuffer.press("shoot")
    elif Input.is_action_just_pressed("p1_pass"):
        InputBuffer.press("pass")
```

### 9.2 可取消窗口

**问题**：按了射门就不能改了，动画播完前玩家被锁定。

**解法**：动作的起手帧（startup frames）内可被新指令覆盖。

```gdscript
# player_state_shooting.gd
const CANCEL_WINDOW_FRAMES := 6  # 前6帧内可取消

var elapsed_frames := 0

func _process(_delta: float) -> void:
    elapsed_frames += 1
    # 仅在起手窗口内检查取消输入
    if elapsed_frames <= CANCEL_WINDOW_FRAMES:
        if InputBuffer.consume("pass"):
            transition_state(Player.State.PASSING)  # 取消射门，改传球
            return
```

### 9.3 辅助瞄准（磁性传球）

**问题**：传球方向差一度、距离差一点就丢了。

**解法**：不是传向方向，而是传向**队友位置**，自动做吸附。

```gdscript
# ball.gd pass_to 的吸附逻辑
func pass_to_nearest_teammate(direction: Vector2, country: String) -> void:
    var best_teammate: Player = null
    var best_angle := INF
    for teammate in _get_teammates(country):
        var angle_to_teammate := abs(direction.angle_to(
            position.direction_to(teammate.position)
        ))
        # 只吸附方向40度以内、距离合理的队友
        if angle_to_teammate < deg_to_rad(40.0) and angle_to_teammate < best_angle:
            best_angle = angle_to_teammate
            best_teammate = teammate
    if best_teammate:
        pass_to(best_teammate.position)  # 传向吸附目标
    else:
        pass_to(position + direction * MAX_PASS_DISTANCE)  # 自由传球
```

### 9.4 冲击反馈（Hitstop — 已有）

```gdscript
# GameManager — 已有，无需修改
if is_high_impact:
    time_since_paused = Time.get_ticks_msec()
    get_tree().paused = true  # 全屏暂停100ms → 打击感
```

可扩展的点：为不同力度分档 hitstop 时长（轻碰撞 50ms、铲球 100ms、射门 150ms）。

---

## 10. 状态机的格斗游戏化

### Seabass 的直接类比

Seabass 本人在与任天堂的开发者对谈（Iwata Asks, 3DS/PES 2011）中明确说：

> **「足球游戏的攻防类似格斗游戏。」**

区别只是：
- 格斗游戏：读对手动作 → 找破绽 → 反击
- 足球游戏：读防线 → 创造空间 → 利用瞬间优势

所以 WE 的设计目标是：「让玩家像真正踢球一样思考」——不是按一个键就完成动作，而是提前判断，球员根据情况执行。

这直接推导出下面的帧数据架构：

### 原则

PS1 没有物理引擎，所以每个动作就是一个**不可被打断的状态段**。这其实是格斗游戏的帧数据思维：

```
MOVING:
  ↓ 按射门（起手6帧可取消，进入 PREPPING_SHOT）

PREPPING_SHOT:
  startup: 8帧（起手，可取消）
  active:   1帧（触球判定帧 → 释放球）
  recovery: 10帧（收招，不可取消）
  → 自动回到 MOVING

TACKLING:
  startup: 4帧
  active:   4帧（铲球判定窗口）
  recovery: 12帧（如果没铲到，长硬直）
```

关键设计决策：

| 决策 | 理由 |
|------|------|
| 起手可取消 | 玩家应该能基于新信息改变决定（读防线 → 找到更好的选择） |
| 判定帧为单帧 | 触球不是"碰撞区存在多帧"，而是"在正确的一帧到达正确的位置"——技巧来自时机 |
| 收招不可取消但可输入缓冲 | 玩家可以提前排队下一个动作（输入缓冲 §9.1），但当前动作必须自然结束 |
| 高风险动作长硬直 | 铲球如果空了应该有惩罚——这是足球博弈的一部分 |

### 状态设计检查清单

本项目已有 15 个球员状态，对照 PS1/PS2 WE 设计原则逐项检查：

| 状态 | 有触球判定帧? | 有取消窗口? | 有硬直帧? | 有误差计算? |
|------|------------|----------|---------|----------|
| MOVING | - | - | - | - |
| PASSING | 应有 `hit_frame` 主动释放球 | 起手可取消 | 传球后收招 | 应叠加球员技术误差 |
| SHOOTING | 应有 `hit_frame` 主动释放球 | 起手可取消 | 射门收招 | 应叠加防守压力误差 |
| TACKLING | 应有 `active_frames` 伤害窗口 | 不应可取消（高风险） | 铲空长硬直 | - |

> 每个状态需要明确定义它的**帧数据**：startup / active / recovery 各占多少帧（或秒），以及哪些窗口内可被新输入打断。

---

## 11. 实现优先级路线图

按手感收益 / 实现成本的性价比排序：

| 优先级 | 任务 | 预计改动量 | 手感提升 | 来源 |
|--------|------|----------|---------|------|
| **P0** | **输入缓冲** | 新建 `utils/input_buffer.gd` + `player.gd` 的 `_input` 改写 | ⭐⭐⭐⭐⭐ | §9.1 |
| **P0** | **辅助磁性传球** | `ball.gd pass_to` 加吸附逻辑 | ⭐⭐⭐⭐ | §9.3 |
| **P1** | **触球帧数据** | 定义 `ContactFrameData` 资源类 + 动画事件回调 | ⭐⭐⭐⭐ | §2.5 |
| **P1** | **可取消窗口** | `player_state_shooting/passing/tackling.gd` 加起手取消逻辑 | ⭐⭐⭐ | §9.2 |
| **P1** | **多因子动作选择** | `ActionSelector` + 射门/传球变体评估 | ⭐⭐⭐⭐ | §2.5 |
| **P2** | **球落地预测** | `ball.gd` 新增 `predict_landing_position()` | ⭐⭐⭐ | §1 |
| **P2** | **AI 归位点 + 阵型阶段** | `ai_behavior_field.gd` 新建阵型归位 + 攻防阶段偏移 | ⭐⭐⭐ | §3, §3.5 |
| **P2** | **AI 决策节流（三级）** | `ai_behavior.gd` 按距离降频 + 无球跑位状态机 | ⭐⭐⭐ | §3 |
| **P2** | **球员身体模型参数** | `player_resource.gd` 扩展 height/balance/body_strength/jump | ⭐⭐⭐ | §5 |
| **P3** | **误差系统** | 新建 `utils/pass_accuracy.gd` + 种子随机数 | ⭐⭐⭐ | §6 |
| **P3** | **球员动作模型参数** | `player_resource.gd` 扩展 shot_power_mult/pass_curve/dribble_freq | ⭐⭐⭐ | §5 |
| **P3** | **身体对抗判定** | `resolve_physical_contest()` 数值规则 | ⭐⭐⭐ | §4.1 |
| **P3** | **hitstop 分档** | `GameManager` 支持多档暂停时长 | ⭐⭐ | §9.4 |
| **P3** | **帧数据规范** | 所有 `player_state_*.gd` 标注 startup/active/recovery | ⭐⭐ | §10 |
| **P3** | **节奏参数系统** | `TeamRhythm` 类 + 球队级参数 | ⭐⭐ | §7 |

> **P0 是纯手感，P1-P2 改变玩法深度，P3 是锦上添花。** 建议按顺序推进。

---

## 12. 反模式（不要做的事）

| 不要做的 | 为什么 |
|----------|--------|
| 给球加 `RigidBody2D` | 你就失去了"给定初速度 → 确定落点"的确定性。WE 的 AI 强就强在能预判球路。 |
| 用物理碰撞做触球判定 | "脚碰到了球吗"应该由动画帧决定，不是 `_on_body_entered`。物理碰撞的结果不可预测也不可调试。 |
| 让 AI 每帧都为所有球员决策 | 你的 CPU 可能撑得住，但这不是好设计——远端球员不需要"思考"，只需要"站在那"。 |
| 追求物理真实参数 | 如果"真实值"导致手感差，直接 overrule 真实值。Seabass 就是这么做的。 |
| 让动画时长定义硬直 | 应该反过来——硬直时长定义动画时长，不要让美术决定的动画长度绑架了手感。 |
| 追求 100% 传球准确率 | 完美的数学传球反而没有足球味——误差创造比赛的故事感和紧张感（§6）。 |
| 给所有球员同一套动作参数 | 同质化是足球游戏的大敌——C罗和阿德里亚诺应该行为不同，不是数值不同（§5）。 |
| 为「更好看」牺牲「更好判定」 | Seabass 的优先级清单说得很清楚：AI > 球物理 > 判定规则 > 动画 > 细节（§8）。 |

---

> **核心方法论（最终版）**：
>
> 不要模拟足球的物理过程。用确定性规则 + 可控噪声来创造可阅读、可学习、可掌控、有故事的博弈体验。
>
> WE 模拟的不是「踢球动作」，而是「足球决策」——
> 以空间判断为核心，以无球 AI 为骨架，以球员个性参数为血肉，以不确定性为灵魂。
>
> "手感"的判断标准永远只有一条：**玩家觉得对不对**，不是算得对不对。
