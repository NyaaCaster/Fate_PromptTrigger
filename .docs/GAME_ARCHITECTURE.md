# Fate_PromptTrigger 游戏架构总纲

> Created: 2026-06-18
> Purpose: 记录 Fate 圣杯战争题材多人游戏的核心架构边界。后续玩法、后端、Agent、记忆、战斗与叙事设计均不得超越本文档定义的方向。

## 1. 项目定位

Fate_PromptTrigger 是一款以《Fate》系列圣杯战争题材为灵感的多人房间制 Web 游戏。

- 每局游戏随机发生在中国一个主要现代城市。
- 每个房间容纳 7 名玩家。
- 7 名玩家各自召唤 1 名英灵，共 7 名英灵参与圣杯战争。
- 游戏基础规则参考冬木市第四次圣杯战争，但具体数值、战斗、召唤、事件和胜负逻辑由本项目独立实现。
- LLM 负责角色演绎、叙事包装、氛围生成、规则解说和结构化候选内容生成。
- 非 LLM 服务器模块负责状态权威、规则判定、战斗结算、情报可见性和事件推进。

## 2. 总体架构原则

本项目采用“规则权威型多智能体叙事系统”。

核心结构不是让多个 LLM 自由互相聊天并决定世界，而是：

```text
中央房间服务器 = 圣杯战争真实世界与规则权威
模块化规则系统 = 行为、战斗、治疗、移动、发现、召唤、胜负等判定
客观房间记忆 = 不可变事件日志、公共状态、私有情报、摘要索引
多个 LLM Agent = 角色扮演、叙事演绎、气氛渲染、规则答疑、候选生成
Prompt Builder = 按 Agent 视野、权限、记忆和上下文构造请求
```

### 2.1 不可突破的边界

- LLM 永远不直接管理真实世界状态。
- LLM 永远不直接决定关键事件是否发生。
- LLM 永远不直接决定战斗、伤害、死亡、治疗、移动、侦查、胜负等硬结果。
- LLM 输出的行为意图、叙事文本、候选档案必须经过服务器规则模块校验后才能写入权威状态。
- 玩家、英灵、NPC、事件、新闻等所有 Agent 的可见情报必须由服务器显式分配，不能靠 LLM 自行判断。
- 房间事件记忆是游戏事实源之一，必须严谨记录、可查询、可重建摘要。

一句话原则：

```text
LLM 只负责演绎与候选生成；真实状态与规则结算永远由服务器控制。
```

## 3. 核心服务分层

推荐后端逻辑分为以下服务模块：

```text
Room Service
├─ Rule Engine                 # 战斗、治疗、移动、发现、胜负、资源消耗等硬规则
├─ Event Log                   # 不可变房间事件日志，记录已发生事实
├─ World State Projector       # 从事件日志生成当前公共状态
├─ Knowledge Service           # 管理谁知道什么、来源、可信度、可见性
├─ Memory Service              # 摘要、长期记忆、tag 检索、时间线查询
├─ Prompt Builder              # 统一构造所有 LLM 请求
├─ Agent Runtime               # 调用不同类型 LLM Agent
└─ Persistence                 # 房间、玩家、英灵、事件、对话、情报、摘要等存储
```

### 3.1 Rule Engine

Rule Engine 是所有硬规则的权威实现。它负责：

- 行动合法性校验
- 技能和宝具调用校验
- 命中、伤害、防御、治疗、资源消耗
- 移动、侦查、隐蔽、遭遇判定
- 状态变化、异常状态、胜负条件
- 玩家主动行为和英灵行为意图的最终结算

LLM 不得绕过 Rule Engine 直接写入这些结果。

### 3.2 Event Log

Event Log 是房间中已发生事实的不可变账本。

- 所有玩家行为、系统触发、规则结算、公开事件、私有事件都应写入事件日志。
- 事件日志 append-only，不应被 LLM 改写。
- 摘要、公共状态、Agent 记忆都应视为从事件日志派生出的缓存或投影。
- 如果摘要与事件日志冲突，以事件日志为准。

建议结构：

```ts
type RoomEvent = {
  id: string;
  roomId: string;
  sequence: number;
  timestamp: number;
  eventType: string;
  actors: string[];
  targets: string[];
  locationId?: string;
  visibility: VisibilityRule;
  publicText?: string;
  privateText?: string;
  mechanicalFacts: Record<string, unknown>;
  source: "player_action" | "rule_engine" | "system_trigger" | "llm_narration";
};
```

### 3.3 World State Projector

World State Projector 从事件日志生成当前公共状态，供新闻天气、事件导演、第三方 NPC、裁判进度查询等读取。

