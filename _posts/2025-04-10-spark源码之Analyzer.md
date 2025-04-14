---
math: true
pin: true
title:    Spark 源码 | Analyzer 解读

date:     2025-04-10
author:   Aiden
image: 
  path : source/internal/data-stream.jpg
categories : ['分布式']
tags : ['计算引擎']
--- 

### Substitution (FixedPoint)

Substitution Batch 主要用于`LogicalPlan`中**特定语法结构或占位符的替换与展开**，确保逻辑计划在元数据绑定后具备可执行的完整结构

#### OptimizeUpdateFields

主要用于优化 `UpdateFields` 表达式（通常在处理复杂数据类型，如`StructType`、`ArrayType`或`MapType`时使用）。

1. **优化`UpdateFields`在`ArrayType`或`MapType`中的操作**

例如 : 当多个`UpdateFields`操作连续作用于同一个结构体（`StructType`）时，**OptimizeUpdateFields**会尝试将它们合并成一个操作，减少中间结果的生成，提高执行效率。

```sql
-- 原始逻辑计划可能包含多个 UpdateFields
UPDATE table SET struct_col.a = 1, struct_col.b = 2
```
优化后可能会合并成一个`UpdateFields`操作，而不是分别执行两次修改。

2. **消除冗余的`UpdateFields`操作**

例如 : 如果某个`UpdateFields`操作没有实际修改数据（如多次更新同一字段为相同的值），优化器可能会移除冗余操作。


#### CTESubstitution

```txt
 分析 WITH 节点，并根据以下条件用 CTE 引用或 CTE 定义替换子计划:
  1. 如果处于兼容模式（legacy mode），或者查询是一个 SQL 命令或 DML 语句，则用 CTE 定义替换，即内联 CTE；
  2. 否则，替换为 CTE 引用 CTERelationRef。是否内联的决定将在查询分析之后，由规则 InlineCTE 来做出。
 
所有在该替换过程中未被内联的 CTE 定义，将会被统一归入一个 WithCTE 节点中，无论是在主查询中，还是在子查询中。
任何不包含 CTE，或其所有 CTE 均已被内联的主查询或子查询，显然将不会包含 WithCTE 节点。
但如果有，它们的 WithCTE 节点会处于原来最外层 With 节点所在的位置。

WithCTE 节点中的 CTE 定义将按照它们被解析的顺序保存。
这意味着，对于任何合法的 CTE 查询，这些定义一定是按依赖关系的拓扑顺序排列的（例如，若有两个CTE定义A和B，且B依赖于A，则 A一定出现在 B 之前）。
否则，这将是一个非法的用户查询，稍后在解析关系时会抛出分析异常。
```

处理 **Common Table Expressions(CTE)** , 即`WITH`子句定义的临时查询结果。它的核心功能是 将逻辑计划中的`CTE`引用替换为实际的子查询，确保后续优化和执行阶段能正确处理`CTE`。

1. **内联展开CTE**

  将SQL中通过`WITH`定义的临时表（如 `WITH t1 AS (SELECT ...)`）替换为直接引用的子查询，消除中间表的概念。

  例如 : 

  ```sql
  WITH t1 AS (SELECT * FROM src) 
  SELECT * FROM t1 WHERE t1.id > 10
  ```

  优化后会被重写为:

  ```sql
  SELECT * FROM (SELECT * FROM src) AS t1 WHERE t1.id > 10
  ```
2. **处理递归 CTE**

  如果CTE是递归的（如 `WITH RECURSIVE`），**CTESubstitution**会确保递归逻辑被正确展开，避免无限循环（Spark 对递归 CTE 的支持较新版本才引入）。

3. **避免重复计算**

  如果同一个 CTE 被多次引用，默认情况下会被内联多次，可能导致重复计算。但Spark的**CTESubstitution**会结合**Cache**或后续优化规则（如`CTERelationDef`）来优化这一问题。

#### WindowsSubstitution

**WindowSubstitution** 主要用于窗口函数（Window Functions）的规范化处理，

1. **窗口函数语法结构的标准化转换**

  将查询中定义的窗口函数（如 `ROW_NUMBER()`、`RANK()` 等）的语法糖转换为Catalyst优化器可识别的Window逻辑节点，
  例如将 `OVER (PARTITION BY ... ORDER BY ...)` 子句转换为包含 `WindowSpecDefinition` 的标准化结构。

2. **窗口函数依赖关系的处理**

  - **解析窗口规范引用**

  若多个窗口函数共享相同的窗口规范（如 WINDOW 子句定义的命名窗口），**WindowsSubstitution** 会将其替换为具体的 `WindowSpecDefinition`，避免重复定义并保证逻辑计划的一致性

  - **拓扑排序保证执行顺序**

  处理窗口函数之间的依赖关系，确保窗口规范的解析顺序符合拓扑排序规则（例如，依赖其他窗口定义的规范需在其之后解析），防止因循环引用导致的解析错误

3. **逻辑计划结构的完整性保障**

  替换逻辑计划中的中间表示（如未绑定的窗口函数占位符），生成完全解析的 Analyzed LogicalPlan，为后续优化器（Optimizer）提供可直接处理的逻辑结构

#### EliminateUnions

用于消除冗余的 **Union** 操作，提升查询效率。

1. **冗余 Union 合并**

  - **相同子查询合并**

  若 **Union** 的子节点包含完全相同的子查询（如 **Union(A, A)**），则将其合并为单个子查询 A。

  - **嵌套 Union 扁平化**

  将多层嵌套的 Union 操作展开为单层结构，例如将 Union(Union(A, B), C) 转换为 Union(A, B, C)6。

2. **无效 Union 消除**

  若 Union 的某个子节点输出为空数据集（例如 Union(A, EmptyRelation)），则直接移除该子节点，仅保留有效分支 A6。

#### SubstituteUnresolvedOrdinals

处理 SQL 语句中的序数位置引用，将其转换为明确的列或表达式，确保逻辑计划的可执行性

1. **序数位置解析**

  在SQL的`ORDER BY`或`GROUP BY`子句中，用户可能通过数字（如 `1、2`）表示列的位置（例如 `ORDER BY 1`）。
  该规则会将这些序数位置转换为对应的实际列名或表达式，避免执行时因位置歧义导致的错误

  ```sql
  -- 示例:将 `ORDER BY 1` 解析为实际列名  
  SELECT name, salary FROM employees ORDER BY 1  
  -- 转换后等价于:  
  SELECT name, salary FROM employees ORDER BY name  
  ```

2. **逻辑计划标准化**

  - **消除未解析的占位符**

  将`UnresolvedOrdinal`（未解析的序数占位符）替换为具体的 `Attribute` 节点，确保逻辑计划中所有引用均与Catalog中的元数据绑定

  - **类型校验支持**

  在转换过程中，验证序数位置对应的列是否存在且类型合法，例如检查 `ORDER BY 3` 是否超出查询结果的列数范围


---

### Disable Hints (Once)

#### ResolveHints.DisableHints

基于`spark.sql.optimizer.disableHints`判断是否移除SQL中所有的**Hints**


---

### Hints (FixedPoint)

主要用于SQL中`Hints`逻辑处理

#### ResolveHints.ResolveJoinStrategyHints

```
允许的连接策略提示（join strategy hint）列表在 [[JoinStrategyHint.strategies]] 中定义，
可以使用连接策略提示指定一组关系别名，例如:"MERGE(a, c)"、"BROADCAST(a)"。
当匹配指定名称的关系（未被重新命名）、子查询或公共表表达式（CTE）时，
将在其上方插入一个连接策略提示的计划节点（join strategy hint plan node）。
```

专门用于 解析并应用用户指定的**Join**策略提示，确保优化器能够根据提示选择合适的物理执行计划。

1. **语法提示转换**

将 SQL 语句中通过注释语法（如 `/*+ BROADCAST(t) */`）指定的 Join 策略提示（如 `BROADCAST`、`MERGE`、`SHUFFLE_HASH`等）转换为逻辑计划中的 `UnresolvedHint` 节点，并绑定到对应的**Join**操作节点

2. **有效性检查**

若同一 Join 操作存在多个冲突的提示（例如同时指定 `BROADCAST` 和 `MERGE`），该规则会根据优先级或默认策略进行选择，并生成警告日志

检查提示的合法性，例如验证被广播的表是否满足大小限制（避免无效的 `BROADCAST` 提示），并在逻辑计划中标记可执行的策略范围

#### ResolveHints.ResolveCoalesceHints

```
COALESCE Hint accepts names "COALESCE", "REPARTITION", and "REPARTITION_BY_RANGE".
```

解析并应用用户指定的 COALESCE 或 REPARTITION 提示，以控制数据分区策略，优化数据分布和计算效率。

1. **分区提示的解析与转换**

将SQL语句中通过注释语法（如 `/*+ COALESCE(3) */` 或 `/*+ REPARTITION(5) */`）指定的分区提示转换为逻辑计划中的`UnresolvedHint`节点，
并绑定到对应的操作节点（如 **SELECT** 或 **JOIN**）

2. **参数合法性校验**

检查提示参数是否合法（如分区数需为整数且大于0），若参数无效则生成警告并忽略该提示

---

### Simple Sanity Check (Once)

这个batch主要进行一些简单的检查校验， 确保语法准确

#### LookupFunctions

负责 **解析和验证 SQL 语句中使用的函数**，确保函数名称正确绑定到具体的实现（内置函数或用户定义函数），并收集函数的元数据（如参数类型、返回类型）供后续优化和执行阶段使用.

1. **函数名称解析**  
   - 在函数注册表（内置函数、UDF、Hive 函数）中查找匹配的函数实现，处理大小写敏感/不敏感的匹配规则。  
   - **示例**：`SELECT CONCAT(a, b)` ➔ 绑定到内置函数 `org.apache.spark.sql.functions.concat`。

2. **函数重载处理**  
   - 根据参数数量和类型选择正确的函数重载版本（如 `SUBSTR(str, start)` 和 `SUBSTR(str, start, length)`）。

3. **验证函数存在性**  
   - 若函数未注册或名称错误，抛出 `AnalysisException`（如 `Function 'my_udf' not found`）。

4. **元数据绑定**  
   - 确定函数的输入参数类型、返回类型及确定性（是否为确定性的），供优化器使用。

5. **处理命名空间**  
   - 按优先级解析函数：临时 UDF > 内置函数 > Hive 函数（若启用 Hive 支持）。

---

### Keep Legacy Outputs (Once)

这个 batch 主要做一些兼容性处理

#### KeepLegacyOutputs

其核心功能是 **在逻辑计划优化过程中保留旧版本 Spark 的输出列结构**，确保升级到新版本后，查询结果的列名、顺序或存在性与旧版本一致，避免因优化规则变更导致的兼容性问题。

1. **保留旧版列名**  
   防止优化规则修改列名（如聚合函数 `SUM(col)` 的默认别名从 `sum(col)` 变为 `sum_col`），维持旧版本的列命名规则。

2. **禁止列剪裁优化**  
   在部分场景中保留未被后续操作引用的列，避免新版本优化器（如 `ColumnPruning`）移除这些列，破坏依赖全列输出的逻辑。

3. **维持输出顺序**  
   确保结果集的列顺序与旧版本一致，即使优化规则重排了逻辑计划中的列。

4. **兼容旧版 UDF 行为**  
   保留旧版本对用户定义函数（UDF）输出类型的处理逻辑（如类型推导规则）。

---

### Resolution (FixedPoint)

Resolution Batch 是逻辑计划解析阶段（Analyzer）的核心流程，主要用于 批量处理逻辑计划中的未解析节点（Unresolved Nodes），将其转换为可执行的元数据绑定状态

#### ResolveTableValuedFunctions

主要用于 解析并绑定用户定义或系统内置的**表值函数**（Table-Valued Functions, TVF），将其从逻辑计划中的未解析状态（Unresolved）转换为元数据绑定的可执行状态。

根据SQL语句中调用的表值函数名称（如 `SELECT * FROM my_tvf(arg1, arg2)`），在`Catalog`或`FunctionRegistry`中查找匹配的注册函数实现，完成函数名称到具体实现的映射

