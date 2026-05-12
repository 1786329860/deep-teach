# Few-Shot 示例 3：架构决策场景（Enhanced Card）

> **场景**：用户正在规划一个内容管理系统（CMS）的后端数据库选型
> **操作**：架构决策 —— 选择 PostgreSQL 作为主数据库
> **卡片类型**：Enhanced Teaching Card（含决策树 + 风险提示）

---

## 用户请求

> 我要做一个内容管理系统，有文章、用户、评论、标签这些核心实体，数据之间关联比较多。后端用 Node.js。帮我选一个数据库。

## AI 正常回复

根据你的项目特征分析，我推荐使用 **PostgreSQL** 作为主数据库。让我先说明理由，然后搭建数据库连接层。

你的项目有以下关键特征：

1. **强关系型数据模型** — 文章-评论-用户-标签之间存在多对多/一对多关系
2. **需要事务一致性** — 如发布文章时同时更新分类计数 + 记录操作日志
3. **复杂查询需求** — 按标签筛选 + 按作者过滤 + 按时间范围排序的组合查询
4. **数据结构相对稳定** — CMS 的 Schema 不会频繁变动

PostgreSQL 在这些场景下是业界标准选择。下面开始实施...

## AI 执行的操作

```bash
npm install pg @types/pg
```

创建 `src/db/index.ts` 数据库连接模块和初始 Schema 设计。

---

🎯 **STEP 1 技术深度解析（增强版）**

📌 **操作：** 架构决策 —— 选择 PostgreSQL 作为 CMS 项目的持久化存储方案

---

### ① 🛠️ 所用技术

| 属性 | 内容 |
|------|------|
| **技术** | PostgreSQL 16+ |
| **分类** | 关系型数据库管理系统（RDBMS / SQL Database） |
| **角色** | 项目的主数据存储引擎，负责所有结构化数据的持久化、查询、事务管理 |

**配套技术：**
- **pg (node-postgres)** 8.11+ — Node.js 的 PostgreSQL 驱动（底层、高性能）
- **ORM 可选**：Prisma / Drizzle ORM / TypeORM（本步骤先用原生 pg 驱动）

### ② 💡 为什么选择这项技术

**项目约束：**
- **数据模型约束**：文章、用户、评论、标签之间有明确的外键关系（多对多标签-文章、一对多用户-评论、一对多文章-评论）
- **一致性约束**：发布文章时需原子性地更新多个表（文章状态 + 分类文章数 + 活动日志），不允许部分成功
- **查询复杂度约束**：需要支持多条件组合筛选（WHERE + JOIN + GROUP BY + ORDER BY + LIMIT/OFFSET 分页）
- **数据量预估约束**：初期 < 10 万条记录，中期可能到百万级，但单表不太会过亿
- **团队技能约束**：团队熟悉 SQL 基础语法，有数据库课程背景

**匹配原因：**

| 约束 | PostgreSQL 如何满足 |
|------|-------------------|
| 强关系模型 | 原生支持 FOREIGN KEY / REFERENCES 约束，自动级联更新/删除 |
| 事务一致性 | 完整的 ACID 支持，MVCC 并发控制，SAVEPOINT 嵌套事务 |
| 复杂查询 | 强大的查询优化器（基于成本的 CBO）、JOIN 性能优秀、支持窗口函数/CTE |
| 数据规模 | 百万级数据在合理索引下毫秒级响应；PG 单表可处理数十亿行 |
| 团队 SQL 基础 | SQL 是计算机专业必修课，学习成本低 |

**权衡取舍（Trade-off）：**
```
✅ 选择了：
   - 强一致性保证（ACID 事务）
   - 成熟的关系建模能力（Normalization / Foreign Keys）
   - 强大的查询语言（SQL 是图灵完备的声明式查询语言）
   - JSONB 支持兼顾灵活性（不是纯 rigid schema）
   - 30 年历史的极端稳定性

❌ 放弃了：
   - MongoDB 的水平扩展便利性（Sharding 开箱即用）
   - MongoDB 的 Schema 灵活性（字段可随时增减无需迁移）
   - MongoDB 的嵌套文档自然表达（Comment 直接嵌入 Article 文档）
   - Redis 的极致读取速度（内存级延迟）

⚖️ 这个交换在本项目中是值得的，因为：
   CMS 的核心是「结构化内容」，关系模型天然匹配；
   一致性比灵活性更重要（不能出现文章发布了但分类计数没更新的情况）；
   PG 的 JSONB 字段保留了应对未来 Schema 变更的能力。
```

