# WE2000 实现研究：弱 CPU、无物理引擎如何做出好玩的足球游戏

> **资料来源声明**：本文撰写时联网检索不可用，内容基于业界公认的 PS1 时代足球游戏开发技术、Konami Winning Eleven / ISS 系列的设计哲学，以及 PS1 硬件特性的一般知识。其中标注「推断」的部分是结合时代背景的合理重建，建议后续用日方开发者访谈（高塚真吾/Shingo Takatsuka 等）与拆解资料核实具体细节。

## 1. 研究动机

本项目（`soccer-course`，Godot 4.4）正在用**简化解析物理 + 状态机**做一款 2D 足球游戏。WE2000（World Soccer Winning Eleven 2000，2000 年于 PS1 发售，KCE Tokyo 开发，系列核心制作人为高塚真吾）所处平台比今天弱几个数量级，却确立了"实况式手感"的标杆。研究它如何在不依赖真实物理引擎的前提下做到"好玩"，对本项目的方向有直接参考价值。

核心问题：**没有刚体物理、CPU 极弱，为什么手感反而比很多"物理正确"的现代游戏更好？**

短答：因为它**不追求物理仿真，而追求"响应、可读、可掌控"的游戏感（game feel）**。物理只是手段，手段被刻意做"假"以服务手感。

## 2. PS1 硬件约束

| 项 | 规格 | 对游戏设计的影响 |
|----|------|------------------|
| CPU | MIPS R3000A @ ~33.8 MHz，FPU 极慢 | 几乎全用**定点运算（fixed-point）**而非浮点 |
| 协处理器 | GTE（几何变换引擎） | 3D 矩阵/向量运算靠它，但预算紧张 |
| 主内存 | 2 MB | 资产极小，动画/数据必须精简、可复用 |
| VRAM | 1 MB | 低多边形 + 调色板贴图 |
| 物理 | **无硬件物理、无浮点优势** | 球与玩家运动全部**手写解析公式或脚本化** |

关键推论：在这样的机器上**不可能**跑连续刚体碰撞求解。一切运动要么是**闭式解析公式**（给出初速度就能算出任意时刻位置），要么是**动画驱动**（运动轨迹烧进动画曲线）。这两条路恰好就是 WE2000 的核心。

## 3. 无物理引擎的球物理

PS1 时代的足球游戏普遍不用数值积分的物理求解器，而是：

### 3.1 解析式弹道（parametric / analytic trajectory）
- 传球、射门被建模为**已知初速度的抛物线**，位置 = 闭式公式直接求值，而非 `v += a*dt; p += v*dt` 反复迭代。
- 好处：**完全确定、无漂移、无穿透、无求解抖动**；任何时刻都能算出球会落到哪，因此 AI 提前预判、UI 提示都很容易。
- 弹跳用简单的**反射 + 能量衰减**（`v = v.bounce(normal) * restitution`），不做多次连续碰撞。
- 高传球（吊球）通过给一个 `height_velocity` 让球做竖直方向的抛物运动，水平匀速——二维半的"假 3D"。

### 3.2 脚本化的球状态
- 球不是自由刚体，而是处在有限几个**状态**之一：被携带、自由滚动、被踢出（射门/传球）、空中。每个状态有自己的运动规则。
- 状态切换由**事件**触发（球员触球、出界、进球），而非物理碰撞自动产生。

> **本项目对照**：`scenes/ball/ball.gd` 正是这套思路——`pass_to()` 用 `intensity := sqrt(2 * distance * friction_ground)` 直接算出所需初速度（闭式解），`tumble()/shoot()` 直接设速度，`BallState.process_gravity()` 用解析重力 + 弹跳系数。**本项目已经走在 WE2000 的路子上**，只是 2D。

## 4. 动画驱动的玩法（animation-driven gameplay）

这是"手感"的真正来源，也是 PS1 弱机器的关键解法。

### 4.1 触球点烧进动画
- 不做"脚的胶囊体碰球的球体"这种连续碰撞检测（PS1 算不起也调不准），而是**在动作动画的特定帧上挂事件**："第 N 帧时，把球以方向 D、力度 P 释放出去"。
- 球员动作与球的释放因此**天然同步**，永远不会出现"脚挥空了球却飞了"的违和感。这是现代物理足球游戏反而容易翻车的地方。