公共状态不是所有真相，而是“当前被公开、可观察、可播报、可作为氛围基础”的世界投影。

建议结构：

```ts
type PublicRoomState = {
  city: string;
  currentPhase: string;
  publicTime: string;
  publicWeather?: string;
  knownIncidents: string[];
  publicLocations: LocationState[];
  publicRumors: string[];
};
```

### 3.4 Knowledge Service

Knowledge Service 管理情报隔离，是防止 NPC 串情报的核心。

它需要表达：

- 谁知道什么
- 谁只是怀疑什么
- 谁被误导了
- 情报来源是什么
- 情报是否公开
- 情报是否可以向其他角色传播

建议结构：

```ts
type KnowledgeItem = {
  id: string;
  roomId: string;
  fact: string;
  truthLevel: "confirmed" | "rumor" | "lie" | "hypothesis";
  source: string;
  knownBy: string[];
  visibleToPublic: boolean;
  tags: string[];
  createdAt: number;
};
```

权限过滤必须先于全文搜索或向量召回。正确顺序是：

```text
先按 roomId / agentId / visibility / knownBy 过滤允许集合
再在允许集合中做 tag、关键词、全文或向量召回
```

禁止先做向量召回再过滤权限。

### 3.5 Memory Service

Memory Service 负责长期记忆、摘要、tag 查询和时间线检索。

它不是真相源，只是事件日志和对话记录的派生辅助层。

推荐分层：

- 公共摘要：房间当前公共状态和近期公开事件。
- Agent 私有摘要：某个 Agent 自己知道的历史与判断。
- 御主关系摘要：英灵与其御主的长期关系记忆。
- 时间线事件索引：按时间、角色、地点、tag 检索。
- 后期可加入全文搜索或向量召回，但必须在权限过滤之后进行。

## 4. Agent 分类与职责

本项目不应把所有 LLM 调用都做成同一种长期 Agent。不同 Agent 类型具有不同状态、记忆和权限。

| 类型 | 本质 | 长期记忆 | 状态写入权限 | 推荐实现 |
|---|---|---:|---:|---|
| 召唤 Agent | 一次性候选生成器 | 否 | 否 | 单次 Prompt + JSON Schema |
| 新闻天气 Agent | 氛围播报器 | 否 | 否 | 公共状态 + 外部 MCP + 文本生成 |
| 事件导演 Agent | 事件演绎器 | 否或弱记忆 | 否 | 事件纲要 + 公共状态 + 叙事输出 |
| 战斗叙事 Agent | 已结算结果描写器 | 否 | 否 | 战斗结果 + 可见性 + 风格描写 |
| 第三方 NPC Agent | 受限剧本交互器 | 弱记忆 | 通常否 | 阶段/选项驱动 + 状态包装 |
| 裁判 Agent | 监督/异常/规则/进度功能集合 | 视状态而定 | 否 | 拆分为多种模式 |
| 英灵 Agent | 长期角色智能体 | 是 | 只能提交意图 | Persona + 私有记忆 + 可见状态 |

## 5. 各 Agent 设计方向

### 5.1 召唤 Agent

召唤 Agent 是最简单的单次 Agent。

输入：

- 玩家输入
- 触媒信息
- 当前城市
- 可用职阶
- 房间召唤规则
- 项目预设的召唤提示词

输出：结构化英灵候选档案。

建议输出 JSON Schema，例如：

```ts
type SummonResult = {
  trueName: string;
  className: "Saber" | "Archer" | "Lancer" | "Rider" | "Caster" | "Assassin" | "Berserker";
  alias: string;
  origin: string;
  alignment: string;
  personality: string;
  parameters: {
    strength: string;
    endurance: string;
    agility: string;
    mana: string;
    luck: string;
    noblePhantasm: string;
  };
  skills: SkillDraft[];
  noblePhantasms: NoblePhantasmDraft[];
  hiddenTraits: string[];
  publicIntro: string;
  privateGMNotes: string;
};
```

系统必须对召唤结果做二次校验：

- 职阶是否可用
- 数值是否超标
- 技能和宝具数量是否合规
- 是否符合城市、触媒和玩家输入
- 是否包含禁用内容或破坏平衡的设定

### 5.2 新闻天气 Agent

新闻天气 Agent 无长期记忆，只负责氛围。

触发方式：由非 LLM 游戏系统规则触发。

输入：

- 当前城市和游戏阶段
- 房间公共状态摘要
- 可公开事件
- MCP 提供的真实世界时间、天气、新闻或联网搜索结果
- 播报风格

输出：亦真亦假的环境报道、天气播报、新闻片段、社交媒体传言等。

