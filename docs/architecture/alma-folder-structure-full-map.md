# Alma 文件夹结构全景（推断版）

> 目标：把目前已知的 Alma 持久化目录、工作区目录、运行期可能涉及的结构，整理成一份“尽量全”的层级地图。
> 说明：本文件混合了**已明确存在**与**结合现有行为合理推断**两部分；其中推断项会单独标注，避免和已证实结构混淆。

## 1. 总览

Alma 的文件结构可以分成四层：

1. **用户级全局配置层**：`~/.config/alma/`
2. **工作区层**：`~/.config/alma/workspaces/<workspace>/`
3. **仓库/项目层**：例如 `~/workspace/ai/verification-workbench/`
4. **运行时临时/派生层**：日志、截图、导出物、缓存、线程附件等

其中最核心的是 `~/.config/alma/`，它既是 Alma 的“人格与记忆目录”，也是大量持久化资产的归档根。

---

## 2. 已证实的全局目录结构

下面这些结构，已经能从当前运行环境、系统说明或 CLI 约定中确认：

```text
~/.config/alma/
├── SOUL.md                     # Alma 的人格、自我认知、外观设定
├── USER.md                     # owner/主用户画像
├── MEMORY.md                   # 长期整理后的记忆
├── HEARTBEAT.md                # 周期任务/自检清单
├── api-spec.md                 # 本地运行 API 规格/端口信息
├── memory/                     # 每日日记式记忆
│   └── YYYY-MM-DD.md
├── people/                     # 人物档案
│   ├── <name>.md
│   └── <name>.avatar.jpg
├── groups/                     # 群聊日志
│   └── <chatId>_<date>.log
├── chats/                      # 私聊日志
│   └── <chatId>_<date>.log
├── selfies/                    # 自拍相册/形象一致性素材
├── skills/                     # 个人技能覆写或新增技能
│   └── <skill-name>/
│       └── SKILL.md
├── plugins/                    # 已安装插件
├── reports/                    # 自动生成报告
├── workspaces/                 # 工作区根目录
│   └── <workspace>/
└── ...                         # 其他 Alma 自身维护文件
```

---

## 3. `~/.config/alma/` 更细一层的建议结构图

下面是在“已知目录”基础上，进一步展开到更可实施的层级。带 `（推断）` 的是高概率合理结构。

```text
~/.config/alma/
├── SOUL.md
├── USER.md
├── MEMORY.md
├── HEARTBEAT.md
├── api-spec.md
│
├── memory/
│   ├── 2026-04-25.md
│   ├── 2026-04-26.md
│   └── 2026-04-27.md
│
├── people/
│   ├── README.md                        （推断：人物档案索引）
│   ├── deltamx.md
│   ├── deltamx.avatar.jpg
│   ├── MAXAI.md                         （推断）
│   └── <other-person>.md
│
├── groups/
│   ├── README.md                        （推断：群索引）
│   ├── <discordChannelId>_<date>.log
│   ├── <telegramChatId>_<date>.log      （推断）
│   └── <feishuChatId>_<date>.log        （推断）
│
├── chats/
│   ├── README.md                        （推断：私聊索引）
│   ├── <chatId>_<date>.log
│   └── ...
│
├── selfies/
│   ├── index.json                       （推断：素材索引）
│   ├── covers/                          （推断）
│   ├── raw/                             （推断：原始自拍）
│   ├── exports/                         （推断：导出图）
│   └── refs/                            （推断：一致性参考图）
│
├── skills/
│   ├── README.md                        （推断：技能说明）
│   ├── skill-hub/                       （推断）
│   │   └── SKILL.md
│   ├── plan-mode/
│   │   └── SKILL.md
│   └── <custom-skill>/
│       ├── SKILL.md
│       ├── assets/                      （推断）
│       └── examples/                    （推断）
│
├── plugins/
│   ├── <plugin-name>/                   （推断）
│   │   ├── manifest.json                （推断）
│   │   ├── package.json                 （推断）
│   │   ├── dist/                        （推断）
│   │   └── README.md                    （推断）
│   └── ...
│
├── reports/
│   ├── people-summary/                  （推断）
│   ├── memory-rollups/                  （推断）
│   ├── workspace-audits/                （推断）
│   └── <generated-report>.md
│
├── workspaces/
│   ├── default/
│   │   ├── .alma/                       （推断：线程级/任务级元数据）
│   │   ├── scratch/                     （推断：临时文件）
│   │   ├── exports/                     （推断：导出物）
│   │   └── ...
│   └── <workspace-name>/
│       └── ...
│
├── cache/                               （推断：缓存）
├── tmp/                                 （推断：临时产物）
├── logs/                                （推断：运行日志）
├── attachments/                         （推断：聊天附件/上传文件）
├── threads/                             （推断：线程级索引快照）
└── providers/                           （推断：provider 派生配置或测试缓存）
```

---

## 4. 工作区层结构

当前已知工作目录是：

```text
~/.config/alma/workspaces/default
```

这说明 Alma 不只是有“全局人格目录”，还会把一次实际工作的上下文落到某个 workspace 下。一个较完整的工作区层结构，合理上大概像这样：

```text
~/.config/alma/workspaces/
└── default/
    ├── .alma/                           # 当前线程/任务生成的局部元数据（推断）
    │   ├── todos-<threadId>.md
    │   ├── notes-<threadId>.md          （推断）
    │   ├── tasks/                       （推断）
    │   ├── runs/                        （推断）
    │   └── attachments/                 （推断）
    ├── scratch/                         # 临时草稿（推断）
    ├── exports/                         # 导出文件（推断）
    ├── downloads/                       # 下载物（推断）
    └── linked-repos/                    # 关联仓库入口（推断）
```