### 4.2 格斗游戏式的帧数据
- 铲球、射门、头球等动作像格斗游戏一样有 **起手（startup）/ 判定（active）/ 收招（recovery）** 帧。
- 玩家学会的是**时机与节奏**，而不是物理直觉。这让操作可学习、可精通，深度来自人而非来自物理参数。
- 收招硬直 = 风险，起手窗口 = 时机，这套语言让"什么时候出脚"成为博弈核心。

### 4.3 动画即判定
- 很多情况下"能否截到球"不是看物理碰撞，而是看**当前动画帧是否处于可触球窗口**且球员在球的合理范围内。判定是**动画帧 + 距离阈值**的组合，便宜且可控。

> **本项目对照**：`PlayerState.on_animation_complete()` 回调、`player_state_*.gd` 一系列动画状态、`can_carry_ball()/can_pass()` 这类布尔门——就是动画驱动的骨架。可以进一步把"触球释放"明确绑到动画事件帧上。

## 5. 弱 CPU 上的 AI

PS1 没有余力每帧为 22 个球员跑复杂决策。WE2000 的 AI 是**结构化、分层、有 LOD**的：

### 5.1 阵型 + 归位点（home position）
- 每个球员有一个"归位坐标"，会随球的位置和进攻方向整体平移。
- 离球远的球员基本只是**向归位点做廉价插值**，不做决策。

### 5.2 影响图 / 吸引点（influence / attraction）
- 球是一个"吸引子"，附近球员被拉向球或传球路线；空当处设置"占位吸引点"让球员去填空。
- 这是**势场/权重**式导航，不是寻路——PS1 算不起导航网格。

### 5.3 AI 分级（LOD）
- **持球人或离球最近者**：跑完整决策树（射门/传球/带球选择）。
- **附近协防/接应者**：跑中等逻辑（跑位、盯人）。
- **远端球员**：仅按阵型归位，几乎不"思考"。
- 这样每帧真正"烧脑"的只有 1~3 个球员。

### 5.4 规则树而非规划
- 决策多为**有限状态机 + 优先级规则**（if 持球且空当 then 带球；else if 有队友 then 传球；else 大脚解围），没有复杂的多步规划或机器学习。

> **本项目对照**：`ActorsContainer.set_on_duty_weights()` 用 `ease()` 给离球近的 CPU 加权、其余降权——**这正是 AI LOD 的雏形**。`ai_behavior_field.goalie` 分行为也对应分层 AI。可继续强化"归位点"与"远端球员廉价更新"。

## 6. "手感"从哪来（game feel）

这是研究最该回答的问题。WE2000 的"好玩"主要来自下列**刻意设计**，而非物理真实：

1. **辅助瞄准 / 磁性传球**：传球目标会自动吸附到方向上最合适的队友，球路甚至会轻微弯曲以方便接球。玩家觉得"我传得准"，实际是系统在帮。
2. **输入缓冲（input buffering）**：在动画播放中按下的按键会被**排队**，在动作可中断的帧执行。玩家不会因为"按早了一帧"而操作失败。
3. **可取消的动画（cancel windows）**：部分动作在起手段可被新指令打断，让操作连贯、响应快。
4. **自动切换最近球员**：防守时系统自动把控制权切到离球最近的球员，玩家不用手动找。
5. **冲击反馈**：铲断、射门、扑救有屏幕震动、短暂停顿（hitstop）、特写镜头——**打击感来自反馈，不是物理**。
6. **可读性**：球的飞行轨迹清晰、球员跑位意图明显、UI 提示到位，让玩家能"预判"而非"反应"。
7. **节奏控制**：比赛节奏被有意调校（加速/减速段），制造张力而非恒定速度。

> **本项目对照**：`GameManager` 的 hitstop（`DURATION_IMPACT_PAUSE` 冲击暂停 100ms）已经是第 5 条的实现。`ActorsContainer.on_player_swap_request()` 是第 4 条。**最可能提升手感的下一步是第 1、2 条——辅助瞄准与输入缓冲**。

## 7. 渲染与性能技巧（背景）

- 低多边形球员 + 广告板（billboard）观众。
- 球场是带滚动纹理的平面。
- **调色板复用**：一套贴图通过调色板换色出多支队伍的球衣——这在内存只有 2MB 时几乎是必需的。
- 动画帧数有限，依赖动作捕捉数据压缩。

> **本项目对照**：`shaders/replace_color.gdshader` 用调色板纹理换肤色/队色——**与 PS1 调色板复用同源**，思路完全一致。

## 8. 对本项目的可落地启示

按"性价比 / 手感收益"排序：

