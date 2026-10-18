# 交付单：per 本地部署 + 双模型 + 接入总部

> 写给：在创始人电脑上运行的本地编码助手（Claude Code、Codex 等均可）
> 写于：2026-10-18，由总部 Chief 起草，创始人已批准方案 B（全程本地、代码不出本机）
> 工作目录：`D:\codex\per`（引擎 + 个人笔记库）；总部文档：`D:\Claude\hq`（本仓库的本地副本）

---

## 0. 先读这些，再动手

1. `D:\codex\per\MIGRATION.md`、`个人超体系统-技术方案.md`、`docs\roadmap.md`、`task_plan.md`、`findings.md`
2. `D:\Claude\hq\CLAUDE.md`、`docs\org\organization.md`、`docs\org\operating-rhythm.md`、`ops\decisions.md`（重点看 D14–D17）

## 1. 铁律（任何阶段都不能违反）

1. **密钥**：`.env`、`1.env` 只由创始人填写。不读出、不打印、不写进日志或提交信息，也不复制到别的文件。需要验证时只输出"已配置 / 未配置"。
2. **私人内容**：`00-inbox/`、`02-daily/`、`08-life/`、`09-memory/`、`_system/workbench-data/` 等笔记内容**只读**：不改写、不删除、不移动。`.gitignore` 现有的排除规则一条都不能删。
3. **不联网上传**：不 `git push`，不新建远程仓库，不把文件发到任何外部服务。调用模型 API 是唯一允许的外部请求。
4. **测试先行**：沿用 per 现有的 TDD 方式，先写失败的测试，再实现。每个阶段结束时，全量测试必须全部通过（当前基线 151/151）。
5. **小步提交**：每个阶段单独开一个分支，提交要小、说明要清楚；合并回 `master` 前先停下来，等创始人确认。
6. **遇到不确定就停下来问**，不要猜。尤其是模型接口参数、环境变量名、脚本命令行参数，都以代码和官方文档为准。

---

## 阶段 0 · 基线检查（只读）

```powershell
cd D:\codex\per
git log --oneline -3
git status --short
git ls-files | Select-String -Pattern '\.env|1\.env|workbench-data|08-life|02-daily|00-inbox'
$env:PYTHONDONTWRITEBYTECODE='1'; $env:PYTHONUTF8='1'
python -m unittest discover -s tests
Get-TimeZone
```

要向创始人报告：
- 最新提交号，以及 `git status` 里那 1 个未提交文件是什么（**不要动它**）。
- `git ls-files` 有没有命中上面这些模式。如果命中，说明密钥或私人内容已经被 git 跟踪，立即停下来报告。
- 测试结果（期望 151/151 OK）。
- 系统时区：期望是香港时间（`China Standard Time` 或 `Asia/Hong_Kong`，都是 UTC+8）。

然后打一个回退标签：`git tag pre-hq-setup-20261018`

---

## 阶段 1 · 本机配置（分支 `setup/local-config`）

1. **项目路径**：把 `_system/config/projects.json` 改成 D 盘的真实路径，用 UTF-8 **无 BOM** 写入：
   - per → `D:/codex/per`
   - goforu → `D:/Claude/goforu`
   - moonrain → `D:/Claude/moonrain`
   - fgame → `D:/Claude/fgame`
   - fenci → `D:/Claude/fenci`
   - general 保持不变

   每个路径都要确认存在。不存在的设为 `enabled: false`，并报告。
2. 跑一次 `python -m _system.scripts.project_brief .`，确认能读到这几个项目的 git 记录。
3. 打开工作台 `python -m _system.scripts.workbench_server`，确认 http://127.0.0.1:8765/ 能打开，「项目巡检」页签里有数据。

**停下来：** 报告结果，等创始人确认后再进入阶段 2。

---

## 阶段 2 · 双模型：智谱 + DeepSeek（分支 `feat/dual-llm`）

**目标：** 两家都能用。按任务选模型，一家失败时自动切到另一家，两家都失败时退回现有的确定性兜底逻辑。

1. **先调研，不写代码：**
   - 读 `_system/scripts/llm_client.py` 和 `.env.example`，弄清现有的环境变量名、默认模型、网关调用方式、失败回退方式。
   - 查 DeepSeek **当前**的官方文档，确认它是否提供 Anthropic 兼容接口（已知曾提供 `https://api.deepseek.com/anthropic`），以及当前的模型名、是否支持 `tool_use` 和 `cache_control`。**以官方文档为准，不要凭记忆写。**
   - 向创始人报告：两家的接口地址、模型名、支持哪些功能，以及打算新增哪些环境变量名。
2. **配置：**
   - 在 `.env.example` 里增加 DeepSeek 一节，只写变量名和注释，不写真实值。
   - 新增 `_system/config/llm_routing.json`（会被 `.gitignore` 排除，由 bootstrap 生成默认值），按任务指定主模型和备用模型。建议的默认值如下，创始人可以随时改这个文件：

   | 任务 | 主模型 | 备用模型 | 理由 |
   |---|---|---|---|
   | 情报雷达 `intel` | 智谱 | 无 | 依赖智谱的 `web_search_prime` 搜索，DeepSeek 没有这个功能 |
   | 随手记分类 `triage` | 智谱 | DeepSeek | 短文本，追求速度 |
   | 思考板解读 `interpret` | DeepSeek | 智谱 | 需要较深的推理 |
   | 需求排序和 PRD 片段 `pm` | DeepSeek | 智谱 | 需要推理 |
   | 复盘 `retro` | DeepSeek | 智谱 | 长文本归纳 |
   | 技能蒸馏 `skill_distill` | DeepSeek | 智谱 | 需要归纳 |
   | 默认 `default` | 智谱 | DeepSeek | |
