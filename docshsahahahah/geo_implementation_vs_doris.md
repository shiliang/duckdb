# DuckDB Geo 实现 vs Doris Geo 实现 —— 效率设计对比分析

本文对比 DuckDB 和 Doris 在空间/地理（Geo）功能上的实现方式，重点分析 DuckDB 中哪些设计决策带来了更高的效率，以及 Doris 可以从中借鉴的设计思路。

---

## 一、整体架构对比

| 维度 | DuckDB | Doris |
|------|--------|-------|
| **类型系统** | `GEOMETRY` 是核心类型系统的一等公民（LogicalTypeId::GEOMETRY = 60） | 无专用几何类型，几何数据存于 VARCHAR/STRING 列 |
| **内部格式** | 标准化 WKB（Well-Known Binary，始终小端序、无 EWKB SRID） | 自定义格式：2字节头 + S2 编码载荷 |
| **存储层** | 自定义 `GeoColumnData`，支持三种存储模式 | 普通 VARCHAR 列存储，存储层无感知 |
| **查询优化** | 空间统计（GeometryStats）+ Zonemap 剪枝 | 无空间感知优化 |
| **空间索引** | 无 R-tree，但通过统计剪枝和碎化实现高效过滤 | 无空间索引 |
| **计算引擎** | 内建 WKT/WKB 解析，基于边界框的快速相交测试 | Google S2 Geometry Library |
| **函数数量** | 7 个核心函数 | 25+ 个函数 |
| **CRS 支持** | 类型级别的 CRS 元数据（EPSG、PROJJSON、WKT2） | 隐式 WGS84，无 SRID 追踪 |

---

## 二、DuckDB 核心效率设计详解

### 1. 原生类型系统 vs VARCHAR 字符串

**DuckDB 的做法：**
```cpp
// GEOMETRY 是 LogicalTypeId 枚举中的一等公民
LogicalType::GEOMETRY();            // 无 CRS
LogicalType::GEOMETRY("EPSG:4326"); // 带 CRS
```

**效率优势：**
- **编译时类型识别**：所有表达式、函数重载、cast 操作可以在编译/绑定阶段识别 GEOMETRY 类型，不需要运行时猜测
- **无冗余序列化**：函数间传递几何数据不需要编解码——WKB blob 直接通过 `string_t` 传递，零拷贝
- **CRS 元数据分离**：CRS 存储在 `LogicalType` 的 `ExtraTypeInfo` 中，不占用数据空间，`ST_SetCRS` 是 O(1) 的 reinterpret cast

**对比 Doris：**
- Doris 在 FE 层面将几何类型映射为 `VarcharType.SYSTEM_DEFAULT`，意味着：
  - 每个函数调用都必须 `from_encoded()` 解码几何体，操作完后丢弃
  - 无法在类型层面做任何编译时优化
  - FE planner 无法区分空间列和普通字符串列，无法做谓词下推

### 2. WKB 作为规范内部格式 vs 自定义 S2 编码

**DuckDB 的做法：**
- 所有几何数据统一存储为 **标准化小端序 WKB**
- `FromBinary()` 做两遍 WKB 标准化：
  1. `AnalyzeWKB` —— 分析字节序、EWKB 标志，计算规范化后的尺寸（不分配内存）
  2. `ConvertWKB` —— 直接写入预分配缓冲区

**效率优势：**
- **零对象树开销**：WKB 即存储格式，无需反序列化为几何对象树，节省大量内存分配
- **ST_AsWKB 是恒等操作**：内部已经是 WKB，只需要 `StringVector::AddHeapReference` 零拷贝返回
- **标准化简化运算**：统一小端序意味着不再需要在每个操作中处理字节序
- **尺寸预计算**：两遍法避免了动态扩容

**对比 Doris：**
- 自定义二进制格式（2 字节头 + S2 编码），每次函数调用需要：
  1. 读取头部的类型标志
  2. 使用 S2 `Decoder` 反序列化为 `S2Point`/`S2Polyline`/`S2Polygon` 等对象
  3. 操作完成后销毁对象