#### ResolveNamespace

负责 **解析 SQL 语句中对象（表、视图、函数等）的命名空间（Namespace）**，确定其所属的 Catalog 和数据库层级，确保跨 Catalog 和多数据库环境下的对象引用语义正确性。

1. **多级标识符解析**  

   处理形如 `catalog.db.table` 或 `db.table` 的多部分名称，拆解为 Catalog、Database 和 Object 层级。  

   **示例**：  
   - `prod.inventory.sales` ➔ Catalog: `prod`, Database: `inventory`, Table: `sales`  
   - `inventory.sales` ➔ 默认 Catalog（如 `spark_catalog`）, Database: `inventory`, Table: `sales`

2. **默认命名空间填充**  

   - 若未显式指定 Catalog，使用 `spark.sql.currentCatalog` 的当前值。  
   - 若未指定 Database，使用 `spark.sql.currentDatabase` 的当前值。

3. **临时对象优先级**  

   确保临时视图（`TEMPORARY VIEW`）的引用优先于持久化对象（即使存在同名表）。

4. **Catalog 实现兼容**  

   支持内置 Catalog（如 `HiveExternalCatalog`）和自定义 Catalog 插件（如 Iceberg、JDBC Catalog）。


#### ResolveCatalogs

```
从 SQL 语句中的多段标识符（multi-part identifiers）中解析 catalog，
并在解析出的 catalog 不是会话 catalog（session catalog）时，
将这些语句转换为对应的 V2 命令。
```

负责 **解析 SQL 语句中显式或隐式引用的 Catalog 实现**，确保多 Catalog 环境下的元数据操作（如跨 Catalog 查询、DDL 操作）能够正确绑定到具体的 Catalog 插件，并为后续的元数据解析（表、视图、函数）提供基础支撑。


1. **Catalog 标识符解析**  
   解析形如 `catalog.db.table` 的标识符，识别其中的 Catalog 名称（如 `iceberg`、`hive`），并验证该 Catalog 是否已在当前会话中注册。

2. **Catalog 实现绑定**  
   将 Catalog 名称映射到具体的 Catalog 插件实现（如 `IcebergCatalog`、`HiveExternalCatalog`），确保后续元数据操作通过插件接口执行。

3. **默认 Catalog 管理**  
   - 处理未显式指定 Catalog 的引用（如 `db.table`），绑定到当前默认 Catalog（`spark.sql.catalog` 配置）。  
   - 支持动态切换默认 Catalog（`USE CATALOG iceberg`）。

4. **临时对象覆盖规则**  
   确保临时视图或函数优先于持久化 Catalog 对象（即使存在同名 Catalog 表）。

5. **Catalog 存在性校验**  
   若指定 Catalog 未注册，抛出 `NoSuchCatalogException`，避免无效元数据操作。

#### ResolveUserSpecifiedColumns

用于 解析并绑定用户在查询中显式指定的列名或表达式，确保列引用在元数据中合法且可访问，并为后续查询优化提供准确的列级语义信息。

- **列名绑定** : 将用户指定的列名（如 `SELECT id, name FROM table`）与目标表或视图中的实际列进行匹配，验证列是否存在于元数据系统中，若未找到则抛出解析错误（如 `ColumnNotFoundException）` 

- **多表引用消歧** : 处理多表 JOIN 场景下的同名列引用（如 `table1.id` vs `table2.id`），通过显式表名前缀或上下文推导明确具体列来源

- **表达式列处理** : 解析用户定义的列表达式（如 SELECT `price*quantity AS total`），推导其数据类型并绑定到逻辑计划中，确保后续操作（如聚合、过滤）可正确引用

- **别名映射** : 将用户定义的列别名（如 `SELECT id AS user_id`）映射到原始列或表达式，生成统一的元数据标识符，避免后续阶段因别名导致的引用歧义

- **类型一致性检查**: 验证用户指定的列或表达式类型是否与上下文兼容（例如聚合函数参数是否为数值类型），若类型冲突则抛出语义错误（如 TypeMismatchException）

- **隐式类型转换** : 在允许的场景下自动插入类型转换逻辑（如将 INT 列隐式转换为 BIGINT 以适配函数参数要求）

#### ResolveInsertInto

专门用于 **解析和验证 `INSERT INTO` 语句的逻辑计划**，确保插入操作的目标表元数据、数据兼容性及分区策略正确绑定，从而生成可执行的插入逻辑节点。

1. **目标表元数据解析**  
   - 验证目标表是否存在且用户具有写入权限。  
   - 解析表存储格式（如 Parquet、ORC）、分区结构（静态/动态）及存储路径。

2. **数据兼容性校验**  
   - 检查插入数据的 Schema 与目标表是否兼容（列数量、顺序、类型可转换）。  
   - **示例**：插入 `INT` 到 `BIGINT` 列允许，但插入 `STRING` 到 `TIMESTAMP` 需隐式转换或报错。

3. **分区策略处理**  
   - **静态分区**：验证分区列值是否合法，路径是否存在。  
   - **动态分区**：根据数据自动推断分区目录，处理 `PARTITION` 子句未覆盖的列。

4. **写入模式适配**  
   - 区分 `INSERT INTO`（追加）和 `INSERT OVERWRITE`（覆盖）语义，生成对应的逻辑操作。  
   - **覆盖模式**：删除目标分区或全表数据（需结合 `spark.sql.sources.partitionOverwriteMode`）。

5. **转换未解析节点**  
   将 `UnresolvedInsertIntoStmt` 转换为 `InsertIntoStatement` 或 `InsertIntoHadoopFsRelationCommand`。

#### ResolveRelations

**解析和绑定 SQL 语句中的表、视图等数据对象**，将逻辑计划中未解析的符号（如表名、别名）转换为具体的元数据引用，确保后续优化和执行阶段能够正确访问数据源。

1. **表/视图名称解析**  
   - 将 `UnresolvedRelation` 节点（未解析的表或视图名）绑定到 Catalog 中的元数据，解析为 `LogicalRelation`（数据源表）或 `HiveTableRelation`（Hive 表）。  
   - **示例**：`SELECT * FROM t` ➔ 解析 `t` 为 Catalog 中注册的表或临时视图。

2. **多级命名空间处理**  
   - 解析形如 `catalog.db.table` 的多部分标识符，结合 `ResolveNamespace` 规则确定 Catalog、数据库层级。  
   - **示例**：`prod.inventory.sales` ➔ Catalog: `prod`, Database: `inventory`, Table: `sales`。

3. **临时对象优先级**  
   - 确保临时视图（`TEMPORARY VIEW`）或临时表的引用优先于持久化对象，即使存在同名表。  
   - **示例**：若存在临时视图 `t` 和表 `default.t`，`SELECT * FROM t` 优先指向临时视图。

4. **子查询与别名处理**  
   - 解析子查询中的表引用和别名（`AS`），确保嵌套查询的正确性。  
   - **示例**：`SELECT * FROM (SELECT id FROM t) AS sub` ➔ 将 `sub` 绑定到子查询结果。

5. **视图展开**  
   - 若引用对象是视图（`VIEW`），递归解析视图定义，将其逻辑计划展开合并到主查询中。

6. **错误检测**  
   - 表/视图不存在时抛出 `AnalysisException`（如 `Table or view not found`）。  
   - 权限不足时抛出 `PermissionDenied` 异常（需集成权限管理框架）。


#### ResolvePartitionSpec

处理分区表的分区规格（Partition Specification）的关键规则，主要解决分区路径的元数据绑定、分区值合法性校验以及动态分区生成等问题。

- **分区路径匹配** : 解析用户指定的分区键值（如 `PARTITION(dt='2024-04-11')`），根据目标表的分区字段定义，验证分区路径在存储系统中的存在性，并绑定对应的分区元数据（如分区列值、存储位置等）

- **静态分区验证** : 若分区值显式指定（如 `dt='2024-04-11'`），检查该分区是否已存在；若不存在且表不允许动态分区，则抛出异常以防止无效写入

- **动态分区推导** : 当插入操作未显式指定分区值（如 `INSERT INTO table SELECT ...`），根据目标表的分区字段自动从插入数据中提取分区列值，并生成对应的分区目录路径

#### ResolveFieldNameAndPosition

处理查询中字段名称和位置的解析与验证规则，其核心功能包括字段存在性校验、歧义消除、位置映射等，确保查询语义的正确性和执行效率

- **字段存在性检查** : 验证查询中引用的字段（如 `SELECT col1, col2 FROM table`）是否存在于目标表或子查询结果的 Schema 中，若字段未定义则抛出 `NoSuchFieldException`，防止无效字段引用

- **按位置引用处理** : 解析`GROUP BY 1`或`ORDER BY 2`等按位置引用字段的操作，将数字位置转换为实际字段名称（如 `GROUP BY col1`），并验证位置是否超出查询结果列数范围

#### AddMetadataColumns

用于动态扩展元数据列的核心规则，其功能覆盖元数据管理、数据操作增强及查询优化支持

```
当节点缺少已解析属性时，为子关系（child relations）的输出添加元数据列（metadata columns）。

对元数据列的引用是通过 [[LogicalPlan.metadataOutput]] 中的列来解析的，但关系的输出中并不会包含这些元数据列，直到该关系被替换为止。
如果这个规则没有将元数据列添加到关系的输出中，分析器将检测到没有任何节点生成这些列。

 此规则仅在某个节点已解析，但其子节点缺少输入的情况下，才会添加元数据列。这可以确保只有在使用时才会添加元数据列。
 
  通过只检查已解析的节点，可以确保 * 展开已经完成，
  避免意外通过 * 选中元数据列。
 
  此规则采用 从上往下（downwards） 的方式解析操作符，
  以避免在投影（projection）阶段过早地丢弃掉元数据列。
```

- **系统元数据扩展** : 在逻辑计划解析时自动为表或子查询添加隐藏的系统元数据列（如 `_file_path`、`_partition`），记录数据来源或存储路径，用户无需显式定义

- **分区字段绑定** : 针对分区表操作（如动态分区写入），自动关联分区字段的元数据信息（如 `dt`、`region`），确保分区剪枝和过滤条件能正确解析


#### DeduplicateRelations

主要用于消除重复关系引用，确保查询逻辑的正确性和执行效率

- **自关联查询处理** : 在**自连接（Self-Join）**或多次引用同一物理表的场景中，自动为重复的表或视图生成唯一别名，避免因同名导致的数据混淆或逻辑错误

```sql
SELECT * FROM table1 a JOIN table1 b ON a.id = b.id;  
```

解析时会为 `table1` 赋予不同别名（如` a `和 `b`），确保逻辑计划中关系对象的唯一性

- **子查询重复引用消除** : 

当子查询在逻辑计划中被多次引用时（如嵌套子查询或 CTE 重复使用），自动合并为单一实例，减少冗余计算


#### ResolveReferences

负责 **解析逻辑计划中的未绑定列名和属性**，确保所有列引用都能正确关联到数据源的元数据（如表、视图或子查询的列）。以下是其核心功能及作用机制：


1. **列名解析与绑定**  
   - 将未解析的列名（如 `id`）绑定到具体的表或视图的列。  
   - **示例**：`SELECT id FROM t` ➔ 解析 `id` 为表 `t` 的 `id` 列。

2. **歧义列处理**  
   - 当多个表存在同名列时，强制要求显式别名限定（如 `t1.id`）。  
   - **示例**：  
     ```sql
     SELECT id FROM t1 JOIN t2 ON t1.id = t2.id  -- 错误！无法确定 `id` 属于哪张表
     ```
     修复：`SELECT t1.id FROM t1 JOIN t2 ON t1.id = t2.id`

3. **别名展开与引用**  
   - 将查询中的列别名（如 `SELECT a AS b`）绑定到原始列，支持后续操作引用别名。  
   - **示例**：  
     ```sql
     SELECT a AS b FROM t WHERE b > 10  -- 将 `b` 解析为 `t.a`
     ```

