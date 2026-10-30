# Mnemosyne Memory Bank

### A Modular Data Foundation for Evolving Agent Memory

**让记忆有组织、有边界、可追溯、可迁移、可复用。**

Mnemosyne Memory Bank 是 Mnemosyne 记忆基础设施中的数据组织与管理层。

> **本仓定位：** Memory Bank 的公开架构与数据契约说明。  
> **实际实现：** 存储、索引、版本和迁移由上游 [`canyueY/mnemosyne`](https://github.com/canyueY/mnemosyne) 统一管理；本仓不另建一套个人长期记忆实现。

> **⚠️ 重点：本仓没有独立记忆库。**  
> 这里只有架构说明、数据契约与边界文档：**不包含数据库、不启动 Memory Bank 服务、不保存个人数据**。  
> 真正的记忆库是 Mnemosyne Node 管理的一个 **Memory Bank 数据目录**（例如 `%LOCALAPPDATA%\Mnemosyne Memory Bank`）。

## 本仓与 Mnemosyne 的边界（必读）

| 层 | 实体 | 是否独立存储 | 说明 |
| --- | --- | --- | --- |
| 代码 | [`canyueY/mnemosyne`](https://github.com/canyueY/mnemosyne) | 否 | Node / Core / API / SDK |
| 运行 | Mnemosyne Node | **是，一个 Node 一个数据根** | 进程、配置、事务、任务 |
| 数据 | Mnemosyne Memory Bank | **是，Node 的物理数据目录** | SQLite、runtime、logs、components |
| 逻辑 | Personal / Projects / Knowledge / Shared / Public / System | 否，同一数据库内的 Bank | 权限、检索范围、隔离 |
| 文档 | **本仓** `mnemosyne-memory-bank` | **否** | 架构与数据契约说明，不是运行时 |

### 同一 Node 内

- 不同项目用不同 **Bank**，不复制数据库；
- 默认检索只查 `personal`；
- `shared` / `public` 不会被自动带进 `personal`；
- 需要显式指定 `banks` 才能跨空间检索；
- 关系、证据、时间仍走同一套记忆生命周期。

### 不同 Node 之间

- 不共享同一个 SQLite；
- 通过 API / SDK / MCP / 导入导出交换；
- 同一份认知必须有明确权威来源，不能双写；
- 跨 Node 联邦按规划放在 2.1+。

推荐形态：

```text
一个用户
  └─ 一个 Mnemosyne Node
       └─ 一个 Memory Bank
            ├─ personal
            ├─ projects
            └─ knowledge
```

> **一句话：`mnemosyne` 有记忆库，本仓没有独立记忆库。**  
> 两者逻辑上重合、可以关联，但不应该各自维护一套数据库；  
> **项目隔离用 Bank，跨设备才用多 Node + 授权互操作。**

它的职责不是替 Agent 思考，也不是负责决定哪些经验一定值得记住，而是为上层记忆服务提供稳定的数据基础：

- 记忆如何存储？
- 不同主体和项目的记忆如何隔离？
- 记忆如何分类、关联和检索？
- 记忆如何保留历史版本？
- 记忆如何迁移、备份和恢复？
- 不同 Agent 如何在授权范围内共享记忆？

Memory Bank 的核心目标是：

**将记忆从分散的文本和上下文，转化为有结构、有边界、有生命周期的数据资产。**

---

## 目录

- [1. 核心思想](#1-核心思想)
- [2. 项目定位](#2-项目定位)
- [3. 核心特点](#3-核心特点)
- [4. 记忆组织模型](#4-记忆组织模型)
- [5. 数据生命周期](#5-数据生命周期)
- [6. 技术实现方案](#6-技术实现方案)
- [7. 模块化与独立运行](#7-模块化与独立运行)
- [8. 安装与使用](#8-安装与使用)
- [9. 数据安全与治理](#9-数据安全与治理)
- [10. 未来方向](#10-未来方向)
- [11. 设计原则](#11-设计原则)

---

## 1. 核心思想

传统记忆库容易出现几个问题：

- 不同项目的记忆混在一起。
- 用户事实、项目事实和知识资料没有明确边界。
- 新信息不断追加，旧信息却没有明确的失效机制。
- 数据只能通过原有 Agent 访问，无法独立复用。
- 记忆迁移依赖某个框架的内部格式。
- 删除、修改和历史追溯缺少统一规则。

Mnemosyne Memory Bank 希望解决的是记忆数据的基础设施问题。

它将记忆视为一种需要长期维护的数据资产，而不是对话结束后生成的一段摘要。

```text
                   Mnemosyne
                       │
                       ▼
                  Memory Bank
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
           User      Project   Knowledge
             │         │         │
             ▼         ▼         ▼
          Memories  Memories  Resources
             │         │         │
             └─────────┼─────────┘
                       ▼
                Evidence / History
                       │
                       ▼
                Retrieval / Export
                       │
                       ▼
                Agent / Application
```

Memory Bank 的核心职责是组织数据、维护边界和提供稳定访问能力。

---

## 2. 项目定位

| 属性 | 说明 |
|---|---|
| 项目名称 | Mnemosyne Memory Bank |
| 所属系统 | Mnemosyne |
| 项目类型 | 记忆数据基础设施 |
| 核心职责 | 记忆存储、组织、隔离、检索和生命周期管理 |
| 上游 | Mnemosyne Core |
| 下游 | DSH、其他 Agent、SDK、外部应用 |
| 设计方向 | 独立、模块化、可迁移、可组合 |

Memory Bank 不等于某一个 Agent 的个人记忆文件夹。

它应该能够在 Agent 停止运行、宿主更换或模型升级之后继续存在。

---

## 3. 核心特点

### 3.1 数据与 Agent 解耦

Memory Bank 不应该依赖某个 Agent 的内部对话对象。

Agent 只是记忆的使用者之一。

同一个 Memory Bank 可以被不同的服务和应用通过授权接口访问。

这意味着：

- 更换模型不必重建记忆库。
- 更换 Agent 宿主不必重建数据层。
- 多个模块可以复用同一套数据资产。
- 记忆数据可以独立备份、迁移和恢复。

### 3.2 分区组织

建议按照不同的记忆归属和使用场景组织：

| 分区 | 作用 | 典型内容 |
|---|---|---|
| User | 用户长期信息 | 明确偏好、稳定事实、个人约束 |
| Project | 项目连续性 | 项目状态、架构决策、开发进度 |
| Knowledge | 可复用知识 | 技术资料、研究结论、外部知识 |
| Session / Working | 短期上下文 | 当前任务、临时状态、近期信息 |

不同分区不意味着必须使用不同数据库。

逻辑隔离和物理隔离可以根据部署规模独立选择。

### 3.3 记忆具有来源

一条记忆应该尽可能保留：

- 内容。
- 来源。
- 创建时间。
- 有效时间。
- 所属主体。
- 所属项目或分区。
- 证据引用。
- 当前状态。
- 相关历史版本。

这样，记忆库保存的不只是一个结论，也保存了结论的上下文。

### 3.4 历史可追溯

记忆发生变化时，尽量保留变化关系。

例如：

```text
旧状态：使用 Python 作为项目主要语言
              │
              ▼
新证据：项目核心服务迁移到 Node.js
              │
              ▼
更新当前状态
              │
              ▼
保留历史版本与变化原因
```

历史记忆与当前记忆具有不同的用途。

当前状态用于现在的决策，历史版本用于复盘和追溯。

### 3.5 可独立运行

Memory Bank 应尽量提供独立的数据访问能力。

上层应用通过明确的数据契约访问，而不是直接依赖内部文件结构或数据库实现细节。

未来可以在不改变上层业务逻辑的前提下，调整底层存储实现。

### 3.6 可迁移、可恢复

记忆数据不能被某个宿主锁死。

应当逐步支持：

- 标准化导出。
- 标准化导入。
- 数据备份。
- 数据校验。
- 版本迁移。
- 重复数据检测。
- 迁移前预览。
- 迁移后验证。

---

## 4. 记忆组织模型

Memory Bank 的核心是建立稳定的逻辑组织方式。

### 4.1 分区层

将数据按照归属和使用范围组织。

```text
Memory Bank
│
├── User
│   ├── Profile
│   ├── Preferences
│   ├── Constraints
│   └── Experiences
│
├── Project
│   ├── Project A
│   │   ├── Architecture
│   │   ├── Decisions
│   │   ├── Progress
│   │   └── Issues
│   │
│   └── Project B
│       ├── Architecture
│       └── Progress
│
├── Knowledge
│   ├── Technical
│   ├── Research
│   └── References
│
└── Working
    ├── Current Task
    └── Temporary Context
```

以上为逻辑组织示意，不要求物理目录必须采用相同结构。

### 4.2 记忆对象层

不同类型的记忆应该有明确的逻辑身份。

| 对象 | 作用 |
|---|---|
| Memory | 一条可管理的记忆记录 |
| Evidence | 支持或反驳记忆的证据 |
| Observation | 原始观察信息 |
| Entity | 记忆涉及的主体或对象 |
| Relation | 不同记忆或实体之间的关系 |
| Version | 记忆的历史版本 |
| Metadata | 来源、标签、时间和扩展属性 |

### 4.3 状态层

记忆不应只有存在和不存在两种状态。

建议支持以下逻辑状态：

- Candidate：候选。
- Active：当前有效。
- Contested：存在冲突。
- Superseded：已被新状态取代。
- Archived：归档。
- Rejected：被否决。

具体状态是否进入物理数据模型，应由核心服务的数据契约决定。

### 4.4 证据关系

记忆与证据之间应当保持明确关联：

```text
Memory
  │
  ├── supported_by ── Evidence A
  │
  ├── supported_by ── Evidence B
  │
  ├── contradicted_by ── Evidence C
  │
  └── superseded_by ── Memory Version 2
```

这使系统能够在新证据出现时重新评估记忆，而不是直接覆盖文本。

---

## 5. 数据生命周期

Memory Bank 不是一个只负责写入和读取的静态容器。

它需要配合上层 Mnemosyne 服务完成记忆生命周期管理。

```text
           Ingestion
               │
               ▼
           Validation
               │
               ▼
           Classification
               │
               ▼
            Storage
               │
               ▼
           Indexing
               │
               ▼
           Retrieval
               │
               ▼
          Update / Review
               │
               ▼
         Version / Archive
```

### 写入

接收符合数据契约的记忆对象，校验必要字段与来源信息。

### 组织

确定归属、分区、关联对象和适用范围。

### 检索

根据当前任务获取相关记忆，并允许上层结合证据、时间和情境进一步排序。

### 更新

更新记忆状态时保留必要的历史关系。

### 归档

将不再适用但仍有历史价值的记忆转入归档状态。

### 删除

删除应遵守明确的权限和生命周期策略，避免误删被其他记录引用的证据。

---

## 6. 技术实现方案

Memory Bank 的实现不应与某个存储引擎永久绑定。

建议分为以下几个技术层次。

### 6.1 Storage Adapter

统一底层数据访问接口。

上层只依赖数据访问契约，而不直接依赖某种数据库的具体操作。

未来可以根据规模和部署需求替换存储后端。

### 6.2 Repository Layer

负责记忆对象的基本操作：

- Create
- Read
- Update
- Archive
- Delete
- Query
- List

Repository 不应该承担复杂的心理学推理和记忆演化决策。

这些属于 Mnemosyne Core 的职责。

### 6.3 Index Layer

负责提升数据查找效率。

可以逐步组合：

- 结构化字段过滤。
- 全文检索。
- 标签检索。
- 关系检索。
- 向量检索。

不同检索方式应当能够独立启用和替换。

### 6.4 Version Layer

负责记录记忆变化。

至少应保证重要状态变化能够关联：

- 原始版本。
- 新版本。
- 变更原因。
- 相关证据。
- 变更时间。

### 6.5 Import / Export Layer

提供标准化的数据交换能力。

建议使用稳定的导入导出格式，并在迁移前支持：

- Schema 校验。
- 冲突检查。
- 重复检测。
- Dry Run。
- 迁移结果验证。

### 6.6 API / SDK Layer

对外提供稳定的数据访问契约。

具体可以由 Mnemosyne Core 的 API 层统一暴露，避免 Memory Bank 与多个宿主分别维护互不兼容的接口。

---

## 7. 模块化与独立运行

Memory Bank 的一个重要目标是成为可以被不同系统复用的数据基础设施。

建议保持以下边界：

```text
                 Agent / Host
                      │
                      ▼
                Mnemosyne Core
                      │
                      ▼
                Memory Interface
                      │
                      ▼
                  Memory Bank
                      │
             ┌────────┼────────┐
             ▼        ▼        ▼
          Storage   Index    Version
             │        │        │
             └────────┼────────┘
                      ▼
                Persistent Data
```

### 为什么要这样设计？

**第一，降低耦合。**

Agent 不需要理解数据库内部结构。

**第二，方便替换。**

更换存储引擎不应该影响认知演化模块。

**第三，支持独立测试。**

存储、索引、版本和迁移可以分别测试。

**第四，支持多宿主。**

DSH、独立 Agent 和其他应用可以共享相同的数据能力。

**第五，降低未来维护成本。**

新增一个 Agent 接入层，不需要重新实现一套 Memory Bank。

---

## 8. 安装与使用

Memory Bank 应由 Mnemosyne Core 统一管理其数据访问与生命周期。

当前具体部署方式取决于所使用的存储后端和服务版本。

### 环境准备

- Git。
- 对应版本的 Mnemosyne Core。
- 所选存储后端及其运行环境。
- 必要的读写权限。

### 获取项目

```bash
git clone https://github.com/canyueY/mnemosyne-memory-bank.git
cd mnemosyne-memory-bank
```

### 配置

部署时需要明确：

- 数据根目录。
- 默认分区。
- 数据访问权限。
- 备份目录。
- 索引配置。
- 版本迁移策略。

### 使用原则

不建议让 Agent 直接修改 Memory Bank 的底层文件。

推荐通过 Mnemosyne 的统一接口完成：

```text
Agent
  │
  ▼
Mnemosyne API / SDK
  │
  ▼
Memory Repository
  │
  ▼
Memory Bank
```

这样可以确保权限、校验、版本和审计逻辑始终生效。

> 当前 README 不预设具体数据库、目录结构或启动脚本。应在对应 Memory Bank 仓库确定实现后，补入可直接复制执行的安装命令和 SDK 示例。

---

## 9. 数据安全与治理

记忆库保存的是长期数据资产，因此数据安全不是附加功能，而是基础能力。

### 数据隔离

不同用户、Agent、项目和工作区应当有明确的访问边界。

### 最小权限

读取、写入、修改、归档和删除应使用不同的权限控制。

### 默认不共享

个人记忆不会自动转化为项目共享记忆或公共知识。

跨分区共享需要明确授权。

### 来源可追溯

不能因为数据被写入 Memory Bank，就默认其内容一定真实。

### 可恢复

备份和恢复必须经过完整性验证。

### 可迁移

存储格式和内部实现不应成为数据的永久锁定机制。

---

## 10. 未来方向

### P0 — Data Contract Stabilization

- 明确 Memory Schema。
- 统一分区与命名空间规则。
- 完善数据版本。
- 定义 Repository 接口。
- 明确删除和归档语义。

### P1 — Independent Storage Infrastructure

- 存储层独立运行。
- 完善事务和并发控制。
- 增加备份、恢复和完整性验证。
- 提供统一的数据访问接口。

### P2 — Advanced Retrieval Infrastructure

- 全文检索。
- 结构化过滤。
- 关系检索。
- 可选向量索引。
- 多信号融合。

### P3 — Universal Integration

- REST API。
- Python / Node SDK。
- MCP。
- DSH Adapter。
- 第三方 Agent Adapter。

### P4 — Data Portability

- 标准化导入导出。
- 跨版本迁移。
- 多实例同步。
- 数据冲突检测。
- 可验证的数据恢复。

### P5 — Memory Federation

在明确授权、数据隔离和一致性规则的前提下，探索跨实例记忆共享。

联邦能力不应以牺牲个人数据边界为代价。

---

## 11. 设计原则

Memory Bank 的长期价值，不应该由它保存了多少条记录来衡量。

更重要的是：

- 数据是否有明确归属？
- 来源是否能够追溯？
- 历史变化是否能够恢复？
- 不同模块是否能够独立使用？
- 更换 Agent 后数据是否仍然可用？
- 新的存储实现是否可以无痛替换？
- 记忆是否能够在长期运行中保持一致？

**Memory Bank 负责保存和组织经验，Mnemosyne 负责理解和演化经验，Agent 负责使用经验。**

三者各司其职，构成一个可以长期发展的记忆基础设施。



---

## 相关仓库

- [Mnemosyne Core](https://github.com/canyueY/mnemosyne) — 记忆服务、证据、图式演化与 API。
- [Mnemosyne Memory Bank](https://github.com/canyueY/mnemosyne-memory-bank) — 本仓，数据组织与边界说明。
