# Parquet 文件格式设计与DuckDB实现

## 概述

Apache Parquet 是一种面向列式存储的开源文件格式，专为大数据处理场景设计。Parquet 文件格式在 Hadoop 生态系统中被广泛使用，提供了高效的数据压缩和编码方案，能够显著提升查询性能。DuckDB 内置了对 Parquet 格式的完整支持，可以直接读取和写入 Parquet 文件。

本文档深入分析 Parquet 文件格式的设计架构、编码机制和 DuckDB 中的具体实现。

## Parquet 文件格式架构

### 整体架构

```mermaid
graph TB
    subgraph "**Parquet文件结构**"
        A["**文件头<br/>(Magic Number)**"]
        B["**RowGroup 1**"]
        C["**RowGroup 2**"]
        D["**RowGroup N**"]
        E["**File Metadata**"]
        F["**文件尾<br/>(Magic Number + Footer Length)**"]
    end
    
    subgraph "**RowGroup结构**"
        G["**ColumnChunk 1**"]
        H["**ColumnChunk 2**"]
        I["**ColumnChunk M**"]
    end
    
    subgraph "**ColumnChunk结构**"
        J["**Dictionary Page<br/>(可选)**"]
        K["**Data Page 1**"]
        L["**Data Page 2**"]
        M["**Data Page N**"]
    end
    
    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    
    B --> G
    B --> H
    B --> I
    
    G --> J
    G --> K
    K --> L
    L --> M
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style F fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style G fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style H fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style I fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style J fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style K fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style L fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style M fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
```

### 文件布局

Parquet 文件采用 **页脚式设计**（Footer Layout），元数据存储在文件末尾：

```mermaid
graph LR
    A["**Magic: PAR1<br/>(4字节)**"]
    B["**Row Group 1<br/>(数据)**"]
    C["**Row Group 2<br/>(数据)**"]
    D["**...**"]
    E["**File Metadata<br/>(元数据)**"]
    F["**Footer Length<br/>(4字节)**"]
    G["**Magic: PAR1<br/>(4字节)**"]
    
    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style F fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style G fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
```

**关键特点：**
1. **文件头**：4字节魔术字"PAR1"（加密文件为"PARE"）
2. **数据区**：多个 RowGroup，每个包含所有列的数据
3. **元数据区**：FileMetadata，包含 Schema、RowGroup 信息、统计数据等
4. **文件尾**：4字节 Footer 长度 + 4字节魔术字"PAR1"

### 核心概念

```mermaid
classDiagram
    class FileMetaData {
        **+int32 version**
        **+vector~SchemaElement~ schema**
        **+int64 num_rows**
        **+vector~RowGroup~ row_groups**
        **+KeyValue[] key_value_metadata**
        **+string created_by**
    }
    
    class RowGroup {
        **+vector~ColumnChunk~ columns**
        **+int64 total_byte_size**
        **+int64 num_rows**
        **+int16 ordinal**
    }
    
    class ColumnChunk {
        **+ColumnMetaData meta_data**
        **+int64 file_offset**
    }
    
    class ColumnMetaData {
        **+Type type**
        **+vector~Encoding~ encodings**
        **+CompressionCodec codec**
        **+int64 num_values**
        **+int64 total_uncompressed_size**
        **+int64 total_compressed_size**
        **+int64 data_page_offset**
        **+int64 dictionary_page_offset**
        **+Statistics statistics**
    }
    
    class SchemaElement {
        **+Type type**
        **+FieldRepetitionType repetition_type**
        **+string name**
        **+int32 num_children**
        **+ConvertedType converted_type**
        **+LogicalType logical_type**
    }
    
    class Statistics {
        **+byte[] max**
        **+byte[] min**
        **+int64 null_count**
        **+int64 distinct_count**
    }
    
    FileMetaData --> RowGroup
    FileMetaData --> SchemaElement
    RowGroup --> ColumnChunk
    ColumnChunk --> ColumnMetaData
    ColumnMetaData --> Statistics
    
    style FileMetaData fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style RowGroup fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style ColumnChunk fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style ColumnMetaData fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style SchemaElement fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style Statistics fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
```

