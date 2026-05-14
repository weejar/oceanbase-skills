# OceanBase DB Skills

96 个 OceanBase 数据库参考指南，覆盖 SQL、PL/SQL、性能调优、安全、迁移、分布式运维、生态工具及 AI 向量特性。

---

## 快速入门

本 Skill 由 96 个专项技能文件组成，按优先级分为 P0-P3 四个层级：

| 层级 | 主题 | 文件数 | 适用人群 |
|------|------|--------|---------|
| **P0** | 核心 DBA | 26 | DBA、运维 |
| **P1** | 开发与迁移 | 28 | 开发、DBA |
| **P2** | 安全、设计、DevOps、特性 | 16 | 架构师、DBA |
| **P3** | 生态工具与 AI | 11 | DBA、开发者 |

---

## 文件索引

详见 [`skills-index.md`](skills-index.md) 获取完整的文件清单和进度跟踪。

---

## 目录结构

```
oceanbase-db-skills/
├── SKILL.md                    # Skill 入口
├── AGENTS.md                   # Agent 路由表
├── CLAUDE.md                   # Claude Code 指引
├── SKILL_GENERATION_PROMPT.md  # 内容生成规范
├── skills-index.md             # 进度跟踪
├── README.md                   # 本文件
└── skills/
    ├── admin/                  # P0: 备份恢复、用户管理、租户管理 (6)
    ├── architecture/           # P0: 分布式架构、多租户、Paxos、OBProxy (7)
    ├── performance/            # P0: 执行计划、索引、统计信息、慢SQL (7)
    ├── monitoring/             # P0: 告警日志、健康巡检、空间管理 (6)
    ├── appdev/                 # P1: 连接池、多语言驱动、事务、锁 (11)
    ├── migrations/             # P1: 各数据库迁移、数据类型映射、验证割接 (9)
    ├── sql-dev/                # P1: SQL规范、DML/DDL、查询改写、分页 (5)
    ├── plsql/                  # P1: 存储过程、触发器、包 (3)
    ├── security/               # P2: 权限、行级安全、加密、审计、网络安全 (5)
    ├── design/                 # P2: 数据建模、分区策略、表组设计 (3)
    ├── devops/                 # P2: OBD部署、Schema管理、版本升级 (3)
    ├── features/               # P2: 物化视图、DBLink、闪回、列存 (5)
    ├── tools/                  # P3: OCP/OMS/ODC/obdiag/obloader/OBD/K8s (8)
    └── ai/                     # P3: 向量检索、AI函数、混合检索 (3)
```

---

## 兼容模式说明

每个技能文件均标注适用版本和兼容模式：

- **MySQL Mode**: 兼容 MySQL 协议和语法
- **Oracle Mode**: 兼容 Oracle PL/SQL 和语法
- 部分特性标注为 V4.3+，需要对应版本支持

---

## 参考文档

- [OceanBase 官方文档（中文）](https://www.oceanbase.com/docs/oceanbase-database-cn)
- [OceanBase 官方文档（英文）](https://en.oceanbase.com/docs/oceanbase-database)
- [OCP 管理平台](https://www.oceanbase.com/docs/ocp)
- [OMS 迁移服务](https://www.oceanbase.com/docs/oms)
- [ODC 开发中心](https://www.oceanbase.com/docs/odc)
- [obdiag 诊断工具](https://www.oceanbase.com/docs/obdiag)
- [OBD 部署器](https://www.oceanbase.com/docs/obd-cn)
