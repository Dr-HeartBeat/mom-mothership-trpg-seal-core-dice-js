# 母舰 (Mothership) RPG SealDice 核心插件架构文档

**版本:** 1.0.0-beta **适用框架:** SealDice (JavaScript Extension) **架构风格:** MVC 变体 + 管道过滤器模式 (Pipeline-Filter Pattern)

## 1. 项目综述 (Executive Summary)

本项目不仅仅是一个简单的 TRPG 投骰指令集，而是一个基于 SealDice 平台构建的微型 **母舰 (Mothership) RPG 规则引擎**。

它旨在解决传统骰子插件中“状态难以追踪”、“规则硬编码严重”以及“复杂结算逻辑缺失”的痛点。通过引入 **MVC 分层** 和 **管道处理机制**，本项目实现了数据、规则与交互的高度解耦，成功模拟了类似 FVTT (Foundry Virtual Tabletop) 的数据流处理体验，支持复杂的规则结算（如压力溢出、伤害抗性、自动惊恐结算、时间回溯等）。

## 2. 核心架构模式 (Core Architecture)

系统严格遵循高内聚低耦合原则，主要由以下四个层级组成：

### 2.1 Model: 数据与状态层

负责数据的定义、存储与持久化，实现了静态规则数据与动态角色状态的分离。

- **静态数据 (`MS_CONSTS`):**
    
    - **角色:** 充当只读数据库 (Read-only Database)。
        
    - **职责:** 存储规则常量（如属性映射 `ATTR_MAP`）、物品库 (`ITEMS`, `WEAPON_DB`)、惊恐表 (`PANIC_TABLE`)、损伤表 (`WOUND_TABLE`) 等不可变数据。
        
    - **优势:** 规则数值调整只需修改此对象，无需侵入逻辑代码。
        
- **动态状态 (`MSStateManager`):**
    
    - **角色:** 充当 ORM (对象关系映射) 及事务管理器。
        
    - **职责:** 封装 SealDice 的底层存储接口 (`seal.vars`)。
        
    - **核心特性:**
        
        - **快照 (Snapshot):** 在执行指令前自动捕获角色关键状态。
            
        - **回溯 (Undo):** 提供 `.ms back` 接口，允许回滚至上一次操作前的状态，极大降低了跑团时的操作容错成本。
            

### 2.2 Controller: 业务逻辑层

系统的核心大脑，由多个专用控制器组成，负责纯粹的规则运算，**不涉及任何 UI 输出**。

- **`MSRules` (通用规则库):**
    
    - 提供基础工具函数，如骰点计算 (`rollCheck`)、压力管理 (`updateStress`)、状态增删 (`manageCondition`)。
        
- **`MSPipeline` (检定管道):**
    
    - 核心逻辑调度器。接收上下文对象，按顺序经过一系列“过滤器”，最终输出修正后的检定目标值。
        
- **`MSDamageSystem` (伤害子系统):**
    
    - 处理复杂的伤害结算流程，区分 **Outbound** (造成伤害) 和 **Inbound** (承受伤害) 上下文。
        
- **`MSEffectEngine` (效果引擎):**
    
    - 数据驱动的规则解释器。解析 JSON 格式的效果描述（如 `{ type: "resource", key: "压力", op: "add", val: 1 }`），处理递归投骰和延时性状态。
        

### 2.3 Context: 上下文对象

在各层之间传递的数据载体 (DTO)，取代了零散的函数参数，是管道模式的核心。

- **`CheckContext`:** 封装一次属性检定的全生命周期数据（基础值、累加修正值、优劣势计数器、标签集合）。
    
- **`Inbound/OutboundContext`:** 封装伤害事件的详细信息（伤害值、类型、穿透性、掩体状态、护甲AP/DR）。
    

### 2.4 View: 表现与交互层

负责解析用户指令参数，调用 Controller 层逻辑，并将结果格式化为富文本反馈。

- **`MSCommands` (指令控制器):**
    
    - 路由层。解析 `.ms atk` 等指令，组装 Context，调用 Controller，处理文本回显。
        
- **`MOM` (Mission Operations Monitor):**
    
    - 视图装饰器 (Decorator)。一个独立的 AI 人格模块，根据上下文结果（成功/失败/死亡/惊恐），向标准输出中注入符合世界观的“风味文本 (Flavor Text)”。
        

## 3. 关键机制深度解析 (Key Mechanisms)

### 3.1 检定管道系统 (The Check Pipeline)

这是本架构中最具扩展性的设计。所有的属性检定不再是简单的 `roll <= value`，而是一个**流式处理过程**。

**工作流程:**

1. **初始化 (Context Init):** 创建 `CheckContext`，载入基础属性值，并打上初始标签（如 `['combat', 'attack', 'ranged']`）。
    