### ③ 📚 技术深度剖析

#### 【核心原理 — PostgreSQL MVCC 多版本并发控制】

PostgreSQL 的并发模型与 MySQL InnoDB 有本质区别。理解 MVCC 是理解 PG 的关键：

```
传统锁模型（如 MySQL MyISAM）:
  写者 → 锁整张表 → 读者等待 → 写完释放 → 读者才能读
  问题：读写互相阻塞，并发性能差

PostgreSQL MVCC 模型:
  
  时间线 ──────────────────────────────────────→
         T1          T2           T3          T4
  
  事务A:  [────── 读取 v1 ────] 
                              [────── 更新为 v2 ──]
                                                  [提交]
  
  事务B:               [────── 读取 v1 ────]      ← 读到的是快照！不受 A 的写影响
                                                   ← 即使 A 还没提交，B 也不阻塞！

核心机制：
  ┌─────────────────────────────────────────────────┐
  │ PG 不覆盖旧数据，而是写入新版本                   │
  │                                                   │
  │  数据页 (Heap Tuple):                            │
  │  ┌────────────────────────────────────────┐     │
  │  │ xmin=100 xmax=0   data="hello"  ← 旧版本  │     │
  │  ├────────────────────────────────────────┤     │
  │  │ xmin=102 xmax=0   data="world"  ← 新版本  │     │
  │  └────────────────────────────────────────┘     │
  │                                                   │
  │  每个 tuple 记录:                                  │
  │  • xmin = 创建该版本的事务 ID                      │
  │  • xmax = 删除/更新该版本的事务 ID (0=有效)        │
  │  • 事务通过 visibility map 判断自己能看到哪个版本  │
  └─────────────────────────────────────────────────┘
```

**MVCC 带来的关键特性：**

1. **读不阻塞写，写不阻塞读**（Reader-Writer Non-Blocking）
2. **每个事务看到的是一致性的时间点快照**（Snapshot Isolation）
3. **代价**： dead tuples 需要定期 VACUUM 清理回收空间

#### 【关键概念】

**概念 1：B-Tree 索引与查询优化器**

```sql
-- CMS 核心查询示例：获取某标签下按热度排序的文章列表
SELECT a.id, a.title, a.created_at, COUNT(c.id) as comment_count
FROM articles a
JOIN article_tags at ON a.id = at.article_id
JOIN tags t ON at.tag_id = t.id
LEFT JOIN comments c ON c.article_id = a.id
WHERE t.slug = 'javascript'
  AND a.status = 'published'
  AND a.created_at >= '2024-01-01'
GROUP BY a.id, a.title, a.created_at
ORDER BY comment_count DESC
LIMIT 20 OFFSET 0;
```

```sql
-- 为此查询优化的复合索引策略
-- 索引1：加速 tag slug 查找
CREATE INDEX idx_tags_slug ON tags(slug);

-- 索引2：加速 article_tags 关联查找
CREATE INDEX idx_article_tags_article_id ON article_tags(article_id);
CREATE INDEX idx_article_tags_tag_id ON article_tags(tag_id);

-- 索引3：加速 articles 过滤 + 排序（覆盖索引）
CREATE INDEX idx_articles_published_filter 
  ON articles(status, created_at DESC) 
  INCLUDE (id, title);  -- PostgreSQL 11+ 的 Covering Index
```

**概念 2：JSONB — 关系型数据库中的灵活利器**

```sql
-- CMS 中文章的"扩展属性"用 JSONB 存储（避免频繁 ALTER TABLE）
CREATE TABLE articles (
  id SERIAL PRIMARY KEY,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  status VARCHAR(20) DEFAULT 'draft',
  author_id INTEGER REFERENCES users(id),
  metadata JSONB DEFAULT '{}',    -- 扩展字段：封面图、SEO信息、自定义配置
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 插入带元数据的文章
INSERT INTO articles (title, content, metadata) VALUES (
  '深入理解 PostgreSQL',
  '...',
  '{"cover_image": "/imgs/pg-cover.jpg", "seo": {"description": "...", "keywords": ["postgres", "database"]}, "reading_time": 15}'
);

-- JSONB 的 GIN 索引支持高效查询
CREATE INDEX idx_articles_metadata ON articles USING GIN (metadata);

-- 查询：找所有阅读时长 > 10 分钟的文章
SELECT title, metadata->>'reading_time' as minutes
FROM articles
WHERE (metadata->>'reading_time')::int > 10;

-- 查询：找包含特定 SEO 关键词的文章
SELECT title
FROM articles
WHERE metadata @> '{"seo": {"keywords": "database"}}';
-- @> 操作符检查 JSON 包含关系，由 GIN 索引加速
```