3. **实现**（测试先行，沿用现有的注入 transport 写法，**测试里不发真实网络请求**）：
   - `llm_client` 支持多个服务商，并按任务路由。
   - 主模型超时或报错时切到备用模型；备用也失败时，抛出现有的 `InterpretationError`，让调用方走原有的兜底逻辑。
   - 每次调用在 `_system/logs/llm-YYYYMMDD.log` 记一行：时间、任务、服务商、模型、耗时、成功或失败、token 用量。**不记录 key，也不记录提示词和回复的正文。**这份日志就是之后经营助理 Ledger 统计成本的依据。
   - key 缺失时，自动跳过那一家；两家都缺时，全部走确定性兜底，不报错退出。
4. **验收：**
   - 全量测试全部通过。
   - 创始人填好两家 key 后，分别对两家跑一次 `python -m _system.scripts.llm_smoke`。如果这个脚本还不支持选择服务商，就给它加一个参数。
   - 输出只显示服务商、模型名、`tool_use` 是否成功、耗时，**不显示 key**。

**停下来：** 报告结果，等创始人确认后合并，再进入阶段 3。

---

## 阶段 3 · Windows 定时任务（分支 `feat/windows-scheduler`）

**背景：** per 原来用 macOS 的 `.plist` 做定时，部分入口还是 `.sh` 脚本（`daily_retro.sh`、`weekly_retro.sh`、`skill_distill.sh`、`intel.sh`、`sync.sh`），在 Windows 上不能直接运行。

1. 先读 `agent_team.py`、`agent_dispatcher.py` 和各个 `.sh` 的内容，确认每个 Agent 真正的 Python 入口和参数。
2. 给每个 Agent 一个统一的 Python 入口：优先复用 `agent_dispatcher`；没有的话，新增 `python -m _system.scripts.run_agent <name>`。`.sh` 文件保留，不删除，方便以后回到 Mac 或 Linux。
3. 新增 `_system/scripts/windows/register_tasks.ps1`，以当前用户身份注册任务（不要用管理员）。任务名统一用 `SuperSelf-<agent>` 前缀，写好对应的 `unregister_tasks.ps1`。时间按香港时间：

   | 时间 | Agent |
   |---|---|
   | 每天 07:00 | growth-planner |
   | 每天 07:30 | intel 情报雷达 |
   | 每天 08:00 | project_brief 项目简报 |
   | 每天 08:30 | pm-agent |
   | 每天 09:00、21:00 | dev-task-tracker |
   | 每 2 小时（08:00–22:00） | triage |
   | 每天 22:00 | retro（日复盘） |
   | 每天 22:30 | skill-distill |
   | 每周一 09:30 | retro（周复盘，和总部的运营节奏一致） |

   - **不注册 `sync`**：创始人决定先不同步到远程。
   - 每个任务的输出追加到 `_system/logs/<agent>-YYYYMMDD.log`，失败时记下错误。
   - 电脑在任务时间关机或睡眠时，开机后补跑一次（启用 `StartWhenAvailable`）。
4. **验收：**
   - 注册后用 `Get-ScheduledTask -TaskName 'SuperSelf-*'` 列出所有任务。
   - 手动触发其中两个（例如 `project_brief` 和 `dev-task-tracker`），确认日志和产出文件都正常生成。

**停下来：** 报告结果，等创始人确认。

---

## 阶段 4 · 接入总部（先出方案，不直接实现）

**目标：** 实现总部 D17 的拆分方案：工作内容进总部，个人内容留在 per。总部的 5 个员工使用 per 的 8 个工具。

这一阶段**只写设计文档和实施计划**，放在 `docs/superpowers/specs/` 和 `docs/superpowers/plans/`，格式沿用已有文档，然后停下来等创始人批准。方案要回答这些问题：

1. **一套引擎、两个笔记库：** per 的脚本大多以笔记库根目录为参数，比如 `project_brief .` 和 `as_root(root)`。能不能让同一套引擎同时服务两个库：工作库 `D:\Claude\hq`（总部）和个人库 `D:\codex\per`？哪些脚本还写死了路径？
2. **目录对齐：** 总部现在用 `ops/briefs`、`ops/standups`、`ops/reviews`、`prd/`；per 用 `11-intel/`、`02-daily/`、`03-reviews/`、`01-projects/*/backlog.md`。建议**总部改用 per 的目录结构，不改 per 的代码**。列出这样做需要改哪些文档。
3. **5 个员工和 8 个工具的对应**：

   | 员工 | 使用的工具 |
   |---|---|
   | Chief | dev-task-tracker、project_brief、retro、growth-planner、agent_team |
   | Scout | intel、oss-scout、pm-agent、triage |
   | Forge | skill-distill、asset-librarian |
   | Probe | 现有测试和 CI；代码体检要新建 |
   | Ledger | 读取阶段 2 的模型调用日志，统计成本；营销功能新建 |

   方案里要说明：工作台的「Agent 团队」页签怎么改成按 5 个员工分组展示；生活和个人内容怎么保证只进个人库，不会混进工作库。
4. **迁移步骤和回退办法。**

---

## 交付给创始人的汇报格式

每个阶段结束时，用中文汇报：
- 做了什么（列出提交号）
- 测试结果
- 发现的问题和风险
- 需要创始人决定的事
- 最后附一句英文练习（创始人的偏好）