- 这意味着即使简单如 `ST_X()` 这种取一个 double 的操作，也需要完整解码整个几何体

### 3. Columnar Shredding（碎化）—— 杀手级优化

**这是 DuckDB 最核心的效率创新。**

#### 基本原理

将 WKB blob 在 checkpoint 时拆解为 DuckDB 原生嵌套类型：

| 几何类型 | 碎化后类型 |
|----------|-----------|
| `POINT XY` | `STRUCT(x DOUBLE, y DOUBLE)` |
| `POINT XYZ` | `STRUCT(x DOUBLE, y DOUBLE, z DOUBLE)` |
| `LINESTRING XY` | `LIST(STRUCT(x DOUBLE, y DOUBLE))` |
| `POLYGON XY` | `LIST(LIST(STRUCT(x DOUBLE, y DOUBLE)))` |

#### 效率优势

1. **列式压缩**：所有 x 坐标连续存储，使用 DuckDB 的通用压缩算法（FSST、Alpine 等），压缩率远高于 WKB blob

2. **向量化执行**：
   ```sql
   -- 碎化后，这个过滤变成了标准 DOUBLE 列过滤
   SELECT * FROM geo_table WHERE x > 5.0 AND y < 10.0
   ```
   - 不需要 WKB 解析即可执行谓词
   - DuckDB 的向量化执行引擎直接操作连续内存的 DOUBLE 数组

3. **统计信息自然生成**：碎化列的子列自动产生 min/max 统计信息：
   - x 列的 min/max → GeometryExtent 的 x 范围
   - y 列的 min/max → GeometryExtent 的 y 范围
   - 不需要额外的统计计算

4. **零碎片化**：碎化通过 48 个模板特化函数实现（6 种几何类型 × 4 种顶点类型 × To/From），直接 WKB → DuckDB 列式向量，无中间表示

#### 触发条件（精心设计的边界条件）

```
Checkpoint 时判断：
├─ 无变更 → 保持原格式
├─ 数量 < GeometryMinimumShreddingSize → 保持 WKB（小表不值得碎化）
├─ 混合类型 / GEOMETRYCOLLECTION / 空几何 → 保持 WKB（不适合列式存储）
└─ 否则 → 碎化为列式格式
```

**关键洞察**：写入总是 WKB（保持写入速度），碎化是 checkpoint 时的读优化——写路径不增加开销。

#### 对比 Doris

Doris 没有碎化机制。几何数据始终以 S2 编码的 blob 存储在 VARCHAR 列中：
- Zone map 基于字节序的 min/max（对空间数据无意义）
- 任何过滤都需要 full decode + full scan
- 无列式压缩优势

### 4. GeometryStats 统计系统 —— 空间谓词剪枝

**DuckDB 的做法：**

```cpp
struct GeometryStatsData {
    GeometryTypeSet type_set;    // 32位位图，标记存在的几何类型
    GeometryExtent extent;       // 8个 double：x/y/z/m min/max
    GeometryStatsFlags flags;    // 空/非空标志
};
```

**`CheckZonemap` 优化能力：**
- 识别 `&&`（边界框相交）和 `ST_Intersects_Extent` 函数
- 移除 CRS 擦除 cast 以找到底层列引用
- 根据 extent 与常量几何体的边界框关系返回：
  - `FILTER_ALWAYS_FALSE` → 完全跳过数据块
  - `FILTER_ALWAYS_TRUE` → 跳过过滤条件
  - `NO_PRUNING_POSSIBLE` → 正常执行

**效率优势：**
- **单轴即可剪枝**：如果只有 x 轴已知，y 轴视为无穷范围，仍能剪枝
- **Sentinel 值设计**：`UNKNOWN_MIN = +inf`，`UNKNOWN_MAX = -inf`，`Merge` 操作不需要特殊分支
- **GeometryTypeSet 位图**：O(1) 查詢 `Has(type)`、`HasSingleType()`，用于碎化决策

**对比 Doris：**
- 完全没有空间统计。VARCHAR 列的统计信息（NDV、字节序 min/max）对空间查询毫无帮助
- 所有空间谓词都导致全表扫描+逐行解码