4. **星号（`*`）展开**  
   - 将 `SELECT *` 扩展为所有列的显式列表（需结合 `ResolveStar` 规则协作）。  
   - **示例**：`SELECT * FROM t` ➔ `SELECT t.id, t.name, ... FROM t`

5. **存在性校验**  
   - 检查列是否在数据源中存在，若不存在则抛出 `AnalysisException`。  
   - **示例**：`SELECT invalid_col FROM t` ➔ `Column 'invalid_col' does not exist`.

6. **子查询作用域处理**  
   - 解析子查询中引用的外层列（关联子查询），标记为 `OuterReference`。  
   - **示例**：  
     ```sql
     SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)  -- `t1.id` 是外层引用
     ```
     
#### ResolveExpressionsWithNamePlaceholders

负责处理逻辑计划中带有命名占位符（例如动态参数或模板化变量）的表达式，将其替换为具体的列、常量或函数调用，确保逻辑计划的完整性和可执行性

- **动态参数解析** :  识别表达式中的命名占位符（如 :`param_name` 或 `{column}`），并通过 `Catalog` 或用户提供的参数映射将其绑定到实际值或列引用
- **模板化查询支持** : 在动态生成的 SQL 模板中（如通过字符串拼接生成的查询），将占位符替换为运行时传入的具体参数，避免硬编码

#### ResolveDeserializer

负责确定数据反序列化的逻辑，将输入数据（如文件、网络流等）转换为 Spark 内部可操作的结构化数据(如 `Row` 对象或特定类型的 `DataSet` )

- **反序列化器匹配** : 根据数据源（如 `Parquet`、`JSON`、`Avro`）的格式和 `Schema` 信息，绑定对应的反序列化器（`Deserializer`），将原始字节流或结构化数据转换为 Spark 的 `Row` 对象或其他数据类型

- **复杂类型处理** : 解析嵌套结构（如 struct、array、map）的反序列化逻辑，确保递归处理字段类型与 Schema 定义一致

#### ResolveNewInstance

负责处理并验证表达式中的对象实例化逻辑（如构造函数调用或工厂方法），确保其类型正确性并转换为`Catalyst`内部可执行的表达式结构。

- **复杂类型支持** : 处理嵌套类型或用户自定义类型（UDT）的实例化，例如将 `new StructType(...)` 转换为结构化数据的内部表示'

- **Catalyst 表达式转换** : 将实例化操作转换为 Catalyst 内部的逻辑表达式（如 `CreateNamedStruct` 或自定义逻辑节点），确保后续优化阶段能正确处理

#### ResolveUpCast

负责处理表达式中的隐式类型提升（Type Promotion），确保不同数据类型在运算或赋值时的兼容性，避免因类型不匹配导致的执行错误。

- **数值类型扩展** : 在混合数值类型的表达式中（如 `Int`+`Long`），自动将低精度类型提升为高精度类型（如`Int`转`Long`），避免溢出或精度损失

- **字符串与时间类型兼容** : 在涉及字符串与时间类型的操作中（如 `CAST('2023' AS DATE)`），验证并提升类型兼容性，确保合法转换

#### ResolveGroupingAnalytics

专用于处理分组聚合操作（如 `GROUP BY`、`GROUPING SETS`、`ROLLUP、CUBE`）的语义解析与逻辑结构转换，确保聚合操作的语法合法性和逻辑正确性。

- **分组列存在性验证** : 检查 `GROUP BY` 子句中引用的列或表达式是否存在于当前作用域的表或子查询中，若未定义则抛出 `AnalysisException`

- **隐式分组处理** : 在包含聚合函数（如 `SUM`、`COUNT`）但未显式声明 `GROUP BY` 的查询中，自动生成全局分组（即所有行视为一个组）

- **多维分组操作转换** : 将 `GROUPING SETS`、`ROLLUP`、`CUBE` 等高级分组语法转换为等效的 `UNION ALL` 逻辑结构，便于后续优化与执行

- **分组层级标识生成** : 为多维分组操作生成 `GROUPING_ID` 或 `GROUPING` 字段，用于标识不同分组层级的组合状态

#### ResolvePivot

专门用于解析和验证 PIVOT 子句的语法结构，并将其转换为逻辑计划中可执行的多列聚合操作，实现数据透视（行转列）功能。

- **PIVOT 子句结构校验** : 验证 `PIVOT` 子句是否符合语法规范，包括 `aggregate_expression`（聚合表达式）、`column_list`（旋转列）和 `expression_lis`t（旋转列值列表）的合法性

- **列存在性检查** : 确保 `column_list` 和聚合表达式中引用的列存在于当前作用域的表或子查询中，否则抛出 `AnalysisException`

- **动态列生成** : 根据`expression_list`中指定的值，将原始数据按`column_list`的取值展开为多列，并为每个值生成对应的聚合列17。例如，`FOR month IN ('Jan', 'Feb')` 会生成 `Jan` 和 `Feb` 两列

#### ResolveOrdinalInOrderByAndGroupBy

```
在许多 SQL 方言中，ORDER BY 和 GROUP BY 子句中使用序号（ordinal positions）是合法的。
本规则的作用是将这些序号位置转换为 SELECT 列表中对应的表达式。Spark 从 2.0 版本开始支持该特性。

具体行为说明:

 如果 ORDER BY 或 GROUP BY 中引用的表达式不是整数，而是可折叠（foldable）的表达式，就忽略它们；

 如果参数 spark.sql.orderByOrdinal 或 spark.sql.groupByOrdinal 被设置为 false，则也会忽略这些序号；

  在 Spark 2.0 之前，ORDER BY 或 GROUP BY 子句中的字面量值（如数字）不会对查询结果产生任何影响。
```

专门用于处理 `ORDER BY` 和 `GROUP BY` 子句中通过列序号（序数）引用列的行为，将其转换为明确的列名或表达式，确保语义正确性和执行可行性。

将 `ORDER BY` 或 `GROUP BY` 子句中的数值序号（如 `1`、`2`）映射为 `SELECT` 列表中对应位置的列名或表达式。

例如，`ORDER BY 1` 会解析为 `SELECT` 列表中的第一列

#### ResolveAggAliasInGroupBy

专门用于处理 `GROUP BY` 子句中引用的聚合别名（如 `SELECT` 列表中定义的别名列），确保别名与底层聚合表达式正确绑定，并验证语义合法性。

将 `GROUP BY` 子句中引用的列别名（如`SELECT SUM(a) AS total GROUP BY total`）解析为对应的聚合表达式（`SUM(a)`），避免直接使用别名导致底层逻辑混淆

#### ResolveMissingReferences

在许多 SQL 方言中，可以对未出现在 SELECT 子句中的属性进行排序。

本规则用于检测这类查询，并将排序所需的属性添加到原始的投影（即 SELECT 列表）中，

以确保它们在排序过程中可用。排序完成后，再添加一个额外的投影操作，将这些临时添加的属性移除。

此外:

`HAVING` 子句也可能会使用未出现在 SELECT 中的分组列。

#### ExtractGenerator

```
从 [[Project]] 操作符的 projectList 中提取 [[Generator]]，并在 [[Project]] 之下创建 [[Generate]] 操作符。

此规则在以下几种情况下会抛出 [[AnalysisException]] 异常:

 - [[Generator]] 被嵌套在表达式中，例如: SELECT explode(list) + 1 FROM tbl

 - 在 projectList 中出现多个 [[Generator]]，例如: SELECT explode(list), explode(list) FROM tbl

 - [[Generator]] 出现在不是 [[Project]] 或 [[Generate]] 的其他操作符中，例如: SELECT * FROM tbl SORT BY explode(list)
```

提取并处理生成器表达式（如 `LATERAL VIEW` 或 `EXPLODE` 函数）的核心规则，其功能聚焦于解析复杂生成逻辑、验证语义合法性，并为后续优化与执行生成结构化逻辑节点


- **语法解构** : 将 SQL 中使用的生成器函数（如 `EXPLODE(array_col)`）解析为逻辑计划中的`Generator`节点，标识其输入表达式（如数组或映射类型的列）与输出结构

- **嵌套生成逻辑处理** : 支持多层生成器嵌套（如 `LATERAL VIEW EXPLODE` 联合使用），递归解析并生成对应的逻辑树结构

#### ResolveGenerate

主要用于解析和验证查询中与生成器（`Generator`）相关的操作（如 `LATERAL VIEW EXPLODE`、`INLINE`、`JSON_TUPLE` 等）。

它的核心功能是将逻辑计划中未解析的生成器表达式（`UnresolvedGenerator`）绑定到具体的生成器实现，并确保生成器的输入和输出符合语义规范。

- **解析生成器表达式**

生成器（`Generator`）用于将复杂数据类型（如数组、Map、Struct）展开为多行数据（行转列操作）。 `ResolveGenerate` 的作用包括:

绑定生成器类型:将逻辑计划中的`UnresolvedGenerator`（未解析的生成器表达式）替换为具体的生成器实现（如 `Explode`、`PosExplode`、`Stack`等）。

验证输入数据类型:确保生成器的输入列是合法的复杂类型（如 `ArrayType`、`MapType`）。

处理生成器的别名:解析生成器输出的别名（例如 `EXPLODE(col) AS (a, b)）`，并验证别名数量与生成器输出的字段数量是否匹配。

- **处理 `LATERAL VIEW` 语法**

在 Spark SQL 中，LATERAL VIEW EXPLODE 是常见的生成器用法，例如:

```sql
SELECT id, a, b 
FROM table 
LATERAL VIEW EXPLODE(array_col) AS a 
LATERAL VIEW EXPLODE(map_col) AS b, c
```

`ResolveGenerate` 会解析这些语法，生成对应的逻辑计划节点（`Generate`），并确保:

生成器的输入列（如 `array_col`、`map_col`）存在且类型正确。

别名（`a`、`b`, `c`）与生成器的输出字段匹配。

- **生成 `Generate` 逻辑节点**

解析完成后，`ResolveGenerate` 会将 `UnresolvedGenerator` 转换为 `Generate` 节点，表示具体的生成器操作。

输入:生成器表达式（如 `Explode(array_col)`）。

输出:展开后的列及其别名。


#### ResolveFunctions

负责解析和验证逻辑计划中的函数调用。它的核心任务是将未解析的函数（`UnresolvedFunction`）绑定到具体的函数实现（内置函数或用户自定义函数），并确保函数参数的类型和数量符合定义。

1. **解析函数名称**

  - **绑定函数实现**:将逻辑计划中的 `UnresolvedFunction` 转换为具体的函数表达式（如 `Sum`、`Explode`、自定义`UDF`等）。

  - **多部分标识符支持**:支持跨数据库或 `Catalog` 的函数名称解析（如 `db.func` 或 `catalog.db.func`）。

2. **验证函数参数**

  - **参数类型匹配**:检查输入参数类型是否与函数定义的签名一致（例如，`LENGTH` 函数要求输入为字符串类型）。

  - **参数数量匹配**:验证参数数量是否合法（例如，`SUBSTR` 需要2或3个参数）。

3. **处理函数重载**

部分函数支持多态签名（如 `COALESCE` 可接受任意数量参数），`ResolveFunctions` 会选择最匹配的实现。

4. **支持自定义 UDF**

  - **UDF（用户自定义函数）**:解析通过 `spark.udf.register` 注册的函数
  - **临时函数与持久化函数**:优先解析临时函数，再查找持久化函数（如 Hive UDF）。

#### ResolveAliases

用于解析逻辑计划中的别名（`Alias`），确保列名、表达式或子查询的别名在上下文中唯一且无歧义。它的核心任务是消除别名冲突，并为后续优化阶段提供明确的引用标识。

1. **解析列或表达式的别名**

  - **显式别名**:处理 `SELECT` 子句中的别名（如 `SELECT col AS alias`）。

  - **隐式别名**:为复杂表达式自动生成唯一别名（如聚合函数 `SUM(col)` 或子查询结果）。

2. **处理子查询别名**

为子查询块（如 `FROM (SELECT ...) AS subq）`分配别名，确保外部查询能正确引用子查询的列。