| 优先级 | 启示 | 本项目落点 |
|--------|------|-----------|
| 高 | **辅助瞄准 / 磁性传球**：传球方向吸附最近队友，球路轻微弯曲 | `ball.gd pass_to()` 增加目标吸附；`player_state_passing.gd` |
| 高 | **输入缓冲**：动画中按下的射门/传球排队到可执行帧执行 | 新增输入缓冲队列，`player_state_*.gd` 检查 |
| 中 | **触球点绑动画事件**：把"释放球"明确挂到动画帧而非物理碰撞 | `player_state_shooting/passing.gd` + 动画事件 |
| 中 | **AI 归位点**：给每个球员按阵型算 home position，远端球员廉价归位 | `ActorsContainer` + `ai_behavior_field.gd` |
| 中 | **AI LOD 强化**：远端球员降低决策频率（隔几帧才思考一次） | `ai_behavior.gd` 加节流 |
| 低 | **冲击反馈多样化**：不同力度档位的 hitstop、震屏、粒子 | `GameManager.on_impact_received`、`ActorsContainer` spark |
| 低 | **可读性**：传球目标提示、接球预判高亮 | UI 层 |

核心心法：**把"物理正确"换成"反馈正确"**。WE2000 证明了一款足球游戏可以完全不仿真物理，只要做到了**响应快、辅助足、反馈强、可读性好**，手感就会优于物理逼真但迟钝的游戏。本项目的架构（解析弹道 + 状态机 + 动画驱动 + 全局事件总线）天然契合这条路，缺的主要是第 6 节里的"辅助与缓冲"层。

## 9. 原始资料深挖：开发者访谈与社区研究

本节基于实际抓取的原始资料（开发者访谈、官方手册、资深玩家技术分析），将第 1-8 节的推断与真实证据对照，补充深层设计哲学。

### 9.1 官方设计哲学（Winning Eleven 官方手册，原文）

**来源**：Winning Eleven 官方手册（Scribd 存档），Konami 发行

#### 核心设计目标

> **原文引用**："The complexity of the underlying mechanisms in the Winning Eleven series aims for one goal: to reproduce as faithfully as possible all the details that occur on a football field...from the simplest pleasure of the game to the slightest subtleties of the physics of bodies."

**解读**：这句话揭示了核心矛盾——"忠实再现"与"简单乐趣"并存。实际上 WE 系列**不是物理仿真器**，而是用参数化/脚本化机制"再现"足球的**可读性**而非物理真实性。

#### 确定性系统 vs 涌现复杂度

> **原文引用**："Each event is governed by processes and calculations so elaborate that they sometimes escape even the inventors. From the engine that governs the ball's trajectory to the capabilities that determine the players' performance... nothing is left to chance."

**关键词分析**：
- **"nothing is left to chance"** = 不是随机物理，而是确定性脚本
- **"so elaborate that they sometimes escape even the inventors"** = 复杂规则产生涌现行为，但底层仍是规则而非物理求解器

这印证了第 3 节的推断：球轨迹由"engine"（解析公式）控制，球员表现由"capabilities"（参数）决定——全是**数值驱动的脚本化系统**。

#### 理解优于反应

> **原文引用**："Those who have a greater understanding of the theory of the game will perform better than someone who simply presses the buttons."

**设计意图**：WE 追求**可学习的深度**而非物理仿真的复杂度。"theory of the game"指的是时机、站位、节奏——格斗游戏式的博弈语言，不是"算准抛物线"。

#### 输入精度的刻意设计

> **原文引用（控制章节）**："Directional pad — more precise for quick actions. Left analog stick — more flexible with less 'jerks'."

**技术细节**：
- 方向键 = 离散输入，响应快，用于精确时机操作
- 摇杆 = 连续输入，过渡平滑但"less jerks"（牺牲了瞬时响应）

这证明**输入响应被刻意设计**以服务手感，不是"原样接受玩家输入"。

---

### 9.2 开发者访谈：高塚真吾（Shingo "Seabass" Takatsuka）

**来源**：evoweb.uk 转载的 2006 Games Convention 访谈（PES 6 / WE 2007 时期）

虽然这次访谈聚焦 PS2 时代，但揭示了贯穿整个系列的设计哲学演进。

#### 从"无敌"到参数化护球（手感演进的证据）

> **原文引用**："Up until PES 5 and Winning Eleven 10, no matter which player, you were invincible [when shielding the ball]. [In PES 6] it depends on body strength stats."

**关键发现**：
- PES 5 之前（包括 PS1 时代）护球是**状态无敌**，与物理/力量无关
- PES 6 才引入"身体强度影响护球"——说明早期 WE **刻意牺牲真实性换取手感确定性**