## 数据组织模型

### RowGroup（行组）

RowGroup 是 Parquet 文件的物理分区单元，包含表中所有列的一个子集行数据。

**特点：**
- **默认大小**：通常为 128MB 或 256MB
- **列存储**：每个 RowGroup 内部按列组织数据
- **独立性**：每个 RowGroup 可以独立读取和处理
- **统计信息**：包含每列的 min/max/null_count 等统计数据

### ColumnChunk（列块）

ColumnChunk 表示 RowGroup 中某一列的所有数据。

```mermaid
graph TB
    subgraph "**ColumnChunk组成**"
        A["**Dictionary Page<br/>(可选)**"]
        B["**Data Page 1**"]
        C["**Data Page 2**"]
        D["**Data Page N**"]
        E["**Column Metadata**"]
    end
    
    A --> B
    B --> C
    C --> D
    D --> E
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
```

### Data Page（数据页）

Data Page 是 Parquet 中最小的数据单元，包含实际的列值。

**页结构：**

```
+------------------------------------------+
|              Page Header                 |
|  +------------------------------------+  |
|  | page_type: DATA_PAGE / DATA_PAGE_V2 | |
|  | uncompressed_page_size              |  |
|  | compressed_page_size                |  |
|  | crc (optional)                      |  |
|  +------------------------------------+  |
+------------------------------------------+
|         Repetition Levels (optional)     |
+------------------------------------------+
|         Definition Levels (optional)     |
+------------------------------------------+
|         Encoded Data (compressed)        |
+------------------------------------------+
```

**三级结构：**

1. **Repetition Levels** - 嵌套数据的重复级别
2. **Definition Levels** - 可选字段的定义级别
3. **Encoded Data** - 实际编码的数据值

## Schema 设计

### Schema 表示

Parquet 使用 **Dremel 论文** 中的嵌套结构表示法。

```mermaid
graph TB
    A["**Root<br/>(duckdb_schema)**"]
    B["**Column: id<br/>INT32, REQUIRED**"]
    C["**Column: name<br/>BYTE_ARRAY, OPTIONAL**"]
    D["**Column: address<br/>STRUCT, OPTIONAL**"]
    E["**Field: street<br/>BYTE_ARRAY, OPTIONAL**"]
    F["**Field: city<br/>BYTE_ARRAY, OPTIONAL**"]
    G["**Column: tags<br/>LIST, OPTIONAL**"]
    H["**Element<br/>BYTE_ARRAY, OPTIONAL**"]
    
    A --> B
    A --> C
    A --> D
    A --> G
    D --> E
    D --> F
    G --> H
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style F fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style G fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style H fill:#ffe1ff,stroke:#333,stroke-width:2px,color:#000
```

### Repetition Type（重复类型）

```mermaid
graph LR
    A["**Repetition Type**"]
    B["**REQUIRED<br/>(必需)**"]
    C["**OPTIONAL<br/>(可选)**"]
    D["**REPEATED<br/>(重复)**"]
    
    A --> B
    A --> C
    A --> D
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
```

- **REQUIRED**: 值必须存在
- **OPTIONAL**: 值可以为 NULL
- **REPEATED**: 值可以重复（用于数组/列表）

## 编码机制

### 编码类型

Parquet 支持多种编码方式，针对不同数据类型和分布优化：

```mermaid
graph TB
    subgraph "**编码算法**"
        A["**PLAIN<br/>(原始编码)**"]
        B["**DICTIONARY<br/>(字典编码)**"]
        C["**RLE<br/>(游程长度编码)**"]
        D["**BIT_PACKED<br/>(位打包)**"]
        E["**DELTA_BINARY_PACKED<br/>(增量二进制打包)**"]
        F["**DELTA_LENGTH_BYTE_ARRAY<br/>(增量长度字节数组)**"]
        G["**DELTA_BYTE_ARRAY<br/>(增量字节数组)**"]
        H["**BYTE_STREAM_SPLIT<br/>(字节流分割)**"]
    end
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style F fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style G fill:#ffe1ff,stroke:#333,stroke-width:2px,color:#000
    style H fill:#ffffe1,stroke:#333,stroke-width:2px,color:#000
```