3. **消除别名冲突**

  - **列别名重复**:若同一作用域中存在重复别名，抛出错误（如 `SELECT a AS col, b AS col）`。

  - **别名作用域隔离**:确保子查询中的别名不会与外部作用域冲突（如嵌套查询中的同名别名）。

#### ResolveSubquery

专门用于解析和验证逻辑计划中的子查询（`Subquery`）。它的核心任务是确保子查询（如`EXISTS`、`IN`、标量子查询等）在语法和语义上合法，并正确绑定到外部查询的上下文。

1. **解析子查询结构**

  - **子查询类型识别**:处理不同类型的子查询，包括: **标量子查询**, **EXISTS/IN 子查询**, **派生表**
  - **绑定子查询到逻辑计划** : 将未解析的子查询（如 `UnresolvedSubquery`）转换为具体的逻辑计划节点（如 `ScalarSubquery`、`ListQuery`）

2. **处理相关子查询**

  - **关联外部列引用**:解析子查询中引用的外部查询列（例如 `WHERE outer.col = subq.col`），并将其转换为 **outer reference（外部引用）**。
 
  - **解相关（可选）**:部分子查询可能需要解相关（转换为 `JOIN` 操作），但这通常由后续规则（如 `DecorrelateInnerQuery`）处理。

#### ResolveSubqueryColumnAliases

专门用于解析子查询中定义的列别名，确保外部查询能够正确引用这些别名。

它的核心任务是处理子查询生成的列别名的作用域和可见性，避免多层嵌套查询中的别名冲突或歧义。

1. **解析子查询的列别名**

  - **别名传递**:将子查询中定义的列别名（如 `SELECT id AS a`）暴露给外部查询，使其能够通过别名（如 a）引用子查询的列。
  - **处理派生表别名**:确保`FROM (SELECT ...) AS subq`中的子查询列别名在外部可见

2. **处理多层嵌套子查询**

  - **作用域隔离**:在多层嵌套子查询中，确保别名仅在直接外层生效，避免跨层引用冲突。

  - **逐层解析**:按嵌套层级由内向外解析别名，保证外层引用内层子查询的别名。

#### ResolveWindowOrder

专门用于解析和验证窗口函数（Window Function）中的 `ORDER BY` 子句。

它的核心任务是确保窗口函数的排序表达式合法且语义正确，并将其绑定到逻辑计划中的具体列或表达式。

1. **解析窗口排序表达式**

  - **绑定列或表达式**:将窗口函数中未解析的 `ORDER BY `表达式（如` UnresolvedAttribute`）转换为具体的列引用或表达式（例如 `SortOrder(col ASC)`）。
  - **验证排序键存在性**:确保 `ORDER BY` 中引用的列或表达式在窗口函数的输入中存在。

2. **验证排序语义**

  - **数据类型可排序性**:检查排序键的数据类型是否支持排序（例如，复杂类型 `ArrayType` 不可直接排序）。
  - **窗口框架依赖**:如果窗口函数定义了框架（如 `ROWS BETWEEN ...`），必须存在 `ORDER BY` 子句（部分框架类型需要明确排序）。

3. **处理隐式排序规则**

  - **默认排序方向**:若未显式指定排序方向`（ASC/DESC）`，默认使用 `ASC`。

  - **空值排序规则**:处理 `NULLS FIRST` 或 `NULLS LAST` 的隐式或显式定义。


#### ResolveWindowFrame

专门用于解析和验证窗口函数（Window Function）中的窗口框架（Window Frame）。

其核心任务是确保窗口框架的定义（如 `ROWS BETWEEN` 或 `RANGE BETWEEN`）合法且语义正确，并将其绑定到逻辑计划中的具体表达式，为后续优化和执行提供明确的行范围定义。

1. **解析窗口框架边界**

  - **边界表达式绑定** :将 `ROWS/RANGE BETWEEN` 中的未解析表达式（如 `UNBOUNDED PRECEDING`、`CURRENT ROW`、`1 FOLLOWING`）转换为具体的逻辑表达式。

  - **验证边界类型** :确保边界表达式的数据类型合法（例如，`ROWS` 的偏移量必须为整数，`RANGE` 的偏移量需与` ORDER BY` 列类型兼容）。