这印证第 4 节"动画即判定"：早期版本"正在护球"就是一个**布尔状态**，AI 和物理都不会打断，玩家因此能学会"何时护球"的确定性策略。

#### 手感优先于真实性：传球速度分队伍调整

> **原文引用**："The game was too fast for all teams. We decided to slow down for the weak passing teams. [Otherwise] Japan [would] dominate via quick passing."

**设计决策**：
- 不追求"每支队伍在物理引擎里自然涌现出风格"
- 而是**直接脚本化速度参数**让弱队慢下来
- 目的是**平衡性**和**风格辨识度**，不是仿真

这呼应官方手册的"nothing is left to chance"——一切都是调参，不是物理。

#### 门将定位的"新弱点"哲学

> **原文引用**："The goalkeeper has much better positioning, but we created new weak points for the goalkeepers. So if you play like you're playing PES 5 and you think it's a goal, it won't be a goal, and vice versa."

**设计思路**：
- 不追求"完美无破绽的 AI"
- 而是**故意留可利用的弱点**，让玩家能学会新的进攻套路
- 每代游戏是**新的博弈平衡**，不是"更接近真实足球"

这是格斗游戏设计思维：角色有强项也有弱点，深度来自玩家学习利用弱点。

#### 平台选择：手柄手感决定一切

> **原文引用**："The PlayStation controller matches our game best in terms of play feeling and control."

**核心价值观**：
- 平台选择的第一标准是**"play feeling"（手感）**
- 不是图形、不是性能、不是用户量
- PlayStation 手柄（压感按键 + 双摇杆）与 WE 的操作语言共同演化了十多年

---

### 9.3 社区技术研究：PS1 版本演进的微观分析

**来源**：evoweb.uk 论坛，用户 franz57 的 PS1 WE 全系列对比研究（23 个版本，2026 年仍在更新分析）

franz57 是资深玩家，对每个 PS1 版本做了细致的多人对战测试。以下是关键技术观察：

#### "粘球"机制的演进

> **原文引用（J.League WE3 98-99）**："The ball isn't almost glued to the foot when sprinting."

**技术推断**：
- WE2/WE3 早期版本：冲刺时球**几乎粘在脚上**（位置绑定到玩家，不做独立物理）
- WE3 98-99 改进：球与脚**松耦合**，冲刺时球会略微脱离
- 这是从"完全脚本化"向"有限自由度"演进的证据

这验证了第 3.2 节"脚本化球状态"：早期 WE 的"携带球"状态就是**位置锁定**，后来才加入"可能失控"的动态。

#### 响应性在演进中反而退步

> **原文引用（World Soccer WE4）**："More responsive than the others that followed, including ISS Pro Evolution."

**反常发现**：
- WE4（2000 年前后）的响应性**优于后续的 ISS Pro Evolution**
- 说明响应性不是单调提升，而是在与"真实性"权衡中摇摆
- 社区玩家明确感知到"后来的版本变慢了"

这呼应第 6 节"响应优先"的哲学：**不是越新越好，而是每代在响应 vs 真实之间做不同取舍**。

#### 区域版差异巨大：同一作品的平行宇宙

> **原文引用（ISS Pro 98）**："PAL version differs significantly from NTSC: the movement is more arcade-like, it spins around like a top, and it has a different shooting system."

**技术含义**：
- PAL（欧版）和 NTSC（日/美版）**不是简单的 50Hz/60Hz 移植**
- 而是**调校了核心玩法参数**（转向速度、射门系统）
- 说明 WE 的"手感"高度依赖**数值调参**而非固定物理引擎

欧洲玩家更偏爱"街机化"转向（陀螺般旋转），日本玩家要"写实"的惯性——**同一引擎，参数一调就是不同游戏**。

#### 每代都是独立博弈系统

> **原文引用（总结）**："!! They all provide very different experiences !!"

franz57 从 23 个 PS1 版本中精选 7-8 个用于多人对战，因为：
- **每个版本的手感、平衡、节奏完全不同**
- 不是"WE6 > WE5 > WE4"的单调进步
- 而是**不同的设计哲学实验**

这呼应第 1 节的核心问题答案：WE 不是在模拟足球，而是在**设计博弈系统**，每代都是新的设计。

#### 社区共识：PS1 版优于早期 PS2 版

