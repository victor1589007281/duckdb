# DuckDB Pipeline 执行引擎

## 概述

DuckDB 使用 Pipeline 执行引擎实现并行查询处理。Pipeline 将查询分解为多个可并行执行的任务，利用多核 CPU 提升性能。

## 整体架构

```mermaid
graph TB
    subgraph "**Pipeline架构**"
        A["**PhysicalPlan**"]
        B["**PipelineBuilder**"]
        C["**MetaPipeline**"]
        D["**Pipeline 1**"]
        E["**Pipeline 2**"]
        F["**Pipeline N**"]
    end
    
    subgraph "**Pipeline组件**"
        G["**Source**"]
        H["**Operator 1**"]
        I["**Operator 2**"]
        J["**Sink**"]
    end
    
    subgraph "**并行执行**"
        K["**Task 1**"]
        L["**Task 2**"]
        M["**Task N**"]
    end
    
    A --> B
    B --> C
    C --> D
    C --> E
    C --> F
    
    D --> G
    G --> H
    H --> I
    I --> J
    
    D --> K
    E --> L
    F --> M
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style F fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style G fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style H fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style I fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style J fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style K fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style L fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style M fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
```

## 核心概念

### Pipeline

Pipeline 是一组可以流式处理的算子序列。

### Event

Event 表示 Pipeline 执行中的异步事件，用于协调并行执行。

```mermaid
classDiagram
    class Event {
        **+vector~Event~ dependencies**
        **+Finish()**
        **+Schedule()**
    }
    
    class PipelineEvent {
        **+Pipeline pipeline**
        **+FinishEvent()**
    }
    
    class PipelineCompleteEvent {
        **+OnCompletion()**
    }
    
    Event <|-- PipelineEvent
    PipelineEvent <|-- PipelineCompleteEvent
    
    style Event fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style PipelineEvent fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style PipelineCompleteEvent fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
```

### Task

Task 是 Pipeline 执行的基本单元，可以在任意线程上执行。

## 执行流程

```mermaid
sequenceDiagram
    participant **Executor** as **Executor**
    participant **Pipeline** as **Pipeline**
    participant **Task** as **Task**
    participant **Thread** as **WorkerThread**
    
    **Executor**->>**Pipeline**: **Initialize()**
    **Pipeline**->>**Task**: **CreateTasks()**
    **Task**-->>**Pipeline**: **Task列表**
    
    loop **并行执行**
        **Pipeline**->>**Thread**: **ScheduleTask()**
        **Thread**->>**Task**: **Execute()**
        **Task**->>**Task**: **处理数据块**
        **Task**-->>**Thread**: **完成**
    end
    
    **Pipeline**->>**Executor**: **AllTasksComplete()**
    **Executor**-->>**Executor**: **返回结果**
```

## 相关源码

- `src/parallel/pipeline.cpp` - Pipeline实现
- `src/parallel/event.cpp` - Event系统
- `src/parallel/executor_task.cpp` - Task实现
- `src/parallel/task_scheduler.cpp` - 任务调度器
- `src/parallel/meta_pipeline.cpp` - MetaPipeline