2. **设置默认窗口框架**

  - **隐式框架定义**:若未显式指定窗口框架，根据 ORDER BY 是否存在设置默认框架:

    - **有 ORDER BY**:默认 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`。

    - **无 ORDER BY**:默认 `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`（整个分区）。

3. **验证框架与上下文的兼容性**

  - `RANGE` 框架依赖: 若使用 RANGE，必须存在 `ORDER BY` 且其列类型支持范围计算（如数值、日期）。

  - **边界顺序合法性**:检查窗口框架的起始和结束位置是否合理（例如，结束边界不能在起始边界之前）。

4. **处理特殊边界值**

  - **UNBOUNDED 处理**:标识无界起始（UNBOUNDED PRECEDING）或无界结束（UNBOUNDED FOLLOWING）。

  - **CURRENT ROW 处理**:标记当前行位置。

#### ResolveNaturalAndUsingJoin

专门用于解析 `NATURAL JOIN` 和 `USING` 子句的隐式连接条件，并将其转换为显式的等值连接（`EQUI-JOIN`）。

它的核心任务是简化用户编写连接查询的语法，同时确保语义正确性，避免列名歧义或错误。

1. **解析 `NATURAL JOIN`**

  - **自动匹配同名列**：对于 `NATURAL JOIN`，自动识别左右表中所有同名的列，并生成等值连接条件（例如 `left.a = right.a AND left.b = right.b`）。

  - **消除结果列重复**：在输出结果中合并同名列，仅保留一列（避免重复）。

2. **解析 USING 子句**

  - **显式指定连接列**：对于 `USING(col1, col2)`，生成等值连接条件（例如 `left.col1 = right.col1 AND left.col2 = right.col2`）。
  
  - **结果列合并**：在输出结果中仅保留 col1 和 col2 的单列（而非 `left.col1` 和 `right.col1`）。


#### ResolveOutputRelation

专门用于解析和验证数据写入操作（如 `INSERT INTO`、`DataFrame.write`）的目标表或输出路径。其核心任务是确保写入操作的目标关系（表、视图、文件路径等）合法且结构兼容，同时处理动态分区、列映射等复杂场景。

1. **解析目标表/路径**  
  - **绑定目标元数据**：将未解析的输出关系（如 `UnresolvedOutputRelation`）绑定到 Catalog 中的具体表或路径。  
    - **表/视图写入**：验证目标表存在且可写（如检查权限、表是否为临时表）。  
    - **文件路径写入**：解析输出路径格式（如 `parquet`、`csv`），验证路径合法性。  

2. **验证列兼容性**  
  - **列名与类型匹配**：确保待插入的列与目标表的列在名称、顺序和数据类型上兼容（允许隐式类型转换）。  
  - **动态分区处理**：若目标表为分区表，解析分区列并验证分区策略（静态分区或动态分区）。  

3. **处理隐式列映射**  
  - **按名称匹配**：当未显式指定列时，按列名自动匹配目标表的列（需完全一致）。  
  - **按顺序匹配**：当指定列列表（如 `INSERT INTO t (a, b)`），按顺序映射到目标表的列。  


#### ExtractWindowExpressions

专门用于从逻辑计划中提取窗口函数（如 `ROW_NUMBER() OVER`、`SUM() OVER`），并将其转换为独立的 `Window` 逻辑节点。

其核心目标是分离窗口函数与普通表达式，确保窗口操作的语义正确性，并为后续优化和执行提供明确的结构化计划。

1. **识别与提取窗口函数**  

  - **遍历表达式树**：扫描逻辑计划中的表达式，识别所有窗口函数（如 `Rank`、`AggregateExpression` + `WindowSpec`）。  
  - **分离普通表达式**：将窗口函数从 `Project` 或 `Aggregate` 节点中剥离，避免与普通聚合或列计算混淆。  

2. **生成 `Window` 逻辑节点**  

  - **结构化窗口操作**：将提取的窗口函数封装到独立的 `Window` 节点中，保留原查询逻辑。  

  - **绑定窗口规范**：将 `PARTITION BY`、`ORDER BY` 和 `WindowFrame` 绑定到对应的窗口函数。  

3. **处理复杂窗口场景**  
  
  - **多窗口定义支持**：允许同一查询中存在多个窗口定义（例如不同分区或排序规则）。  
  
  - **共享窗口规范优化**：合并重复的窗口定义（如相同 `PARTITION BY` 和 `ORDER BY` 的窗口函数复用同一逻辑）。  

#### GlobalAggregates

专门用于优化全局聚合操作（即没有 `GROUP BY` 的聚合查询，如 `SELECT SUM(value) FROM table`）。

其核心目标是将单阶段全局聚合拆分为 **局部聚合（Partial Aggregation）** 和 **全局聚合（Final Aggregation）**，以减少数据传输量并提升执行效率。

1. **拆分聚合阶段** 

  - **局部聚合（Partial）**：在数据分片（如 Map 阶段）预先计算部分聚合结果（例如 `SUM` 的中间值）。  

  - **全局聚合（Final）**：将局部聚合结果合并为最终结果（例如对多个分片的 `SUM` 中间值求和）。  

2. **优化全聚合查询**  

  适用于以下类型的查询：  
  
  - 无分组的聚合（如 `SELECT COUNT(*) FROM table`）。  
  
  - 仅含聚合函数的查询（如 `SELECT MAX(salary), AVG(age) FROM employees`）。  

3. **减少数据 Shuffle**  
  
  - 通过局部聚合降低需要 Shuffle 的数据量（例如减少传输数据行数）。  
  
  - 避免全量数据在全局阶段的重复计算。  

4. **支持多种聚合函数**  

  - **完全可分聚合**（如 `SUM`、`COUNT`）：可拆分为局部和全局阶段。  
  
  - **部分可分聚合**（如 `AVG`、`STDDEV`）：需保留中间状态（如 `SUM` + `COUNT`）。 

#### ResolveAggregateFunctions

专门用于解析和验证聚合函数（如 `SUM`、`COUNT`、`GROUP_CONCAT`）的合法性，并确保聚合操作的语义正确性。

其核心任务是处理聚合查询中的未解析表达式，验证聚合函数的位置、参数及分组规则的合法性。

1.**解析聚合函数表达式**  
  
  - **绑定聚合函数实现**：将逻辑计划中的 `UnresolvedFunction`（如 `SUM(col)`）转换为具体的聚合函数表达式（如 `Sum(col)`）。  
  
  - **验证聚合函数参数**：确保参数类型合法（例如 `SUM` 的参数必须是数值类型）。  

2. **验证聚合上下文**  

  - **禁止非法位置**：确保聚合函数仅出现在允许的上下文中（例如 `SELECT` 或 `HAVING` 子句，而非 `WHERE` 或 `JOIN` 条件）。  
  
  - **处理嵌套聚合**：禁止嵌套聚合（如 `SUM(MAX(col))`），除非显式允许（如某些数据库的扩展语法）。  

3. **处理分组键（Grouping Keys）**  

  - **验证分组表达式**：确保 `GROUP BY` 中的列或表达式是合法的（例如非聚合表达式）。  
  
  - **隐式分组处理**：当查询包含聚合函数但无 `GROUP BY` 时，隐式转换为全局聚合（`GROUP BY 1`）。  

4. **处理 `DISTINCT` 聚合**  

  - **解析 `DISTINCT` 语义**：将 `COUNT(DISTINCT col)` 转换为去重后的聚合逻辑。  

  - **验证 `DISTINCT` 支持性**：确认聚合函数是否支持 `DISTINCT`（例如 `SUM(DISTINCT col)` 合法，但 `MIN(DISTINCT col)` 可能冗余）。  


#### TimeWindowing

专门用于优化基于时间窗口的聚合操作（如 `GROUP BY window(time_column, "1 hour")`）。

其核心任务是将时间窗口表达式转换为高效的窗口计算逻辑，确保时间窗口的分组和聚合在底层执行计划中被正确优化。

1. **解析时间窗口定义**  
  
  - **时间列绑定**：识别时间窗口的基准列（如事件时间或处理时间列），并验证其数据类型为 `TimestampType`。  
  
  - **窗口参数解析**：解析窗口大小（如 `"5 minutes"`）、滑动步长（如 `"1 minute"`）和延迟阈值（用于处理迟到数据）。  

2.  **生成窗口元数据**  

  - **窗口开始/结束时间**：将时间窗口表达式转换为明确的窗口起始（`window_start`）和结束（`window_end`）列。  

  - **窗口合并优化**：合并相邻或重叠的时间窗口，减少重复计算（适用于批处理场景）。  

3.  **处理迟到数据**  

  - **水印（Watermark）集成**：结合水印机制过滤超出容忍阈值的迟到数据，避免状态无限增长（流处理场景）。  

4. **优化窗口聚合逻辑**  

  - **窗口分组下推**：将窗口定义下推到数据源（如 Kafka 时间戳提取），减少后续处理的数据量。  

  - **状态管理优化**：针对流处理，优化状态存储策略（如增量聚合）。  

#### SessionWindowing

专门用于优化基于会话窗口（Session Window）的聚合操作（如 `GROUP BY session_window(time_column, "5 minutes")`）。

其核心任务是动态划分会话窗口，处理事件间隔驱动的窗口逻辑，确保会话窗口的分组和聚合在批处理或流处理中高效执行。


1. **解析会话窗口定义** 

  - **事件时间列绑定**：识别会话窗口的基准列（如事件时间列），并验证其数据类型为 `TimestampType`。  

  - **会话间隔参数解析**：解析会话间隔参数（如 `"5 minutes"`），确定窗口动态闭合的静默时间阈值。  

2. **动态划分会话窗口**  

  - **窗口边界计算**：根据事件时间及间隔参数，动态合并连续事件到同一会话窗口（若事件间隔 ≤ 静默时间，则合并；否则拆分）。  

  - **状态管理（流处理）**：跟踪每个键（如用户ID）的最后事件时间，决定是否开启新会话窗口。  

3. **生成窗口元数据**  

  - **窗口开始/结束时间**：生成会话窗口的起始（`session_start`）和结束（`session_end`）时间列。  

  - **窗口唯一标识**：为每个会话窗口生成唯一ID（适用于多会话合并场景）。  

4. **处理迟到数据与状态清理**  

  - **水印（Watermark）集成**：结合水印机制过滤超时数据，清理过期会话状态（流处理场景）。  

  - **容错与状态恢复**：确保会话状态可持久化，支持故障恢复（如使用检查点机制）。

#### ResolveInlineTables

```sql
SELECT * FROM VALUES (1, 'a'), (2, 'b') AS t(id, name)
```

专门用于解析查询中的内联表（Inline Table）或值列表（如 `VALUES` 子句）。

其核心任务是将未解析的内联表结构（如 `UnresolvedInlineTable`）转换为具体的逻辑计划节点（如 `LocalRelation`），并验证其数据的一致性和合法性。

1. **解析内联表结构**  

  - **绑定行数据**：将 `VALUES` 子句中的行数据（如 `(1, 'a'), (2, 'b')`）转换为内存中的 `LocalRelation` 节点。  
  
  - **分配列名**：为内联表的列生成默认列名（如 `col1`、`col2`）或使用显式别名（如 `VALUES ... AS t(a, b)`）。  

2. **验证数据一致性**

  - **列数一致性**：确保所有行的列数相同（例如，`VALUES (1), (2, 3)` 会因列数不同而报错）。  
  
  - **数据类型推断**：自动推断每列的数据类型，确保所有行的对应列类型兼容（例如 `INT` 和 `STRING` 无法隐式兼容）。  

3. **类型强制转换**  
  
  - **统一数据类型**：若某列存在不同类型但可转换的值（如 `1` 和 `1.5`），将其强制转换为共同类型（如 `DOUBLE`）。  

4. **处理别名与作用域**  

  - **解析列别名**：处理 `AS` 子句中的列别名（如 `VALUES (1, 'a') AS t(id, name)`）。  

  - **避免列名冲突**：确保别名在查询上下文中唯一（例如，避免重复列名）。 


#### ResolveLambdaVariables

```sql
SELECT transform(array_col, x -> x + 1) FROM table
```

专门用于解析 Lambda 表达式中的变量（如 `x -> x + 1`），确保 Lambda 变量在逻辑计划中正确绑定到其上下文，并验证其作用域和类型的合法性。

其核心任务是处理高阶函数（如 `transform`、`filter`）中的 Lambda 参数，将其转换为可执行的表达式。

1. **解析 Lambda 变量**  

  - **绑定变量到参数**：将未解析的 Lambda 变量（如 `UnresolvedNamedLambdaVariable`）绑定到高阶函数的参数（例如 `x` 对应 `array` 的元素）。  
  
  - **验证变量作用域**：确保 Lambda 变量仅在定义它们的 Lambda 表达式内部使用，防止变量逃逸到外部作用域。  

2. **处理嵌套 Lambda 表达式**  
  
  - **多层 Lambda 支持**：解析嵌套 Lambda 表达式（如 `x -> y -> x + y`），逐层绑定变量到对应参数。  

  - **变量名冲突处理**：当嵌套 Lambda 变量同名时，确保内层变量覆盖外层变量（按词法作用域）。  

3. **类型推断与验证**  
  
  - **类型一致性检查**：推断 Lambda 参数的类型（如 `x` 的类型由输入数组元素类型决定），并验证 Lambda 表达式返回类型是否符合预期。  
  
  - **隐式类型转换**：若 Lambda 表达式参数类型不一致但可转换（如 `INT` 转 `DOUBLE`），自动插入类型转换。  

4. **替换未解析节点**  
  
  - **生成 `NamedLambdaVariable`**：将 `UnresolvedNamedLambdaVariable` 替换为具体的 `NamedLambdaVariable`，携带类型和作用域信息。  

#### ResolveTimeZone

专门用于解析和统一时间戳相关的时区设置，确保时间戳类型的数据在计算、存储和展示时具有一致的时区语义。

其核心任务是处理时间戳类型的时区转换逻辑，避免因时区不一致导致的数据错误。

1. **统一时区设置** 

  - **会话时区绑定**：将未明确指定时区的时间戳绑定到 Spark 会话的默认时区（由 `spark.sql.session.timeZone` 配置）。  
  
  - **时区敏感操作**：处理时间戳函数（如 `from_utc_timestamp`、`to_utc_timestamp`）的时区参数，验证其合法性。  

2. **时间戳类型转换**  

  - **隐式类型转换**：在混合时区的时间戳运算中，自动转换为统一的时区（例如将 `TIMESTAMP_LTZ` 转为 `TIMESTAMP`）。  

  - **字面量解析**：将字符串字面量（如 `'2023-10-01 12:00:00'`）解析为带时区的时间戳类型。  

3. **验证时区一致性**  
  
  - **禁止混合时区操作**：若时间戳列或表达式来自不同时区，且无法隐式转换，抛出错误。  
  
  - **函数参数验证**：确保时间戳函数的时区参数合法（例如 `from_utc_timestamp` 的时区必须是字符串常量）。  

4. **处理夏令时（DST）**  

  - **时区偏移计算**：在涉及夏令时的时区（如 `America/New_York`）中，正确计算时间戳的 UTC 偏移。

#### ResolveRandomSeed

专门用于解析和处理随机数生成函数（如 `rand()`、`randn()`）的种子参数（`seed`）。

其核心任务是确保随机函数的种子在逻辑计划中正确绑定，并验证种子值的合法性，以保证随机操作的可重复性和一致性。

1. **解析种子参数**

  - **显式种子绑定**：解析用户指定的种子值（如 `rand(123)`），确保其类型为整数（`Long`）。  
  
  - **隐式种子生成**：若未指定种子（如 `rand()`），自动生成随机种子（通常基于 Spark 任务上下文）。  

2. **验证种子合法性**  
  
  - **种子类型检查**：确保种子参数为合法整数（例如，`rand("invalid")` 会因种子类型错误而失败）。  
  
  - **种子范围检查**：允许种子为任意 `Long` 值（无需特定范围）。  

3. **处理多随机函数调用**  
  
  - **种子唯一性**：为同一查询中的多个随机函数分配不同种子（若未显式指定），避免随机结果重复。  
  
  - **种子传播**：在分布式计算中，确保同一分区的相同种子生成相同的随机序列。  

4. **保证可重复性**  
  
   - **固定种子优化**：在需要可重复结果的场景（如测试），显式种子确保多次运行结果一致。

#### ResolveBinaryArithmetic

专门用于解析和验证二元算术表达式（如 `+`、`-`、`*`、`/`）的类型兼容性，并进行隐式类型转换以确保运算的合法性。

其核心任务是确保操作数类型兼容，并在必要时自动插入类型转换（如 `CAST`），避免因类型不匹配导致运行时错误。

1. **类型兼容性验证** 

  - **操作数类型检查**：验证二元算术表达式（如 `a + b`）的左右操作数类型是否兼容。  
    - 允许类型组合：如 `INT + DOUBLE`、`DECIMAL + LONG`。  
    - 禁止类型组合：如 `STRING + INT`、`BOOLEAN / DATE`。  

2. **隐式类型转换**  
  - **自动插入 `CAST`**：若操作数类型不匹配但可隐式转换，自动插入类型转换表达式。  
    - 例如：`INT` ➔ `DOUBLE`、`LONG` ➔ `DECIMAL`。  
    - 转换优先级：根据 Spark 的类型提升规则（如 `DOUBLE` 优先级高于 `INT`）。  

3. **结果类型推断**

  - **推断表达式结果类型**：根据操作符和转换后的操作数类型确定结果类型。  
    - 例如：`INT + DOUBLE` ➔ `DOUBLE`、`DECIMAL * DECIMAL` ➔ `DECIMAL`。  

#### ResolveUnion

专门用于解析和验证 `UNION` 操作的合法性，确保多个子查询的联合操作在列数量、类型和名称上兼容。

其核心任务是解决 `UNION` 的结构一致性，并在必要时插入隐式类型转换或列别名，以生成统一的逻辑计划。


1. **列数量一致性验证** 

  - **检查子查询列数**：确保所有 `UNION` 的子查询具有相同的列数，否则抛出分析异常。

2. **列类型兼容性验证**

  - **隐式类型转换**：若对应列类型不兼容但可转换（如 `INT ➔ DOUBLE`），自动插入 CAST 表达式。
  
  ```sql
  SELECT age (INT) FROM users
  UNION
  SELECT height (DOUBLE) FROM employees  -- 自动转换为 DOUBLE
  ```

  - **类型冲突报错**：若类型无法转换（如 STRING ➔ DATE），终止解析并报错。

3. **列名称统一** : 以首个查询列名为准：统一 `UNION` 结果集的列名，后续子查询的列名被忽略（但需保持列顺序一致）

4. 处理嵌套 `UNION` : 递归解析嵌套结构：支持多层嵌套的 `UNION` 操作，逐层验证列对齐规则。

5. 消除重复 `UNION` 节点 : 合并相邻 `UNION`：优化逻辑计划，减少冗余的 `UNION` 节点（如`UNION(UNION(a, b), c) ➔ UNION(a, b, c)`）。


#### RewriteDeleteFromTable

专用于优化 `DELETE FROM` 语句的逻辑计划，将其转换为高效的物理操作（如数据重写、过滤、分区删除等）。

其核心目标是保证删除操作的正确性，同时最小化I/O和计算开销，尤其在处理大规模数据集时提升性能。

1. **逻辑删除转物理重写**  
  
  - **文件级删除优化**：对于基于文件的数据源（如 Parquet、Delta Lake），将 `DELETE` 转换为数据重写操作（读取原始数据 → 过滤目标行 → 写新文件 → 替换元数据）。  
  
  - **分区删除**：若删除条件匹配分区键，直接删除对应分区目录（Hive表场景）。  

2. **谓词下推与过滤优化**  
  
  - **提前过滤数据**：将 `WHERE` 条件下推到数据扫描阶段，减少需处理的数据量。  
  
  - **索引利用（如 Delta Lake）**：利用数据统计信息（如 Min/Max、布隆过滤器）跳过无关文件。  

3. **处理事务性删除**  

  - **ACID 事务支持**：在支持事务的数据源（如 Delta Lake）中，将 `DELETE` 转换为事务日志（Delta Log）的原子操作。  

  - **版本回滚**：生成数据版本快照，支持 `DELETE` 操作的撤销（Time Travel）。  

4. **动态覆盖优化**  

  - **动态分区覆盖**：仅重写受影响的分区文件，而非全表覆盖。  
  
  - **小文件合并**：在删除后触发小文件合并（如通过 `OPTIMIZE` 命令）。  


#### typeCoercionRules

`spark.sql.ansi.enabled`


| **场景**                | **默认模式**                  | **ANSI 模式**                     |
|-------------------------|-------------------------------|-----------------------------------|
| 字符串转数值            | 非法字符串返回 `NULL`          | 抛出 `NumberFormatException`      |
| 浮点数转整数            | 隐式截断（如 `3.14` → `3`）   | 必须显式 `CAST`                   |
| 宽松时间格式解析         | 自动解析（如 `20231001`）      | 仅支持严格格式                    |
| 布尔类型转换             | 允许 `1 = TRUE`                | 禁止隐式转换                      |

##### AnsiTypeCoercion.typeCoercionRules

其核心目标是 **遵循 ANSI SQL 标准**，通过更严格的类型检查来限制隐式类型转换，避免潜在的数据精度丢失或意外行为。

1. **禁用不安全的隐式转换**  
   - **字符串 ➔ 数值类型**：仅在字符串为合法数值时允许转换（否则抛出错误）。  
     ```sql
     SELECT CAST('123' AS INT)        -- 允许（ANSI 模式）
     SELECT 'abc' + 1                 -- 报错（默认模式可能返回 NULL）
     ```
   - **浮点数 ➔ 整数**：禁止隐式截断（需显式 `CAST`）。  
     ```sql
     SELECT 3.14::INT                 -- 报错（ANSI 模式）
     SELECT CAST(3.14 AS INT)           -- 显式允许（结果为 3）
     ```

2. **严格的时间类型转换**  
   - **字符串 ➔ 时间戳**：仅支持严格格式（如 `yyyy-MM-dd HH:mm:ss`），宽松格式（如 `20231001`）需显式转换。  
     ```sql
     SELECT '2023-10-01'::DATE        -- 允许
     SELECT '20231001'::DATE           -- 报错（需 `TO_DATE` 指定格式）
     ```

3. **禁止布尔类型隐式转换**  
   - **数值/字符串 ➔ BOOLEAN**：必须显式转换。  
     ```sql
     SELECT 1 = TRUE                    -- 报错（ANSI 模式）
     SELECT CAST(1 AS BOOLEAN)         -- 显式允许
     ```

4. **联合查询类型对齐更严格**  
   - **UNION 类型不兼容时报错**：默认模式可能隐式转换，而 ANSI 模式直接报错。  
     ```sql
     SELECT 1 AS col UNION SELECT 'a'  -- 报错（ANSI 模式）
     ```

##### TypeCoercion.typeCoercionRules

其核心目标是 **隐式地解决表达式中的类型冲突**，通过插入类型转换（如 `CAST`）使操作合法化，同时尽可能减少用户显式转换的工作量。

1. **隐式类型转换**  

   - **数值运算兼容性**：自动将低精度类型提升为高精度类型（如 `INT ➔ DOUBLE`）。  
     ```sql
     SELECT 1 + 2.5  -- 1 (INT) 隐式转为 1.0 (DOUBLE)
     ```
   - **字符串与数值混合运算**：将字符串隐式转换为数值（若字符串可解析为数值）。  
     ```sql
     SELECT '100' * 3  -- 字符串 '100' 转为 100 (INT)
     ```
   - **时间类型兼容性**：允许宽松的时间格式隐式转换（如 `20231001` ➔ `DATE`）。  
     ```sql
     SELECT date_col > '20231001'  -- 字符串转为 DATE 类型
     ```

2. **联合查询类型对齐**  
   - 在 `UNION` 操作中，自动对齐子查询的列类型（按首个查询的列类型转换）。  
     ```sql
     SELECT 1 AS col UNION SELECT 2.5  -- 2.5 (DOUBLE) 转为 1.0 (DOUBLE)
     ```

3. **函数参数适配**  
   - 自动转换函数参数类型以满足签名要求。  
     ```sql
     SELECT concat('id:', 100)  -- 100 (INT) 隐式转为 '100' (STRING)
     ```

4. **布尔类型宽松处理**  
   - 允许非布尔类型在布尔上下文中隐式转换（如 `0` 和 `1` 作为布尔值）。  
     ```sql
     SELECT IF(1, 'true', 'false')  -- 1 视为 TRUE
     ```

#### ResolveWithCTE

专门用于解析查询中的 **公共表表达式（CTE）**（即 `WITH` 子句定义的临时表）。

其核心任务是确保 CTE 的定义在逻辑计划中正确绑定，解决 CTE 的作用域、递归引用及名称冲突问题，从而生成可执行的逻辑计划。

1. **解析 CTE 定义**  

  - **绑定 CTE 名称**：将 `WITH` 子句中定义的临时表名称（如 `cte_name`）绑定到其对应的子查询逻辑计划。  

  - **作用域管理**：确保 CTE 仅在定义它的查询或后续 CTE 中可见（例如，CTE 不能跨 `UNION` 分支引用）。  

2. **处理递归 CTE**  

  - **递归引用验证**：若使用 `WITH RECURSIVE`，验证递归终止条件并防止无限循环。  

  - **生成迭代逻辑**：将递归 CTE 转换为迭代执行计划（Spark 3.0+ 支持有限递归查询）。  

3. **名称冲突处理**  
  - **禁止重复定义**：同一作用域内不允许重复的 CTE 名称。  

  - **优先使用 CTE**：当 CTE 名称与物理表名冲突时，优先解析为 CTE 定义。  

4. **逻辑计划替换**  

  - **替换未解析节点**：将逻辑计划中的 `UnresolvedWith` 节点替换为具体的 `SubqueryAlias` 节点。  

#### FindDataSourceTable

专门用于解析查询中引用的 **数据源表**（如 Hive 表、Parquet 文件、JDBC 表等），将其绑定到具体的表元数据或外部数据源。

其核心任务是将逻辑计划中的未解析表名（`UnresolvedRelation`）转换为指向实际数据源的 `LogicalRelation` 节点，确保后续操作可以访问正确的表结构和数据。

1. **解析表名到数据源**  

  - **Catalog 查询**：通过 Spark 的 `SessionCatalog` 查找表名对应的元数据（Hive 表、临时视图等）。  
  
  - **外部数据源加载**：若表名未注册到 Catalog，尝试通过 `DataSource` API 从文件路径、JDBC 等外部数据源加载表结构。  

2. **绑定表结构（Schema）** 

  - **Schema 推断**：对文件类数据源（如 Parquet、CSV），自动推断列名和类型。  

  - **显式 Schema 定义**：若用户通过 `schema` 参数指定表结构，直接应用该 Schema。  

3. **处理不同数据源类型**  

  - **Hive 表**：从 Hive Metastore 获取表元数据（列、分区、存储格式等）。  
  
  - **文件表**：根据路径和格式（如 `spark.read.parquet("/path")`）解析数据。  

  - **JDBC 表**：通过 JDBC 连接获取表的 Schema 信息。  

4. **(4) 处理临时视图**  
  
  - **临时视图绑定**：若表名是临时视图（`createOrReplaceTempView` 创建），绑定到其对应的逻辑计划。  

#### ResolveSQLOnFile

专门用于解析直接在文件路径上执行的 SQL 查询（如 `SELECT * FROM parquet. '/path/to/file'`），将其转换为可执行的逻辑计划。

其核心任务是将文件路径、格式及用户指定的选项绑定到具体的文件数据源，并推断或应用表结构（Schema），最终生成 `LogicalRelation` 节点，

使 Spark 能够正确读取和处理文件数据。

1. **解析文件路径与格式**  

  - **识别文件格式**：通过 SQL 语句中的格式标识符（如 `parquet.`、`csv.`、`json.`）确定文件类型。  

   ```sql
   SELECT * FROM csv.`/data/file.csv`  -- 解析为 CSV 格式
   ```

2.  **Schema 处理**

  - **自动推断 Schema**：对无 Schema 定义的文件（如 CSV、JSON），根据文件内容推断列名和类型。
  - **应用用户定义 Schema**：若用户通过 schema 参数或 DDL 显式定义列结构，直接应用该 Schema。

3. **文件选项解析**

  - 读取选项注入：解析用户指定的文件读取选项（如 CSV 的 header、delimiter），传递给底层数据源。

  ```sql
  SELECT * FROM csv.`/data/file.csv` OPTIONS (header 'true', delimiter '|')
  ```

4. **分区发现**

  - **自动识别分区列**：若文件路径是分区目录结构（如 `/data/date=2023-10-01`），提取分区列（date）并合并到表 Schema

#### FallBackFileSourceV2

用于在特定条件下将 **文件数据源 V2（File Data Source V2）** 的操作回退到 **V1 实现**。

其核心目标是确保 Spark 在尝试使用 V2 API 时，若遇到不支持的场景或配置，能够无缝降级到稳定且兼容的 V1 数据源实现，从而保障查询的稳定性和兼容性。

1. **V2 到 V1 的回退机制**  

  - **检测 V2 数据源的限制**：当 V2 数据源无法处理某些操作（如特定谓词下推、分区处理）时，自动触发回退。  

  - **配置驱动回退**：若用户显式配置使用 V1 数据源（如 `spark.sql.sources.useV1SourceList=parquet`），强制回退。  

2. **逻辑计划重写**

  - **替换数据源节点**：将逻辑计划中的 `FileTable`（V2）节点替换为 `HadoopFsRelation`（V1）节点。  

  - **保留原始语义**：确保回退后的逻辑计划与原始查询语义一致，避免结果差异。  

3. **(3) 错误处理与兼容性**  
  
  - **处理 V2 未实现的特性**：如 V2 数据源未支持某些文件格式或优化规则时，回退到 V1。  

  - **异常降级**：当 V2 数据源抛出未处理异常时，通过回退到 V1 避免作业失败。  

#### ResolveEncodersInScalaAgg

专门用于解析 Scala 自定义聚合函数（如 `TypedImperativeAggregate`）中的 **编码器（Encoder）**，确保聚合操作的中间状态和结果能够正确序列化/反序列化。

其核心任务是绑定聚合函数的输入/输出类型到合适的编码器，并验证类型兼容性，避免因编码器缺失或类型错误导致的运行时失败。

1. **编码器绑定**  
   - 为自定义 Scala 聚合函数（如 `Aggregator` 实现类）解析输入、中间状态（`Buffer`）和输出的编码器。  
   - 若用户未显式指定编码器，根据聚合类型自动推断（如 `IntType` ➔ `IntEncoder`）。

2. **类型兼容性验证**  
   - 检查聚合函数声明类型与实际数据类型的兼容性（如 `Aggregator[IN, BUF, OUT]` 的 `IN` 是否与输入列匹配）。  
   - 确保中间状态类型（`BUF`）可被 Spark 序列化（如支持 `Product` 类型或基本类型）。

3. **处理复杂类型**  
   - 支持嵌套类型（如 `case class`、`Array[StructType]`）的编码器解析。  
   - 处理泛型聚合函数的类型擦除问题（如通过 `TypeTag` 捕获实际类型）。

#### ResolveSessionCatalog

其核心任务是在逻辑计划中解析并绑定与当前 `SparkSession` 关联的元数据对象（如表、视图、函数等）。

它确保对数据库、表或视图的引用正确绑定到当前会话的目录（`SessionCatalog`）中的元数据，从而保证后续优化和执行能正确访问数据源的 Schema 和属性。

1. **解析多级标识符**
  
   - **处理 `database.table` 格式的引用**：验证数据库是否存在，并解析到具体的表对象。  

   - **临时视图优先**：若表名与临时视图冲突，优先使用临时视图而非持久化表。  

   ```sql
   SELECT * FROM default.employees  -- 解析为 Hive 库 `default` 的表 `employees`
   ```

2. **元数据绑定**

  - 将 `UnresolvedRelation` 转换为 `LogicalRelation`：

  例如，从逻辑计划中的符号 `UnresolvedRelation('employees')` 绑定到 `LogicalRelation`（链接到 Hive/Parquet表或临时视图）。

  - 验证对象存在性：若表/视图未注册到 `SessionCatalog`，抛出 AnalysisException。

3. **处理临时对象**

  - **临时视图解析**：解析 `CREATE TEMP VIEW` 创建的视图。
  
  - **全局临时视图隔离**：确保 `CREATE GLOBAL TEMP VIEW` 的视图在跨会话时可见性正确（通过 global_temp 库限定）。

4. **路径解析与多 Catalog 环境**

  - **跨 Catalog 访问的支持**：在 Spark 3.x 后，支持 `catalog.database.table` 的多 Catalog 行为（需结合 CatalogManager）。

  - **默认 Catalog 选择**：若未显式指定 Catalog，使用 SessionCatalog 的默认设置（通常是 hive）。

#### ResolveWriteToStream

专门用于解析和验证流式查询的写入操作（即 `Dataset.writeStream`），确保流式数据可正确写入目标 Sink。

其核心任务是识别逻辑计划中的流式写入需求，验证 Sink 的兼容性，并生成优化的物理写入计划。

1. **解析流式 Sink**  

  - **绑定 Sink 类型**：将逻辑计划中的 `StreamingWrite` 节点转换为具体 Sink 的实现（如 `KafkaSink`、`FileSink` 或 `ForeachBatchSink`）。  
  
  - **验证 Sink 可用性**：确保选择的 Sink 支持流式写入（例如，JDBC Sink 可能仅支持批处理）。  

2. **验证输出模式（Output Mode）**  
  
  - **检查模式与查询兼容性**：  
    - `Append` 模式：要求查询不包含聚合或窗口操作。  
    - `Complete`/`Update` 模式：验证是否支持聚合操作。  
  
  - **水印（Watermark）与事件时间**：若查询定义了事件时间，确保 Sink 支持基于水印的输出。  

3. **处理检查点（Checkpoint）与容错**  

  - **路径验证**：若启用检查点，验证检查点路径是否合法且可访问。  
  - **容错语义**：保证 Sink 支持至少一次（At-Least-Once）写入语义。  

4. **生成物理写入计划**  

  - **物理计划转换**：将逻辑写入节点转换为对应的 `StreamExecution` 物理节点（如 `MicroBatchExecution`）。  

  - **注入触发器（Trigger）**：根据用户指定的触发策略（如 `ProcessingTime`、`Once`）设置执行周期。  


#### customResolutionRules(用户自定义逻辑)

用户可以在 `SparkSessionExtensions` 自动注入处理逻辑

```scala
class PaimonStatisticsExtensions extends (SparkSessionExtensions => Unit){
  override def apply(extensions: SparkSessionExtensions): Unit = {
    extensions.injectResolutionRule(spark => PaimonStatisticsAnalysis(spark))
  }
}