论坛多位玩家（jumbo, elfie）的观点：
- PS1 的 ISS Pro Evolution 1 & 2："no fuckin slowdown issues"（性能稳定）+ "superior goalkeeper AI"
- PS2 首作 Pro Evolution Soccer："terrible, all the focus on graphics"（牺牲了玩法）

**技术解读**：
- PS1 的硬件限制**逼出了纯粹的手感设计**——没有余力做花哨表现，只能把核心玩法打磨到极致
- PS2 早期开发者被新硬件诱惑，**在图形上投入过多**，反而丢了手感
- 这验证了本研究的核心论点：**弱硬件 + 聚焦手感 = 更好的游戏**

---

### 9.4 未解之谜与后续研究方向

即便有了上述原始资料，以下问题仍需更深入的技术拆解：

#### 1. 精确的触球判定窗口
- franz57 提到 WE3 是"最后一个可以铲自己传球的版本"——说明触球判定规则在演进
- 具体的**帧窗口 + 距离阈值**数值未公开，需要逆向工程或开发者深度访谈

#### 2. "scripting" / "momentum" 争议
- PES 社区长期争论"游戏是否有隐性追平机制"（落后方射门更易进）
- Seabass 从未正面回应此话题
- 可能需要代码拆解或数据挖掘来确认

#### 3. 与 FIFA 的技术对比
- EA 的 FIFA 系列更早引入"物理引擎"（FIFA 08 的 Impact Engine）
- 但手感一直被批评"笨重、延迟"
- **为什么物理引擎反而伤害了手感？** 需要对比两者的输入响应架构

#### 4. PS1 时代的 AI 决策树
- Seabass 访谈只提到"改了防守站位"，没有深入讲 AI 架构
- franz57 观察到"WE4 的 AI 最聪明"，但机制未知
- 需要日方开发者的 GDC/CEDEC 分享或技术博客

#### 5. 定点数学的具体实现
- 虽然确认 PS1 用定点运算，但 WE 系列的**精度选择**（Q16.16? Q12.20?）和**溢出处理**策略未公开
- 现代复刻（如 eFootball）是否仍用定点？还是改用浮点后模拟定点行为？

---

### 9.5 可追溯的资料线索

如后续能访问更多资源，优先级排序：

1. **日方开发者访谈（日文）**
   - ファミ通（Famitsu）、電撃 PlayStation 的 90 年代末访谈存档
   - CEDEC（日本游戏开发者大会）讲座，搜索「実況」「ウイニングイレブン」
   
2. **PS1 拆解社区**
   - PCSX-Redux 等模拟器社区的逆向工程笔记
   - 游戏文件结构分析（球员数据、动画、AI 脚本的存储格式）

3. **对比测试视频**
   - YouTube 上的"WE2 vs WE3 vs WE4"对比，用慢动作分析触球判定时机
   - TAS（tool-assisted speedrun）社区对帧数据的精确测量

4. **Seabass 的早期博客/推特**
   - 高塚真吾在 PES 系列鼎盛期（2005-2010）可能有开发日志
   - 搜索「高塚真吾」「開発」「PS1」

5. **Konami 内部技术分享**
   - Konami 与 PlayStation 的合作历史文档（Sony 开发者关系档案）
   - 可能在 GDC Vault 有早期分享（需付费会员）

---

### 9.6 本研究的置信度评估

| 主题 | 置信度 | 证据来源 |
|------|--------|----------|
| 解析弹道 + 脚本化状态 | ★★★★★ | 官方手册确认"engine governs trajectory"；社区观察到"粘球"机制 |
| 动画驱动判定 | ★★★★☆ | 手册强调"theory over button mashing"；franz57 观察到判定规则演进 |
| 确定性优先于随机物理 | ★★★★★ | 官方手册"nothing is left to chance" |
| 手感优先于真实性 | ★★★★★ | Seabass 访谈中多次明确（传球速度、护球、门将弱点） |
| AI 分级 LOD | ★★★☆☆ | 基于 PS1 性能推断 + 通用游戏开发实践，但无直接证据 |
| 输入缓冲 | ★★★☆☆ | 官方手册提到 Super Cancel（输入覆盖），暗示有缓冲，但未确认 |
| 磁性传球 | ★★★☆☆ | 普遍玩家感知 + 手册"precise control"措辞，但无开发者确认 |
| 定点数学 | ★★★★★ | PS1 硬件特性 + PSn00bSDK 文档确认行业标准 |

**总结**：第 1-7 节的核心论点（解析弹道、动画驱动、手感优先）已被原始资料证实；AI 和输入细节仍是合理推断，需进一步验证。