2. **管道过滤 (Pipeline Processing):** `MSPipeline.process(context)` 让上下文依次经过以下过滤器：
    
    - **状态过滤器 (Conditions):** 遍历角色身上的状态（如“受伤”、“恐惧”），检查状态定义的 `modifiers` 是否匹配 Context 的标签。若匹配，则修改 Context 的数值或优劣势。
        
    - **装备过滤器 (Equipment):** 检查穿戴的护甲或持有的武器，应用被动修正（如“动力甲”导致“速度”劣势）。
        
    - **机制过滤器 (Mechanics):** 处理核心规则限制（如“武器在近距不可用”）。
        
3. **结果输出:** 管道输出最终修正后的目标值 (Target Number) 和优劣势模式 (Advantage/Disadvantage)。
    

**代码映射示例:**

```
// 伪代码演示
const ctx = new CheckContext(mctx, "战斗", 40);
ctx.addTag("ranged"); // 打上远程标签
MSPipeline.process(ctx); 
// -> 自动检测到角色有"眼部受伤"状态(匹配ranged标签)，施加劣势
// -> 最终输出: { target: 40, mode: "dis" }
```

### 3.2 伤害结算系统 (Damage System)

伤害逻辑被拆分为 **Outbound** (输出) 和 **Inbound** (承受) 两个独立阶段，支持 **Dry Run (预计算)** 机制。

- **Outbound Pipeline:** 处理攻击方的加成（如“狂暴”状态增加伤害骰）。
    
- **Inbound Pipeline:** 处理受击方的减免。
    
    - **级联抵消逻辑:** 掩体 (Cover AP/DR) -> 护甲 (Armor AP/DR) -> 生命值 (HP) -> 损伤 (Wounds)。
        
    - **Dry Run 预警:** 在实际扣血前，系统会先进行一次模拟计算。如果判定伤害会导致 HP 归零且用户未指定损伤类型（如 `gunshot`），系统会中断流程并提示用户补充参数，防止误操作导致的流程卡顿。
        

### 3.3 状态管理与回溯 (State & Undo)

为了解决 TRPG 跑团中常见的误操作问题，系统内置了类数据库的事务管理。

- **快照 (Snapshot):** 在执行任何“写操作”指令（如 `.ms hit`, `.ms panic`, `.ms make`）之前，`MSStateManager` 会自动捕获当前角色的关键变量（HP、压力、属性、状态列表）。
    
- **持久化:** 快照被序列化为 JSON 字符串存储。
    
- **回滚:** 用户输入 `.ms undo` 或 `.ms back` 时，系统反序列化最近的快照并覆盖当前状态，实现“时光倒流”。
    

## 4. 数据流视图：一次攻击指令的生命周期

以用户输入 `.ms atk shotgun long` (使用霰弹枪进行远距离攻击) 为例：

```
sequenceDiagram
    participant User
    participant MSCommands (View)
    participant MSRules (Controller)
    participant MSPipeline (Controller)
    participant MOM (View Decorator)

    User->>MSCommands: .ms atk shotgun long
    MSCommands->>MSRules: 解析武器与距离
    MSCommands->>MSCommands: 创建 CheckContext (tags: combat, attack, ranged)
    
    MSCommands->>MSPipeline: process(CheckContext)
    Note over MSPipeline: 1. 检查状态 (如: 眼部受伤->劣势)
    Note over MSPipeline: 2. 检查武器属性 (如: 霰弹枪远距无惩罚)
    MSPipeline-->>MSCommands: 返回修正后的 Target & Mode

    MSCommands->>MSRules: rollCheck(Target, Mode)
    MSRules-->>MSCommands: 返回 检定结果 (成功/失败/暴击)

    alt 攻击命中
        MSCommands->>MSDamageSystem: 计算伤害 (Outbound)
        MSCommands-->>User: 显示伤害值 & 快捷指令 (.ms hit ...)
    end

    MSCommands->>MOM: react("combat", result)
    MOM-->>MSCommands: 返回 AI 吐槽文本
    MSCommands-->>User: 输出最终战报
```

## 5. 扩展性指南 (Extensibility)

本插件的设计使得由非程序员进行内容扩展变得非常简单，通常只需修改 `MS_CONSTS` 即可。

### 5.1 新增状态 (House Rules)

在 `MS_CONSTS.CONDITIONS` 中定义新的状态对象：

```
"太空眩晕": {
    summary: "所有智力检定劣势",
    // 标签匹配系统：自动匹配 tag 为 'intellect' 的检定上下文
    modifiers: [{ target: "intellect", type: "dis" }]
}
```

### 5.2 新增物品特效

在 `MS_CONSTS.ITEMS` 或 `WEAPON_DB` 中添加：

```
"重型机枪": {
    ...,
    // 自动在 Pipeline 中生效：持有此武器时，速度检定获得劣势
    modifiers: [{ target: "speed", type: "dis" }]
}
```