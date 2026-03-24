---
outline: deep
---

# dbt 中 Jinja 模板的使用：适当性与替代方案 {#dbt-jinja-templates}

## dbt 与 Jinja 模板 {#dbt-and-jinja}

[dbt（data build tool）](https://www.getdbt.com/) 是目前最流行的数据转换框架之一，它将 [Jinja](https://jinja.palletsprojects.com/) 模板引擎深度集成到 SQL 文件中。这意味着你的 `.sql` 文件实际上是 Jinja 模板，在执行前会先被编译成纯 SQL。

### 典型用法 {#typical-usage}

**模型配置（`config` 块）**

```sql
{{ config(
    materialized='incremental',
    unique_key='event_id',
    incremental_strategy='merge',
    schema='analytics'
) }}

SELECT
    event_id,
    user_id,
    event_time
FROM {{ source('raw', 'events') }}
{% if is_incremental() %}
WHERE event_time > (SELECT MAX(event_time) FROM {{ this }})
{% endif %}
```

**引用其他模型**

```sql
SELECT
    o.order_id,
    c.customer_name
FROM {{ ref('orders') }} o
JOIN {{ ref('customers') }} c ON o.customer_id = c.customer_id
```

**宏（Macros）**

```sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100.0)::NUMERIC(10, 2)
{% endmacro %}

SELECT
    order_id,
    {{ cents_to_dollars('amount_cents') }} AS amount_dollars
FROM {{ ref('orders') }}
```

---

## 这种方式合适吗？ {#is-it-appropriate}

### 优势 {#advantages}

**1. 解决 SQL 的核心痛点**

纯 SQL 缺乏抽象能力——没有函数、没有变量、没有条件逻辑。Jinja 填补了这一空白，让 SQL 具备了：

- **DRY（Don't Repeat Yourself）**：通过宏复用复杂逻辑，避免跨模型复制粘贴
- **条件逻辑**：`{% if is_incremental() %}` 等语句让增量加载无需维护两套 SQL
- **跨数据库兼容**：同一套宏可以针对不同数据库（BigQuery、Snowflake、Redshift 等）生成不同 SQL 方言

**2. `ref()` 和 `source()` 的价值**

`{{ ref('model_name') }}` 不只是字符串替换——它还向 dbt 声明了模型间的依赖关系，使得 dbt 能够：

- 自动构建 DAG（有向无环图）
- 按正确顺序执行模型
- 在 lineage graph 中可视化数据流向

这一特性是 dbt 最核心的价值之一，而它恰恰依赖 Jinja 实现。

**3. `config()` 块的必要性**

`{{ config(...) }}` 让模型的物化方式、分区、cluster 策略等配置紧贴 SQL 定义，避免配置与代码分离带来的维护负担。

### 劣势与风险 {#disadvantages-and-risks}

**1. SQL 文件不再是纯 SQL**

IDE 的 SQL 语法高亮、格式化、lint 工具往往无法正确处理 Jinja 语法。开发者需要额外的工具链支持（如 [dbt Power User](https://marketplace.visualstudio.com/items?itemName=innoverio.vscode-dbt-power-user) VS Code 插件）。

**2. 调试难度增加**

当 Jinja 模板生成错误的 SQL 时，需要使用 `dbt compile` 将模板编译为最终 SQL，才能定位问题。这增加了调试的间接性。

**3. 过度使用导致可读性下降**

Jinja 宏嵌套过深时，代码可读性会急剧下降。例如：

```sql
-- 难以阅读的过度封装示例
{{ generate_surrogate_key([
    dbt_utils.generate_surrogate_key(['user_id', 'session_id']),
    'event_type'
]) }}
```

**4. 学习曲线**

数据分析师通常熟悉 SQL，但不一定了解 Jinja 或模板编程概念。对于新成员，这会增加入门门槛。

---

## 是否有更好的方式？ {#are-there-better-alternatives}

### 方案一：将复杂逻辑移至宏或包 {#option-1-macros-and-packages}

对于频繁重复的逻辑，应优先使用宏封装，而非在模型中直接编写复杂 Jinja：

```sql
-- macros/incremental_filter.sql
{% macro incremental_filter(column) %}
{% if is_incremental() %}
WHERE {{ column }} > (SELECT MAX({{ column }}) FROM {{ this }})
{% endif %}
{% endmacro %}

-- models/events.sql
{{ config(materialized='incremental', unique_key='event_id') }}

SELECT event_id, user_id, event_time
FROM {{ source('raw', 'events') }}
{{ incremental_filter('event_time') }}
```

[dbt-utils](https://github.com/dbt-labs/dbt-utils) 和 [dbt-expectations](https://github.com/calogica/dbt-expectations) 等社区包提供了大量成熟宏，应优先使用，而非重复造轮子。

### 方案二：使用 `dbt_project.yml` 统一配置 {#option-2-project-level-config}

不是每个模型都需要在 SQL 文件内写 `config` 块。`dbt_project.yml` 支持为整个目录批量配置：

```yaml
# dbt_project.yml
models:
  my_project:
    staging:
      +materialized: view
      +schema: staging
    marts:
      +materialized: table
      +schema: marts
```

这样，绝大多数模型就不再需要 `{{ config(...) }}` 块，SQL 文件更接近纯 SQL，可读性更好。只有需要覆盖默认值的模型才单独在文件内声明 `config`。

### 方案三：Python 模型（dbt 1.3+）{#option-3-python-models}

dbt 1.3 起支持 Python 模型（通过 Snowpark 或 PySpark），可以将复杂的条件逻辑移至 Python：

```python
# models/python/my_model.py
def model(dbt, session):
    dbt.config(materialized="table")

    df = dbt.ref("source_model")
    # 用 Pandas/PySpark 实现复杂逻辑
    return df.filter(df["status"] == "active")
```

适用场景：需要机器学习、复杂统计运算、或无法用 SQL 优雅表达的逻辑。

### 方案四：预处理器或代码生成工具 {#option-4-preprocessors}

部分团队选择用 Python 脚本或 [SQLGlot](https://github.com/tobymao/sqlglot) 等工具在 dbt 之外生成 SQL，然后将纯 SQL 提交到 dbt。这种方式保持了 SQL 文件的纯净，但失去了 dbt Jinja 生态的便利。通常只在极少数场景下值得考虑。

---

## Mage-ai 的做法 {#mage-ai-approach}

[Mage-ai](https://www.mage.ai/) 是另一个现代数据管道框架，其设计哲学与 dbt 截然不同。

### Mage-ai 的核心模型 {#mage-ai-core-model}

Mage-ai 使用**原生 Python 函数**作为数据块（Block）的基本单元，而不是 SQL + Jinja：

```python
# 数据加载块（Data Loader）
@data_loader
def load_data_from_bigquery(*args, **kwargs) -> DataFrame:
    query = """
    SELECT user_id, event_time
    FROM `project.dataset.events`
    WHERE DATE(event_time) = CURRENT_DATE()
    """
    return BigQuery.with_config(ConfigFileLoader('io_config.yaml')).load(query)
```

```python
# 数据转换块（Transformer）
@transformer
def transform(df: DataFrame, *args, **kwargs) -> DataFrame:
    df['amount_dollars'] = df['amount_cents'] / 100.0
    return df[df['status'] == 'active']
```

### 两者的核心差异 {#key-differences}

| 维度 | dbt | Mage-ai |
|------|-----|---------|
| **主要语言** | SQL + Jinja 模板 | Python（原生） |
| **SQL 支持** | 核心，`.sql` 文件 | 支持，但通常内嵌于 Python |
| **模板/宏** | Jinja 模板引擎 | Python 函数，无模板层 |
| **DAG 定义** | 通过 `ref()` 隐式声明 | 通过 UI 或 YAML 显式定义 |
| **调试体验** | 需 `dbt compile` 查看编译后 SQL | 直接运行 Python，所见即所得 |
| **适用人群** | SQL 分析师、数据工程师 | Python 工程师、数据科学家 |
| **增量逻辑** | Jinja `{% if is_incremental() %}` | Python 条件判断 |
| **可读性** | SQL 中混入模板语法 | 纯 Python，无额外语法层 |
| **IDE 支持** | 需专用插件 | 标准 Python 工具链 |

### Mage-ai 中的 SQL 块 {#mage-ai-sql-blocks}

Mage-ai 也支持 SQL 块，但方式更简洁——不使用 Jinja，而是通过 UI 配置连接和参数：

```sql
-- Mage-ai 的 SQL 数据加载块
SELECT
    order_id,
    customer_id,
    amount
FROM orders
WHERE created_at >= '{{ start_date }}'  -- 由 Pipeline 运行时参数注入
```

Mage-ai 的模板变量（`{{ start_date }}`）只用于运行时参数传递，**不像 dbt 那样深度嵌入业务逻辑**。模型间的依赖和条件逻辑全部通过 Python 表达。

---

## 总结与建议 {#summary-and-recommendations}

### dbt Jinja 的使用原则 {#dbt-jinja-principles}

dbt 的 Jinja 集成**总体上是合适的**，但应遵循以下原则：

1. **`ref()` 和 `source()` 是核心，必须使用**——它们是 dbt 依赖管理的基础
2. **`config()` 块尽量统一到 `dbt_project.yml`**——减少每个 SQL 文件的模板噪音
3. **宏适度封装，避免嵌套过深**——以可读性为优先标准
4. **条件逻辑（如增量策略）优先通过增量模型标准模式实现**，而非自定义复杂 Jinja
5. **复杂业务逻辑优先考虑 Python 模型**（dbt 1.3+），保持 SQL 的简洁

### 如何选择 dbt vs Mage-ai {#dbt-vs-mage-ai}

- **选择 dbt**：团队以 SQL 为主，主要做数据建模和转换，需要强大的血缘追踪和文档生成能力
- **选择 Mage-ai**：团队以 Python 为主，需要复杂的编排逻辑、机器学习集成、或更灵活的管道控制

两者并不互斥——很多数据平台会将 dbt 用于数据建模层，将 Mage-ai 或 Airflow 用于上层编排。