#### 【实现细节】

- **数据结构**：
  - **Heap File**：数据以 Append-only 方式追加到数据页（8KB pages），每行是一个 Heap Tuple
  - **B-Tree Index**：平衡多路搜索树，叶子节点包含指向 heap tuple 的物理位置指针（CTID）
  - **Free Space Map (FSM)**：跟踪每个数据页的空闲空间，供 INSERT 快速定位
  - **Visibility Map**：标记哪些数据页的所有 tuple 对所有事务可见（加速 VACUUM 和 Index-Only Scan）

- **算法复杂度**：
  - B-Tree 查找：O(log n)，n = 索引条目数
  - Hash Join（等值连接）：O(N + M)
  - Sort Merge Join（范围连接）：O(N log N + M log M)
  - Nested Loop Join（小表驱动大表）：O(N * M) — 最差但有索引时退化很快
  - VACUUM 全表清理：O(total_tuples)

- **关键机制**：
  1. **查询优化器（Planner/Optimizer）**：基于统计信息（`ANALYZE` 收集的直方图/MCV/相关性）估算每种执行计划的成本（Cost = CPU cost + I/O cost），选择成本最低的计划
  2. **WAL (Write-Ahead Logging)**：每次数据修改先写 WAL 日志再写数据页，崩溃恢复时重放 WAL 保证 durability
  3. **Checkpoint**：周期性将 dirty buffer 刷盘并截断 WAL，平衡性能与恢复时间
  4. **Autovacuum Daemon**：后台进程自动清理 dead tuples、冻结旧事务 ID、更新统计信息

- **执行流程（一次 SELECT）：
  ```
  Parser → Analyzer → Rewriter → Planner → Executor
                                              ↓
                                    [InitPlan] → [ExecScan]
                                                      ↓
                                            [Index Scan / Seq Scan]
                                                      ↓
                                            [Heap Fetch (if needed)]
                                                      ↓
                                            [Result assembly]
  ```

#### ⚠️ **常见陷阱**

| # | 陷阱 | 后果 | 避免/解决方法 |
|---|------|------|--------------|
| 1 | **N+1 查询问题** | 循环中逐条查询关联数据，O(N) 次数据库往返 | 使用 JOIN 或 `WHERE IN (...)` 批量查询；ORM 中使用 eager loading (`include`/`populate`) |
| 2 | **缺少索引导致全表扫描** | 数据量增长后查询从毫秒变秒级 | 用 `EXPLAIN ANALYZE` 分析执行计划；为 WHERE/JOIN/ORDER BY 列建索引 |
| 3 | **忘记 VACUUM 导致表膨胀** | dead tuples 占据空间不释放，表体积持续增长，查询变慢 | 确保 autovacuum 运行正常；大批量 DELETE 后手动 `VACUUM ANALYZE table` |
| 4 | **长事务阻止 VACUUM** | autovacuum 无法清理该事务之前的 dead tuples，表快速膨胀 | 监控 `pg_stat_activity` 中的长事务；设置 `idle_in_transaction_session_timeout` |
| 5 | **OID 自增耗尽风险**（SERIAL 类型） | SERIAL 内部用 INTEGER（最大 21 亿），高写入场景可能耗尽 | 新项目统一使用 `BIGSERIAL` / `GENERATED ALWAYS AS IDENTITY` |

### ④ 🔄 可替代方案对比