case class PaimonStatisticsAnalysis(session: SparkSession) extends Rule[LogicalPlan] {
  override def apply(plan: LogicalPlan): LogicalPlan = {
    plan.resolveOperatorsDown {
      case prePlan@InsertIntoStatement(DataSourceV2Relation(_, _, Some(catalog), _, _),
                                        _, _, subQuery, _, _)
        if catalog.name == "paimon" && !subQuery.isInstanceOf[InsertPaimonStatistics] =>

        prePlan.copy(query = InsertPaimonStatistics(subQuery))
    }
  }
}
```

---

### Remove TempResolvedColumn (Once)

#### RemoveTempResolvedColumn

其核心功能是 **清理逻辑计划中临时生成的、已解析的中间列**，以消除冗余计算并优化执行计划。

1. **清理临时列**  
  移除在解析过程中生成的临时列（如处理复杂表达式、嵌套字段时生成的中间列），避免它们在最终执行计划中保留。
2. **优化 Project 节点**  
  修剪 `Project` 操作中的冗余输出列（如未被后续操作引用的中间列）。
3. **消除冗余计算**  
  减少不必要的数据传输和存储，提升查询性能。

---

### Apply Char Padding (Once)

#### ApplyCharTypePadding

`ApplyCharTypePadding` 是 Spark SQL **Analyzer 阶段** 的关键规则，专门用于处理 **定长字符类型（`CHAR`）的填充逻辑**，确保在比较、计算或存储时自动补齐空格，以满足 SQL 标准对 `CHAR` 类型的语义要求。其核心功能如下：

1. **自动填充空格**  
  将 `CHAR(N)` 类型的列或字面值填充到定义的长度 `N`（不足部分用空格补齐）。  
  ```sql
  -- 假设 col 类型为 CHAR(5)
  SELECT col FROM t WHERE col = 'abc'  -- 'abc' 会填充为 'abc  ' (5字符)
  ```

2. **统一比较语义**
  确保 `CHAR` 类型比较时基于填充后的值进行，避免因长度差异导致错误结果。
  ```sql
  -- 即使实际存储为 'abc', 比较时视为 'abc  '
  SELECT * FROM t WHERE char_col = 'abc  '
  ```

3. **兼容 ANSI SQL 标准**
  严格遵循 ANSI SQL 对 CHAR 类型的定义，避免隐式截断或未填充导致的语义歧义。

---

### Post-Hoc Resolution (Once)

#### ResolveCommandsWithIfExists

`ResolveCommandsWithIfExists` 是 Spark SQL **Analyzer 阶段** 的关键规则，专门用于解析和处理 **DDL/DML 命令中的 `IF EXISTS` 或 `IF NOT EXISTS` 子句**，确保在存在性条件明确的情况下正确执行命令（如 `DROP TABLE`、`ALTER VIEW` 等）且避免抛出冗余错误。

其核心目标是实现对数据库对象（表、视图、函数等）的**条件化操作**。

1. **解析条件子句**  
  识别并处理 SQL 命令中的 `IF EXISTS` 或 `IF NOT EXISTS` 修饰符，将其逻辑绑定到操作中。例如：
  ```sql
  DROP TABLE IF EXISTS table_name;   -- 仅当表存在时删除
  CREATE TABLE IF NOT EXISTS table_name ...; -- 仅当表不存在时创建
  ```

2. **存在性验证整合**
  将对象的存在性检查（如元数据查询）与命令操作合并到逻辑计划中，生成原子性操作逻辑。例如，在 `DROP VIEW IF EXISTS` 中：
  - 先检查视图是否存在（通过查询 Catalog）。
  - 仅在存在时执行删除动作，否则跳过。

3. **错误抑制**
  根据条件（如 `IF EXISTS`）自动忽略特定错误场景（如删除不存在的对象），避免运行时抛出 `AnalysisException`

4. **统一 SQL 兼容性**
  确保 Spark SQL 处理条件化 `DDL/DML` 时兼容标准 SQL 语义（如 MySQL、PostgreSQL 行为）。

#### DetectAmbiguousSelfJoin

用于 **检测自连接（Self-Join）中潜在的列引用歧义**，避免因未明确限定列来源而导致的逻辑错误或结果不可预测。其核心目标是确保自连接操作中的列引用具有明确的上下文语义。

1. **检测歧义列引用**  
  当同一张表以不同别名被多次引用（如自连接）时，若查询中引用的列未通过别名明确限定来源，则抛出 `AnalysisException`。  
  ```sql
  -- 示例：自连接未限定列来源
  SELECT id FROM t AS t1 JOIN t AS t2 ON t1.id = t2.parent_id
  WHERE id > 10  -- 错误！无法确定 id 属于 t1 还是 t2
  ```

2. **强制列完全限定**
  在自连接场景中，要求所有列引用必须通过别名（如 t1.id）或表名（如 t.id）明确指定来源表实例。

3. **防止隐式错误匹配**
  避免因列名相同但实际来源不同（如多版本表、Schema 演化场景）导致的错误数据匹配。


#### PreprocessTableCreation

用于 **在表创建（`CREATE TABLE`）前执行预处理**，确保表定义（如 Schema、存储属性、分区规则等）的合法性和一致性，并转换为标准化的逻辑计划。其核心目标是解决外部数据源兼容性、属性统一化及元数据预校验问题。

1. **存储格式与数据源适配**  
   - 根据 `USING` 子句或 `STORED AS` 推断数据源类型（如 Parquet、CSV、Hive）。
   - 默认格式处理（如未指定 `USING` 时，使用 `spark.sql.sources.default` 配置）。

2. **表属性标准化**  
   - 转换 Hive 风格属性（如 `TBLPROPERTIES`、`COMMENT`）为 Spark 内部元数据格式。
   - 处理 `LOCATION` 与数据库默认路径的冲突（外部表 vs 托管表）。

3. **分区与分桶规范校验**  
   - 验证分区列是否存在于 Schema 中。
   - 分桶列类型检查（仅允许可哈希类型），并生成分桶 ID 表达式。

4. **Schema 预处理**  
   - 合并用户显式定义的 Schema 与数据源推断的 Schema（如 `CREATE TABLE ... AS SELECT`）。
   - 处理 Hive 风格的数据类型转换为 Spark 类型（如 `STRING` ➔ `StringType`）。

5. **存在性条件处理**  
   - 解析 `IF NOT EXISTS`，结合 `SessionCatalog` 检查表是否已存在，避免重复创建。

#### PreprocessTableInsertion

用于 **预处理数据插入操作（如 `INSERT INTO`、`INSERT OVERWRITE`）**，确保插入逻辑与目标表的存储格式、分区策略、数据源特性等兼容，并生成标准化的写入计划。其核心目标是解决插入操作中的元数据适配、动态分区处理及写入优化问题。

1. **元数据适配与验证**  
   - 验证插入数据的 Schema 是否与目标表兼容（列名、类型、顺序）。  
   - 处理隐式类型转换（如插入 `INT` 到 `BIGINT` 列，需在兼容模式下自动转换）。

2. **分区处理**  
   - **静态分区插入**：解析分区键的显式值（如 `PARTITION (dt='2023-10-01')`）。  
   - **动态分区插入**：根据数据自动推断分区值，生成动态分区写入策略。  
   - 校验分区列是否存在且类型匹配，避免写入非法分区路径。

3. **存储格式与数据源适配**  
   - 根据目标表的 `USING` 或 `STORED AS` 定义，选择对应的数据源写入器（如 `ParquetFileFormat`）。  
   - 处理外部表（`EXTERNAL`）的路径权限及存在性检查。

4. **写入模式处理**  
   - 解析 `INSERT INTO`（追加）和 `INSERT OVERWRITE`（覆盖）的语义差异。  
   - 在覆盖模式下，清理目标分区或目录（需结合 `spark.sql.sources.partitionOverwriteMode` 配置）。

5. **优化写入计划**  
   - 合并小文件（通过 `coalesce` 或 `repartition` 优化文件数量）。  
   - 转换为物理写入节点（如 `InsertIntoHadoopFsRelationCommand`）。

#### DataSourceAnalysis

用于 **解析和验证与外部数据源（如 Parquet、JDBC、CSV）相关的逻辑计划节点**，确保数据源的元数据、存储格式及访问参数符合预期，并将其转换为可直接执行的物理操作。其核心目标是解决数据源读写操作的类型适配、Schema 推断及执行计划生成问题。

```sql 
-- 读取外部数据源
SELECT * FROM parquet.`/path/to/data`  -- 解析为 Parquet 数据源