### 1. PLAIN（原始编码）

最简单的编码方式，直接存储原始值。

**适用场景：**
- 高基数数据
- 随机分布的数据
- 作为字典编码的回退方案

### 2. DICTIONARY（字典编码）

使用字典存储唯一值，数据存储字典索引。

```mermaid
graph TB
    subgraph "**字典编码示例**"
        A["**原始数据<br/>['Apple', 'Banana', 'Apple', 'Cherry', 'Banana']**"]
        B["**字典<br/>{0: 'Apple', 1: 'Banana', 2: 'Cherry'}**"]
        C["**编码数据<br/>[0, 1, 0, 2, 1]**"]
    end
    
    A --> B
    B --> C
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
```

**优势：**
- 适合低基数列
- 显著减少存储空间
- 提高查询性能

### 3. RLE（游程长度编码）

压缩连续重复值的序列。

**示例：**
```
原始: [1, 1, 1, 1, 2, 2, 3, 3, 3]
编码: [(4, 1), (2, 2), (3, 3)]
```

### 4. DELTA_BINARY_PACKED（增量二进制打包）

存储值之间的差值（delta），并使用位打包压缩。

```mermaid
graph LR
    A["**原始值<br/>[100, 102, 105, 107]**"]
    B["**增量<br/>[100, 2, 3, 2]**"]
    C["**位打包<br/>(紧凑存储)**"]
    
    A --> B
    B --> C
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
```

**适用场景：**
- 有序整数序列
- 时间戳列
- 自增 ID

### 5. DELTA_LENGTH_BYTE_ARRAY

对字符串长度使用增量编码，值使用原始编码。

**示例：**
```
原始: ["cat", "dog", "bird", "fish"]
长度: [3, 3, 4, 4]
长度增量: [3, 0, 1, 0]
```

### 6. BYTE_STREAM_SPLIT

将多字节值分割为多个字节流，提高压缩率。

```mermaid
graph TB
    subgraph "**字节流分割**"
        A["**浮点数<br/>[1.23, 4.56, 7.89]**"]
        B["**字节表示<br/>(4字节/值)**"]
        C["**分割字节流**"]
        D["**Byte 0: [b0, b0, b0]<br/>Byte 1: [b1, b1, b1]<br/>Byte 2: [b2, b2, b2]<br/>Byte 3: [b3, b3, b3]**"]
    end
    
    A --> B
    B --> C
    C --> D
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
```

**优势：**
- 提高浮点数压缩率
- 适合科学计算数据

## 压缩算法

Parquet 支持多种压缩算法，可以在 ColumnChunk 级别指定：

```mermaid
graph TB
    subgraph "**压缩算法**"
        A["**UNCOMPRESSED<br/>(无压缩)**"]
        B["**SNAPPY<br/>(快速压缩)**"]
        C["**GZIP<br/>(高压缩率)**"]
        D["**LZ4<br/>(极快压缩)**"]
        E["**ZSTD<br/>(平衡方案)**"]
        F["**BROTLI<br/>(高压缩率)**"]
    end
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style F fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
```

### 压缩算法对比

| 算法 | 压缩速度 | 解压速度 | 压缩率 | 适用场景 |
|------|---------|---------|-------|---------|
| **UNCOMPRESSED** | ✓✓✓✓✓ | ✓✓✓✓✓ | ✗ | 已压缩数据 |
| **SNAPPY** | ✓✓✓✓ | ✓✓✓✓✓ | ✓✓ | 实时查询 |
| **LZ4** | ✓✓✓✓✓ | ✓✓✓✓✓ | ✓✓ | 极致性能 |
| **ZSTD** | ✓✓✓ | ✓✓✓✓ | ✓✓✓✓ | 平衡方案 |
| **GZIP** | ✓✓ | ✓✓✓ | ✓✓✓✓ | 存储优先 |
| **BROTLI** | ✓ | ✓✓ | ✓✓✓✓✓ | 最小存储 |

## DuckDB 中的 Parquet 实现

### 架构概览