| 对比维度 | ✅ **本选方案（PostgreSQL 16）** | 🔶 **替代 A（MongoDB 7.0）** | 🔶 **替代 B（MySQL 8.0）** | 🔶 **替代 C（SQLite）** |
|---------|--------------------------------|---------------------------|-------------------------|---------------------|
| **数据模型** | 关系型（Relational / SQL） | 文档型（Document / BSON） | 关系型（Relational / SQL） | 关系型（嵌入式 / SQL） |
| **核心优势** | ACID + SQL 标准 + JSONB + 扩展生态极强 | Schema 灵活 + 水平扩展简单 + 嵌套文档自然 | 最广泛的托管服务 / 云生态 | 零配置 / 单文件部署 / 极简 |
| **主要劣势** | 水平扩展需外部工具（Citus/Patroni） | 弱事务支持（直到 4.0 才有多文档事务）；无真正 JOIN | JSON 支持弱于 PG；窗口函数/CTE 支持较晚 | 无并发写入；不适合服务端多连接场景 |
| **适用场景** | 结构化数据、强一致需求、复杂查询、GIS/全文检索 | 快速原型、非结构化数据、需要敏捷迭代、海量水平扩展 | Web 应用通用选择、云托管优先、LAMP 栈 | 嵌入式应用、桌面应用、开发测试、低流量 API |
| **ACID 事务** | ✅ 完整支持（MVCC Snapshot Isolation） | ⚠️ 多文档事务（4.0+），有限隔离级别 | ✅ 完整支持（InnoDB MVCC） | ✅ 完整支持（SERIALIZABLE） |
| **JOIN 性能** | ⭐⭐⭐⭐⭐ 优秀的优化器 | ⚠️ $lookup 性能较差（内存限制） | ⭐⭐⭐⭐ 优秀 | ⭐⭐⭐ 够用 |
| **JSON 支持** | ⭐⭐⭐⭐⭐ JSONB（二进制，GIN 索引，完整操作符） | ⭐⭐⭐⭐⭐ 原生文档（BSON） | ⚠️ JSON 类型（无 GIN 索引，性能一般） | ⭐⭐⭐ JSON1 扩展 |
| **扩展性** | 丰富的 PostGIS / pg_trgm / hstore 等扩展 | GridFS（大文件存储） | 较少官方扩展 | 通过加载扩展 |
| **性能基准** (sysbench OLTP_RO) | ~2500 qps（单核参考值） | ~1800 qps（WiredTiger 引擎） | ~2300 qps（InnoDB） | N/A（单连接） |
| **学习曲线** | 中等（SQL + 关系建模） | 平滑（JavaScript 友好，JSON 直觉） | 中等（与 PG 类似） | 低（最简单的 SQL 数据库） |
| **社区 Stars** | GitHub 14k+ / 30+ 年历史 | GitHub 25k+ / 15+ 年历史 | Oracle 维护 / 28+ 年历史 | 公共领域 / 20+ 年历史 |

> **对比总结：** 对于 CMS 这种「结构化内容 + 强关联 + 事务需求」的场景，PostgreSQL 是教科书式的正确答案。MongoDB 的文档模型在处理深层嵌套数据时有优势，但 CMS 的文章-评论-标签模型天然适合关系建模——强行用嵌入式文档会导致数据冗余和更新异常（违反数据库规范化范式）。MySQL 是 PG 最接近的竞争者，但在 JSON 支持、扩展生态、查询优化器成熟度上 PG 略胜一筹。SQLite 仅适用于开发/测试环境。

### ⑤ ⭐ 本选技术的优越性

**关键优势：**

1. **SQL + JSONB 的混合力量 — 兼得鱼与熊掌**
   - 说明：PG 不是纯粹死板的关系数据库。JSONB 类型让你可以在同一张表中同时拥有严格的结构化列（title, status, created_at）和灵活的半结构化列（metadata）。这意味着 CMS 的核心字段享受类型安全、索引加速、外键约束，同时扩展属性可以随时增减字段而无需 DDL 迁移。
   - 数据支撑：Benchmark 显示 PG 的 JSONB 查询性能达到原生 MongoDB 的 **70-90%**（在 GIN 索引加持下），同时保留了 ACID 事务能力。

2. **业界最强的开源查询优化器**
   - 说明：PG 的基于成本的优化器（CBO）经过 30+ 年迭代，能智能选择 Index Scan vs Seq Scan vs Bitmap Index Scan，自动重写子查询为 JOIN，并行化大型查询。
   - 数据支撑：TPC-H 基准测试中，PG 在 22 个复杂查询中的总执行时间优于同配置下的 MySQL 约 **15-30%**（来源：PostgreSQL 官方 Wiki Benchmark）。

