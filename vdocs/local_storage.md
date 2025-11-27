# DuckDB LocalStorage 模块

## 概述

LocalStorage 是 DuckDB 的事务级临时存储模块，用于存储事务中尚未提交的数据修改（插入、更新、删除）。每个事务都有自己的 LocalStorage，事务提交时才将数据合并到全局存储。

## 整体架构

```mermaid
graph TB
    subgraph "**LocalStorage架构**"
        A["**Transaction**"]
        B["**LocalStorage**"]
        C["**LocalTableStorage**"]
        D["**RowGroupCollection**"]
        E["**VersionManager**"]
    end
    
    subgraph "**数据修改**"
        F["**Inserts<br/>(插入)**"]
        G["**Updates<br/>(更新)**"]
        H["**Deletes<br/>(删除)**"]
    end
    
    A --> B
    B --> C
    C --> D
    C --> E
    
    C --> F
    C --> G
    C --> H
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style F fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style G fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style H fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
```

## 核心组件

```mermaid
classDiagram
    class LocalStorage {
        **+ClientContext context**
        **+map~DataTable~ table_storage**
        **+Get(DataTable) LocalTableStorage**
        **+Append(DataTable, DataChunk)**
        **+Delete(DataTable, Vector)**
        **+Update(DataTable, ...)**
        **+Commit()**
        **+Rollback()**
    }
    
    class LocalTableStorage {
        **+DataTable table**
        **+RowGroupCollection row_groups**
        **+UpdateSegment updates**
        **+DeletedEntries deletes**
        **+Scan()**
        **+Append()**
    }
    
    class RowGroupCollection {
        **+vector~RowGroup~ row_groups**
        **+idx_t total_rows**
        **+Append()**
        **+Scan()**
    }
    
    LocalStorage --> LocalTableStorage
    LocalTableStorage --> RowGroupCollection
    
    style LocalStorage fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style LocalTableStorage fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style RowGroupCollection fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
```

## MVCC 支持

LocalStorage 实现多版本并发控制（MVCC）：

```mermaid
graph TB
    subgraph "**事务隔离**"
        A["**Transaction 1<br/>LocalStorage 1**"]
        B["**Transaction 2<br/>LocalStorage 2**"]
        C["**Transaction 3<br/>LocalStorage 3**"]
    end
    
    subgraph "**全局存储**"
        D["**Persistent Storage**"]
    end
    
    A -->|**Commit**| D
    B -->|**Commit**| D
    C -->|**Rollback**| E["**丢弃**"]
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style D fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
```

## 提交流程

```mermaid
sequenceDiagram
    participant **Transaction** as **Transaction**
    participant **LocalStorage** as **LocalStorage**
    participant **DataTable** as **DataTable**
    participant **Storage** as **PersistentStorage**
    
    **Transaction**->>**LocalStorage**: **Commit()**
    
    loop **每个修改的表**
        **LocalStorage**->>**DataTable**: **Merge(LocalTableStorage)**
        **DataTable**->>**Storage**: **AppendRowGroups()**
        **Storage**-->>**DataTable**: **完成**
        **DataTable**->>**DataTable**: **UpdateVersionInfo()**
    end
    
    **LocalStorage**->>**LocalStorage**: **Clear()**
    **LocalStorage**-->>**Transaction**: **提交成功**
```

## 相关源码

- `src/storage/local_storage.cpp` - LocalStorage主类
- `src/storage/data_table.cpp` - 数据表与LocalStorage集成
- `src/storage/table/row_group.cpp` - 行组管理
- `src/transaction/` - 事务管理