```mermaid
classDiagram
    class ParquetReader {
        **+CachingFileSystem fs**
        **+ParquetFileMetadataCache metadata**
        **+ParquetOptions parquet_options**
        **+ParquetColumnSchema root_schema**
        **+InitializeScan()**
        **+Scan()**
        **+GetFileMetadata()**
    }
    
    class ColumnReader {
        **+ParquetColumnSchema schema**
        **+ColumnEncoding encoding**
        **+CompressionCodec codec**
        **+DictionaryDecoder dict_decoder**
        **+DeltaBinaryPackedDecoder dbp_decoder**
        **+InitializeRead()**
        **+Read()**
        **+Skip()**
    }
    
    class ParquetWriter {
        **+BufferedFileWriter writer**
        **+FileMetaData file_meta_data**
        **+CompressionCodec codec**
        **+vector~ColumnWriter~ column_writers**
        **+Flush()**
        **+Finalize()**
    }
    
    class ColumnWriter {
        **+ParquetWriter writer**
        **+ColumnWriterState state**
        **+MemoryStream temp_writer**
        **+CompressPage()**
        **+FlushDictionary()**
        **+FlushDataPage()**
    }
    
    ParquetReader --> ColumnReader
    ParquetWriter --> ColumnWriter
    
    style ParquetReader fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style ColumnReader fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style ParquetWriter fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style ColumnWriter fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
```

### 读取流程

```mermaid
sequenceDiagram
    participant **Client** as **客户端**
    participant **Reader** as **ParquetReader**
    participant **FS** as **文件系统**
    participant **ColReader** as **ColumnReader**
    participant **Decoder** as **解码器**
    
    **Client**->>**Reader**: **打开Parquet文件**
    **Reader**->>**FS**: **读取文件尾部**
    **FS**-->>**Reader**: **返回Footer**
    **Reader**->>**Reader**: **解析FileMetadata**
    
    **Client**->>**Reader**: **InitializeScan()**
    **Reader**->>**ColReader**: **创建列读取器**
    **ColReader**->>**FS**: **读取ColumnChunk元数据**
    
    **Client**->>**Reader**: **Scan(DataChunk)**
    **Reader**->>**ColReader**: **Read(num_values)**
    **ColReader**->>**FS**: **读取Data Page**
    **FS**-->>**ColReader**: **返回压缩数据**
    **ColReader**->>**ColReader**: **解压缩**
    **ColReader**->>**Decoder**: **解码数据**
    **Decoder**-->>**ColReader**: **返回解码值**
    **ColReader**-->>**Reader**: **填充Vector**
    **Reader**-->>**Client**: **返回DataChunk**
```

### 写入流程

```mermaid
sequenceDiagram
    participant **Client** as **客户端**
    participant **Writer** as **ParquetWriter**
    participant **ColWriter** as **ColumnWriter**
    participant **Encoder** as **编码器**
    participant **FS** as **文件系统**
    
    **Client**->>**Writer**: **创建Parquet文件**
    **Writer**->>**FS**: **写入魔术字(PAR1)**
    **Writer**->>**Writer**: **初始化Schema**
    **Writer**->>**ColWriter**: **创建列写入器**
    
    **Client**->>**Writer**: **Write(DataChunk)**
    **Writer**->>**ColWriter**: **Prepare()**
    **ColWriter**->>**Encoder**: **编码数据**
    **Encoder**-->>**ColWriter**: **返回编码数据**
    **ColWriter**->>**ColWriter**: **压缩页面**
    **ColWriter**->>**FS**: **写入Data Page**
    
    **Client**->>**Writer**: **Flush()**
    **Writer**->>**ColWriter**: **FlushRowGroup()**
    **ColWriter**->>**FS**: **写入剩余页面**
    
    **Client**->>**Writer**: **Finalize()**
    **Writer**->>**FS**: **写入FileMetadata**
    **Writer**->>**FS**: **写入Footer Length**
    **Writer**->>**FS**: **写入魔术字(PAR1)**
```

### 核心读取器实现

#### ColumnReader 解码流程