3. **不可比拟的扩展生态**
   - 说明：通过扩展机制，PG 可以变成 GIS 数据库（PostGIS）、全文搜索引擎（pg_trgm/tsvector）、时序数据库（TimescaleDB）、图数据库（Apache Age）。对于 CMS 来说 PostGIS 用于基于位置的内容推荐、pg_trgm 用于模糊搜索都是开箱即用的能力。
   - 数据支撑：PostGIS 是业界最先进的开源空间数据库，被 Google、Uber、Grab 等用于生产环境。

**📊 生态成熟度评估：**

| 指标 | 数据 | 评级 |
|------|------|------|
| GitHub Stars | 14.5k+ | ⭐⭐⭐⭐⭐ |
| DB-Engines 排名 | #4（2024年，全部数据库类型） | 顶级 |
| 文档质量 | 9.5/10（官方文档极其详尽，含教程/参考/ internals） | 卓越 |
| 维护频率 | 每年一个大版本发布，活跃的补丁维护 | 极其活跃 |
| 生产验证 | Apple (iCloud)、Instagram、Shopify、Slack、Uber、Netflix | 全球顶级公司广泛验证 |
| 云托管 | AWS RDS/Azure Cloud SQL/GCP Cloud SQL/Supabase/Neon/Railway | 全平台覆盖 |
| 社区支持 | PostgreSQL 邮件列表 + IRC + Discord + Stack Overflow [postgresql] 标签 96K+ | 响应快且质量高 |

### ⑥ 🔗 知识延伸与迁移

**🔄 思想迁移 — 「关系模型 + ACID」思想的跨领域应用：**

| 领域 | 应用 | 类似思想 |
|------|------|---------|
| **分布式系统** | 分布式事务（2PC/Saga 模式） | 将 ACID 思想推广到跨服务场景 |
| **编程语言** | 关系代数映射（LINQ / SQLAlchemy Core） | 将 SQL 声明式查询思想带入通用编程 |
| **数据工程** | Data Warehouse 建模（Star Schema / Snowflake） | 同样的规范化思想应用于分析型数据库 |
| **版本控制** | Git 分支合并（类似 Merge Join） | 三路合并算法与数据库 JOIN 有理论联系 |
| **知识图谱** | RDF Triple Store（三元组存储） | 关系模型的另一种形式化表达 |

**📖 完整学习路径：**

```
前置知识 → [关系代数基础, SQL 语言(DML/DDL/DCL), 数据库 normalization 范式]
                    ↓
          【当前技术：PostgreSQL】
                    ↓
    ├── 进阶方向 A：性能优化
    │     ↓
    │   [EXPLAIN ANALYZE 读懂执行计划 → 索引设计策略 → 分区表 → 连接池(PgBouncer)]
    │     ↓
    │   [高并发调优: shared_buffers/work_mem/maintenance_work_mem]
    │
    ├── 进阶方向 B：高可用与扩展
    │     ↓
    │   [流复制(Streaming Replication) → Patroni 高可用集群 → Citus 水平分片]
    │
    ├── 进阶方向 C：现代 ORM 层
    │     ↓
    │   [Prisma（类型安全的 ORM） / Drizzle ORM（类 SQL 体验）/ TypeORM（装饰器风格）]
    │
    └── 进阶方向 D：PostgreSQL 特有能力
          ↓
        [PostGIS 地理信息 / pg_search 全文搜索 / TimescaleDB 时序数据 / pgvector 向量相似度(AI/RAG)]
```

**📚 推荐资源（按优先级排序）：**

| 类型 | 资源 | 说明 | 来源 |
|------|------|------|------|
| 📖 官方文档 | PostgreSQL Documentation | 全世界最好的开源软件文档之一 | https://www.postgresql.org/docs/current/ |
| 📕 经典书籍 | 《Database Internals》Ch. 7-8 | Alex Petrov 著，深入理解存储引擎与 MVCC | O'Reilly 出版 |
| 📕 经典书籍 |《Designing Data-Intensive Applications》Ch. 3 Storage & Retrieval | Martin Kleppmann 著，LSM-Tree vs B-Tree 对比的经典论述 | O'Reilly 出版 |
| 💻 交互学习 | PostgreSQL Tutorial (postgresqltutorial.com) | 从安装到高级特性的完整免费教程 | https://www.postgresqltutorial.com/ |
| 🎬 视频 | FreeCodeCamp - SQL Full Course | 4 小时从零到精通 SQL | YouTube |
| 🛠️ 工具 | pgAdmin 4 / DBeaver / TablePlus | GUI 管理、查询调试、EXPLAIN 可视化 | 各自官网 |