-- 处理 JDBC 和 Hive 数据源的适配
INSERT INTO jdbc_table SELECT * FROM hive_table

-- 未指定 Schema，需推断 CSV 列名和类型
CREATE TABLE t USING csv OPTIONS (path '/data/csv') 
```

1. **数据源关系解析**  
   - 将 `UnresolvedRelation`（未解析的表/视图）转换为具体数据源的 `LogicalRelation` 节点，绑定数据源的 Schema 和元数据。  
   - **示例**：`SELECT * FROM t` ➔ 解析 `t` 为 `LogicalRelation(ParquetFileFormat, path:/data/t)`。

2. **读写操作适配**  
   - **读操作**：解析 `DataSourceV2` 表的扫描逻辑，生成 `BatchScan` 或 `MicroBatchScan` 节点。  
   - **写操作**：将 `InsertIntoStatement` 转换为具体数据源的写入命令（如 `WriteToDataSourceV2`）。

3. **Schema 推断与验证**  
   - **显式 Schema**：直接使用用户提供的 Schema 进行数据解析。  
   - **隐式推断**：若未指定 Schema，根据数据源格式（如 JSON、CSV）自动推断字段名和类型。  
   - **兼容性检查**：验证查询 Schema 与数据源 Schema 的兼容性（如列缺失、类型不匹配）。

4. **数据源属性处理**  
   - 解析数据源选项（如 `OPTIONS`、`path`、`compression`），验证其合法性并传递给底层读写器。  
   - **示例**：`df.write.option("header", "true").csv(...)` ➔ 设置 CSV 写入器的表头选项。

5. **分区与过滤下推优化**  
   - 将分区列和过滤条件（`WHERE` 子句）推送到数据源层，减少数据读取量。  
   - **示例**：`SELECT * FROM t WHERE dt='2023-10-01'` ➔ 仅读取 `dt=2023-10-01` 的分区目录。


#### ReplaceCharWithVarchar

**将逻辑计划中的定长字符类型（`CHAR(N)`）隐式转换为变长字符类型（`VARCHAR(N)` 或 `STRING`）**，旨在提升兼容性并减少定长类型处理带来的性能开销。

1. **类型语义统一化**  
  将 `CHAR(N)` 类型统一为 `VARCHAR(N)` 或 `STRING` 类型（取决于配置），简化后续处理逻辑。  
  **示例**：  
  ```sql
  CREATE TABLE t (c CHAR(10));  -- 实际解析为 c VARCHAR(10) 或 STRING
  ```

2. **避免隐式填充开销**
  
  `CHAR(N)` 在比较或存储时需填充空格以满足长度要求（如 `ApplyCharTypePadding` 规则），而 `VARCHAR` 仅保留实际内容。替换后可减少填充操作的计算和存储成本。

3. **兼容非标准场景**
  
  某些数据源或查询引擎（如旧版 Hive）不支持 `CHAR` 类型，统一转换为 `VARCHAR` 可避免兼容性问题。

#### customPostHocResolutionRules (用户自定义逻辑)

用户可以在 `SparkSessionExtensions` 自动注入处理逻辑

```scala
class PaimonStatisticsExtensions extends (SparkSessionExtensions => Unit){
  override def apply(extensions: SparkSessionExtensions): Unit = {
    extensions.injectPostHocResolutionRule(spark => PaimonStatisticsAnalysis(spark))
  }
}