```mermaid
graph TB
    A["**开始读取**"]
    B["**读取Page Header**"]
    C{**Page类型?**}
    D["**Dictionary Page**"]
    E["**Data Page**"]
    F["**读取并解压数据**"]
    G["**解析Definition Levels**"]
    H["**解析Repetition Levels**"]
    I{**编码类型?**}
    J["**Dictionary解码**"]
    K["**Delta Binary解码**"]
    L["**RLE解码**"]
    M["**Plain解码**"]
    N["**填充Vector**"]
    O["**完成**"]
    
    A --> B
    B --> C
    C -->|**字典页**| D
    C -->|**数据页**| E
    D --> E
    E --> F
    F --> G
    G --> H
    H --> I
    I -->|**DICTIONARY**| J
    I -->|**DELTA_BINARY_PACKED**| K
    I -->|**RLE**| L
    I -->|**PLAIN**| M
    J --> N
    K --> N
    L --> N
    M --> N
    N --> O
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style F fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style G fill:#ffe1ff,stroke:#333,stroke-width:2px,color:#000
    style H fill:#ffffe1,stroke:#333,stroke-width:2px,color:#000
    style I fill:#e1e1ff,stroke:#333,stroke-width:2px,color:#000
    style J fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style K fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style L fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style M fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style N fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style O fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
```

### 核心写入器实现

#### ColumnWriter 编码流程

```mermaid
graph TB
    A["**接收DataChunk**"]
    B{**字典编码?**}
    C["**构建字典**"]
    D["**编码为索引**"]
    E{**字典大小检查**}
    F["**使用字典编码**"]
    G["**回退到Plain**"]
    H["**应用编码算法**"]
    I["**压缩页面**"]
    J["**写入页头**"]
    K["**写入数据**"]
    L["**更新统计信息**"]
    M["**完成**"]
    
    A --> B
    B -->|**是**| C
    B -->|**否**| H
    C --> D
    D --> E
    E -->|**字典小**| F
    E -->|**字典大**| G
    F --> I
    G --> H
    H --> I
    I --> J
    J --> K
    K --> L
    L --> M
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style F fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
    style G fill:#ffe1ff,stroke:#333,stroke-width:2px,color:#000
    style H fill:#ffffe1,stroke:#333,stroke-width:2px,color:#000
    style I fill:#e1e1ff,stroke:#333,stroke-width:2px,color:#000
    style J fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style K fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style L fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style M fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
```

### 解码器实现

DuckDB 实现了多种解码器类来处理不同的编码格式：

```mermaid
classDiagram
    class DictionaryDecoder {
        **+vector~Value~ dictionary**
        **+RleBpDecoder index_decoder**
        **+Read()**
        **+LoadDictionary()**
    }
    
    class DeltaBinaryPackedDecoder {
        **+int64 last_value**
        **+vector~int64~ delta_buffer**
        **+Read()**
        **+ReadBlock()**
    }
    
    class DeltaByteArrayDecoder {
        **+DeltaBinaryPackedDecoder prefix_decoder**
        **+DeltaBinaryPackedDecoder suffix_decoder**
        **+Read()**
    }
    
    class DeltaLengthByteArrayDecoder {
        **+DeltaBinaryPackedDecoder length_decoder**
        **+Read()**
    }
    
    class RLEDecoder {
        **+RleBpDecoder decoder**
        **+Read()**
    }
    
    class ByteStreamSplitDecoder {
        **+idx_t element_size**
        **+vector~byte_stream~ streams**
        **+Read()**
    }
    
    style DictionaryDecoder fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style DeltaBinaryPackedDecoder fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style DeltaByteArrayDecoder fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style DeltaLengthByteArrayDecoder fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style RLEDecoder fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style ByteStreamSplitDecoder fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
```

## Parquet 优化技术

### 1. 谓词下推（Predicate Pushdown）

利用列统计信息跳过不满足条件的 RowGroup 和 Page：