---

### ⑦ 🌳 决策树（Decision Tree）

```
                        ┌──────────────────────────────┐
                        │     你的数据有关系吗？         │
                        │ （实体间有关联/引用需求？）    │
                        └──────────────┬───────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
              【有明确关系】      【弱关系/嵌套】      【无关系/独立文档】
                    │                  │                  │
          ┌─────────┼─────────┐       │            ┌──────┴──────┐
          ▼         ▼         ▼       ▼            ▼             ▼
    需要 ACID?   可以容忍   最终一致?    MongoDB    Firebase      CouchDB
    (事务?)    最终一致?   即可?                           Firestore
          │         │         │
     ┌────┴────┐    │    ┌────┴────┐
     ▼         ▼    ▼    ▼         ▼
 PostgreSQL  MySQL  DynamoDB Cassandra CockroachDB
 (推荐✅)
     
     ┌─────────────────────────────────────────────────┐
     │ 本项目的路径:                                     │
     │ 有明确关系(文章-评论-用户-标签) → 需要ACID(发布事务)│
     │ → PostgreSQL ✅                                   │
     └─────────────────────────────────────────────────┘
```

**决策路径说明：**

1. **第一层判断 — 数据关系强度**：如果数据实体之间几乎没有关联（如日志收集、物联网传感器读数），文档型或键值数据库更自然。CMS 的文章-评论-用户-标签有大量外键关系 → 走左分支。

2. **第二层判断 — 一致性要求**：如果可以接受短暂的不一致（如社交网络的点赞数），NoSQL 方案可行。但 CMS 的「文章发布」操作涉及多表原子更新 → 必须走强一致性分支 → PostgreSQL。

3. **特殊情况 — 如果数据量达到亿级且必须水平扩展**：考虑 Citus（PG 的分片扩展）或转向 Cassandra/DynamoDB（牺牲一致性换扩展性）。

### ⑧ ⚠️ 风险提示与缓解措施

| 风险类型 | 具体风险 | 可能性 | 影响程度 | 缓解措施 |
|---------|---------|--------|---------|---------|
| **运维风险** | Autovacuum 配置不当导致表膨胀，磁盘占用超预期 | 中 | 中 | 监控 `pg_stat_user_tables` 的 `dead_tuple_count`；设置合理的 `autovacuum_vacuum_scale_factor`；生产环境配置监控告警 |
| **性能风险** | 复杂 JOIN 查询随着数据量增长退化，缺乏合适索引时全表扫描 | 中 | 高 | 建立 SQL 审查规范：所有上线查询必须跑 `EXPLAIN ANALYZE`；建立慢查询日志（`log_min_duration_statement = 1000`）；定期 review 慢查询 |
| **扩展风险** | 单机 PostgreSQL 达到性能上限（CPU/IO/连接数瓶颈）后垂直扩展成本指数上升 | 低（中期） | 高 | 初期：连接池 PgBouncer 减少连接开销；中期：读写分离（流复制 Standby 只读）；长期：Citus 分片或迁移到托管云数据库（Supabase/RDS） |
| **Schema 变更风险** | CMS 功能迭代需要频繁 ALTER TABLE，大表 DDL 会锁表 | 中 | 中 | 使用 `ALTER TABLE ... ADD COLUMN ... IF NOT EXISTS` 非阻塞语法；大表变更使用 `pg_repack` 在线重建；灵活字段优先用 JSONB |
| **团队技能风险** | 团队成员 SQL 能力不足写出低效查询（N+1 / 缺索引 / 全表扫） | 中 | 高 | Code Review 必须审查 SQL；提供 ORM 层（Prisma/Drizzle）降低直接写 SQL 的频率；建立常用查询模板库 |
| **备份风险** | 未配置自动化备份导致数据丢失不可恢复 | 低 | 极高 | 第一天就配置 `pg_dump` 定时任务 + WAL 归档 + Point-in-Time Recovery (PITR)；使用 `pgbackrest` 或云厂商自动备份功能 |