case class PaimonStatisticsAnalysis(session: SparkSession) extends Rule[LogicalPlan] {
  override def apply(plan: LogicalPlan): LogicalPlan = {
    plan.resolveOperatorsDown {
      case prePlan@InsertIntoStatement(DataSourceV2Relation(_, _, Some(catalog), _, _),
                                        _, _, subQuery, _, _)
        if catalog.name == "paimon" && !subQuery.isInstanceOf[InsertPaimonStatistics] =>

        prePlan.copy(query = InsertPaimonStatistics(subQuery))
    }
  }
}
```

---

### Remove Unresolved Hints (Once)

#### ResolveHints.RemoveAllHints

**在解析并处理所有查询提示（Hints）后，从逻辑计划中彻底移除这些提示**，确保后续优化和执行阶段不受残留提示的干扰，保持逻辑计划的简洁性和正确性。

1. **清理已处理的提示**  
  将已成功解析且应用的提示（如 `BROADCASTJOIN`、`MERGE` 等）从逻辑计划中删除，避免冗余信息影响优化器决策。
  
2. **移除无效或未识别的提示**  
  若存在无法解析的提示（如拼写错误或 Spark 不支持的提示），统一移除以避免计划污染。

3. **保障逻辑计划纯净性**  
  确保优化器处理的逻辑计划中仅包含结构化操作（如 `Join`、`Filter`），不含原始提示符。

---

### Nondeterministic (Once)

#### PullOutNondeterministic


其核心功能是 **将逻辑计划中的非确定性表达式（Non-deterministic Expressions）提取到更高的作用域**，确保这些表达式仅计算一次，避免因重复执行导致结果不一致或性能损耗。

- **非确定性表达式隔离**  
  识别并提取如 `rand()`、`uuid()`、`current_timestamp()` 等非确定性表达式，将其提升至查询的最外层或公共祖先节点，确保在整个查询生命周期内仅计算一次。

- **保证结果一致性**  
  防止同一非确定性表达式在查询的不同位置多次执行产生不同结果（如 `SELECT rand(), rand()` 返回两个不同值）。

- **优化执行计划**  
  减少重复计算非确定性表达式的开销，提升查询性能。

---

### UDF (Once)

#### HandleNullInputsForUDF

用于 **处理用户定义函数（UDF）输入参数中的空值（`NULL`）**，确保在调用 UDF 时自动插入空值检查逻辑或类型转换，避免因未处理 `NULL` 导致的运行时错误或结果异常。

其核心目标是提升 UDF 的健壮性和与 SQL 语义的兼容性。

1. **空值安全调用**  
   - 当 UDF 参数可能为 `NULL` 时，自动包装输入参数为 `Coalesce` 或 `If` 表达式，防止 UDF 接收未预期的 `NULL`。  
   - **示例**：将 `udf(name)` 转换为 `If(IsNull(name), null, udf(Coalesce(name)))`。

2. **类型兼容性处理**  
   - 若 UDF 声明为非空类型参数，但输入可能为 `NULL`，自动插入类型转换（如 `CAST`）或抛出分析异常。

3. **保留 SQL 语义**  
   - 遵循 SQL 的 `NULL` 传播规则，当任意输入为 `NULL` 时，确保 UDF 返回 `NULL`（除非 UDF 显式处理 `NULL`）。

#### ResolveEncodersInUDF

专门用于 **解析用户定义函数（UDF）中涉及复杂类型（如自定义对象、集合类型）的编码器（Encoder）**，确保 UDF 的输入和输出类型能够被正确序列化/反序列化。

其核心目标是解决 UDF 与 Dataset API 集成时的类型编码兼容性问题。

1. **编码器绑定**  
   - 为 UDF 的输入参数和返回类型隐式推导或显式绑定对应的 Spark 编码器（`ExpressionEncoder`）。  
   - **示例**：UDF 参数为自定义 `case class` 时，自动生成该类的编码器。

2. **类型兼容性验证**  
   - 检查 UDF 签名中的类型是否支持 Spark 的编码机制（如非原生类型需有隐式 `Encoder` 可用）。  
   - 若类型无法编码，抛出 `AnalysisException`（如不支持的嵌套类型）。

3. **优化编码逻辑**  
   - 合并多层编码操作，避免冗余序列化（如 UDF 链式调用时复用中间编码结果）。  

4. **处理泛型类型**  
   - 通过类型擦除或 `TypeTag` 保留泛型信息，确保编码器正确匹配（如 `List[T]` 中的 `T` 需明确）。

---

### UpdateNullability (Once)

#### UpdateAttributeNullability

其核心功能是 **更新逻辑计划中属性的可空性（Nullability）信息**，确保每个表达式或列的 `nullable` 属性与其实际语义一致。

通过动态推断列的潜在空值可能性，优化器可以更准确地决策执行计划，避免因错误的可空性假设导致的数据不一致或性能损失。

1. **动态推断可空性**  
   - 根据数据源元数据、操作符语义及表达式逻辑，推断列或表达式是否可能包含 `NULL` 值。  
   - **示例**：  
     - `LEFT JOIN` 后右表列变为可空（即使原定义为非空）。  
     - `SELECT a + b` 中，若 `a` 或 `b` 可空，则结果列可空。

2. **更新逻辑计划属性**  
   - 修正逻辑计划节点（如 `Project`、`Aggregate`、`Join`）输出列的 `nullable` 属性。  
   - **示例**：聚合函数 `COUNT(*)` 的结果永远非空，而 `AVG(col)` 可能为 `NULL`（若输入全为空）。

3. **优化执行计划**  
   - 为后续优化规则（如谓词下推、常量折叠）提供准确的空值信息，避免无效优化。  
   - **示例**：若某列被推断为非空，优化器可安全应用 `IS NOT NULL` 过滤。

---

### Subquery (Once)

#### UpdateOuterReferences

`UpdateOuterReferences` 是 Spark SQL **Analyzer 阶段** 的关键规则，其核心功能是 **处理逻辑计划中嵌套查询或关联子查询对外部作用域列的引用（即外部引用，Outer References）**，确保这些引用能够正确绑定到外层查询的列，从而保障关联子查询的语义正确性和执行计划生成的准确性。

1. **解析外部引用**  
   识别子查询中引用的外层查询的列，并将其与外部作用域的对应属性绑定。

   **示例**：  
   ```sql
   SELECT * FROM outer_table o 
   WHERE EXISTS (SELECT 1 FROM inner_table i WHERE i.id = o.id)  -- o.id 是外部引用
   ```

2. **标记引用作用域**
  在逻辑计划中标记外部引用所属的作用域层级，防止内部查询错误地引用非预期外层列。

3. **生成关联逻辑节点**
  将隐式关联子查询转换为显式的逻辑计划节点（如 LateralJoin），明确依赖关系。

---

### Cleanup (FixedPoint)

#### CleanupAliases

其核心功能是 **清理逻辑计划中冗余的列或表达式别名（Alias）**，简化计划结构并提升后续优化规则和执行效率。

1. **消除冗余别名**  
   移除逻辑计划中不必要的别名（`Alias` 节点），这些别名通常由解析器生成或优化阶段引入，但对最终执行无实际意义。  
   **示例**：  
   ```sql
   SELECT col AS c FROM t  -- 解析后的逻辑计划可能保留别名 `c`
   ```
  优化后直接使用原始列名，避免冗余元数据传递。

2. **合并重复别名**
  若同一表达式被多次赋予相同别名，合并为一个别名节点，减少重复计算。

3. **优化聚合与投影** 
  在聚合（Aggregate）或投影（Project）操作中，清理未引用的别名，避免无效数据处理。

4. **提升可读性与性能**
  简化后的逻辑计划更简洁，减少优化器和执行引擎处理无关节点的开销。

---

### HandleAnalysisOnlyCommand (Once)

#### HandleAnalysisOnlyCommand

1. **拦截纯分析命令**  
   识别无需执行引擎处理的命令（如 `EXPLAIN`、`SHOW`、`DESCRIBE`），直接通过分析阶段完成操作。

2. **短路执行优化**  
   跳过优化器（Optimizer）和物理计划生成阶段，减少计算资源浪费。

3. **元数据快速访问**  
   直接通过 `SessionCatalog` 或外部 Catalog 获取表/列元数据，避免触发全表扫描或数据加载。

4. **支持交互式查询**  
   快速响应用户的元数据操作（如查看表结构），提升交互体验。

| **命令类型**       | **示例**                                 | **作用**                               |
|--------------------|-----------------------------------------|---------------------------------------|
| **元数据查询**     | `SHOW TABLES`、`DESCRIBE TABLE`         | 展示表/列定义、存储属性等。             |
| **执行计划解释**   | `EXPLAIN FORMATTED <QUERY>`             | 直接解析并返回逻辑/物理计划，不执行查询。 |
| **数据探查**       | `ANALYZE TABLE ... COMPUTE STATISTICS`  | 收集统计信息（如行数、列大小）并存储。    |
| **配置管理**       | `SET -v`、`SHOW DATABASES`              | 显示 Spark 配置参数或数据库列表。        |