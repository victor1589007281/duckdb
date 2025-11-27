# DuckDB Planner 和 Binder 模块

## 概述

Planner 和 Binder 是 DuckDB 计算层的核心组件，负责将 Parser 生成的抽象语法树（AST）转换为逻辑执行计划。Binder 负责名称解析和类型检查，Planner 负责生成逻辑算子树。

## 整体架构

```mermaid
graph TB
    subgraph "**Planner架构**"
        A["**SQLStatement<br/>(AST)**"]
        B["**Binder**"]
        C["**BoundStatement**"]
        D["**LogicalPlan**"]
    end
    
    subgraph "**Binder组件**"
        E["**BindContext**"]
        F["**ExpressionBinder**"]
        G["**TableBinder**"]
        H["**ColumnBinding**"]
    end
    
    subgraph "**LogicalOperator**"
        I["**LogicalGet**"]
        J["**LogicalFilter**"]
        K["**LogicalJoin**"]
        L["**LogicalAggregate**"]
        M["**LogicalProjection**"]
    end
    
    A --> B
    B --> C
    C --> D
    
    B --> E
    B --> F
    B --> G
    B --> H
    
    D --> I
    D --> J
    D --> K
    D --> L
    D --> M
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style F fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style G fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style H fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style I fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style J fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style K fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style L fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style M fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
```

## Binder 核心功能

```mermaid
classDiagram
    class Binder {
        **+ClientContext context**
        **+BindContext bind_context**
        **+CatalogSearchPath catalog_search_path**
        **+Bind(SQLStatement) BoundStatement**
        **+BindExpression() BoundExpression**
        **+BindTable() BoundTableRef**
    }
    
    class BindContext {
        **+map~string~ bindings**
        **+vector~TableBinding~ table_bindings**
        **+AddBinding()**
        **+GetBinding()**
    }
    
    class ExpressionBinder {
        **+Binder binder**
        **+BindContext context**
        **+Bind(ParsedExpression)**
        **+BindColumn()**
        **+BindFunction()**
    }
    
    class LogicalOperator {
        **+LogicalOperatorType type**
        **+vector~LogicalOperator~ children**
        **+vector~LogicalType~ types**
        **+ResolveTypes()**
    }
    
    Binder --> BindContext
    Binder --> ExpressionBinder
    Binder --> LogicalOperator
    
    style Binder fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style BindContext fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style ExpressionBinder fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style LogicalOperator fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
```

## 绑定流程

```mermaid
sequenceDiagram
    participant **Planner** as **Planner**
    participant **Binder** as **Binder**
    participant **Catalog** as **Catalog**
    participant **Logical** as **LogicalOperator**
    
    **Planner**->>**Binder**: **Bind(SQLStatement)**
    **Binder**->>**Binder**: **解析表引用**
    **Binder**->>**Catalog**: **查找表定义**
    **Catalog**-->>**Binder**: **返回TableEntry**
    
    **Binder**->>**Binder**: **绑定列引用**
    **Binder**->>**Binder**: **类型检查**
    **Binder**->>**Binder**: **绑定函数**
    **Binder**->>**Catalog**: **查找函数定义**
    **Catalog**-->>**Binder**: **返回FunctionEntry**
    
    **Binder**->>**Logical**: **创建逻辑算子**
    **Logical**-->>**Binder**: **LogicalOperator树**
    **Binder**-->>**Planner**: **返回BoundStatement**
```

## LogicalOperator 类型

```mermaid
graph TB
    subgraph "**扫描算子**"
        A["**LogicalGet<br/>(表扫描)**"]
        B["**LogicalTableFunction<br/>(表函数)**"]
    end
    
    subgraph "**连接算子**"
        C["**LogicalJoin<br/>(连接)**"]
        D["**LogicalCrossProduct<br/>(笛卡尔积)**"]
    end
    
    subgraph "**聚合算子**"
        E["**LogicalAggregate<br/>(聚合)**"]
        F["**LogicalWindow<br/>(窗口)**"]
    end
    
    subgraph "**过滤投影**"
        G["**LogicalFilter<br/>(过滤)**"]
        H["**LogicalProjection<br/>(投影)**"]
    end
    
    subgraph "**修改算子**"
        I["**LogicalInsert<br/>(插入)**"]
        J["**LogicalUpdate<br/>(更新)**"]
        K["**LogicalDelete<br/>(删除)**"]
    end
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style C fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style F fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style G fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style H fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style I fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style J fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style K fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
```

## 相关源码

- `src/planner/planner.cpp` - Planner主类
- `src/planner/binder.cpp` - Binder主类
- `src/planner/bind_context.cpp` - 绑定上下文
- `src/planner/expression_binder.cpp` - 表达式绑定器
- `src/planner/operator/` - 逻辑算子实现

