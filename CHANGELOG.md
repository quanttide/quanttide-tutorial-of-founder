# CHANGELOG

所有显著变更都将记录在此文件中。

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.0.0/)。

版本遵循语义化版本规范：0.0.x（探索期）→ 0.x.y（验证期）→ x.y.z（正式期）

---

## [Unreleased]

### 新增

- `game/` 拆分为 `game/`（游戏设计，暂无内容）与 `war/`（兵棋推演），三篇兵棋教程迁入 `war/`；`index.md`、`README.md`、`myst.yml` 目录同步
- `war/` 三篇兵棋教程重写并重排为递进三部曲：会推棋 → 会造棋 → 会做实验；篇间互指、术语分工（规则校准＝定性写约束，参数标定＝数值倒推），旧版存量措辞一并清理
- `war/counterfactual-wargaming.md`：反事实推演——把历史决策重放两遍：先照历史打验证引擎，再换命令看差异；含前置声明、分叉选择、蓝方AI、参数标定、不变量清单、三组推演、五个翻车点与三个"改条件"番外实验

- `agent/skill.md`：在 CLI 中嵌入并安装 Skill——用 agent-skills-rs 注册、安装、维护锁文件与升级检测；自 `data/context/work/toolkit/` 迁入，正文未改
- `war/wargaming-intro.md`：兵棋推演入门——概念源流、六大要素、回合流程、新手想定与避坑清单；自 `data/context/game/library/` 迁入，重写后作为三部曲第一篇
- `war/historical-wargame-design.md`：用真实历史造一盘棋——史料核证、想定设计、规则校准、复演验证，以豫东战役为例；原名 `yudong-campaign.md`，重写时删去反事实一节（移入专篇，三个"改条件"实验一并带走）
- `create/creative-work-intro.md`、`create/ai-native-creative-work.md`：两篇创意工作教程，自 `data/context/work/tutorial/` 迁入，去掉飞书导出的 UUID 后缀，改名 `creative-work-intro.md`（原《创意工作入门》）与 `ai-native-creative-work.md`（原《AI 原生创意工作》）；`index.md`、`README.md`、`myst.yml` 目录同步
- 初始化教程仓库：`README.md`、`index.md`（入口，待补充）、`myst.yml`