```mermaid
graph TB
    A["**查询: WHERE age > 30**"]
    B["**检查RowGroup 1<br/>min_age=20, max_age=25**"]
    C["**检查RowGroup 2<br/>min_age=28, max_age=35**"]
    D["**检查RowGroup 3<br/>min_age=35, max_age=45**"]
    
    A --> B
    A --> C
    A --> D
    
    B -->|**跳过**| E["**不读取**"]
    C -->|**可能匹配**| F["**读取并过滤**"]
    D -->|**全部匹配**| G["**直接读取**"]
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style F fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style G fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
```

### 2. 列裁剪（Column Pruning）

只读取查询需要的列：

```mermaid
graph LR
    A["**SELECT name, age<br/>FROM users**"]
    B["**跳过id列**"]
    C["**读取name列**"]
    D["**读取age列**"]
    E["**跳过address列**"]
    
    A --> B
    A --> C
    A --> D
    A --> E
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style C fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
```

### 3. 元数据缓存

DuckDB 使用 `ParquetFileMetadataCache` 缓存文件元数据：

```mermaid
classDiagram
    class ParquetFileMetadataCache {
        **+FileMetaData metadata**
        **+idx_t footer_size**
        **+bool is_cached**
        **+GetMetadata()**
        **+SetMetadata()**
    }
    
    class CachingFileSystem {
        **+map~string~ cached_files**
        **+OpenFile()**
        **+GetCachedHandle()**
    }
    
    ParquetFileMetadataCache --> CachingFileSystem
    
    style ParquetFileMetadataCache fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style CachingFileSystem fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
```

### 4. 并行读取

DuckDB 支持并行读取多个 RowGroup：

```mermaid
graph TB
    subgraph "**并行读取架构**"
        A["**主线程**"]
        B["**线程1: RowGroup 1**"]
        C["**线程2: RowGroup 2**"]
        D["**线程3: RowGroup 3**"]
        E["**线程4: RowGroup 4**"]
        F["**合并结果**"]
    end
    
    A --> B
    A --> C
    A --> D
    A --> E
    B --> F
    C --> F
    D --> F
    E --> F
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style F fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
```

## Parquet 扩展功能

### 1. GeoParquet

DuckDB 支持 GeoParquet 规范，用于存储地理空间数据：

```json
{
  "version": "1.0.0",
  "primary_column": "geometry",
  "columns": {
    "geometry": {
      "encoding": "WKB",
      "geometry_types": ["Point", "LineString"],
      "bbox": [-180.0, -90.0, 180.0, 90.0],
      "projjson": {...}
    }
  }
}
```

### 2. 加密支持

DuckDB 支持加密的 Parquet 文件（魔术字为"PARE"）：

```mermaid
graph LR
    A["**PARE<br/>(加密文件)**"]
    B["**AES-GCM加密<br/>RowGroup**"]
    C["**加密元数据**"]
    D["**Footer**"]
    E["**PARE<br/>(文件尾)**"]
    
    A --> B
    B --> C
    C --> D
    D --> E
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
```

### 3. Bloom Filter

Parquet 支持 Bloom Filter 用于快速成员测试：

```mermaid
graph TB
    A["**列数据**"]
    B["**构建Bloom Filter**"]
    C["**存储在元数据**"]
    D{**查询过滤**}
    E["**Bloom Filter测试**"]
    F["**肯定不存在**"]
    G["**可能存在**"]
    H["**跳过RowGroup**"]
    I["**读取并验证**"]
    
    A --> B
    B --> C
    D --> E
    E --> F
    E --> G
    F --> H
    G --> I
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style F fill:#ffe1ff,stroke:#333,stroke-width:2px,color:#000
    style G fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style H fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
    style I fill:#e1ffff,stroke:#333,stroke-width:2px,color:#000
```

## 使用示例

### DuckDB 读取 Parquet

```sql
-- 直接查询Parquet文件
SELECT * FROM 'data.parquet' WHERE age > 30;

-- 查看Parquet元数据
SELECT * FROM parquet_metadata('data.parquet');

-- 查看Schema信息
SELECT * FROM parquet_schema('data.parquet');

-- 查看文件元数据
SELECT * FROM parquet_file_metadata('data.parquet');
```

### DuckDB 写入 Parquet