外部 MCP / 搜索 / 新闻结果是第三方不可信文本，只能放入 `<search_context>`，不得进入 system。

### 5.3 事件导演 Agent

事件导演 Agent 不决定事件是否发生，只演绎系统已筛选出的事件纲要。

流程：

```text
EventSelector 非 LLM 模块根据规则筛选事件
→ EventDirector LLM 根据事件纲要和公共状态生成叙事
→ 系统固化事件记录和后续可见性
→ 相关英灵、玩家、裁判或 NPC 在后续请求中读取该事件
```

事件导演不得自行扩大事件影响，不得自行修改战斗、死亡、暴露、胜负等关键状态。

### 5.4 第三方 NPC Agent

第三方 NPC 通常不需要完整长期 Agent。它们本质是阶段化、场景化、选项驱动的受限剧本交互器。

推荐输入：

- NPC 静态身份
- 当前阶段或场景
- 允许交互选项
- 已消耗选项
- NPC 可见公共状态或有限私有情报
- 玩家当前选择或询问

推荐轻状态：

```ts
type ScriptedNpcState = {
  npcId: string;
  stageId: string;
  visibleKnowledgeIds: string[];
  unlockedOptions: string[];
  exhaustedOptions: string[];
  disposition?: number;
};
```

### 5.5 裁判 Agent

裁判 Agent 不应是单一 Prompt，而应按功能拆分为三种模式。

#### 监督 / 异常处理

- 本质类似第三方 NPC Agent。
- 由系统规则编排出场。
- 只演绎固定交互、异常提示和监督播报。
- 不直接决定惩罚、胜负或规则结果。

#### 规则播报

- 类似系统帮助答疑。
- 可接入知识库 MCP、RAG、SQLite/向量数据库等。
- 通常与游戏公共状态无关，除非用户询问当前局面下的规则适用。
- 输出应偏“教条式解说”，降低创造性。
- RAG 内容只能作为 `<search_context>`，不得进入 system。

#### 进度查询

- 类似存档数据解说。
- 输入是房间记忆、当前状态、玩家可见信息和用户查询。
- 只解释已记录事实和当前可见状态。
- 不访问其他玩家或 NPC 的隐藏信息。

### 5.6 战斗叙事 Agent

战斗叙事 Agent 只负责描写 Rule Engine 已结算的结果。

输入：

- 行动发起者
- 行动目标
- 使用技能或宝具
- 命中、伤害、消耗、状态变化等已结算结果
- 对不同观众的可见程度
- 叙事风格要求

输出：战斗演出文本。

禁止战斗叙事 Agent 自行决定是否命中、伤害多少、谁死亡、是否暴露位置或是否触发反击。

### 5.7 英灵 Agent

英灵 Agent 是唯一真正需要长期交互能力的核心 Agent。

它与御主保持长期 1 对 1 关系，同时可能与其他玩家、英灵、NPC 产生事件关联。

英灵 Agent 的记忆建议分为四层：

```text
1. Persona / 静态角色档案
2. BondMemory / 与御主的长期关系记忆
3. EpisodicTimeline / 时间线事件记忆
4. KnowledgeState / 当前已知情报与信念
```

#### Persona / 静态角色档案

召唤后固定，包括：

- 真名 / 假名
- 职阶
- 性格
- 价值观
- 战斗倾向
- 对御主初始态度
- 技能、宝具、隐藏特质

#### BondMemory / 与御主的长期关系记忆

适合用结构化摘要维护，不必一开始依赖向量库。

建议结构：

```ts
type MasterBondMemory = {
  servantId: string;
  masterPlayerId: string;
  summary: string;
  trustLevel: number;
  conflictPoints: string[];
  promises: string[];
  secretsShared: string[];
  lastUpdatedAt: number;
};
```

#### EpisodicTimeline / 时间线事件记忆

适合按 tag、角色、地点、阶段、关键词触发查询。

建议结构：

```ts
type TimelineEvent = {
  id: string;
  roomId: string;
  timestamp: number;
  phase: string;
  locationId?: string;
  actors: string[];
  witnesses: string[];
  tags: string[];
  publicLevel: "public" | "witnessed" | "private" | "secret";
  fact: string;
  mechanicalResult?: string;
};
```

#### KnowledgeState / 当前已知情报与信念

用于区分确认事实、传闻、误导和怀疑。

英灵请求时只能读取：

- 它亲历的事件
- 它被告知的事件
- 它可以公开知道的事件
- 与本轮话题相关且权限允许的 tag 命中事件
- 它与御主的长期关系摘要

## 6. 房间记忆与公共状态