### 5. 函数统计回调 —— 计划时间优化

**DuckDB 的一个高级优化**是函数注册时可以附带统计回调，在查询计划阶段就推导统计信息。

**示例：`ST_GeomFromWKB` 的 `FromWKBStats` 回调：**

```cpp
// 不解析所有行，仅比较 WKB 头部 5 字节（字节序 + 类型元数据）的 min/max
// 如果所有行的头部相同，则可以推导出几何类型
```

效率优势：
- **避免查询计划阶段的几何解析**
- **常量化折叠**：当统计信息表明函数总是返回 NULL 时（如从 XYZM 几何体提取 Z），在绑定阶段就替换为 `BoundConstantExpression`

**对比 Doris：**
所有函数在运行时逐行执行，FE planner 对几何函数无任何特殊优化

### 6. 编译期模板分发 vs 运行时类型切换

**DuckDB 的做法（C++ 模板元编程）：**

```cpp
struct VertexXY {   constexpr static bool HAS_Z = false, HAS_M = false;
    constexpr static idx_t WIDTH = 2;
    using STRUCT_TYPE = VectorStructType<double, double>;
    // ...
};
struct VertexXYZ {  constexpr static bool HAS_Z = true, HAS_M = false;
    constexpr static idx_t WIDTH = 3;
    using STRUCT_TYPE = VectorStructType<double, double, double>;
    // ...
};

// 对所有 4 种顶点类型模板化
template <class V>
void ParseVerticesInternal(BlobReader &reader, ...) {
    reader.Reserve(vert_count * sizeof(V)); // 一次 Reserve 减少边界检查
    for (idx_t i = 0; i < vert_count; i++) {
        coord[i].x = reader.Read<double>(le);
        if constexpr (V::HAS_Z) coord[i].z = reader.Read<double>(le);
        if constexpr (V::HAS_M) coord[i].m = reader.Read<double>(le);
    }
}
// 运行时通过 switch 分派到 4 个特化，消除运行时维度检查
```

**效率优势：**
- `if constexpr` 在编译期消除 Z/M 判断分支
- `sizeof(V)` 编译期已知，`Reserve` 一次到位
- `STRUCT_TYPE` 携带 DuckDB 列式类型映射，碎化函数直接写入列式向量
- 48 个模板特化覆盖所有 (几何类型 × 顶点类型 × To/From) 组合，无运行时虚函数开销

**对比 Doris：**
- 使用 O(n*m) 的 switch-case 类型分发：`GeoPoint::intersects()` 内有 switch(rhs->type()) 覆盖 POINT/LINE/POLYGON/MULTI_POLYGON/CIRCLE
- 每次函数调用需要完整解码到 S2 堆对象
- S2 库虽然内部优化好，但编解码开销不可忽略

### 7. 存储代理模式 —— 运行时透明度

**DuckDB 的 `GeoColumnData` 使用代理模式：**

```cpp
class GeoColumnData : public ColumnData {
    shared_ptr<ColumnData> base_column;  // 实际存储委托给此列
    GeometryStorageType storage_type;    // WKB 或碎化类型
    
    // 扫描时：
    if (storage_type == WKB) {
        base_column->Scan(scan_state, ...);        // 直接转发，零开销
    } else {
        base_column->Scan(scan_state, tmp_chunk);  // 扫描碎化列
        Reassemble(tmp_chunk, result);              // 重组为 WKB
    }
};
```

**效率设计点：**
- **WKB 路径零额外开销**：不分配 `tmp_chunk`，不调用 `Reassemble`
- **碎化路径的 Reassemble** 在按行组扫描时批量执行，分摊重组开销
- **存储格式可演化**：列可以在生命周期内从 WKB → 碎化（checkpoint 时）

### 8. 顶点提取优化 —— 统计驱动的常量化

`vertex_extract(geom, 'x')` 函数在绑定阶段通过 `VertexExtractStats` 回调：