```sql
-- 基本写入
COPY (SELECT * FROM users) TO 'output.parquet';

-- 指定压缩算法
COPY users TO 'output.parquet' (COMPRESSION 'ZSTD');

-- 指定行组大小
COPY users TO 'output.parquet' (ROW_GROUP_SIZE 100000);

-- 多种选项组合
COPY users TO 'output.parquet' (
    COMPRESSION 'ZSTD',
    ROW_GROUP_SIZE 100000,
    ROW_GROUP_SIZE_BYTES '128MB'
);
```

## 性能优化建议

### 1. 选择合适的行组大小

- **推荐大小**：128MB - 256MB
- **过小**：增加元数据开销，减少并行度
- **过大**：降低读取效率，增加内存压力

### 2. 选择合适的压缩算法

```mermaid
graph TB
    A{**使用场景**}
    B["**实时查询<br/>→ SNAPPY/LZ4**"]
    C["**存储优先<br/>→ ZSTD/GZIP**"]
    D["**平衡方案<br/>→ ZSTD**"]
    E["**科学计算<br/>→ ZSTD + BSS编码**"]
    
    A --> B
    A --> C
    A --> D
    A --> E
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
    style D fill:#fff4e1,stroke:#333,stroke-width:2px,color:#000
    style E fill:#f5e1ff,stroke:#333,stroke-width:2px,color:#000
```

### 3. 优化列顺序

将查询频繁的列放在前面，利用列裁剪减少I/O：

```
推荐顺序:
1. 频繁查询的过滤列
2. 频繁查询的选择列
3. 其他列
```

### 4. 利用分区

```mermaid
graph TB
    subgraph "**分区策略**"
        A["**年份分区<br/>year=2023/**"]
        B["**月份分区<br/>month=01/**"]
        C["**Parquet文件**"]
    end
    
    A --> B
    B --> C
    
    style A fill:#e1f5ff,stroke:#333,stroke-width:2px,color:#000
    style B fill:#ffe1e1,stroke:#333,stroke-width:2px,color:#000
    style C fill:#e1ffe1,stroke:#333,stroke-width:2px,color:#000
```

## 总结

Parquet 文件格式的核心优势：

1. **列式存储** - 高效的分析查询性能
2. **高压缩率** - 多种编码和压缩算法
3. **Schema 演化** - 支持添加/删除列
4. **谓词下推** - 利用统计信息跳过数据
5. **嵌套数据** - 原生支持复杂数据类型
6. **跨平台** - 广泛的生态系统支持

DuckDB 的 Parquet 实现特点：

1. **零拷贝读取** - 高效的内存管理
2. **并行处理** - 多线程读取 RowGroup
3. **智能缓存** - 元数据和数据缓存
4. **完整编码支持** - 支持所有主流编码
5. **扩展功能** - GeoParquet、加密、Bloom Filter

## 相关源码文件

### 核心读取
- `extension/parquet/parquet_reader.cpp` - Parquet 读取器
- `extension/parquet/column_reader.cpp` - 列读取器
- `extension/parquet/parquet_metadata.cpp` - 元数据解析

### 解码器
- `extension/parquet/decoder/dictionary_decoder.cpp` - 字典解码
- `extension/parquet/decoder/delta_binary_packed_decoder.cpp` - Delta 解码
- `extension/parquet/decoder/delta_byte_array_decoder.cpp` - Delta 字节数组解码
- `extension/parquet/decoder/delta_length_byte_array_decoder.cpp` - Delta 长度解码
- `extension/parquet/decoder/rle_decoder.cpp` - RLE 解码
- `extension/parquet/decoder/byte_stream_split_decoder.cpp` - 字节流分割解码

### 核心写入
- `extension/parquet/parquet_writer.cpp` - Parquet 写入器
- `extension/parquet/column_writer.cpp` - 列写入器
- `extension/parquet/writer/primitive_column_writer.cpp` - 原始类型写入

### 扩展功能
- `extension/parquet/geo_parquet.cpp` - GeoParquet 支持
- `extension/parquet/parquet_crypto.cpp` - 加密支持
- `extension/parquet/parquet_statistics.cpp` - 统计信息处理