这层的作用，更像“当前工作现场”，而 `~/.config/alma/` 根目录更像“长期人格 + 记忆 + 资产仓库”。

---

## 5. 仓库/项目层结构（以 verification-workbench 为例）

当前我们实际在整理的仓库位于：

```text
~/workspace/ai/verification-workbench/
```

它本身不是 Alma 的运行目录，而是**承接分析结果、设计稿、逆向理解文档**的项目仓库。

当前可见主结构可以整理为：

```text
verification-workbench/
├── README.md
└── docs/
    └── architecture/
        ├── overall-design.md
        ├── data-model-outline.md
        ├── context-recovery.md
        ├── roadmap.md
        ├── design-conversation-notes.md
        ├── playwright-and-ui-design-notes.md
        ├── earlier-design-context-notes.md
        ├── plugin-skill-loop-design.md
        ├── skill-and-knowledge-management.md
        ├── runtime-and-state-machine-design.md
        ├── system-architecture-overview.md
        ├── extension-points-and-feature-integration.md
        ├── implementation-plan.md
        ├── search-read-write-validation-loop.md
        ├── search-read-write-validation-schema.md
        ├── search-read-write-validation-runtime-contract.md
        ├── pre-development-checklist.md
        ├── workflow-scheduling-and-automation-design.md
        ├── workflow-schema-design.md
        ├── workflow-runtime-contract.md
        ├── error-and-governance-model.md
        ├── accuracy-assurance-and-evaluation-design.md
        ├── observability-and-debuggability-design.md
        ├── standards-and-database-design.md
        ├── architecture-summary-and-blueprint.md
        ├── alma-runtime-reverse-engineering.md
        ├── alma-runtime-architecture-diagram.md
        ├── alma-runtime-database-map.md
        ├── alma-message-graph-model.md
        ├── alma-runtime-sequence-flows.md
        ├── alma-runtime-confidence-boundaries.md
        └── alma-folder-structure-full-map.md
```

这个仓库扮演的是“研究归档 + 设计资产中心”，不是实际的全部运行时文件所在地。

---

## 6. 运行时可能还有哪些目录

结合 Alma 的行为模式，运行时大概率还会伴随下面这几类目录或派生文件：

### 6.1 线程与任务派生物

```text
.alma/
├── todos-<threadId>.md
├── task-<taskId>.json                    （推断）
├── run-<runId>.json                      （推断）
├── timeline-<threadId>.md                （推断）
└── checkpoints/                          （推断）
```

### 6.2 浏览器/截图/自动化产物

```text
tmp/                                      （推断）
├── screenshots/
├── browser-captures/
├── playwright/
└── webfetch/
```

### 6.3 媒体与附件

```text
attachments/                              （推断）
├── discord/
├── telegram/
├── uploads/
└── generated/
```

### 6.4 语音与图像派生物

```text
media/                                    （推断）
├── tts/
├── stt/
├── images/
└── selfie-outputs/
```

---

## 7. 从职责角度理解这些目录

如果不按“文件夹名字”看，而按职责看，Alma 的目录结构其实是在做五件事：

1. **定义我是谁**：`SOUL.md`、`USER.md`
2. **记住发生过什么**：`MEMORY.md`、`memory/`、`people/`、`groups/`、`chats/`
3. **沉淀能力**：`skills/`、`plugins/`
4. **组织实际工作现场**：`workspaces/`、`.alma/`
5. **保存派生产物**：`reports/`、`selfies/`、附件/缓存/截图/日志

也就是说，Alma 的文件系统并不是普通 app 的“配置目录 + 缓存目录”这么简单，它更像一个：

- 人格仓库
- 记忆仓库
- 技能仓库
- 工作区编排层
- 运行产物归档层

几层叠起来的系统。

---

## 8. 建议后续再补的两份图

如果要把“文件夹结构”继续补完整，下一步最值的是两份：

1. **目录结构图（tree + 职责注释版）**
   - 每个目录旁边标“输入/输出/谁写入/谁读取”
2. **目录与数据流对应图**
   - 例如 thread 消息从哪里来、落到哪里、如何进入 memory / people / reports

这样就不只是“树状目录”，而是“目录和运行机制对上号”。

---

## 9. 证实边界

### 已证实
- `~/.config/alma/` 是全局配置根
- `SOUL.md / USER.md / MEMORY.md / HEARTBEAT.md` 存在且有明确职责
- `memory/ people/ groups/ chats/ selfies/ skills/ plugins/ reports/` 是正式约定目录
- `workspaces/` 存在，且当前工作区是 `default`
- 线程级文件会落到 workspace 下，例如 `.alma/todos-<threadId>.md`

### 高置信推断
- workspace 下还会有任务、附件、导出物、临时文件等局部结构
- 全局目录下还会有缓存、日志、附件、providers 派生结构等运行时目录
- `selfies/`、`skills/`、`plugins/` 内部会有自己的索引或资源子目录

### 仍待源码/实机验证
- 线程索引、任务执行记录、运行检查点具体文件名
- 插件目录内部的真实 manifest / dist 结构
- provider 配置究竟全落在单文件、数据库，还是带独立子目录
- 浏览器自动化、截图、上传附件的最终持久化路径