1. 检查统计信息中是否只有 POINT 类型
2. 检查维度是否存在（如 Z 需要 XYZ/XYZM 类型）
3. 如果维度不存在（如从 XY geometry 提取 Z），将函数替换为 `BoundConstantExpression(NULL)`
4. 如果只有 POINT 类型，传播 GeometryExtent 中的 min/max 到输出统计

这意味着 `SELECT vertex_extract(geom, 'z') FROM points_2d` 在查询计划阶段就被优化掉，不需要执行。

---

## 三、Doris 有而 DuckDB 无的设计（功能差距）

Doris 在某些方面提供了更多功能，但这是以性能为代价的：

| 功能 | Doris | DuckDB | 说明 |
|------|-------|--------|------|
| **函数数量** | 25+ | 7 | DuckDB 更专注核心功能，较少但更高效 |
| **S2 地理计算** | 完整 S2 集成（距离、面积、方位角等） | 无 | DuckDB 暂无球面距离/面积计算 |
| **WKT 解析器** | Flex/Bison 生成 LALR(1) 解析器 | 手写递归下降 | DuckDB 手写解析器更轻量、编译更快 |
| **MultiPolygon 支持** | 部分支持 | 全面支持 | DuckDB 支持完整 OGC WKT 规范 |

---

## 四、DuckDB 的优势要点总结（按重要性排序）

### 最高影响

1. **Shredding（碎化）**：将空间数据转化为列式存储，同时获得压缩、向量化执行、统计信息三大优势。这是列存数据库做空间数据的范式级创新。

2. **WKB 即内部格式**：省去所有中间对象树，ST_AsWKB 为零拷贝恒等操作。Doris 的 S2 编码要求每次函数调用都完整编解码。

3. **原生类型系统**：GEOMETRY 是一等公民，CRS 是元数据而不是数据，所有基础设施自动获得空间感知能力。

4. **GeometryStats + Zonemap 剪枝**：空间谓词可以跳过整块数据，不依赖传统 R-tree 索引就实现了高效的过滤。

### 中高影响

5. **模板元编程消除运行时分支**：`if constexpr` 在编译期消除维度检查，所有几何类型 × 顶点类型的组合都在编译期生成特化代码。

6. **函数统计回调**：在查询计划阶段推导函数输出统计，实现常量化折叠，减少运行时开销。

7. **两遍 WKB 解析**：先分析尺寸再写入，避免动态扩容和冗余分配。

8. **单轴剪枝**：即使只收集到一个维度的统计信息也能用于过滤，比其他系统更激进。

### 设计哲学差异

| 哲学 | DuckDB | Doris |
|------|--------|-------|
| **数据流** | 保持 WKB blob 不动，直到需要时才碎化 | 每次操作都要编码/解码 |
| **优化时机** | 尽可能在查询计划/绑定阶段做 | 全部在运行阶段逐行做 |
| **存储适配** | 存储层感知几何类型，自主选择最优格式 | 存储层无感知，字符串存储 |
| **编译期 vs 运行时** | 用模板把运行时开销转为编译期 | 运行时 switch-case 分发 |
| **元数据管理** | CRS 挂在类型上零开销 | 无 CRS 概念 |

---

## 五、给 Doris 设计改进的启发

1. **引入 GEOMETRY 原生类型**：即使底层仍用 VARCHAR 存储，在 FE 类型系统中引入 GEOMETRY 类型就可以激活类型感知的优化空间。

2. **列式碎化思路**：Doris 的列存引擎同样适合做碎化。在 Segment 写入时将几何数据拆解为 DOUBLE 列，可以获得极大的查询性能提升。

3. **统计信息利用**：利用碎化后的列自然产生的 min/max 统计来做 zonemap 空间剪枝，不需要 S2 索引。

4. **WKB 作为存储规范**：从 S2 自定义格式迁移到标准化 WKB 可以简化系统、提高兼容性，并在写入端保持高性能（WKB 写入比 S2 编码更快）。

5. **函数统计回调**：在 Doris 的 Function 注册机制中增加统计回调接口，让空间函数可以向优化器提供输出统计信息。

---

*本文基于 DuckDB main (2026-07) 和 Doris main (2026-07) 的源码分析。*