房间记忆和当前公共状态是整个游戏的核心地基。

不建议把它设计成自由行动的“世界 Agent”。更推荐设计为非 LLM 的 Room Memory Service，并在需要时使用 LLM Summarizer 生成摘要缓存。

### 6.1 权威层

- Event Log：不可变事件日志。
- Rule Engine State：战斗、资源、位置、状态、胜负等硬数据。
- Knowledge Items：情报及其可见性。

### 6.2 派生层

- PublicRoomState：公共状态投影。
- AgentPrivateSummary：Agent 私有摘要。
- MasterBondMemory：御主关系摘要。
- PhaseSummary：阶段摘要。
- SearchIndex / VectorIndex：检索索引。

派生层可以由 LLM 辅助生成，但不具备真相权威。派生层出错时，以权威层为准。

## 7. LLM 请求构造规则

所有 LLM 对话、Agent 调用、RAG、MCP、Prompt Trigger 和 provider-specific request formatting 必须遵守项目规则中的提示词框架。

统一采用四段结构：

```text
静态前缀
- Agent 类型说明
- 不可变身份/职责
- 永久规则
- 固定授权锚点：<session_rules> 是系统注入规则，<search_context> 是外部资料

稳定历史
- 只放真实 user/assistant 对话
- 单次 Agent 可为空
- 不插入动态状态
- 不把多轮历史扁平化成一个巨型 user 字符串

最新 user
- 本轮玩家输入或系统触发请求
- 外部 MCP / RAG / 搜索内容放入 <search_context>

动态尾部
- 当前可见房间状态
- 当前 Agent 私有记忆
- 当前事件纲要
- 系统已结算结果
- 本轮输出格式要求
```

不同 Agent 只改变注入内容，不改变框架边界。

## 8. LLM 输出类型

LLM 输出必须按用途分为两类。

### 8.1 叙事输出

叙事输出可以直接展示给玩家，但不应直接修改权威状态。

```ts
type NarrativeOutput = {
  text: string;
  tone?: string;
};
```

### 8.2 意图输出

意图输出不能直接生效，必须提交给 Rule Engine 或对应系统模块处理。

```ts
type IntentOutput = {
  actionType: string;
  actorId: string;
  targetId?: string;
  params: Record<string, unknown>;
};
```

示例流程：

```text
英灵 Agent：“Master，我将从侧巷突袭那名枪兵。”
→ 输出台词 + 行为意图
→ Rule Engine 判断是否能突袭
→ 结算命中、消耗、伤害、暴露状态
→ Event Log 写入事实
→ Battle Narrator 描写结果
```

## 9. 长期记忆实现阶段建议

### 9.1 MVP 阶段

优先使用普通数据库即可：

- events：结构化事件日志
- conversations：真实对话
- agent_memories：Agent 摘要
- knowledge_items：情报可见性
- summaries：公共摘要、私有摘要、阶段摘要

检索方式：

- roomId
- agentId
- phase
- tags
- actors
- location
- 最近 N 条

### 9.2 中期

加入全文搜索：

- SQLite FTS5
- PostgreSQL full-text
- Meilisearch / Typesense

### 9.3 后期

加入向量召回：

- 用于模糊语义回忆
- 用于“上次我们谈到背叛时”这类自然语言查询
- 用于与宝具、事件、人物关系相关的相似召回

但向量召回必须在权限过滤之后执行。

## 10. 后续设计优先级

后续详细设计应优先落地以下五个核心对象或模块：

1. `RoomEvent`：房间事件日志结构
2. `KnowledgeItem`：情报可见性结构
3. `AgentState`：英灵/NPC/系统 Agent 状态结构
4. `PromptBuilder`：四段式 LLM 请求构造器
5. `SummonAgent`：英灵召唤 Agent 的 JSON 输出 schema

在这些模块明确之前，不应先行扩展复杂 UI、自由 Agent 通信或高自由度剧情生成。

## 11. 最终架构结论

本项目应采用：

```text
中央规则系统 + 客观房间记忆 + 情报可见性服务 + 多种短/长期 LLM 演绎器 + 英灵长期 Agent
```

这是一种多智能体架构，但不是无约束的自治 Agent 群。它的本质是“中央权威状态驱动的多智能体叙事架构”。

后续所有系统设计必须遵守：

- 规则权威归服务器。
- 世界真相归事件日志和规则状态。
- 情报可见性归 Knowledge Service。
- LLM 只在被授权的视野内生成叙事、解释、候选和意图。
- 房间记忆是所有叙事与氛围的地基。
- Prompt 架构必须遵守项目规则中的四段结构和信任边界。
