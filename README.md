# kubeblocks-addon-docs

> Addon 开发与测试经验沉淀。目标：**避免冷启动** —— 让 3 个月后接手一个新 KubeBlocks Addon 的团队能直接复用经验，不重新发明轮子。

## 内容

- **方法论** (`docs/addon-*-guide.md`)：通用判定流程、排障 framework、设计原则；不绑定单一引擎
- **引擎案例** (`docs/cases/<engine>/`)：valkey / mariadb / oracle 的具体闭环现场
- **跨引擎案例** (`docs/cases/methodology/`)：跨引擎方法论实证

## 入口

- **索引**：[`docs/SKILL-INDEX.md`](docs/SKILL-INDEX.md) 列出所有主题文档与案例材料
- **冷启动 onboarding**（TBD）：`docs/addon-cold-start-checklist.md`（规划中，按 phase 0-5 顺序读完即可独立排障）

## 写作约定

每篇 `docs/*.md` 在 H1 后必须有标准化 intro block：

```
> **Audience**: addon dev / test / TL
> **Status**: stable | draft | superseded
> **Applies to**: any KB addon | engine X
> **Applies to KB version**: 1.0.x / 1.1.x release tags | main (1.2 unreleased) | any
> **Affected by version skew**: <可选，已知跨版本断点>
```

## Acceptance bars

每篇 doc 由两条独立标准评判：

| 维度 | 标准 |
|---|---|
| **Utility** | 3 个月后另一个 addon 团队接手能不能直接用上 |
| **Readability** | 段落短、术语首次出现解释、流程类用编号列表、案例附录给摘要先放结论 |

## 一篇一主题

主题文档不混写。reconfigure / switchover / TLS / backup / 升级 / 排障等各自独立成篇。后续增量优先迭代原文件，不另开平行文档。
